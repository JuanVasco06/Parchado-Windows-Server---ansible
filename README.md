# Parchado Windows Server — Ansible / AWX

Automatización de parchado de Windows Server, multi-cliente, multi-hipervisor,
ejecutada desde AWX sobre nodos ejecutores remotos ubicados en la red de cada
cliente.

---

## 1. Objetivo del proyecto

Un único playbook (`playbooks/parchado_windows_2019_full.yml`), reutilizable
para todos los clientes, que:

- Valida el estado del servidor antes de parchar (espacio en disco, reinicio
  pendiente, procesos de Windows Update activos).
- Crea un snapshot/checkpoint previo, soportando **VMware** y **Hyper-V**.
- Instala actualizaciones de Windows en ciclo hasta dejar el servidor limpio.
- Gestiona firmas de Microsoft Defender (opcional, desactivable por host).
- Deja el panel de "Configuración > Windows Update" del servidor reflejando
  el estado real (no cacheado).
- Valida el resultado después de parchar.
- Notifica un resumen consolidado a Microsoft Teams vía Power Automate.

**Principio de diseño:** el playbook nunca cambia por cliente. Todo lo que
diferencia a un cliente de otro (hipervisor, credenciales, red, antivirus,
umbral de reintentos) vive en la capa de configuración de AWX —
Organización, Inventario, Credenciales, Grupo de instancias, Variables —
nunca en el código.

---

## 2. Arquitectura multi-cliente

| Pieza AWX | ¿Se comparte entre clientes? | Contenido |
|---|---|---|
| Proyecto (Git) | ✅ Uno solo | Este repositorio y el playbook |
| Organización | ❌ Una por cliente | — |
| Inventario | ❌ Uno por cliente | Hosts Windows + `hypervisor_type` + datos del hipervisor |
| Credenciales | ❌ Una(s) por cliente | Machine (WinRM) + vCenter (si aplica) |
| Grupo de instancias | ❌ Uno por cliente | El nodo ejecutor físico que vive en/cerca de la red del cliente |
| Plantilla de trabajo | ❌ Una por cliente | Combina Proyecto (compartido) + Inventario, Credenciales y Grupo de instancias (del cliente) |

### Por qué el nodo ejecutor debe estar en la red del cliente

Las tareas de snapshot (`delegate_to: localhost` para VMware,
`delegate_to: "{{ hyperv_host }}"` para Hyper-V) se ejecutan físicamente
desde el nodo ejecutor asignado a esa plantilla. Por eso cada cliente
necesita su propio nodo con ruta de red al vCenter o al host Hyper-V de esa
infraestructura — ver el manual de onboarding (sección 3 de este repo) para
el procedimiento completo.

### Malla de Receptor (Mesh)

Los nodos ejecutores remotos no reciben conexiones entrantes desde AWX
(bloqueado por firewall en la mayoría de clientes). En su lugar, cada nodo
**marca hacia afuera** contra un nodo *hop* (`awx-cgm-mesh`), publicado vía
`AWXMeshIngress` en el hostname `mesh-dev.e-global.com.co` (puerto 443,
WebSocket sobre TLS con mTLS). Ver manual de onboarding para el detalle de
configuración de peers.

---

## 3. Clientes activos

| Cliente | Organización AWX | Inventario | Nodo ejecutor | Hipervisor | Estado |
|---|---|---|---|---|---|
| GMP | GMP | — | GMP | (confirmar) | ✅ Producción |
| COOSALUD | Coosalud | Inventario COOSALUD | `COOSALUD` (host real: `infiniti`, Ubuntu 24.04) | Ninguno configurado aún (pruebas) | ✅ Validado en pruebas |

---

## 4. El playbook: `parchado_windows_2019_full.yml`

### 4.1 Estructura (6 plays, en orden)

```
1. INICIO      → hosts: localhost   — muestra resumen de configuración del job
2. VALIDACION  → hosts: windows     — precheck: disco, reinicio pendiente, búsqueda de parches
3. SNAPSHOT    → hosts: windows     — snapshot/checkpoint multi-hipervisor (condicional)
4. PATCH       → hosts: windows     — instalación de actualizaciones + reconciliación USO
5. POSTCHECK   → hosts: windows     — validación posterior, agrega resumen para Teams
6. CIERRE      → hosts: localhost   — notificación consolidada a Teams
```

### 4.2 Snapshot multi-hipervisor

Cada host Windows del inventario declara su plataforma con `hypervisor_type`:

- `hypervisor_type: vmware` → snapshot vía `vmware.vmware.vm_snapshot`,
  hablando con vCenter, delegado a `localhost` (el propio nodo ejecutor).
  Requiere `vmware_vm_name`, `vmware_datacenter`, y credenciales vCenter
  inyectadas como variables de entorno (`VMWARE_HOST`, `VMWARE_USER`,
  `VMWARE_PASSWORD`) vía Custom Credential Type de AWX.

- `hypervisor_type: hyperv` → checkpoint vía `Checkpoint-VM` (PowerShell),
  delegado al host Hyper-V (`hyperv_host`), que debe existir en el
  inventario dentro de un grupo `hyperv_hosts` separado del grupo `windows`
  (para que nunca se intente parchar el host Hyper-V por error).

Si `enable_snapshot: true` pero el host no tiene `hypervisor_type` válido,
el playbook lo advierte en el log en vez de fallar en silencio.

### 4.3 Gestión de Microsoft Defender

Controlado por `enable_defender_management` (default `true`). En clientes
con antivirus de terceros (Symantec, CrowdStrike, etc.), se desactiva por
host para evitar el error `"La actualización de las definiciones de virus y
spyware se completó con errores"`, que es esperado cuando Defender no es el
AV activo.

### 4.4 Registro del servicio Microsoft Update

Antes de escanear, el playbook registra el servicio "Microsoft Update"
(GUID `7971f918-a847-4430-9279-4a52d1efe18d`) vía COM. Sin esto, `win_updates`
puede no ver actualizaciones de Defender/Office en servidores donde solo
está registrado "Windows Update" (parches de SO).

### 4.5 Ciclo de instalación

`win_updates` corre en un ciclo (`until`) hasta que una pasada completa no
encuentre nada nuevo por instalar y no haya fallidas, en vez de solo
reintentar cuando hay errores. Controlado por `patch_max_passes` /
`patch_retries` (default 3 pasadas) y `patch_retry_delay` (default 180s).

### 4.6 Reconciliación de USO (evita falsas alarmas en el panel)

**Problema resuelto:** el playbook instala actualizaciones vía **WUA**
(Windows Update Agent, API COM). El panel "Configuración > Windows Update"
lee de **USO** (Update Session Orchestrator), un subsistema distinto con su
propio almacén de estado. Sin reconciliar, el panel puede seguir mostrando
actualizaciones como *"Pending install"* indefinidamente, aunque
`Get-HotFix` y la propia API confirmen que ya están instaladas — generando
falsas alarmas al revisar el servidor manualmente.

**Causa raíz identificada:** el almacén de USO **no** es
`C:\Windows\SoftwareDistribution` (ese es de WUA). Es
`C:\ProgramData\USOPrivate\UpdateStore` y `C:\ProgramData\USOShared`.
Resetear `SoftwareDistribution` no corrige el panel — se probó y se
descartó explícitamente.

**Solución implementada** (tarea *"Reconciliar estado de USO..."*, en
`PATCH`, después de la instalación):

1. Detiene `UsoSvc`.
2. Renombra (respalda, no borra) `USOPrivate\UpdateStore` y `USOShared`.
3. Reinicia `UsoSvc` y dispara `UsoClient StartScan`.
4. Mata el proceso `SystemSettings` — es una app UWP: cerrar la ventana
   solo la *suspende*, no la termina, y puede seguir mostrando su estado
   viejo en memoria.
5. Limpia respaldos con más de 7 días.
6. Reporta el conteo real de pendientes (vía API) en el log del job.

Controlado por `reset_uso_store` (default `true`). Validado exitosamente en
dos clientes distintos (confirmado por captura: panel pasa de *"Updates
available"* a *"You're up to date"*).

### 4.7 Notificación a Teams

Vía Power Automate (webhook con Adaptive Card), variable de entorno
`TEAMS_WEBHOOK_URL`. Incluye:

- `timeout: 90` (el default de `ansible.builtin.uri` es 30s; el flujo de
  Power Automate puede tardar más).
- 2 reintentos con `delay: 5`.
- `ignore_errors: true` + tarea de advertencia: si Teams falla tras los
  reintentos, el job **no** se marca como `Fallido` — el parchado en sí no
  se ve afectado por un problema de notificación.

---

## 5. Referencia de variables

| Variable | Default | Alcance recomendado | Descripción |
|---|---|---|---|
| `cliente_nombre` | `'No definido'` | Inventario (nivel raíz) | Nombre del cliente, mostrado en logs y tarjeta Teams |
| `ambiente_objetivo` | `'PRUEBAS'` | Plantilla / Inventario | PRUEBAS / PRODUCCIÓN |
| `enable_patch_install` | `false` | Plantilla | Interruptor maestro: instala o solo valida |
| `enable_snapshot` | `false` | Plantilla | Crea snapshot/checkpoint antes de parchar |
| `enable_teams_notification` | `false` | Plantilla | Envía resumen a Teams |
| `enable_defender_management` | `true` | Host / Grupo | `false` en clientes con AV de terceros |
| `reset_uso_store` | `true` | Host | Reconciliación de USO (ver 4.6). `false` si un host no debe tocarse |
| `hypervisor_type` | *(sin default)* | Host | `vmware` o `hyperv` |
| `vmware_vm_name` / `vmware_datacenter` | — | Host | Solo si `hypervisor_type: vmware` |
| `vm_name_hypervisor` / `hyperv_host` | — | Host | Solo si `hypervisor_type: hyperv` |
| `windows_update_categories` | `['*']` | Grupo | Categorías de Windows Update a incluir |
| `patch_reboot` | `true` | Grupo | Reiniciar si el parchado lo requiere |
| `patch_reboot_timeout` | `7200` | Grupo | Segundos de espera al reiniciar |
| `patch_max_passes` / `patch_retries` | `3` | Grupo | Pasadas del ciclo de instalación |
| `patch_retry_delay` | `180` | Grupo | Segundos entre pasadas |
| `fail_on_pending_reboot` | `false` | Grupo | Falla el job si queda reinicio pendiente |
| `servicios_criticos` | `[]` | Host | Lista de servicios a validar en el postcheck |

**Variables de entorno** (inyectadas por Credenciales de AWX, no por
`extra_vars`): `VMWARE_HOST`, `VMWARE_USER`, `VMWARE_PASSWORD`,
`TEAMS_WEBHOOK_URL`.

---

## 6. Known issues / configuración global de AWX requerida

Estos ajustes son **globales** en la instancia de AWX (`awx-cgm`), no del
playbook. Se documentan aquí porque afectan a todo nodo ejecutor Ubuntu que
se agregue en el futuro — ver el manual de onboarding, sección de
troubleshooting, para el detalle completo de cada uno.

| Setting AWX | Problema que resuelve | Valor aplicado |
|---|---|---|
| `AWX_ISOLATION_SHOW_PATHS` | El EE monta `/etc/pki/ca-trust` y `/usr/share/pki` (rutas RHEL) que no existen en Ubuntu → `mounting overlay failed` | `["/etc/pki/ca-trust:/etc/pki/ca-trust:O", "/usr/share/pki:/usr/share/pki:O", "/etc/ssl/certs:/etc/ssl/certs:O"]` (aditivo, no reemplaza los paths RHEL por si otros nodos los necesitan) |
| `DEFAULT_CONTAINER_RUN_OPTIONS` | Se investigó IPv6 de slirp4netns como causa de un timeout hacia Teams; **descartado** — la causa real era el timeout de 30s del módulo `uri` (ver 4.7) | `["--network", "slirp4netns:enable_ipv6=false,mtu=1500"]` (queda aplicado, es inofensivo, pero no era la causa raíz del problema que se investigaba) |

**Lección aprendida:** ante un error idéntico en nodos distintos (ej. mismo
timeout de Teams en COOSALUD y en el nodo de GMP), la causa casi nunca es de
red específica de un cliente — hay que sospechar primero de timeouts fijos
en el propio módulo/tarea antes de perseguir MTU, firewall o DNS.

---

## 7. Estructura del repositorio

```
Parchado-Windows-Server---ansible/
├── README.md                                  (este archivo)
├── MANUAL_ONBOARDING_NODO_EJECUTOR.md          (alta de nuevo cliente/nodo, paso a paso)
├── docs/
├── group_vars/
│   └── windows.yml                             (defaults compartidos, ver sección 5)
├── inventories/
│   └── [inventario por cliente]
├── inventoriespruebas/
└── playbooks/
    └── parchado_windows_2019_full.yml
```

---

## 8. Próximos pasos

- [ ] Probar el flujo completo de snapshot en un host Hyper-V real
      (`enable_snapshot: true`, `hypervisor_type: hyperv`) — pendiente
      desde el diseño inicial.
- [ ] Confirmar sistema operativo del nodo ejecutor de GMP (para saber si el
      ajuste aditivo de `AWX_ISOLATION_SHOW_PATHS` puede simplificarse).
- [ ] Evaluar si el tiempo extra que agrega la reconciliación de USO
      (~80s/servidor con `serial: 1`) es aceptable a medida que crece el
      número de servidores por cliente; considerar paralelizar si se
      vuelve un cuello de botella.
- [ ] Definir Tipo de Credencial personalizado de vCenter en AWX si se
      onboardean clientes VMware (hoy solo hay validación con COOSALUD,
      que es Hyper-V/sin hipervisor confirmado).
- [ ] Documentar procedimiento de rotación de credenciales (Machine y
      vCenter) por cliente.

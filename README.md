# Automatización de Parchado Windows Server con AWX + Ansible

Repositorio para la automatización del parchado de servidores Windows Server mediante **AWX**, **Ansible** y **WinRM**.

El objetivo principal es centralizar, controlar y auditar el proceso de actualización de servidores Windows desde AWX, permitiendo ejecutar validaciones previas, instalación de parches, reinicios controlados, postchecks y generación de evidencias operativas.

---

## 1. Objetivo del proyecto

Este proyecto permite automatizar el proceso de parchado de servidores Windows Server, inicialmente orientado a **Windows Server 2019**, usando AWX como plataforma de orquestación.

El flujo contempla:

* Validación de conectividad WinRM.
* Identificación del sistema operativo.
* Validación de espacio en disco.
* Detección de reinicio pendiente.
* Búsqueda de actualizaciones disponibles.
* Preparación de servicios Windows Update.
* Actualización de firmas de Microsoft Defender.
* Ejecución de comandos similares al botón **Retry** de Windows Update.
* Instalación de actualizaciones con reintentos.
* Reinicio automático cuando aplique.
* Validación posterior al parchado.
* Consulta de hotfix instalados.
* Consulta de eventos recientes de Windows Update.
* Registro de logs en el servidor Windows.
* Integración opcional con snapshot VMware.
* Integración opcional con notificación a Microsoft Teams.

---

## 2. Arquitectura general

```text
Usuario / Operador
      |
      | HTTPS
      v
AWX / Ansible Controller
      |
      | WinRM 5985 / 5986
      v
Servidor Windows Server
      |
      | Windows Update / Microsoft Update / WSUS
      v
Fuente de actualizaciones
```

Opcionalmente, el flujo puede integrarse con VMware para crear un snapshot previo al parchado:

```text
AWX
 |
 | API 443
 v
VMware vCenter
 |
 v
Snapshot previo de la VM
```

---

## 3. Estructura del repositorio

```text
.
├── README.md
├── docs/
│   └── arquitectura.md
├── group_vars/
│   └── windows.yml
├── inventories/
│   └── pruebas/
│       └── hosts.yml
├── playbooks/
│   └── parchado_windows_2019_full.yml
└── requirements.yml
```

---

## 4. Requisitos

### 4.1 Requisitos en AWX

* AWX desplegado y funcional.
* Proyecto sincronizado desde Git.
* Inventario de servidores Windows.
* Credencial tipo Machine para acceso al servidor Windows.
* Execution Environment con soporte para Windows.

El Execution Environment debe incluir, como mínimo:

```text
ansible.windows
community.windows
pywinrm
requests
requests-ntlm
pypsrp
```

Si se usará VMware:

```text
community.vmware
vmware.vmware
pyvmomi
```

---

### 4.2 Requisitos en Windows Server

El servidor Windows debe tener WinRM habilitado.

Para laboratorio puede usarse WinRM por HTTP:

```text
Puerto 5985
```

Para ambientes productivos se recomienda WinRM por HTTPS:

```text
Puerto 5986
```

Validaciones recomendadas en Windows:

```powershell
Get-Service WinRM
winrm enumerate winrm/config/listener
winrm get winrm/config/service/auth
```

Servicios requeridos para Windows Update:

```powershell
Get-Service wuauserv,bits,cryptsvc,TrustedInstaller
```

---

## 5. Inventario de ejemplo

Archivo:

```text
inventories/pruebas/hosts.yml
```

Ejemplo:

```yaml
---
all:
  children:
    windows:
      hosts:
        win2019-prueba-01:
          ansible_host: 10.200.254.15
          vmware_vm_name: "WIN2019-PRUEBA-01"
          ambiente: "PRUEBAS"
          servicios_criticos:
            - WinRM
            - W32Time
            - EventLog
```

El grupo debe llamarse exactamente:

```text
windows
```

Esto es importante porque el playbook usa:

```yaml
hosts: windows
```

---

## 6. Variables principales

Las variables pueden definirse en AWX como **Extra Variables** o en:

```text
group_vars/windows.yml
```

Ejemplo recomendado para ejecución completa:

```yaml
ansible_connection: winrm
ansible_port: 5985
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore
ansible_shell_type: powershell
ansible_winrm_operation_timeout_sec: 120
ansible_winrm_read_timeout_sec: 180

windows_update_categories:
  - '*'

enable_patch_install: true
enable_vmware_snapshot: false
enable_teams_notification: false

patch_reboot: true
patch_reboot_timeout: 7200

patch_retries: 3
patch_retry_delay: 180

fail_on_pending_reboot: false
```

---

## 7. Descripción de variables

| Variable                               | Descripción                                                               |
| -------------------------------------- | ------------------------------------------------------------------------- |
| `ansible_connection`                   | Tipo de conexión Ansible. Para Windows debe ser `winrm`.                  |
| `ansible_port`                         | Puerto WinRM. `5985` para HTTP, `5986` para HTTPS.                        |
| `ansible_winrm_transport`              | Transporte de autenticación. Ejemplo: `ntlm` o `negotiate`.               |
| `ansible_winrm_server_cert_validation` | Validación de certificado. En laboratorio puede ser `ignore`.             |
| `ansible_shell_type`                   | Shell remoto. Para Windows usar `powershell`.                             |
| `ansible_winrm_operation_timeout_sec`  | Timeout de operación WinRM. Recomendado aumentar para parches.            |
| `ansible_winrm_read_timeout_sec`       | Timeout de lectura WinRM. Debe ser mayor al timeout de operación.         |
| `windows_update_categories`            | Categorías de Windows Update a instalar. `'*'` incluye todas.             |
| `enable_patch_install`                 | Habilita o deshabilita la instalación real de parches.                    |
| `patch_reboot`                         | Permite reinicio automático si Windows Update lo requiere.                |
| `patch_reboot_timeout`                 | Tiempo máximo de espera para reinicio y reconexión.                       |
| `patch_retries`                        | Número de reintentos de instalación si quedan actualizaciones fallidas.   |
| `patch_retry_delay`                    | Espera entre reintentos, en segundos.                                     |
| `enable_vmware_snapshot`               | Habilita snapshot VMware previo al parchado.                              |
| `enable_teams_notification`            | Habilita notificación a Teams al cierre del flujo.                        |
| `fail_on_pending_reboot`               | Si está en `true`, falla el Job si queda reinicio pendiente en postcheck. |

---

## 8. Modos de ejecución recomendados

### 8.1 Modo consulta / validación

Este modo permite validar conectividad, SO, disco, reinicio pendiente y actualizaciones disponibles, sin instalar parches.

```yaml
ansible_connection: winrm
ansible_port: 5985
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore
ansible_shell_type: powershell
ansible_winrm_operation_timeout_sec: 120
ansible_winrm_read_timeout_sec: 180

windows_update_categories:
  - '*'

enable_patch_install: false
enable_vmware_snapshot: false
enable_teams_notification: false

patch_reboot: false
patch_reboot_timeout: 7200

patch_retries: 3
patch_retry_delay: 180

fail_on_pending_reboot: false
```

---

### 8.2 Modo instalación completa

Este modo instala todas las actualizaciones aplicables.

```yaml
ansible_connection: winrm
ansible_port: 5985
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore
ansible_shell_type: powershell
ansible_winrm_operation_timeout_sec: 120
ansible_winrm_read_timeout_sec: 180

windows_update_categories:
  - '*'

enable_patch_install: true
enable_vmware_snapshot: false
enable_teams_notification: false

patch_reboot: true
patch_reboot_timeout: 7200

patch_retries: 3
patch_retry_delay: 180

fail_on_pending_reboot: false
```

---

### 8.3 Modo instalación con snapshot VMware

Este modo crea snapshot previo antes de instalar parches.

```yaml
ansible_connection: winrm
ansible_port: 5985
ansible_winrm_transport: ntlm
ansible_winrm_server_cert_validation: ignore
ansible_shell_type: powershell
ansible_winrm_operation_timeout_sec: 120
ansible_winrm_read_timeout_sec: 180

windows_update_categories:
  - '*'

enable_patch_install: true
enable_vmware_snapshot: true
enable_teams_notification: false

patch_reboot: true
patch_reboot_timeout: 7200

patch_retries: 3
patch_retry_delay: 180

fail_on_pending_reboot: false

vmware_validate_certs: false
vmware_datacenter: "Datacenter"
```

Las credenciales de VMware se deben manejar como variables de entorno o credenciales seguras en AWX:

```text
VMWARE_HOST
VMWARE_USER
VMWARE_PASSWORD
```

---

## 9. Flujo del playbook

El playbook principal se encuentra en:

```text
playbooks/parchado_windows_2019_full.yml
```

El flujo está dividido en las siguientes etapas:

### 9.1 INICIO

Muestra información general de la ejecución:

* Ambiente.
* Estado de snapshot VMware.
* Estado de notificación Teams.
* Estado de instalación.
* Estado de reinicio.
* Categorías de Windows Update.

---

### 9.2 VALIDACIÓN / PRECHECK

Ejecuta validaciones previas:

* Prueba WinRM con `win_ping`.
* Obtiene hostname.
* Obtiene información del sistema operativo.
* Valida espacio libre en disco C.
* Detecta reinicio pendiente.
* Revisa procesos activos de Windows Update.
* Busca actualizaciones disponibles sin instalar.

---

### 9.3 VMWARE SNAPSHOT

Si `enable_vmware_snapshot` está en `true`, crea un snapshot previo al parchado.

El snapshot usa el formato:

```text
AWX_PRE_PATCH_<hostname>_<fecha_hora>
```

Ejemplo:

```text
AWX_PRE_PATCH_win2019-prueba-01_20260610_140000
```

---

### 9.4 PATCH

La etapa de parchado fue mejorada para comportarse de forma similar al botón **Retry** de Windows Update.

Realiza las siguientes acciones:

1. Asegura que los servicios requeridos estén iniciados:

   * `wuauserv`
   * `bits`
   * `cryptsvc`
   * `TrustedInstaller`

2. Actualiza firmas de Microsoft Defender mediante:

```powershell
Update-MpSignature
```

3. Fuerza acciones de Windows Update similares a Retry:

```powershell
UsoClient StartScan
UsoClient StartDownload
UsoClient StartInstall
```

4. Espera antes de instalar para evitar conflictos con descargas en progreso.

5. Ejecuta instalación de actualizaciones con:

```yaml
ansible.windows.win_updates
```

6. Aplica reintentos si quedan actualizaciones fallidas.

7. Genera log local en:

```text
C:\Windows\Temp\ansible-win-updates.log
```

8. Falla el Job si quedan actualizaciones fallidas.

---

### 9.5 POSTCHECK

Valida el estado posterior al parchado:

* Último arranque.
* Reinicio pendiente.
* Últimos hotfix instalados.
* Nuevas actualizaciones pendientes.
* Eventos recientes de Windows Update.
* Servicios automáticos detenidos.
* Servicios críticos definidos en inventario.

También genera log de búsqueda posterior en:

```text
C:\Windows\Temp\ansible-win-updates-postcheck.log
```

---

### 9.6 CIERRE

Envía notificación a Teams si está habilitado.

Para habilitarlo se requiere:

```yaml
enable_teams_notification: true
```

Y la variable de entorno:

```text
TEAMS_WEBHOOK_URL
```

---

## 10. Logs y evidencias

El flujo genera evidencias en AWX y en el servidor Windows.

### 10.1 Evidencias en AWX

* Resultado del Job.
* Salida de precheck.
* Salida de instalación.
* Cantidad de actualizaciones encontradas.
* Cantidad de actualizaciones instaladas.
* Cantidad de actualizaciones fallidas.
* Resultado del postcheck.
* Estado final del Job.

---

### 10.2 Evidencias en Windows

Logs principales:

```text
C:\Windows\Temp\ansible-win-updates-search.log
C:\Windows\Temp\ansible-win-updates.log
C:\Windows\Temp\ansible-win-updates-postcheck.log
```

Consultar log principal:

```powershell
Get-Content C:\Windows\Temp\ansible-win-updates.log -Tail 150
```

Consultar eventos de Windows Update:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-WindowsUpdateClient/Operational" -MaxEvents 50 |
Select-Object TimeCreated, Id, Message |
Format-List
```

Consultar hotfix instalados:

```powershell
Get-HotFix |
Sort-Object InstalledOn -Descending |
Select-Object -First 30 HotFixID, Description, InstalledOn, InstalledBy
```

---

## 11. Validaciones manuales útiles

### 11.1 Validar servicios Windows Update

```powershell
Get-Service wuauserv,bits,cryptsvc,TrustedInstaller |
Select-Object Name, Status, StartType
```

---

### 11.2 Validar reinicio pendiente

```powershell
$pending = $false

if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending") {
  $pending = $true
}

if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired") {
  $pending = $true
}

if (Test-Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\PendingFileRenameOperations") {
  $pending = $true
}

"Reinicio pendiente: $pending"
```

---

### 11.3 Validar procesos activos de Windows Update

```powershell
Get-Process | Where-Object {
    $_.ProcessName -match "wusa|TiWorker|TrustedInstaller|UsoClient|MoUsoCoreWorker"
} | Select-Object ProcessName, Id, CPU, StartTime
```

---

### 11.4 Forzar retry manual desde PowerShell

```powershell
UsoClient StartScan
Start-Sleep -Seconds 20
UsoClient StartDownload
Start-Sleep -Seconds 30
UsoClient StartInstall
```

---

### 11.5 Actualizar firmas de Microsoft Defender

```powershell
Update-MpSignature
```

---

## 12. Rollback y pruebas de laboratorio

Para repetir pruebas reales de parchado, la opción recomendada es usar snapshots.

### 12.1 Rollback recomendado

```text
vCenter > VM > Snapshots > Revert to Snapshot
```

Flujo recomendado:

```text
1. Crear snapshot antes del parchado.
2. Ejecutar Job AWX.
3. Validar resultado.
4. Revertir snapshot.
5. Repetir prueba.
```

---

### 12.2 Desinstalar KB manualmente para pruebas

Si no existe snapshot, se puede intentar desinstalar un KB específico.

Listar KB instalados:

```powershell
Get-HotFix |
Sort-Object InstalledOn -Descending |
Select-Object -First 30 HotFixID, Description, InstalledOn, InstalledBy
```

Desinstalar un KB:

```powershell
wusa /uninstall /kb:5094123 /norestart
```

Con ejecución silenciosa:

```powershell
wusa /uninstall /kb:5094123 /quiet /norestart
```

Validar procesos:

```powershell
Get-Process | Where-Object {
    $_.ProcessName -match "wusa|TiWorker|TrustedInstaller"
} | Select-Object ProcessName, Id, CPU, StartTime
```

Validar si quedó reinicio pendiente:

```powershell
$pending = $false

if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending") {
  $pending = $true
}

if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired") {
  $pending = $true
}

"Reinicio pendiente: $pending"
```

Reiniciar:

```powershell
Restart-Computer -Force
```

> Nota: no todos los KB permiten desinstalación. No se recomienda remover paquetes críticos con DISM en servidores productivos.

---

## 13. Consideraciones importantes

### 13.1 AWX puede marcar exitoso aunque Windows falle si no se controla

El playbook actual mejorado falla explícitamente si `failed_update_count` es mayor que cero.

Esto evita que un Job quede en verde cuando Windows Update dejó actualizaciones fallidas.

---

### 13.2 Windows Update puede quedar con operaciones en progreso

Si Windows muestra:

```text
We can't install some updates because other updates are in progress.
Restarting your computer may help.
```

Se recomienda:

1. No ejecutar AWX inmediatamente.
2. Revisar procesos `TiWorker` y `TrustedInstaller`.
3. Reiniciar si hay reinicio pendiente.
4. Volver a ejecutar AWX cuando el estado esté limpio.

---

### 13.3 Actualizaciones de Defender

Las actualizaciones tipo:

```text
KB2267602 - Security Intelligence Update for Microsoft Defender Antivirus
```

pueden fallar o quedar pendientes en Windows Update.

Por eso el playbook ejecuta previamente:

```powershell
Update-MpSignature
```

Esto ayuda a resolver actualizaciones de firmas de Defender antes de ejecutar el módulo general de Windows Update.

---

## 14. Ejecución desde AWX

### 14.1 Proyecto

```text
PRJ - Parchado Windows Server 2019
```

### 14.2 Inventario

```text
INV - Windows Server 2019 - Pruebas
```

### 14.3 Job Template

```text
JT - Parchado Windows 2019 - Pruebas
```

### 14.4 Playbook

```text
playbooks/parchado_windows_2019_full.yml
```

### 14.5 Credencial

Credencial tipo Machine con usuario administrador local o de dominio sobre los servidores Windows.

Ejemplos:

```text
Administrator
DOMINIO\usuario_ansible
```

---

## 15. Actualización del repositorio

Después de modificar el playbook o este README:

```bash
git status
git add .
git commit -m "Update Windows patching playbook documentation"
git push origin main
```

Luego sincronizar el proyecto en AWX:

```text
Projects > PRJ - Parchado Windows Server 2019 > Sync
```

---

## 16. Resultado esperado

Una ejecución correcta debe finalizar con una salida similar a:

```text
Actualizaciones encontradas: X
Actualizaciones instaladas: X
Reinicio requerido: False
Fallidas: 0
```

Y en el postcheck:

```text
Reinicio pendiente después de parchar: False
Actualizaciones pendientes después del parchado: 0
Actualizaciones fallidas en postcheck: 0
```

---

## 17. Próximas mejoras

Mejoras recomendadas para próximas versiones:

* Separar plantillas por modo:

  * Check Only
  * Parchado completo
  * Parchado con snapshot
  * Parchado con aprobación
* Crear Workflow Template con aprobación manual antes de producción.
* Agregar notificación Teams con resumen detallado.
* Generar evidencia HTML o CSV por servidor.
* Integrar rollback automatizado con VMware.
* Crear Instance Groups por cliente o ambiente.
* Implementar nodos ejecutores externos mediante AWX Mesh/Receptor.
* Usar WinRM HTTPS en ambientes productivos.
* Separar inventarios por ambiente:

  * DEV
  * QA
  * PREPROD
  * PROD

---

## 18. Estado actual

El flujo actual ya permite ejecutar parchado Windows desde AWX usando WinRM, instalar todas las categorías de actualización con:

```yaml
windows_update_categories:
  - '*'
```

Además, incorpora mejoras para manejar escenarios donde Windows Update queda en estado de descarga, actualización pendiente, conflicto por procesos en curso o fallos parciales.

El objetivo operativo es que AWX sea la consola central de ejecución, auditoría y evidencia del parchado Windows.

ansible-playbook -i inventories/pruebas/hosts.yml playbooks/parchado_windows_2019_full.yml

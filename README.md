# Parchado Windows Server 2019 con AWX + Ansible

Este repositorio contiene playbooks para ejecutar parchado controlado de servidores Windows Server 2019 desde AWX.

## Servidor inicial de pruebas

- IP: 10.200.254.15
- Sistema operativo objetivo: Windows Server 2019
- Ambiente: PRUEBAS

## Flujo principal

1. Validación de conectividad WinRM.
2. Precheck del servidor.
3. Búsqueda de actualizaciones disponibles.
4. Creación de snapshot en VMware.
5. Instalación de actualizaciones Windows.
6. Reinicio controlado si aplica.
7. Postcheck del servidor.
8. Notificación a Microsoft Teams.

## Requisitos

- AWX operativo.
- WinRM habilitado en el servidor Windows.
- Usuario administrador local o de dominio.
- Acceso desde AWX hacia Windows por puerto 5985 o 5986.
- Acceso desde AWX hacia vCenter por puerto 443.
- Webhook o Workflow de Microsoft Teams.

## Colecciones Ansible

Ver archivo requirements.yml.

## Playbook principal

```bash
ansible-playbook -i inventories/pruebas/hosts.yml playbooks/parchado_windows_2019_full.yml
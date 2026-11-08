# 02 - Instalación

Guía para desplegar el laboratorio descrito en [01 - Arquitectura](01-arquitectura.md): Wazuh all-in-one, agentes en Windows y Linux, y las integraciones con Sysmon, Suricata y VirusTotal.

> **Versión de Wazuh utilizada:** 4.14 (verificar en la [documentación oficial](https://documentation.wazuh.com/current/quickstart.html) si hay una versión más reciente y ajustar las URL).
> Completa los espacios marcados con `[ ]` y agrega tus capturas en `docs/img/` a medida que avances.

## Índice

1. [Requisitos previos](#1-requisitos-previos)
2. [Preparación del servidor Wazuh](#2-preparación-del-servidor-wazuh)
3. [Instalación de Wazuh (all-in-one)](#3-instalación-de-wazuh-all-in-one)d
4. [Acceso al dashboard](#4-acceso-al-dashboard)
5. [Agente en el endpoint Linux](#5-agente-en-el-endpoint-linux)
6. [Agente en el endpoint Windows](#6-agente-en-el-endpoint-windows)
7. [Integración de Sysmon](#7-integración-de-sysmon)
8. [Integración de Suricata](#8-integración-de-suricata)
9. [Integración de VirusTotal](#9-integración-de-virustotal)
10. [Verificación final y snapshots](#10-verificación-final-y-snapshots)
11. [Problemas frecuentes](#11-problemas-frecuentes)

---

## 1. Requisitos previos

- Las 4 VMs creadas según la arquitectura (Wazuh Server, Windows, Linux y Kali).
- Conectividad entre las VMs en la red host-only `192.168.56.0/24`.
- Acceso a internet temporal (adaptador NAT) para descargar paquetes.
- Wazuh Server con **Ubuntu Server 22.04 LTS**, 4 vCPU, 8 GB de RAM y 50 GB de disco.

## 2. Preparación del servidor Wazuh

### 2.1 Actualizar el sistema y fijar el hostname

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname wazuh-server
```

### 2.2 Configurar IP estática en la red del laboratorio

Edita el archivo de netplan (el nombre varía, por ejemplo `/etc/netplan/00-installer-config.yaml`). Reemplaza `enp0s8` por el nombre real de tu interfaz host-only (verifícalo con `ip a`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:            # Adaptador NAT (internet)
      dhcp4: true
    enp0s8:            # Adaptador host-only (laboratorio)
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

Aplica los cambios y verifica:

```bash
sudo netplan apply
ip a
ping -c 3 192.168.56.1
```

>  Captura : `docs/img/02-ip-wazuh-server.png`

## 3. Instalación de Wazuh (all-in-one)

El asistente de instalación oficial despliega en el mismo host el **Wazuh Manager**, el **Wazuh Indexer** y el **Wazuh Dashboard**.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

La instalación tarda varios minutos. Al terminar, el asistente muestra la URL de acceso y las credenciales del usuario `admin`.

Si necesitas recuperar las contraseñas más adelante:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

### 3.1 Desactivar el repositorio de Wazuh (recomendado)

Para evitar actualizaciones accidentales que puedan romper el entorno:

```bash
sudo sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

### 3.2 Verificar los servicios

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

Los tres deben aparecer como `active (running)`.

>  Captura  `docs/img/02-servicios-wazuh.png`

## 4. Acceso al dashboard

1. Desde el anfitrión, abre `https://192.168.56.10`.
2. El navegador mostrará una advertencia de certificado (es un certificado autofirmado, esperado en el laboratorio). Acepta la excepción.
3. Inicia sesión con el usuario `admin` y la contraseña generada.
4. **Cambia la contraseña por defecto** si lo deseas, siguiendo la documentación oficial de Wazuh.

>  Captura: `docs/img/02-dashboard-inicial.png`

## 5. Agente en el endpoint Linux

La forma más segura de obtener el comando exacto es desde el dashboard: **Agents → Deploy new agent**, eligiendo Linux, DEB amd64 y la IP del servidor (`192.168.56.10`). El comando generado tiene este formato:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_<VERSION>_amd64.deb
sudo WAZUH_MANAGER='192.168.56.10' dpkg -i ./wazuh-agent_<VERSION>_amd64.deb
```

Habilita e inicia el agente:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

Verifica en el dashboard (**Agents**) que aparezca como **Active**.

>  Captura: `docs/img/02-agente-linux-activo.png`

## 6. Agente en el endpoint Windows

Igual que en Linux, genera el comando desde **Deploy new agent** (Windows, MSI). En PowerShell **como administrador**, el formato es:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-<VERSION>.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='192.168.56.10'
NET START WazuhSvc
```

Verifica que el agente aparezca como **Active** en el dashboard.

>  Captura: `docs/img/02-agente-windows-activo.png`

## 7. Integración de Sysmon

Se instala en el **endpoint Windows**.

### 7.1 Instalar Sysmon con una configuración base

1. Descarga Sysmon desde [Microsoft Sysinternals](https://learn.microsoft.com/sysinternals/downloads/sysmon).
2. Descarga la configuración de [SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) (`sysmonconfig-export.xml`) y guárdala en `configs/sysmon/` del repositorio.
3. En PowerShell como administrador, desde la carpeta de Sysmon:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

4. Comprueba que el servicio esté activo:

```powershell
Get-Service Sysmon64
```

### 7.2 Hacer que el agente de Wazuh lea los eventos de Sysmon

Edita `C:\Program Files (x86)\ossec-agent\ossec.conf` y agrega, dentro de `<ossec_config>`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Reinicia el agente:

```powershell
Restart-Service -Name wazuh
```

### 7.3 Validar

Ejecuta algún comando en el endpoint (por ejemplo, abre `cmd.exe` y ejecuta `whoami`) y busca eventos de Sysmon en el dashboard (**Threat Hunting / Events**, filtrando por `data.win.system.providerName: Microsoft-Windows-Sysmon`).

>  Captura: `docs/img/02-sysmon-eventos.png`

## 8. Integración de Suricata

Se instala en el **endpoint Linux**.

### 8.1 Instalar Suricata

```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
sudo apt install suricata -y
```

### 8.2 Configurar la interfaz y la red local

Edita `/etc/suricata/suricata.yaml`:

- En `vars → address-groups → HOME_NET`, define la red del laboratorio: `"[192.168.56.0/24]"`.
- En la sección `af-packet`, define la interfaz host-only (por ejemplo `enp0s8`).

### 8.3 Actualizar reglas e iniciar el servicio

```bash
sudo suricata-update
sudo systemctl enable suricata
sudo systemctl restart suricata
sudo systemctl status suricata
```

### 8.4 Enviar los eventos a Wazuh

Edita `/var/ossec/etc/ossec.conf` en el endpoint Linux y agrega:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Reinicia el agente:

```bash
sudo systemctl restart wazuh-agent
```

### 8.5 Validar

Desde Kali, ejecuta un escaneo (`nmap -sS 192.168.56.30`) y verifica que aparezcan alertas de Suricata en el dashboard.

>  Si Suricata no ve tráfico entre otras VMs, recuerda habilitar el modo promiscuo en el adaptador host-only del hipervisor (ver [01 - Arquitectura](01-arquitectura.md#63-suricata-endpoint-linux)).

>  Captura: `docs/img/02-suricata-alertas.png`

## 9. Integración de VirusTotal

Se configura en el **servidor Wazuh** y enriquece las alertas de integridad de archivos (FIM).

### 9.1 Obtener la API key

Crea una cuenta gratuita en VirusTotal y copia tu API key. La cuenta gratuita tiene límites de consultas.

> ⚠️ **La API key no debe subirse al repositorio.** En `configs/ossec.conf` deja siempre el marcador `TU_API_KEY_AQUI`.

### 9.2 Configurar la integración

Edita `/var/ossec/etc/ossec.conf` en el servidor y agrega:

```xml
<integration>
  <name>virustotal</name>
  <api_key>TU_API_KEY_AQUI</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

### 9.3 Activar FIM en tiempo real en el agente

Para que haya archivos que consultar, configura el monitoreo de un directorio en el agente (por ejemplo, en el endpoint Linux, en `/var/ossec/etc/ossec.conf`):

```xml
<syscheck>
  <directories realtime="yes">/root</directories>
</syscheck>
```

Reinicia ambos componentes:

```bash
# En el servidor
sudo systemctl restart wazuh-manager
# En el agente
sudo systemctl restart wazuh-agent
```

### 9.4 Validar

Crea un archivo de prueba en el directorio monitoreado y verifica en el dashboard las alertas del grupo `syscheck` con datos de VirusTotal.

> Captura: `docs/img/02-virustotal-alerta.png`

## 10. Verificación final y snapshots

| Verificación | Estado |
|---|---|
| Servicios de Wazuh activos en el servidor | [ ] |
| Agente Linux **Active** en el dashboard | [ ] |
| Agente Windows **Active** en el dashboard | [ ] |
| Eventos de Sysmon visibles | [ ] |
| Alertas de Suricata visibles | [ ] |
| Alerta de FIM con datos de VirusTotal | [ ] |
| Kali con conectividad a los endpoints | [ ] |

Cuando todo esté en orden, **toma un snapshot de cada VM** con el nombre `base-configurada-AAAA-MM-DD`. Antes de cada caso de uso, toma otro snapshot para poder restaurar el estado limpio.

---

*Documento anterior: [01 - Arquitectura](01-arquitectura.md) · Siguiente: [03 - Configuración](03-configuracion.md)*

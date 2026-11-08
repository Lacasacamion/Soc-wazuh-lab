# 03 - Configuración

Con la infraestructura instalada ([02 - Instalación](02-instalacion.md)), este documento deja el laboratorio **listo para los casos de uso**: endurecimiento del servidor, configuración centralizada de agentes, fuentes de logs, monitoreo de integridad de archivos (FIM), reglas personalizadas y gestión de la configuración en el repositorio.

> Completa los espacios marcados con `[ ]` y agrega tus capturas en `docs/img/` a medida que avances.

## Índice

1. [Sincronización de hora](#1-sincronización-de-hora)
2. [Hardening del servidor Wazuh](#2-hardening-del-servidor-wazuh)
3. [Grupos de agentes y configuración centralizada](#3-grupos-de-agentes-y-configuración-centralizada)
4. [Fuentes de logs en Linux](#4-fuentes-de-logs-en-linux)
5. [Fuentes de logs en Windows](#5-fuentes-de-logs-en-windows)
6. [Monitoreo de integridad de archivos (FIM)](#6-monitoreo-de-integridad-de-archivos-fim)
7. [Reglas y decoders personalizados](#7-reglas-y-decoders-personalizados)
8. [Suricata: reglas locales](#8-suricata-reglas-locales)
9. [Active Response (preparación)](#9-active-response-preparación)
10. [Módulos incluidos de Wazuh](#10-módulos-incluidos-de-wazuh)
11. [Gestión de la configuración en el repositorio](#11-gestión-de-la-configuración-en-el-repositorio)
12. [Verificación final](#12-verificación-final)

---

## 1. Sincronización de hora

La correlación de eventos entre máquinas depende de que todas tengan la **misma hora y zona horaria**. Un desfase hace que las alertas aparezcan desordenadas y complica el análisis.

En las VMs Linux (Wazuh Server, endpoint y Kali):

```bash
sudo timedatectl set-timezone America/Lima   # usar la zona horaria que corresponda
sudo timedatectl set-ntp true
timedatectl
```

En Windows (PowerShell como administrador):

```powershell
Set-TimeZone -Id "SA Pacific Standard Time"
w32tm /resync
```

Verificación: la hora debe coincidir en las cuatro VMs.

## 2. Hardening del servidor Wazuh

### 2.1 Firewall (UFW)

Se permite únicamente lo necesario desde la red del laboratorio:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp     # SSH (administración)
sudo ufw allow from 192.168.56.0/24 to any port 1514 proto tcp   # Eventos de los agentes
sudo ufw allow from 192.168.56.0/24 to any port 1515 proto tcp   # Registro de agentes
sudo ufw allow from 192.168.56.0/24 to any port 443 proto tcp    # Dashboard
sudo ufw enable
sudo ufw status verbose
```

> ⚠️ Configura la regla de SSH **antes** de ejecutar `ufw enable`, o perderás el acceso remoto al servidor.
> Los puertos `9200` (Indexer) y `55000` (API) no se abren porque, en el despliegue all-in-one, los componentes se comunican localmente.

### 2.2 Credenciales

- Cambiar las contraseñas por defecto generadas por el instalador, siguiendo la documentación oficial de Wazuh para el cambio de contraseñas del Indexer y de la API.
- Guardarlas en un gestor de contraseñas. **Nunca** en el repositorio.

### 2.3 Respaldo de la configuración

Antes de cada cambio importante:

```bash
sudo mkdir -p /root/backups
sudo tar czf /root/backups/wazuh-etc-$(date +%F).tar.gz /var/ossec/etc
```

Además de los snapshots de las VMs, esto permite restaurar solo la configuración de Wazuh.

> 📸 Captura sugerida: `docs/img/03-ufw-status.png`

## 3. Grupos de agentes y configuración centralizada

Wazuh permite definir la configuración de los agentes **desde el servidor**, por grupos, en lugar de editar el `ossec.conf` de cada máquina.

### 3.1 Crear los grupos y asignar agentes

```bash
sudo /var/ossec/bin/agent_groups -a -g linux -q
sudo /var/ossec/bin/agent_groups -a -g windows -q
sudo /var/ossec/bin/manage_agents -l          # listar IDs de agentes
sudo /var/ossec/bin/agent_groups -a -i <ID_AGENTE_LINUX> -g linux -q
sudo /var/ossec/bin/agent_groups -a -i <ID_AGENTE_WINDOWS> -g windows -q
```

### 3.2 Archivos de configuración compartida

Cada grupo tiene su archivo en `/var/ossec/etc/shared/<grupo>/agent.conf`.

**Grupo `linux`** (`/var/ossec/etc/shared/linux/agent.conf`):

```xml
<agent_config os="Linux">
  <syscheck>
    <directories check_all="yes" realtime="yes" report_changes="yes">/etc/ssh,/root</directories>
  </syscheck>
</agent_config>
```

**Grupo `windows`** (`/var/ossec/etc/shared/windows/agent.conf`):

```xml
<agent_config os="Windows">
  <localfile>
    <location>Microsoft-Windows-PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  <syscheck>
    <directories check_all="yes" realtime="yes">C:\Users\Public</directories>
  </syscheck>
</agent_config>
```

### 3.3 Validar y verificar

```bash
sudo /var/ossec/bin/verify-agent-conf
```

En el dashboard: **Agents → Groups** debe mostrar cada grupo con su agente. Los agentes reciben la configuración en pocos minutos.

> 📸 Captura sugerida: `docs/img/03-grupos-agentes.png`

## 4. Fuentes de logs en Linux

### 4.1 Autenticación y SSH (necesario para el caso de fuerza bruta SSH)

Verifica que el agente del endpoint Linux esté recolectando los logs de autenticación. Revisa en `/var/ossec/etc/ossec.conf` del agente que exista una entrada para `/var/log/auth.log` (o `journald`, según la versión):

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

Si no está, agrégala y reinicia el agente:

```bash
sudo systemctl restart wazuh-agent
```

### 4.2 Suricata

Ya configurado en [02 - Instalación](02-instalacion.md#8-integración-de-suricata) mediante `/var/log/suricata/eve.json`.

### 4.3 Prueba rápida

Genera un intento de SSH fallido desde Kali y comprueba que aparezca la alerta de autenticación fallida en el dashboard:

```bash
ssh usuario_inexistente@192.168.56.30
```

## 5. Fuentes de logs en Windows

### 5.1 Auditoría de inicios de sesión

Para generar los eventos `4624` (inicio exitoso) y `4625` (inicio fallido), habilita la auditoría (PowerShell como administrador):

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /get /subcategory:"Logon"
```

### 5.2 PowerShell Script Block Logging

Registra el contenido de los scripts ejecutados (evento `4104`), clave para detectar comandos ofuscados:

```powershell
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1
```

El canal `Microsoft-Windows-PowerShell/Operational` ya se incluyó en el `agent.conf` del grupo `windows` (sección 3.2).

### 5.3 Sysmon

Ya instalado e integrado en [02 - Instalación](02-instalacion.md#7-integración-de-sysmon). Si el ruido es excesivo, ajusta el archivo de configuración de Sysmon (`configs/sysmon/`) y recárgalo:

```powershell
.\Sysmon64.exe -c sysmonconfig-export.xml
```

### 5.4 Validación

- Falla un inicio de sesión a propósito y busca el evento `4625` en el dashboard.
- Ejecuta `powershell -Command "Write-Output prueba"` y busca el evento `4104` y el evento de creación de proceso de Sysmon.

> 📸 Captura sugerida: `docs/img/03-eventos-windows.png`

## 6. Monitoreo de integridad de archivos (FIM)

El módulo `syscheck` detecta creación, modificación y eliminación de archivos.

| Parámetro | Uso |
|---|---|
| `realtime="yes"` | Detección inmediata (en lugar de esperar al escaneo periódico) |
| `check_all="yes"` | Verifica hash, permisos, propietario y tamaño |
| `report_changes="yes"` | Guarda el diff de archivos de texto modificados |

Directorios monitoreados en este laboratorio:

| Máquina | Directorios | Motivo |
|---|---|---|
| Endpoint Linux | `/etc/ssh`, `/root` | Cambios de configuración SSH y archivos del usuario privilegiado |
| Endpoint Windows | `C:\Users\Public` | Zona de escritura común, útil para simular descargas de malware |

La integración con VirusTotal (documentada en [02 - Instalación](02-instalacion.md#9-integración-de-virustotal)) consulta los hashes de las alertas del grupo `syscheck`.

### Prueba

```bash
echo "prueba fim" | sudo tee /root/prueba-fim.txt
echo "cambio" | sudo tee -a /root/prueba-fim.txt
sudo rm /root/prueba-fim.txt
```

Deben aparecer tres alertas (creado, modificado, eliminado) en **File Integrity Monitoring**.

> 📸 Captura sugerida: `docs/img/03-fim-alertas.png`

## 7. Reglas y decoders personalizados

Las reglas propias se escriben en el servidor y **nunca** se modifican las reglas por defecto de Wazuh.

| Elemento | Ruta en el servidor | Ruta en el repositorio |
|---|---|---|
| Reglas locales | `/var/ossec/etc/rules/local_rules.xml` | `detections/custom-rules/local_rules.xml` |
| Decoders locales | `/var/ossec/etc/decoders/local_decoder.xml` | `detections/custom-rules/local_decoder.xml` |

Los IDs de reglas personalizadas deben estar en el rango **100000 - 120000**. En este laboratorio se usa el rango `100100+`.

### 7.1 Reglas de ejemplo

```xml
<group name="local,lab,">

  <!-- Escaneo SYN interno detectado por la regla local de Suricata (ver sección 8) -->
  <rule id="100100" level="10">
    <if_sid>86601</if_sid>
    <field name="alert.signature">^LAB - Posible escaneo SYN interno</field>
    <description>LAB: escaneo SYN interno detectado por Suricata</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>

  <!-- PowerShell con comando codificado (Sysmon, evento 1) -->
  <rule id="100101" level="12">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)powershell.*\s-e(nc|ncodedcommand)?\s</field>
    <description>LAB: PowerShell ejecutado con comando codificado</description>
    <mitre>
      <id>T1059.001</id>
    </mitre>
  </rule>

</group>
```

> ⚠️ El ID `86601` (alerta genérica de Suricata) y el grupo `sysmon_event1` pertenecen a las reglas por defecto de Wazuh. Verifica que existan en tu versión con `wazuh-logtest` antes de dar la regla por buena.

### 7.2 Probar con `wazuh-logtest`

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Pega un evento de ejemplo (un JSON de `eve.json` o un log de Sysmon) y comprueba qué regla se dispara y con qué nivel.

### 7.3 Aplicar los cambios

```bash
sudo systemctl restart wazuh-manager
sudo tail -n 30 /var/ossec/logs/ossec.log    # buscar errores de sintaxis en las reglas
```

> 📸 Captura sugerida: `docs/img/03-wazuh-logtest.png`

## 8. Suricata: reglas locales

Como se documentó en [05 - Troubleshooting](05-troubleshooting.md#5-supresión-de-alertas-internas-en-reglas-predeterminadas-de-suricata), las reglas estándar no alertan sobre escaneos dentro de `HOME_NET`. Se mantienen reglas propias para tráfico interno (este-oeste).

Ubicación recomendada: un archivo independiente, para que `suricata-update` no lo sobrescriba.

```bash
sudo nano /var/lib/suricata/rules/local.rules
```

Registrar el archivo en `/etc/suricata/suricata.yaml`:

```yaml
rule-files:
  - suricata.rules
  - local.rules
```

Validar y reiniciar:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
sudo systemctl restart suricata
```

Guardar una copia versionada en `configs/suricata/local.rules` del repositorio.

## 9. Active Response (preparación)

Active Response permite que Wazuh ejecute una acción automática (por ejemplo, bloquear una IP) cuando se dispara una regla. **Se deja preparado pero desactivado**; se activará en el caso de uso de respuesta activa.

En `/var/ossec/etc/ossec.conf` del servidor (bloque comentado):

```xml
<!--
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>180</timeout>
</active-response>
-->
```

Consideraciones antes de activarlo:

- Usar `<white_list>` para excluir la IP del anfitrión y del servidor Wazuh, y no quedarte sin acceso.
- Tomar un snapshot de la VM antes de la prueba.
- Probar siempre con un `timeout` corto.

## 10. Módulos incluidos de Wazuh

Estos módulos vienen con el despliegue y conviene revisarlos en el dashboard:

| Módulo | Utilidad en el laboratorio |
|---|---|
| **Security Configuration Assessment (SCA)** | Evalúa la configuración de los endpoints frente a benchmarks (por ejemplo, CIS) y sugiere correcciones |
| **Vulnerability Detection** | Identifica software vulnerable instalado en los agentes |
| **MITRE ATT&CK** | Muestra las alertas organizadas por táctica y técnica, útil para el informe de cada caso |
| **Threat Hunting / Events** | Búsqueda y filtrado de eventos para el análisis de incidentes |

> 📸 Captura sugerida: `docs/img/03-modulos-dashboard.png`

## 11. Gestión de la configuración en el repositorio

Estructura de archivos de configuración versionados:

```
configs/
├── ossec.conf                  # Manager (sin secretos)
├── agent/
│   ├── ossec-linux.conf
│   ├── ossec-windows.conf
│   ├── agent-linux-group.conf  # shared/linux/agent.conf
│   └── agent-windows-group.conf
├── sysmon/
│   └── sysmonconfig-export.xml
└── suricata/
    └── local.rules
detections/
└── custom-rules/
    ├── local_rules.xml
    └── local_decoder.xml
```

### Sanitizar antes de hacer commit

Antes de copiar un archivo al repositorio, sustituye cualquier dato sensible:

```bash
sudo cp /var/ossec/etc/ossec.conf ./configs/ossec.conf
sudo chown $USER: ./configs/ossec.conf
sed -i 's|<api_key>.*</api_key>|<api_key>TU_API_KEY_AQUI</api_key>|' ./configs/ossec.conf
grep -n -i -E "api_key|password|token|secret" ./configs/ossec.conf
```

El último comando permite revisar que no queden credenciales. Mantén además actualizado el `.gitignore` para excluir `wazuh-install-files.tar`, `wazuh-passwords.txt`, claves y certificados.

## 12. Verificación final

| Verificación | Estado |
|---|---|
| Hora y zona horaria sincronizadas en las 4 VMs | [ ] |
| UFW activo con reglas mínimas en el servidor | [ ] |
| Contraseñas por defecto cambiadas y guardadas fuera del repo | [ ] |
| Grupos `linux` y `windows` creados con agentes asignados | [ ] |
| `verify-agent-conf` sin errores | [ ] |
| Alerta de SSH fallido visible en el dashboard | [ ] |
| Eventos `4625` y `4104` visibles desde Windows | [ ] |
| Alertas FIM (crear / modificar / eliminar) visibles | [ ] |
| Reglas personalizadas probadas con `wazuh-logtest` | [ ] |
| Reglas locales de Suricata cargadas y alertando | [ ] |
| Configuración sanitizada y copiada al repositorio | [ ] |
| Snapshot `configuracion-completa-AAAA-MM-DD` tomado en cada VM | [ ] |

---

*Documento anterior: [02 - Instalación](02-instalacion.md) · Siguiente: [04 - Lecciones aprendidas](04-lecciones-aprendidas.md) · Ver también: [05 - Troubleshooting](05-troubleshooting.md)*
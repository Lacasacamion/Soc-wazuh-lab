# Soc-wazuh-lab

##  Resumen y Objetivo
Diseño, despliegue y operación de un laboratorio de **Centro de Operaciones de Seguridad (SOC)** utilizando **Wazuh SIEM/XDR**. Este repositorio documenta la arquitectura de red, el endurecimiento (*hardening*) de endpoints, la ingesta de telemetría avanzada (Sysmon/FIM), el desarrollo de reglas de detección mapeadas a **MITRE ATT&CK** y la elaboración de informes de respuesta a incidentes (*Incident Reports*).

---

##  Arquitectura del Laboratorio
El entorno virtualizado simula una red corporativa monitorizada frente a escenarios de intrusión:

* **SIEM Manager:** Ubuntu Server (Wazuh Server, Indexer & Dashboard).
* **Endpoints Monitoreados:** Windows (Agente Wazuh + Sysmon) y Linux Ubuntu/Debian (Agente Wazuh + Auditd/FIM).
* **Atacante (Red Team):** Kali Linux para emulación de técnicas adversarias.

 *Ver detalle completo de red y hardware asignado en: [`docs/01-arquitectura.md`](docs/01-arquitectura.md)*

---

##  Casos de Uso e Investigación de Incidentes

| ID | Escenario de Ataque | Técnica MITRE ATT&CK | Documentación del Caso | Informe SOC (IR) |
| :--- | :--- | :--- | :--- | :--- |
| **UC-01** | Fuerza Bruta sobre servicio SSH | `T1110.001` - Password Guessing | [Ver Caso 01](use-cases/01-ssh-brute-force/README.md) | [IR-001](incident-reports/IR-001-ssh-brute-force.md) |
| **UC-02** | Reconocimiento y Escaneo de Red (Nmap) | `T1046` - Network Service Discovery | [Ver Caso 02](use-cases/02-escaneo-nmap/) | *En progreso* |
| **UC-03** | Ejecución de PowerShell Sospechoso | `T1059.001` - PowerShell | [Ver Caso 03](use-cases/03-powershell-sospechoso/) | *En progreso* |

 *Ver matriz completa de cobertura en: [`detections/mitre-mapping.md`](detections/mitre-mapping.md)*

---

##  Estructura del Proyecto

```text
Soc-wazuh-lab/
├── docs/                # Documentación de arquitectura, instalación, hardening y lecciones
├── configs/             # Archivos ossec.conf sanitizados y configuraciones de Sysmon/Agentes
├── detections/          # Reglas XML propias, decoders y mapeo técnico MITRE ATT&CK
├── use-cases/           # Ejecución técnica de ataques, evidencias y lógica de detección
├── incident-reports/    # Informes ejecutivos y técnicos de incidentes (formato SOC)
└── scripts/             # Automatización de despliegue y emulación de ataques
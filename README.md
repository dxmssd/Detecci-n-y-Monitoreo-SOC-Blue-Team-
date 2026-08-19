# Deteccion-y-Monitoreo-SOC-Blue-Team

# Security Incident Simulation & Detection Lab

## Escenario 1: Detección de Integridad de Archivos (FIM)
- **Táctica:** Persistence / Defense Evasion
- **Acción:** Creación y asignación de permisos 777 en `/bin/wazuh_test_backdoor`.
- **Detección en Wazuh:** Rule ID `554` (File added to system), Rule ID `550`.
- **Validación:** Eventos capturados en el módulo FIM con hashes MD5/SHA256.

---

## Escenario 2: Ataque de Fuerza Bruta SSH y Mitigación
- **Táctica:** MITRE ATT&CK T1110 (Brute Force) / T1021.004 (SSH).
- **Simulación:** Script automatizado de intentos recurrentes con `sshpass`.
- **Detección en Wazuh:** 
  - Rule ID `5710`: SSH authentication failure.
  - Rule ID `5712`: SSHD brute force trying to get access (Severidad 10).
- **Respuesta Activa (Active Response):** Bloqueo automático de IP origen vía `firewall-drop` durante 60s.

---

## Escenario 3: Agotamiento de Recursos / DoS Capa 7 (Slowloris)
- **Táctica:** MITRE ATT&CK T1499 (Endpoint Denial of Service).
- **Simulación:** Script de retención de sockets HTTP contra Apache2.
- **Impacto:** Caída de disponibilidad (`curl timeout`) con uso mínimo de ancho de banda.
- **Detección:** Logs de error de Apache y saturación de workers.
# Documentación Técnica — Topología FortiGate P5

**Estudiante:** Ashley Fabian
**Matrícula:** 2025-0773
**Asignación:** Proyecto 5 (P5) — Seguridad Perimetral con FortiGate
**Direccionamiento base (según matrícula):** 25.7.73.0
**Video de demostración:** https://youtu.be/ds18HLsziEI

---

## 1. Objetivo de la red

Diseñar e implementar, en su totalidad a través de la interfaz gráfica (GUI) de FortiGate, una topología de red segmentada que separe el tráfico de usuarios y de servidores, controle el acceso a Internet mediante NAT, y aplique múltiples capas de seguridad perimetral: filtrado de contenido web, control de aplicaciones, detección de escaneos de red y un Firewall de Aplicaciones Web (WAF) para proteger un servidor web interno. El objetivo es demostrar, de forma práctica, cómo un firewall de próxima generación puede reforzar las políticas de una organización (acceso restringido entre segmentos, bloqueo de fuga de datos hacia dominios no autorizados, mitigación de amenazas comunes como escaneo de puertos e inyección de código) sin perder disponibilidad para el tráfico legítimo.

---

## 2. Topología de red

### 2.1 Diagrama lógico

```
                      Internet
                         │
                    [Cloud/NAT GNS3VM]
                         │
                       port1 (WAN, DHCP)
                    ┌────────────┐
                    │  FortiGate │
                    └────────────┘
                  port2         port3
                    │             │
               [Switch1]      [Switch2]
                    │             │
            [Kali-Cliente]  [Kali-Servidor]
             (LAN Usuarios)  (LAN Servidores
                              + Apache + WAF)
```

> 📸 **Captura sugerida:** Canvas completo de GNS3 con la topología, incluyendo un recuadro de texto visible con nombre y matrícula del estudiante.
![](Topologia.png)

### 2.2 Direccionamiento IP

| Segmento | Red / CIDR | Interfaz FortiGate | Host(s) |
|---|---|---|---|
| WAN (Internet) | Asignada por DHCP del entorno GNS3VM | port1 — `192.168.241.x /24` | — |
| LAN Usuarios | `25.7.73.0/25` | port2 = `25.7.73.1/25` | Kali-Cliente = `25.7.73.2` (DHCP) |
| LAN Servidores | `25.7.74.128/28`* | port3 = `25.7.74.129/28` | Kali-Servidor = `25.7.74.130` (estático) |

\* *Nota de diseño: el direccionamiento se derivó del bloque de matrícula 25.7.73.0. Durante la implementación se detectó que el adaptador de red físico del host (loopback) ya utilizaba el rango 25.7.73.0/26, por lo que la LAN de Servidores se reubicó a 25.7.74.128/28 para evitar solapamiento, manteniendo el mismo bloque de matrícula (25.7.7x.x).*

### 2.3 Equipos utilizados

| Equipo | Rol | Notas |
|---|---|---|
| Cloud (GNS3VM) | Salida a Internet | Nodo Cloud de GNS3 enlazado a la interfaz de red de la GNS3 VM, no del host Windows directamente |
| FortiGate-VM64-KVM | Firewall / Gateway | v7.0.9, licencia de evaluación |
| Switch1 | Switch L2 | Segmento LAN Usuarios |
| Switch2 | Switch L2 | Segmento LAN Servidores |
| Kali-Cliente | Estación de usuario | VM VMware integrada a GNS3, usada para pruebas de navegación y escaneo (nmap) |
| Kali-Servidor | Servidor Web | VM VMware integrada a GNS3, corriendo Apache2 |

---

## 3. Configuración implementada

### 3.1 Interfaces de red

Configuración realizada en **Network → Interfaces**:

- **port1 (WAN):** modo DHCP, rol WAN.
- **port2 (LAN Usuarios):** modo estático, `25.7.73.1/255.255.255.128`, rol LAN, acceso administrativo HTTPS/PING habilitado.
- **port3 (LAN Servidores):** modo estático, `25.7.74.129/255.255.255.240`, rol LAN, acceso administrativo HTTPS/PING habilitado.

> 📸 **Captura:** Network → Interfaces (vista de lista con las 3 interfaces).
![](PUERTOSCONINT.png)

> 📸 **Captura:** Consola — `get system interface physical` mostrando las máscaras /25 y /28.
![](Getsystem.png)

### 3.2 DHCP en LAN Usuarios

Activado directamente en la configuración de **port2**, con:
- Rango: `25.7.73.2` – `25.7.73.126`
- Máscara: `255.255.255.128`
- Gateway: IP de la interfaz (`25.7.73.1`)
- DNS: `8.8.8.8`

> 📸 **Captura:** Sección DHCP Server dentro de port2.
![](DHCPport2.png)

> 📸 **Captura:** Terminal Kali-Cliente — `ip addr show eth0` con IP asignada por DHCP.
![](image.png)

### 3.3 Ruta por defecto

Configurada en **Network → Static Routes**: destino `0.0.0.0/0.0.0.0` vía `port1`, con gateway obtenido automáticamente por DHCP.

> 📸 **Captura:** Network → Static Routes.
![](StaticRoutes.png)

### 3.4 NAT

Implementado como *policy-based NAT* dentro de las políticas de firewall de salida (no como módulo aparte, ya que FortiGate integra NAT en la política misma):

- **Usuarios-a-Internet:** port2 → port1, NAT activado (Use Outgoing Interface Address).
- **Servidores-a-Internet:** port3 → port1, NAT activado (Use Outgoing Interface Address).

> 📸 **Captura:** Policy & Objects → Firewall Policy (lista con columna NAT activa).
![](Firewallpolicy.png)

> 📸 **Captura:** Edit de una política mostrando el switch NAT.
![](NAt.png)

### 3.5 Acceso a Internet

Verificado mediante ping desde el propio FortiGate y desde ambas máquinas Kali hacia `8.8.8.8`, con 0% de pérdida de paquetes.

> 📸 **Captura:** Consola FortiGate — `execute ping 8.8.8.8` exitoso.
![](\executeping.png)

> 📸 **Captura:** Terminal Kali-Cliente y Kali-Servidor — `ping -c 4 8.8.8.8` exitoso.
![](\Ping8KC.png)
![](\Ping8KS.png)

### 3.6 Política: solo HTTP de Usuarios hacia Servidores

Política **Usuarios-a-Servidores-HTTP**:
- Incoming: port2 | Outgoing: port3
- Source: all | Destination: Server-Web (25.7.74.130)
- Service: **HTTP únicamente**
- Action: Accept | NAT: desactivado (tráfico interno)

Todo tráfico no contemplado en esta política (o en las de salida a Internet) es bloqueado automáticamente por la política de denegación implícita de FortiGate — no fue necesario crear una regla de bloqueo explícita.

**Evidencia de funcionamiento:**
```
$ curl -I http://25.7.74.130
HTTP/1.1 200 OK                     ← HTTP permitido

$ ping -c 3 25.7.74.130
100% packet loss                    ← ICMP bloqueado (todo lo demás)
```

> 📸 **Captura:** Edit de la política Usuarios-a-Servidores-HTTP.
![](\USacctionacept.png)
> 📸 **Captura:** Terminal con ambos resultados (curl y ping).
![](\kaliclientecurlsipingno.png)

### 3.7 Bloqueo de redes sociales

Perfil **Web Filter → Bloqueo-RedesSociales-ITLA**, categoría FortiGuard **"Social Networking"** en modo Block. Aplicado a la política Usuarios-a-Internet con SSL Inspection en modo `certificate-inspection`.

**Evidencia:**
```
$ curl -I https://www.facebook.com
Connection timed out            ← Bloqueado
```

> 📸 **Captura:** Perfil Web Filter con la categoría Social Networking en Block.
![](\socialnetworkingblock.png)

> 📸 **Captura:** Terminal con el timeout de Facebook.
![](\nofacebook.png)

### 3.8 Bloqueo de itla.edu.do y subdominios

Dentro del mismo perfil de Web Filter, sección **Static URL Filter**:
- `itla.edu.do` → Block (Wildcard)
- `*.itla.edu.do` → Block (Wildcard, cubre subdominios)

**Evidencia:**
```
$ curl -I https://www.itla.edu.do
Connection timed out            ← Bloqueado
```

> 📸 **Captura:** Static URL Filter con las entradas de itla.edu.do.
![](\italblock.png)

> 📸 **Captura:** Terminal con el timeout de itla.edu.do.
![](\noitla.png)

### 3.9 Bloqueo de llamadas de WhatsApp

Perfil **Application Control → Bloqueo-WhatsApp-Calls**, con la firma `WhatsApp_VoIP.Call` en modo **Block**, aplicado a la política Usuarios-a-Internet.

**Evidencia (log de Forward Traffic):**
```
Application Control Sensor : Bloqueo-WhatsApp-Calls
Application Name           : Whatsapp_Web
Security Action            : Blocked
Policy                     : Usuarios-a-Internet
Source IP                  : 25.7.73.2
Destination IP              : 157.240.14.52 (Meta/WhatsApp)
```

**Limitación documentada:** la base de firmas de Application Control (APP-DB) de la licencia de evaluación data de 2015 y no recibe actualizaciones de FortiGuard (verificado en System → FortiGuard). Debido a esto, la firma `WhatsApp_VoIP.Call` no logra diferenciar con precisión el tráfico de mensajería de texto del de llamadas de voz/video dentro de la misma sesión TLS de WhatsApp Web, resultando en el bloqueo de la sesión completa en lugar de únicamente la función de llamada. Esta limitación es propia del entorno de laboratorio (licencia eval sin updates) y no de la configuración aplicada, la cual sigue exactamente el procedimiento correcto para bloquear específicamente esa firma.

> 📸 **Captura:** Perfil Application Control con WhatsApp_VoIP.Call en Block.
![](\whatsappblockedit.png)

> 📸 **Captura:** Log completo de Forward Traffic con los campos citados arriba.
![](\BloqueoWhatsApp.png)

> 📸 **Captura:** Navegador mostrando el timeout al intentar cargar web.whatsapp.com.
![](\nowhastapp.png)

### 3.10 Detección y bloqueo de escáneres de red

Configurado mediante **DoS Policy** (Network/Policy & Objects → DoS Policy) — la herramienta correcta de FortiGate para detectar anomalías de tráfico como escaneos, a diferencia de IPS que apunta a firmas de exploits específicos:

- **Política:** Anti-Escaneo-LAN
- **Incoming Interface:** port2
- **Anomalía:** `tcp_port_scan` → Enable, Action: Block, Threshold: 100
- **Anomalía:** `icmp_sweep` → Enable, Action: Block

**Evidencia (log de Anomaly):**
```
Attack Name   : tcp_port_scan
Threat Level  : Critical
Action        : clear_session
Message       : anomaly: tcp_port_scan, 101 > threshold 100,
                repeats 592 times since last log, pps 95
Policy        : Anti-Escaneo-LAN
Source IP     : 25.7.73.2
```

Prueba realizada con `nmap -sS -Pn -p- -T4 25.7.74.130` (escaneo SYN de los 65,535 puertos con temporización agresiva), disparando la anomalía de inmediato.

> 📸 **Captura:** DoS Policy con la anomalía tcp_port_scan configurada.
![](\policyportscanDoS.png)

> 📸 **Captura:** Terminal ejecutando el nmap.
![](\nmapenKali.png)

> 📸 **Captura:** Log de Anomaly con los campos citados.
![](\IPSDoSPolicyCRITICAL.png)

### 3.11 WAF en el servidor web

Perfil **Web Application Firewall → WAF-ServidorWeb**, con las firmas:
- SQL Injection → Enable/Block
- Cross Site Scripting → Enable/Block
- Generic Attacks → Enable/Block

Aplicado a la política **Usuarios-a-Servidores-HTTP**, la cual requirió cambiar su **Inspection Mode** de *Flow-based* a **Proxy-based**, ya que el WAF de FortiGate solo está disponible en políticas basadas en proxy.

**Evidencia — intento de XSS:**
```
$ curl "http://25.7.74.130/?search=<script>alert(1)</script>"
→ Página de bloqueo del FortiGate:
  "This transfer is blocked by a Web Application Firewall"
  Event ID: 10000057 | Event Type: signature
```

**Evidencia — intento de SQL Injection:**
```
$ curl "http://25.7.74.130/?id=1%27%20OR%20%271%27=%271"
→ Página de bloqueo del FortiGate:
  "This transfer is blocked by a Web Application Firewall"
  Event ID: 30000040 | Event Type: signature
```

**Confirmación de que el tráfico legítimo no se ve afectado:**
```
$ curl -I http://25.7.74.130
HTTP/1.1 200 OK
```

> 📸 **Captura:** Perfil WAF con las 3 categorías en Block.
![](\WAFenable.png)

> 📸 **Captura:** Edit de la política mostrando Inspection Mode: Proxy-based y WAF activado.
![](\proxybasedyWAF.png)

> 📸 **Captura:** Página de bloqueo del WAF (intento XSS).
![](\xss.png)
![](\xss2.png)

> 📸 **Captura:** Página de bloqueo del WAF (intento SQLi).
![](\SQLinjecction.png)
![](\SQLinjecction2.png)

> 📸 **Captura:** curl -I devolviendo 200 OK (tráfico legítimo).
![](\200OK.png)

---

## 4. Resumen de cumplimiento

| # | Requisito | Cumplido |
|---|---|---|
| 1 | Configuración 100% por GUI | ✅ |
| 2 | Acceso a Internet | ✅ |
| 3 | LAN Usuarios /25 | ✅ |
| 4 | LAN Servidores /28 | ✅ |
| 5 | IP en interfaces | ✅ |
| 6 | DHCP en LAN Usuarios | ✅ |
| 7 | Ruta por defecto | ✅ |
| 8 | NAT | ✅ |
| 9 | Solo HTTP Usuarios→Servidores | ✅ |
| 10 | Bloqueo de redes sociales | ✅ |
| 11 | Bloqueo de llamadas WhatsApp | ✅ (con limitación documentada de firma) |
| 12 | Bloqueo de itla.edu.do y subdominios | ✅ |
| 13 | Detección/bloqueo de escáneres de red | ✅ |
| 14 | WAF en servidor web | ✅ |

---

## 5. Notas y limitaciones del entorno

- La licencia de evaluación de FortiGate-VM64-KVM no permite actualizar las bases de firmas de FortiGuard (IPS/Application Control), quedando fijas en versiones de 2015. Esto afecta la granularidad del bloqueo de WhatsApp (sección 3.9), pero no impide demostrar que la configuración y el bloqueo funcionan correctamente.
- El acceso a Internet se logró conectando el nodo Cloud de GNS3 a la interfaz de red de la **GNS3 VM** (no del host Windows directamente), tras descartar los adaptadores VMnet/NAT nativos de Windows, que presentaron fallos de resolución ARP en este entorno.
- Se ajustó el MTU de port1 a 1400 para mitigar problemas de fragmentación TLS observados en el entorno virtualizado (QEMU/GNS3).

---

**Fin del documento.**

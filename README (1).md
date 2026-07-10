# Documentación Técnica — Configuración de Seguridad Perimetral con FortiGate

**Dispositivo:** FortiGate v7.6.2 build3462
**Entorno:** PNETLab
**Matrícula asociada al direccionamiento:** 2025-0689

---

## 1. Objetivo de la Red

Implementar un firewall perimetral FortiGate que segmente una red corporativa simulada en dos zonas (usuarios y servidores), garantizando:

- Conectividad controlada a Internet para la LAN de usuarios, con inspección profunda del tráfico saliente.
- Publicación de un servidor web interno accesible únicamente por HTTP desde la LAN de usuarios, protegido con un Web Application Firewall (WAF).
- Restricción de acceso a redes sociales y al dominio institucional `itla.edu.do`.
- Bloqueo de llamadas de voz/video por WhatsApp, permitiendo el uso de mensajería de texto.
- Detección y bloqueo de escaneos de red (Nmap, escáneres de vulnerabilidades) mediante IPS y políticas Anti-DoS.
- Registro (logging) de todos los eventos de seguridad relevantes para su posterior auditoría.

---

## 2. Topología de Red

### 2.1 Direccionamiento IP

| Zona | Interfaz física | Rol | IP / Máscara | Rango útil | Gateway |
|---|---|---|---|---|---|
| WAN | port1 | Salida a Internet | `10.0.0.220/24` | — | `10.0.0.1` |
| LAN Usuarios | port2 | Red interna de usuarios | `10.6.89.1/25` (255.255.255.128) | 10.6.89.2 – 10.6.89.126 | 10.6.89.1 |
| LAN Servidores | port3 | Red interna de servidores | `10.6.89.129/28` (255.255.255.240) | 10.6.89.130 – 10.6.89.142 | 10.6.89.129 |

**Servidor web publicado:** `10.6.89.130`

### 2.2 Diseño de subredes

No se utilizaron VLANs: al no solaparse los bloques `10.6.89.0/25` (usuarios) y `10.6.89.128/28` (servidores), la segmentación se resolvió con interfaces físicas dedicadas (port2 / port3), simplificando el diseño y facilitando la aplicación de políticas independientes por zona.

### 2.3 Evidencia — Interfaces configuradas

![Interfaces físicas](img/13-interfaces-fisicas.png)

*Panel `Network > Interfaces` mostrando las tres interfaces físicas activas: `port1` (WAN, 10.0.0.220/24, con acceso PING/HTTPS/SSH/HTTP), `Lan-usuarios (port2)` (10.6.89.1/25, PING/HTTPS) y `LAN-Servidores (port3)` (10.6.89.129/28, PING/HTTPS).*

> **Observación:** la IP de `port1` en esta implementación quedó en `10.0.0.220/24` en lugar del `10.0.0.2/24` planteado originalmente; ambas son válidas dentro del mismo segmento WAN, pero conviene documentar el valor real usado.

---

## 3. Enrutamiento

Se configuraron dos rutas: la ruta por defecto hacia Internet a través de `port1`, y una ruta explícita hacia la subred de servidores a través de `port3`.

![Tabla de rutas estáticas](img/11-static-routes.png)

*`Network > Static Routes`: `0.0.0.0/0` con gateway `10.0.0.1` vía `port1`, y `10.6.89.128/28` vía la interfaz `LAN-Servidores (port3)`.*

---

## 4. Asignación Dinámica de Direcciones (DHCP)

La LAN de usuarios recibe direccionamiento automático mediante un servidor DHCP configurado directamente sobre la interfaz `port2`.

![Configuración DHCP](img/12-dhcp-server.png)

*`Network > Interfaces > port2 > DHCP Server`: rango de direcciones `10.6.89.2 – 10.6.89.126`, máscara `255.255.255.128`, gateway por defecto igual a la IP de la interfaz, DNS heredado del sistema y tiempo de concesión (lease) de `604800` segundos (7 días).*

---

## 5. Políticas de Firewall

Se definieron políticas independientes por flujo de tráfico, cada una con los perfiles de seguridad correspondientes a su función:

| # | Nombre | Origen → Destino | Servicio | NAT | Perfiles de seguridad aplicados |
|---|---|---|---|---|---|
| 1 | `USR-a-SRV-HTTP` | LAN-Usuarios → LAN-Servidores | HTTP | Deshabilitado | WAF (`WAF-ServidorWeb`), IPS |
| 2 | `USR-a-WAN` | LAN-Usuarios → port1 (Internet) | ALL | Habilitado | Control de Aplicaciones (`AppCtrl-Bloqueos`), DNS Filter (`DNS-Bloqueo-ITLA`), SSL Inspection |

![Políticas de firewall](img/09-firewall-policies.png)

*`Policy & Objects > Firewall Policy`: la política `USR-a-SRV-HTTP` permite tráfico HTTP de usuarios hacia servidores sin NAT (comunicación interna) y aplica WAF; la política `USR-a-WAN` permite salida a Internet con NAT habilitado y aplica control de aplicaciones y filtrado DNS.*

El **Implicit Deny** al final de la lista bloquea cualquier tráfico no contemplado explícitamente, incluyendo comunicación directa no autorizada entre `port2` y `port3` fuera de HTTP.

---

## 6. Filtrado Web y DNS — Bloqueo de Redes Sociales e `itla.edu.do`

Para bloquear de forma efectiva tanto el acceso HTTP como HTTPS (donde el filtrado por URL es limitado por el cifrado), se implementó un **DNS Filter** con reglas estáticas de dominio.

![Perfil DNS Filter](img/08-dnsfilter-itla.png)

*`Security Profiles > DNS Filter > DNS-Bloqueo-ITLA`: filtro de dominio estático activo con las entradas `itla.edu.do` (tipo simple) y `*.itla.edu.do` (tipo wildcard, para subdominios), ambas con acción "Redirect to Block Portal".*

### 6.1 Validación — Tráfico bloqueado

![Log de bloqueo itla.edu.do](img/04-webfilter-log-itla-bloqueado.png)

*Registro de tráfico web mostrando las peticiones `http://itla.edu.do/` y `http://itla.edu.do/favicon.ico` desde el host `10.6.89.3`, ambas con estado `Blocked`, confirmando que la política de filtrado opera correctamente.*

---

## 7. Control de Aplicaciones — Restricción de WhatsApp y Redes Sociales

El perfil de Control de Aplicaciones combina el bloqueo por categoría (Social Media, Proxy) con overrides específicos por aplicación para permitir mensajería de texto en WhatsApp mientras se bloquean las llamadas de voz/video.

![Perfil de control de aplicaciones](img/07-appctrl-perfil-categorias.png)

*`Security Profiles > Application Control`: las categorías `Social Media` y `Proxy` están bloqueadas globalmente. En la tabla de overrides se aplica, en orden de prioridad: (1) `WhatsApp_VoIP.Call` → Block, (2) `HTTP.BROWSER_Firefox` → Allow, (3) `Whatsapp_Web`, `Whatsapp` y `WhatsApp.File_Transfer` → Block.*

### 7.1 Validación — Tráfico bloqueado

![Log de bloqueo de aplicaciones](img/02-appctrl-log-whatsapp-redes.png)

*Registro de tráfico de aplicaciones desde `10.6.89.3` mostrando bloqueos efectivos sobre `Whatsapp` (dos eventos, resueltos vía servidores DNS de Fortiguard), `Instagram` y `Facebook`, todos con acción `Block`.*

---

## 8. Prevención de Intrusiones (IPS) — Detección de Escaneo de Red

Se configuró un perfil IPS con firmas específicas orientadas a la detección de herramientas de reconocimiento y escaneo (Nmap, Acunetix, escáneres de servidores de aplicaciones), y se complementó con una política Anti-DoS que detecta anomalías de tráfico a nivel de capa 3/4.

![Perfil IPS](img/06-ips-perfil-ipsbase.png)

*`Security Profiles > Intrusion Prevention > IPS-Base`: firmas agrupadas por severidad con acción `Block` y `Packet Logging` habilitado; incluye firmas específicas como `Acunetix.Web.Vulnerability.Scanner` y `Apache.Struts.2.DefaultActionMapper.Redirection`.*

![Política Anti-DoS](img/10-dospolicy-antiscan.png)

*`Policy & Objects > DoS Policy > DoS-AntiScan` sobre `port1`: anomalías L3 (`ip_src_session`, `ip_dst_session`) y L4 (`tcp_syn_flood`, `tcp_port_scan`, `tcp_src_session`, `tcp_dst_session`, `udp_flood`, `udp_scan`, `udp_src_session`, `udp_dst_session`) con acción `Block`, logging activo y umbrales configurados (por ejemplo, `tcp_port_scan` con umbral de 1000).*

### 8.1 Validación — Escaneo detectado

![Log de eventos de escaneo](img/03-ips-log-warning-scan.png)

*Registro de eventos de seguridad con nivel `Warning` originados desde `10.6.89.3`, con acción `blocked`, evidenciando la detección activa de actividad de escaneo por parte del IPS/Anti-DoS.*

---

## 9. Web Application Firewall (WAF) — Protección del Servidor Web

El servidor web (`10.6.89.130`) está protegido mediante un perfil WAF aplicado en la política `USR-a-SRV-HTTP`, con firmas orientadas a los ataques web más comunes.

![Firmas WAF](img/05-waf-firmas-sqli-xss.png)

*`Security Profiles > Web Application Firewall`: firmas `Cross Site Scripting` y `SQL Injection` / `SQL Injection (Extended)` habilitadas con acción `Block`; `Cross Site Scripting (Extended)` en modo `Monitor`.*

### 9.1 Validación — Ataques bloqueados

![Log de ataques bloqueados por IPS/WAF](img/01-ips-waf-log-ataques.png)

*Registro de eventos de seguridad originados desde `10.6.89.3` (severidad `High`/`Medium`), todos con acción `dropped`, correspondientes a intentos de explotación reales: `HTTP.URI.SQL.Injection`, `Adobe.XML.Entity.Injection`, `Easy.Hosting.Control.Panel.FTP.Account.Security.Bypass`, `Oxide.WebServer.Directory.Traversal`, `VMware.Server.Directory.Traversal` y `Majordomo2.Directory.Traversal`. Esta evidencia confirma que el WAF/IPS está deteniendo activamente ataques dirigidos al servidor web publicado.*

---

## 10. Resumen de Configuraciones Aplicadas

| Componente | Nombre del objeto/perfil | Función |
|---|---|---|
| Interfaces | port1, port2 (Lan-usuarios), port3 (LAN-Servidores) | Segmentación WAN / LAN Usuarios / LAN Servidores |
| Ruta estática | `0.0.0.0/0` → port1 / `10.6.89.128/28` → port3 | Enrutamiento a Internet y hacia servidores |
| DHCP | Servidor sobre port2 | Asignación automática a LAN de usuarios |
| Política de firewall | `USR-a-SRV-HTTP` | Acceso HTTP usuarios→servidor, con WAF |
| Política de firewall | `USR-a-WAN` | Salida a Internet con NAT, Control de Apps y DNS Filter |
| DNS Filter | `DNS-Bloqueo-ITLA` | Bloqueo de `itla.edu.do` y subdominios |
| Application Control | `AppCtrl-Bloqueos` | Bloqueo de redes sociales y llamadas WhatsApp |
| IPS | `IPS-Base` | Detección/bloqueo de escaneos y explotación |
| DoS Policy | `DoS-AntiScan` | Bloqueo de anomalías de escaneo (L3/L4) en port1 |
| WAF | Perfil sobre `USR-a-SRV-HTTP` | Protección contra XSS/SQLi al servidor web |

---

## 11. Conclusiones

Los registros de log capturados confirman que las políticas configuradas están operando de forma efectiva: se bloquea el acceso a redes sociales y al dominio institucional, se restringen selectivamente las funciones de WhatsApp, y tanto el IPS como el WAF detienen intentos reales de escaneo y explotación contra el servidor web interno, todo mientras se mantiene el logging necesario para trazabilidad y auditoría.

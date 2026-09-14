# Guía detallada — Homework No. 2
## Design of an IDS based on Rules Using SNORT
**Curso:** LIS4062 Information Security — Autumn 2026  
**Profesor:** Dr. Vicente Alarcón Aquino  
**Entrega:** Lunes 21 de septiembre 2026, 11:59 PM  
**Modalidad:** Trabajo en equipo (máximo 3 integrantes)

---

## 1. Resumen de la tarea (qué se pide)

Diseñar un **IDS basado en reglas con Snort** capaz de **identificar** (detectar y alertar) al menos estos tipos de actividad maliciosa o de reconocimiento:

| # | Actividad | Enfoque del laboratorio |
|---|-----------|-------------------------|
| 1 | Ping / ICMP | Reconocimiento y/o PoD (Ping of Death) |
| 2 | DoS / DDoS (TCP SYN Flood) | Ataque de denegación de servicio |
| 3 | ARP spoofing / ARP poisoning | Envenenamiento de tabla ARP en LAN |
| 4 | DNS Poisoning | Redirección DNS falsa (tras ARP spoof) |
| 5 | Traceroute | Reconocimiento de ruta |
| 6 | Nmap scans | Escaneo de puertos / OS / servicios |

**Entregables obligatorios:**
1. **Reporte PDF** por cada integrante del equipo (mismo contenido), con nombre: `HW2_ID_ID_ID.pdf`
2. **Video en inglés** (cámara encendida, participación de **todos**), con URL incluida en el reporte
3. Subida al módulo *Self-Autonomous Learning Activities* en Blackboard  
   - Si no se sube individualmente → calificación **0.0**  
   - Entregas tardías **no se aceptan**

---

## 2. Estructura obligatoria del reporte

Incluye **todas** estas secciones (en este orden recomendado):

1. **Cover page** (portada)
2. **Abstract**
3. **Introduction**
4. **Methodology**
5. **Results and Discussion**
6. **Conclusions**
7. **Bibliography**

**Reglas de formato/citas:**
- Toda figura, tabla y referencia de la bibliografía debe citarse en el texto.
- Portada debe incluir: Title y Homework #, Student Name, ID, Course, Date, Autumn Term.
- En el reporte debe aparecer el **URL del video**.

### 2.1 Qué poner en cada sección (plantilla útil)

**Abstract (150–250 palabras):** objetivo del IDS, entorno (Escenario A: 1 laptop / 2 VMs, o Escenario B: 3 laptops), ataques evaluados, resultado cualitativo (qué se detectó), conclusión breve.

**Introduction:** qué es un IDS, Snort (NIDS basado en reglas), por qué detectar SYN flood / ARP-DNS / recon; objetivo del homework; alcance y limitaciones (laboratorio aislado; colocación del sensor según el escenario).

**Methodology:** topología (A o B), SO, IPs, dónde corre Snort, reglas escritas, herramientas usadas para generar tráfico de prueba, procedimiento de captura (Wireshark/Snort alerts), criterios de éxito (“alerta aparece en `alert`/`fast` log”).

**Results and Discussion:** capturas de pantalla de alertas Snort, reglas usadas, comparación antes/después, falsos positivos, limitaciones. Discute por qué cada regla dispara.

**Conclusions:** aprendizajes, efectividad del IDS, mejoras (Suricata, umbrales, IPS, listas blancas).

**Bibliography:** al menos las referencias del PDF del curso [1–11] + documentación oficial de Snort + cualquier fuente adicional citada.

---

## 3. Conceptos clave (para el reporte y el video)

### 3.1 IDS vs IPS
- **IDS (Intrusion Detection System):** detecta y alerta; no necesariamente bloquea.
- **IPS:** puede bloquear/mitigar. En este homework el foco es **IDS con Snort**.

### 3.2 Snort (visión práctica)
Snort inspecciona paquetes y aplica reglas del estilo:

```text
action protocol src_ip src_port -> dst_ip dst_port (opciones)
```

Ejemplo conceptual:

```text
alert tcp any any -> $HOME_NET any (msg:"Possible SYN flood"; flags:S; threshold:...; sid:1000001; rev:1;)
```

Componentes típicos en Linux:
- Binario `snort`
- `snort.conf` / `snort.lua` (según versión)
- Archivo(s) de reglas `.rules`
- Logs: `alert`, `fast`, `json` (según configuración), PCAP opcional

### 3.3 TCP SYN Flood (DoS)
Handshake normal: SYN → SYN/ACK → ACK.  
En SYN flood el atacante envía muchos SYN y **no completa** el handshake → muchas conexiones *half-open* (`SYN_RECEIVED`).  
Verificación auxiliar en víctima: `netstat -n -p tcp` (o `ss -ant`) buscando muchos estados `SYN-RECV`.

### 3.4 ARP / DNS poisoning
- **ARP spoofing:** asociar la MAC del atacante a la IP del gateway/víctima.
- **DNS poisoning (en LAN):** suele apoyarse en MITM (ARP) y responder DNS falso (plugin DNS spoof de Ettercap / módulos de Bettercap).
- El enunciado sugiere Bettercap + Kali + Ettercap, y editar `etter.dns` para redirigir (ej. Facebook → otro sitio).

### 3.5 Reconocimiento (Network Mapping / PoD)
Con `traceroute`, `ping`/`hping3` y `nmap` se obtiene layout, hosts, puertos, SO, etc.  
**PoD (Ping of Death):** histórico ICMP malformado/oversized; hoy se usa sobre todo para estudiar detección ICMP anómala.

> **Ética / seguridad:** realiza **solo** en tu laboratorio aislado (VMs o red privada del equipo). No ataques a redes ajenas, campus o Internet.

---

## 4. Escenarios de laboratorio

Elige **exactamente uno** y descríbelo en Methodology. No hace falta un segundo escenario ni hardware extra (hub, switch managed, cuarta VM).

| Situación del equipo | Escenario | Qué se usa |
|----------------------|-----------|------------|
| Una sola laptop | **A** | VirtualBox: **2 VMs** (Kali + Ubuntu) |
| Simulación en red real | **B** | **3 laptops** (Attacker, Victim, IDS/gateway) |

---

### Escenario A — Una laptop, 2 VMs (VirtualBox)

**Idea:** el laboratorio cabe en un solo host. No se crea una VM de IDS ni una de gateway: Snort corre en la víctima y el **host** (`192.168.56.1`) es el gateway lógico para ARP/DNS.

#### Topología (fija)

```text
[ Host Linux + VirtualBox ]
   Host-Only adapter = 192.168.56.1   (gateway / DNS lógico del lab)
            |
     Host-Only  192.168.56.0/24
            |
     +------+------+
     |             |
 [VM1 Kali]    [VM2 Ubuntu]
  Attacker      Victim + Snort + HTTP/SSH + Wireshark
  .10           .20
```

**Roles:**
| Nodo | SO | IP | Rol |
|------|----|----|-----|
| VM1 Attacker | Kali Linux | 192.168.56.10/24 | Genera el tráfico de prueba |
| VM2 Victim + IDS | Ubuntu Server/Desktop | 192.168.56.20/24, GW `.1` | Servicios objetivo, Snort, Wireshark, `ss`/`netstat` |
| Host (no es VM) | Linux del equipo | 192.168.56.1/24 | Tercer nodo L2: gateway que se suplanta en ARP/DNS |

`HOME_NET`: `192.168.56.0/24`. Snort escucha la NIC Host-Only de VM2 (`snort -i <iface>`).

#### Por qué 2 VMs bastan (y cumplen el enunciado)

| Actividad | Cómo se cubre |
|-----------|----------------|
| Ping / ICMP | Kali → `.20`; Snort en VM2 ve `itype:8` |
| TCP SYN Flood | Servicio en VM2; Kali inunda `.20`; alertas + `ss -ant` / `SYN-RECV` |
| Nmap | Escaneo a `.20`; reglas SYN/NULL/FIN/XMAS |
| Traceroute | Kali → `.20` (1 hop). Sigue generando probes ICMP/UDP; documentar la limitación |
| ARP spoof | Envenenar ARP de VM2 respecto al gateway **host** `.1` |
| DNS poisoning | Tras el ARP, DNS spoof hacia un **dominio de prueba** del lab (Ettercap/Bettercap) |

En Methodology: el sensor está **en el host protegido**, no en un SPAN. Es válido para el homework; no es un NIDS con mirroring.

#### Configuración VirtualBox (checklist)
1. Crear **una** red **Host-Only** (`vboxnet0` / `192.168.56.0/24`). El host debe quedar con `.1`. No uses Internal Network: ahí no existe el tercer nodo para ARP/DNS.
2. Cada VM: adaptador **Host-Only** en esa red (tráfico del lab). Si necesitan `apt`, un **segundo** adaptador NAT solo para instalar paquetes; las pruebas y las IPs de esta guía van por Host-Only, no por NAT.
3. IPs estáticas **en la NIC Host-Only** (netplan o NetworkManager). En VM2, default gateway y DNS de esa NIC = `192.168.56.1`. Durante las pruebas, no uses la NIC NAT como ruta a la víctima.
4. Promiscuous mode en el adaptador Host-Only de VM2: **Allow All** (útil para ver ARP).
5. Verificar: `ping` VM1 ↔ VM2 y VM2 ↔ host `.1`.
6. Instalar Snort **solo en VM2** (sección 5). Herramientas de prueba **solo en Kali** (sección 6).

#### Recursos mínimos
- Host: 8 GB RAM (16 GB más holgado)
- Kali: 2–4 GB RAM, 2 CPU
- Ubuntu Victim+Snort: 2 GB RAM, 2 CPU
- Disco: ~30–40 GB libres (2 ISOs + 2 VMs)

#### Pasos de preparación (Escenario A)
1. Instalar VirtualBox.
2. Descargar ISO Kali + Ubuntu.
3. Crear **exactamente 2 VMs**, red Host-Only, IPs de la tabla.
4. Actualizar SO: `sudo apt update && sudo apt upgrade -y`
5. En VM2: para **las pruebas**, gateway/DNS de la NIC Host-Only = `.1`; servicio HTTP (nginx o `python3 -m http.server 80`). El NAT, si existe, solo para instalar paquetes **antes** del plan de pruebas.
6. `ping` Attacker ↔ Victim y Victim ↔ host.
7. Instalar Snort en VM2 (sección 5).
8. Herramientas de prueba solo en Attacker (sección 6 — alto nivel).
9. Escribir reglas (sección 7) y ejecutar el plan de pruebas (sección 8).

---

### Escenario B — 3 laptops (red real)

**Idea:** un integrante / un rol / una laptop. El IDS **es el gateway** entre dos segmentos, usando las dos interfaces que ya trae esa laptop (Ethernet + Wi‑Fi). Así Snort ve el tráfico unicast **sin** hub, SPAN ni switch managed.

#### Topología (fija)

```text
[Laptop 1 Kali Attacker]
  192.168.10.10/24  GW .1
        |
        |  Ethernet (cable directo Attacker↔IDS)
        v
[Laptop 3 Ubuntu IDS]
  eth0  192.168.10.1/24     ← segmento Attacker
  wlan0 192.168.20.1/24     ← segmento Victim
  ip_forward=1
  Snort en ambas interfaces (o la que reciba el flujo)
        |
        |  Wi‑Fi del lab (AP del equipo, WAN/Internet desconectado)
        v
[Laptop 2 Ubuntu Victim]
  192.168.20.20/24  GW .1   HTTP/SSH, Wireshark, ss/netstat
```

**Asignación de roles:**
| Persona / Laptop | Rol | Software principal |
|------------------|-----|--------------------|
| Integrante A | Attacker | Kali: hping3, nmap, Bettercap/Ettercap |
| Integrante B | Victim | Ubuntu: web/SSH, Wireshark, `ss`/`netstat` |
| Integrante C | IDS / gateway | Ubuntu: forwarding + Snort + reglas + logs |

`HOME_NET`: la red de la víctima, `192.168.20.0/24`. `EXTERNAL_NET`: `192.168.10.0/24` (o `!$HOME_NET`).

#### Red y visibilidad (checklist)
1. **Aislar:** desconectar WAN/Internet del AP. El lab no debe salir a campus ni a la red doméstica en uso.
2. **Dos segmentos, dos NICs del IDS:** Ethernet hacia Attacker (cable directo; NICs modernas hacen auto-MDIX) y Wi‑Fi hacia Victim (AP que el equipo ya use). No añadas un cuarto dispositivo ni un hub.
3. En IDS: `net.ipv4.ip_forward=1` (sysctl) para que el tráfico Attacker → Victim atraviese la laptop C.
4. Rutas: Attacker GW = `192.168.10.1`; Victim GW y DNS = `192.168.20.1`.
5. IPs estáticas (no DHCP del AP en el lab).
6. Verificar: `ping` Attacker ↔ IDS, Victim ↔ IDS, y **Attacker ↔ Victim** (enrutado). Si este último falla, Snort no verá SYN flood ni nmap.

| Actividad | Cómo se cubre |
|-----------|----------------|
| Ping, SYN flood, traceroute, nmap | Kali envía a `192.168.20.20`; el paquete pasa por el IDS; Snort alerta |
| ARP spoof | En el segmento Victim: el gateway a suplantar es el IDS `192.168.20.1`. Kali debe estar **en ese L2** (asociar el Wi‑Fi de Kali al AP del Victim **durante la Fase 3**) |
| DNS poisoning | Tras el ARP, DNS spoof a un dominio de **prueba**; la Victim resuelve vía el camino MITM |

En Methodology explica: (1) el IDS está **en línea** como gateway, por eso ve unicast sin SPAN; (2) ARP es de enlace, por eso la Fase 3 se hace en el Wi‑Fi de la víctima.

#### Pasos de preparación (Escenario B)
1. Acordar roles, IPs de la topología y qué NIC es eth/wlan en el IDS.
2. Cable Attacker–IDS; Victim e IDS en el Wi‑Fi del lab; WAN desconectada.
3. Activar forwarding en IDS; fijar GWs; probar los tres `ping` (incluido Attacker ↔ Victim).
4. Instalar Snort en IDS (sección 5); interfaz(es) del forwarding.
5. En Victim: servicio HTTP y Wireshark.
6. En Attacker: herramientas del enunciado; dejar listo el Wi‑Fi para la Fase 3.
7. Sincronizar reloj y grabar evidencia por ataque (sección 8).

---

## 5. Instalación y configuración de Snort (defensivo)

> Ajusta nombres de paquetes según la versión de Ubuntu. Snort 2.x usa `snort.conf`; Snort 3 usa `snort.lua`.

### 5.1 Instalación rápida (Ubuntu, paquete)

```bash
sudo apt update
sudo apt install -y snort
```

Durante la instalación puede pedir la red a proteger (`HOME_NET`). Ejemplo: `192.168.56.0/24` (Escenario A, Host-Only) o `192.168.20.0/24` (Escenario B, red de la víctima).

Verifica:

```bash
snort -V
```

### 5.2 Variables importantes
En configuración, define:
- `HOME_NET`: red a proteger — A: `192.168.56.0/24`; B: `192.168.20.0/24`
- `EXTERNAL_NET`: A: `any` o `!$HOME_NET`; B: `192.168.10.0/24` (segmento Attacker)

Instala Snort **donde está el sensor**:
- Escenario A: en **VM2** (Victim)
- Escenario B: en **laptop IDS** (integrante C)

### 5.3 Crear archivo de reglas del equipo

Ejemplo:

```bash
sudo mkdir -p /etc/snort/rules
sudo nano /etc/snort/rules/local_hw2.rules
```

Incluye el archivo desde la config principal (Snort 2 ejemplo):

```text
include $RULE_PATH/local_hw2.rules
```

### 5.4 Ejecutar Snort en modo IDS (consola)

```bash
# Cambia eth0 por la interfaz del sensor (ip a)
# A: NIC Host-Only de VM2     B: NIC del IDS que ve el flujo (eth0 y/o wlan0)
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

Otras opciones útiles:
- `-A fast` / logs en `/var/log/snort/`
- `-k none` (a veces en labs con checksums raros en VMs)
- Probar regla contra un PCAP: `snort -c snort.conf -r captura.pcap -A console`
- Escenario B: si el tráfico entra por una NIC y sale por otra, corre Snort en la interfaz donde confirmes el PCAP (o una instancia por NIC)

### 5.5 Evidencias a guardar
- Contenido de `local_hw2.rules`
- Salida de alertas en consola
- Archivos en `/var/log/snort/`
- Capturas Wireshark (`.pcap`) alineadas con cada prueba
- `ss -ant` / `netstat` en Victim durante SYN flood

---

## 6. Herramientas de generación de tráfico (alto nivel)

El enunciado sugiere herramientas; úsalas **solo en el lab**. Aquí no se detalla un playbook ofensivo paso a paso: consulta la documentación oficial y el material del curso / HW1.

| Actividad | Herramientas citadas / útiles | Qué debe observar el IDS |
|-----------|-------------------------------|---------------------------|
| TCP SYN Flood | hping3, Scapy, LOIC, RUDY, DDoSIM, Engage Packet Builder | Muchos TCP SYN hacia un puerto; umbral/threshold |
| ARP poisoning | Bettercap, Ettercap | Tráfico ARP anómalo / replies sospechosos |
| DNS poisoning | Ettercap DNS spoof (`etter.dns`), Bettercap | DNS spoof / respuestas inconsistentes en MITM |
| Ping / PoD | ping, hping3 | ICMP echo excesivo o anómalo |
| Traceroute | traceroute / tracert | TTL bajos / ICMP time-exceeded / UDP high ports |
| Nmap | nmap | SYN scan, NULL/FIN/XMAS, version/OS probes |

**Verificación auxiliar SYN flood (víctima):**

```bash
ss -ant | grep SYN-RECV | wc -l
# o
netstat -n -p tcp
```

Si hay muchas conexiones en `SYN_RECEIVED` / `SYN-RECV`, hay indicios de SYN flood (como indica el PDF).

**ARP/DNS (alineado al enunciado):**
- Usar Bettercap/Ettercap en Kali dentro de la LAN del lab.
- Para DNS spoof con Ettercap: preparar entradas en `etter.dns` (ej. redirección de un dominio de prueba a otra IP controlada por el equipo).
- Recordar: DNS poisoning en este contexto de LAN inicia típicamente con ARP poisoning.

**Reconocimiento:**
- `ping` / `hping3` al rango del lab
- `traceroute` hacia Victim/gateway
- `nmap` para hosts, puertos, servicios y OS (en el rango privado del equipo)

---

## 7. Reglas Snort sugeridas (núcleo del homework)

> **Importante:** ajusta `HOME_NET`, interfaces y `sid` únicos (rango local típico: `1000000+`).  
> Prueba y **justifica** cada regla en Results. Evita reglas demasiado genéricas que alerten todo el tráfico legítimo.

Guarda estas reglas en `local_hw2.rules` y ve activándolas por prueba.

### 7.1 Ping / ICMP flood o reconocimiento ICMP

```text
# Echo Request hacia HOME_NET
alert icmp any any -> $HOME_NET any ( \
  msg:"HW2 ICMP Echo Request to HOME_NET"; \
  itype:8; \
  classtype:misc-activity; \
  sid:1001001; rev:1; )

# Umbral: muchos pings en poco tiempo (posible flood)
alert icmp any any -> $HOME_NET any ( \
  msg:"HW2 Possible ICMP Flood"; \
  itype:8; \
  threshold: type both, track by_src, count 50, seconds 10; \
  classtype:denial-of-service; \
  sid:1001002; rev:1; )
```

### 7.2 TCP SYN Flood / DoS

```text
alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible TCP SYN Flood"; \
  flags:S,12; \
  threshold: type both, track by_dst, count 100, seconds 5; \
  classtype:attempted-dos; \
  sid:1002001; rev:1; )

# Variante más específica a un servicio (ej. HTTP 80)
alert tcp any any -> $HOME_NET 80 ( \
  msg:"HW2 SYN Flood targeting TCP/80"; \
  flags:S,12; \
  threshold: type both, track by_dst, count 50, seconds 3; \
  classtype:attempted-dos; \
  sid:1002002; rev:1; )
```

Notas:
- `flags:S` = SYN; el modificador `12` (o equivalentes según versión) ayuda a matchear SYN “puro” según documentación de tu versión.
- Combina evidencia Snort + `ss`/`netstat` en Results.

### 7.3 ARP spoofing

Snort clásico tiene limitaciones con ARP según build/config; opciones:

**Opción A — regla ARP (si el build lo soporta):**
```text
alert arp any any -> any any ( \
  msg:"HW2 ARP frame observed (baseline/policy)"; \
  sid:1003001; rev:1; )
```

**Opción B — detección práctica en lab (recomendada para discusión):**
1. Regla/preprocesador o monitoreo de inconsistencias IP-MAC.
2. Evidencia cruzada: Wireshark filtro `arp`, Bettercap/Ettercap logs, y alerta Snort si hay módulo ARP.
3. En Methodology explica si usaron regla ARP nativa, script auxiliar, o correlación Wireshark + Snort para tráfico post-MITM.

Ejemplo de alerta relacionada a MITM HTTP tras spoof (si aplica):
```text
alert tcp any any -> $HOME_NET 80 ( \
  msg:"HW2 HTTP after suspected MITM path - review ARP evidence"; \
  flow:to_server,established; \
  content:"GET "; \
  classtype:policy-violation; \
  sid:1003002; rev:1; )
```

> Sé honestos en el reporte: ARP spoof se demuestra con PCAP + herramienta; Snort puede complementar.

### 7.4 DNS Poisoning / DNS sospechoso

```text
# Muchas respuestas DNS o tráfico DNS anómalo hacia clientes
alert udp any 53 -> $HOME_NET any ( \
  msg:"HW2 DNS response to client - monitor for spoofing"; \
  threshold: type both, track by_src, count 20, seconds 10; \
  classtype:bad-unknown; \
  sid:1004001; rev:1; )

# Consulta DNS saliente (reconocimiento/exfil básica de queries)
alert udp $HOME_NET any -> any 53 ( \
  msg:"HW2 DNS query from HOME_NET"; \
  classtype:misc-activity; \
  sid:1004002; rev:1; )
```

Para el experimento de redirección (Facebook → otro sitio):
- Muestra en Results: query DNS + respuesta falsa (Wireshark) + alerta Snort.
- Explica que la detección robusta de DNS spoof requiere baseline (IP legítima del dominio) o IDS con contexto; en lab se valida con evidencia del MITM.

### 7.5 Traceroute

Traceroute suele usar UDP con TTL creciente o ICMP.

```text
alert icmp any any -> $HOME_NET any ( \
  msg:"HW2 ICMP toward HOME_NET (possible traceroute probe)"; \
  itype:8; \
  ttl:<5; \
  classtype:network-scan; \
  sid:1005001; rev:1; )

alert udp any any -> $HOME_NET 33434:33534 ( \
  msg:"HW2 Possible UDP traceroute probe"; \
  classtype:network-scan; \
  sid:1005002; rev:1; )
```

Ajusta el rango de puertos según el traceroute observado en tu PCAP.

### 7.6 Nmap scans

```text
# SYN scan (stealth) - muchos SYN sin completar a distintos puertos
alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap SYN scan"; \
  flags:S,12; \
  threshold: type both, track by_src, count 20, seconds 5; \
  classtype:network-scan; \
  sid:1006001; rev:1; )

# NULL scan
alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap NULL scan"; \
  flags:0; \
  classtype:network-scan; \
  sid:1006002; rev:1; )

# FIN scan
alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap FIN scan"; \
  flags:F,12; \
  classtype:network-scan; \
  sid:1006003; rev:1; )

# XMAS scan
alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap XMAS scan"; \
  flags:FPU,12; \
  classtype:network-scan; \
  sid:1006004; rev:1; )
```

### 7.7 Buenas prácticas al escribir reglas
1. Un `sid` único por regla; documenta una tabla sid ↔ ataque.
2. Empieza sin `threshold`; luego ajústalo para reducir ruido.
3. Prueba de a una regla/ataque.
4. Evita `content` frágiles si no los validaste en PCAP.
5. Incluye `msg` claros con prefijo `HW2` para filtrar evidencia.
6. Menciona falsos positivos (ej. `nmap` legítimo de admin).

---

## 8. Plan de pruebas paso a paso (ambos escenarios)

Sigue este orden en el laboratorio y en el video.

### Fase 0 — Baseline
1. Levantar la red del escenario elegido:
   - **A:** `ping` Attacker ↔ Victim y Victim ↔ host `.1`
   - **B:** `ping` Attacker ↔ IDS, Victim ↔ IDS y Attacker ↔ Victim (enrutado)
2. Arrancar Snort en el sensor (VM2 en A; laptop IDS en B) con reglas cargadas.
3. Generar tráfico legítimo breve (HTTP GET, un ping).
4. Confirmar que **no** hay ráfaga de alertas absurdas (o documentar las que sí).

### Fase 1 — ICMP / Ping
1. Desde Attacker: ping a Victim.
2. Verificar alerta `1001001` / flood si aplica.
3. Guardar: screenshot consola Snort + PCAP Wireshark (`icmp`).

### Fase 2 — TCP SYN Flood
1. En Victim: servicio escuchando (80/443/22).
2. Arrancar monitoreo `ss -ant`.
3. Desde Attacker: herramienta SYN flood del enunciado (hping3/Scapy/etc.) **solo al IP del lab**.
4. Correlacionar: alertas `100200x` + muchos `SYN-RECV`.
5. Detener el test; no saturar el host hasta colgarlo si no es necesario.

### Fase 3 — ARP + DNS Poisoning
1. Documentar ARP table antes (`ip neigh` / `arp -a`) en Victim (y en el gateway: host `.1` en A, IDS `192.168.20.1` en B).
2. Ejecutar ARP poisoning **en el L2 de la víctima** (Bettercap/Ettercap), suplantando el gateway del lab:
   - **A:** gateway = host `192.168.56.1` (Kali ya está en ese Host-Only)
   - **B:** gateway = IDS `192.168.20.1` (Kali se asocia al Wi‑Fi de la víctima para esta fase)
3. Preparar `etter.dns` (o equivalente) con dominio de **prueba** controlado.
4. Desde Victim, resolver/navegar al dominio de prueba; observar redirección.
5. Evidencia: PCAP ARP + DNS, tablas ARP alteradas, alertas Snort relacionadas.
6. Restaurar red (parar spoof, flush ARP). En B, Kali puede volver al Ethernet para el resto de pruebas.

### Fase 4 — Traceroute
1. `traceroute` / `tracert` hacia Victim (A: 1 hop; B: 2 hops vía IDS).
2. Validar alertas TTL/UDP.
3. Guardar salida del comando + alerta. Documentar si el path es corto.

### Fase 5 — Nmap
1. Escaneos controlados al host Victim (SYN, y opcionalmente NULL/FIN/XMAS).
2. OS/service detection si el enunciado lo pide para “mapping”.
3. Mostrar alertas de scan + discusión de qué reveló nmap (puertos/OS) en Results (como atacante) y qué detectó Snort (como defensor).

### Fase 6 — Empaquetado de evidencia
Para cada ataque, ten al menos:
- [ ] Regla(s) usadas (texto)
- [ ] Screenshot alerta Snort
- [ ] Screenshot/herramienta atacante (sin exponer datos sensibles)
- [ ] PCAP o filtro Wireshark
- [ ] Breve explicación “por qué matcheó”

---

## 9. Video en inglés (requisitos y guion sugerido)

**Requisitos del curso:**
- Idioma: **inglés**
- Cámara **encendida**
- Participación visible de **cada** integrante
- Explica el homework
- URL del video **dentro del PDF**

**Duración sugerida:** 8–15 minutos (claro y completo > largo y vacío).

### Guion propuesto (reparto para 3)

| Min | Quién | Contenido |
|-----|-------|-----------|
| 0:00–1:00 | Todos a cámara | Presentación: nombres, IDs, HW2, objetivo IDS/Snort |
| 1:00–3:00 | Integrante C | Topología (A: 2 VMs Host-Only, o B: 3 laptops + IDS gateway), HOME_NET, instalación Snort |
| 3:00–6:00 | Integrante A | Demo SYN flood + muestra alerta; explica handshake/half-open |
| 6:00–9:00 | Integrante B | Demo ARP/DNS poisoning + evidencia Victim/Wireshark + regla |
| 9:00–12:00 | Integrante C/A | Traceroute + Nmap + alertas de scan |
| 12:00–14:00 | Todos | Resultados, limitaciones, conclusiones |
| Cierre | Todos | Gracias / link repo o nada si no aplica |

**Plataformas:** YouTube (unlisted), Drive, Stream, etc. Verifica que el profesor pueda abrir el link sin login raro.

---

## 10. Checklist de entrega Blackboard

- [ ] PDF nombrado `HW2_ID_ID_ID.pdf` (IDs de los integrantes)
- [ ] **Cada** miembro sube el PDF
- [ ] Portada completa (Title, HW#, Name, ID, Course, Date, Autumn Term)
- [ ] Secciones: Abstract → … → Bibliography
- [ ] Figuras/tablas numeradas y citadas en texto
- [ ] URL del video en el reporte
- [ ] Bibliografía con referencias del PDF + extras usadas
- [ ] Entrega antes de **Mon 21 Sep 2026 11:59 PM**

---

## 11. Rúbrica para calificar los entregables

Usa esta rúbrica (100 pts) para autoevaluación del equipo antes de subir.

### 11.1 Reporte escrito (60 pts)

| Criterio | Excelente | Aceptable | Deficiente | Pts |
|----------|-----------|-----------|------------|-----|
| **Portada y formalidades** | Todos los campos + nombre de archivo correcto | Falta menor | Mal nombre / sin datos | /5 |
| **Abstract** | Claro, completo, refleja resultados reales | Genérico pero correcto | Vacío o no relacionado | /5 |
| **Introduction** | Contextualiza IDS/Snort/ataques con citas | Superficial | Sin marco teórico | /5 |
| **Methodology** | Topología (A: 2 VMs o B: 3 laptops), IPs, dónde corre Snort, tools, reglas, procedimiento reproducible | Faltan detalles de red/Snort | No se entiende el lab | /10 |
| **Reglas Snort** | Reglas para **todos** los tipos pedidos, explicadas y con SID | Cubren la mayoría | Pocas o copiadas sin explicación | /10 |
| **Results & Discussion** | Evidencia por ataque (alertas+PCAP+análisis) | Evidencia parcial | Solo teoría sin pruebas | /15 |
| **Conclusions** | Críticas, limitaciones, trabajo futuro | Conclusión breve | Ausente | /5 |
| **Bibliography & citas** | Referencias del curso + citas en texto | Bibliografía incompleta | Sin citas | /5 |

### 11.2 Video (25 pts)

| Criterio | Excelente | Aceptable | Deficiente | Pts |
|----------|-----------|-----------|------------|-----|
| **Inglés + cámaras** | Todos con cámara; inglés comprensible | Uno con problemas menores | Sin cámaras / no inglés | /8 |
| **Participación equitativa** | Cada integrante explica una parte | Uno domina pero todos hablan | Solo una persona | /7 |
| **Demo técnica** | Se ve Snort detectando ≥3 tipos de actividad | Demo parcial | Solo slides sin lab | /7 |
| **URL funcional en PDF** | Link abre y video completo | Link con permiso confuso | Sin link / roto | /3 |

### 11.3 Cumplimiento técnico del objetivo (15 pts)

| Criterio | Pts |
|----------|-----|
| Detección Ping/ICMP documentada | /2 |
| Detección SYN Flood / DoS documentada | /3 |
| Detección/análisis ARP spoofing documentado | /3 |
| Detección/análisis DNS poisoning documentado | /3 |
| Detección traceroute documentada | /2 |
| Detección Nmap documentada | /2 |

### 11.4 Penalizaciones (según enunciado / buenas prácticas)
| Situación | Efecto |
|-----------|--------|
| Algún integrante no sube el PDF | **0.0** para ese integrante (según Activity instructions) |
| Entrega tardía | No aceptada |
| Sin URL de video | Fuerte descuento / posible no cumplimiento |
| Evidencia de ataques fuera del lab / a terceros | Inválido éticamente; no hacerlo |
| Reglas sin evidencia | Descuento fuerte en Results |

### 11.5 Escala sugerida
| Puntos | Nivel |
|--------|-------|
| 90–100 | Excelente |
| 80–89 | Notable |
| 70–79 | Satisfactorio |
| 60–69 | Suficiente (mejorar evidencia/reglas) |
| <60 | Insuficiente |

---

## 12. Recomendaciones finales (para sacar mejor nota)

1. **Prioriza evidencia:** el profesor quiere ver reglas + alertas reales, no solo teoría del SYN flood.
2. **Tabla resumen** en Results: Ataque | Herramienta | SID | ¿Detectado? | Screenshot.
3. **Un escenario bien hecho** (A **o** B) es suficiente. En Methodology declara colocación del sensor: A = Snort en la víctima; B = Snort en el gateway. No montes VMs/laptops extra.
4. **Seguridad:** red aislada; dominios de prueba; no credenciales reales; detén floods al obtener evidencia.
5. **Coherencia video ↔ PDF:** mismas IPs, mismas reglas, mismas figuras.
6. **Cita referencias del PDF:** Kurose/Ross [1], Snort [4], Bettercap [6], Ettercap [7], Kali [8], etc.
7. **Menciona Suricata [5]** en Introduction/Conclusions como alternativa IDS moderna (aunque implementen Snort).
8. Si algo no se puede detectar bien con regla simple (ARP/DNS), **explícalo con rigor** y apoya con Wireshark; eso suma en Discussion.
9. Haz una **prueba de carga del PDF y del link** el día anterior a la entrega.
10. En Methodology incluye diagrama de red (draw.io/Excalidraw) del escenario elegido.

---

## 13. Diagrama rápido para pegar en Methodology

### Escenario A (1 laptop, 2 VMs, Host-Only)
```text
[Host 192.168.56.1]  gateway lógico (ARP/DNS)
         |
   Host-Only 192.168.56.0/24
         |
    +----+----+
    |         |
[Kali .10]  [Ubuntu Victim+Snort .20]
 Attacker    HTTP/SSH; ss/netstat; Wireshark
             snort -i <host-only nic> + local_hw2.rules
```

### Escenario B (3 laptops, IDS como gateway)
```text
[Kali Attacker 192.168.10.10]
        | Ethernet
        v
[Ubuntu IDS 10.1 / 20.1]  ip_forward=1; Snort; local_hw2.rules
        | Wi-Fi lab (sin Internet)
        v
[Ubuntu Victim 192.168.20.20]  HTTP/SSH; Wireshark; ss/netstat
```
Fase 3 ARP/DNS: Kali se une al Wi‑Fi de la víctima y suplanta `192.168.20.1`.

---

## 14. Mapa de referencias del enunciado (para Bibliography)

1. Kurose & Ross — *Computer Networking: A Top-Down Approach*  
2. Engage Packet Builder — http://www.engagesecurity.com/products/engagepacketbuilder  
3. Wireshark — https://www.wireshark.org/  
4. Snort — https://www.snort.org/  
5. Suricata — https://suricata-ids.org/  
6. Bettercap — https://www.bettercap.org/  
7. Ettercap — https://www.ettercap-project.org/  
8. Kali Linux — https://www.kali.org/  
9. TutorialsPoint — https://www.tutorialspoint.com/  
10. DDoSIM — https://ddosim.live/  
11. RUDY — https://www.invicti.com/learn/rudy-attack  

Además: documentación oficial de reglas Snort (Options, Thresholding, Flags) según la versión instalada.

---

## 15. Cronograma sugerido del equipo (sin fechas intermedias rígidas)

1. **Kickoff:** Escenario A (1 laptop / 2 VMs) o B (3 laptops); asignar roles; chat de evidencias.
2. **Lab up:** red + Snort instalado + ping OK.
3. **Reglas v1:** ICMP + SYN + Nmap.
4. **Pruebas v1:** capturas y ajustes de threshold.
5. **ARP/DNS:** experimento + evidencia extra Wireshark.
6. **Traceroute/PoD-ICMP:** cerrar cobertura del enunciado.
7. **Redacción PDF** en paralelo (Introduction/Methodology temprano).
8. **Grabación video** cuando las demos ya salgan en un take limpio.
9. **QA rúbrica** (sección 11) + subida individual a Blackboard.

---

*Documento guía preparado a partir de: `HW2_LIS4062_Autumn_26` y `Activity instructions` (Due Date: Monday, 21 September 2026 11:59 PM).*

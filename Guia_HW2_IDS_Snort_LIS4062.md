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

**Abstract (150–250 palabras):** objetivo del IDS, entorno (escenario 1 o 2), ataques evaluados, resultado cualitativo (qué se detectó), conclusión breve.

**Introduction:** qué es un IDS, Snort (NIDS basado en reglas), por qué detectar SYN flood / ARP-DNS / recon; objetivo del homework; alcance y limitaciones (laboratorio aislado).

**Methodology:** topología, SO/VMs, instalación Snort, reglas escritas, herramientas usadas para generar tráfico de prueba, procedimiento de captura (Wireshark/Snort alerts), criterios de éxito (“alerta aparece en `alert`/`fast` log”).

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

Elige **uno** (o documenta ambos si el equipo tiene recursos). En el reporte describe claramente el escenario usado.

### Escenario A — Una sola laptop Linux + VirtualBox

**Idea:** todo el laboratorio vive en una laptop; las “máquinas” son VMs en una red interna virtual.

#### Topología recomendada (3–4 VMs)

```text
[ Host Linux + VirtualBox ]
        |
   VirtualBox Host-Only / Internal Network  (ej. 192.168.56.0/24)
        |
   +----+----+------------+--------------+
   |         |            |              |
[VM1]     [VM2]        [VM3]          [VM4 opcional]
Attacker  Victim/      Snort IDS      Gateway/DNS
(Kali)    Target       (Ubuntu)       (Ubuntu)
```

**Roles sugeridos:**
| VM | SO | IP ejemplo | Rol |
|----|----|------------|-----|
| Attacker | Kali Linux | 192.168.56.10 | Genera tráfico de prueba |
| Victim | Ubuntu Server/Desktop | 192.168.56.20 | Objetivo (servicios web/SSH) |
| IDS | Ubuntu Server | 192.168.56.30 | Snort en modo sniffer |
| Gateway/DNS (opc.) | Ubuntu | 192.168.56.1 | Simula router/DNS interno |

#### Configuración VirtualBox (checklist)
1. Crear red **Host-Only** o **Internal Network** (preferible Internal para aislamiento total).
2. Todas las VMs en la **misma** red.
3. Para que el IDS vea tráfico entre Attacker y Victim:
   - **Opción 1 (simple):** poner Snort en la misma red y generar tráfico hacia Victim; capturar en interfaz de Victim **o**
   - **Opción 2 (mejor para NIDS):** bridge/tap o “promiscuous mode” en el adaptador del IDS + tráfico que pase por un punto visible (en Internal Network puro a veces el IDS no ve tráfico entre otras VMs).
4. Solución práctica muy usada en una sola laptop:
   - Instalar **Snort en la Victim** (HIDS/NIDS local sobre su interfaz), **o**
   - Usar un switch virtual + port mirroring (avanzado), **o**
   - Hacer que el IDS sea gateway (Victim con default route vía IDS) → el IDS ve el tráfico (recomendado para demo clara).

**Topología “IDS como gateway” (recomendada en 1 laptop):**

```text
Attacker (56.10) ---> IDS/Gateway (56.30) ---> Victim (56.20)
                         |
                      Snort -i eth0
```

En Victim: ruta por defecto hacia `192.168.56.30`.  
En Attacker: misma subred; tráfico a Victim pasa o es visible según diseño. Alternativa: dos interfaces en IDS (attacker-net / victim-net).

#### Recursos mínimos sugeridos
- Host: 16 GB RAM (ideal), 8 GB mínimo apretado
- Kali: 2–4 GB RAM, 2 CPU
- Ubuntu Victim: 1–2 GB
- Ubuntu IDS: 1–2 GB
- Disco: ~40–60 GB libres

#### Pasos de preparación (Escenario A)
1. Instalar VirtualBox + Extension Pack (si se necesita).
2. Descargar ISO Kali + Ubuntu.
3. Crear VMs, asignar red Internal/Host-Only.
4. Actualizar SO: `sudo apt update && sudo apt upgrade -y`
5. Fijar IPs estáticas (netplan o NetworkManager).
6. Verificar conectividad: `ping` entre VMs.
7. Instalar Snort en IDS (sección 5).
8. Instalar herramientas de prueba solo en Attacker (sección 6 — alto nivel).
9. Escribir reglas Snort (sección 7).
10. Ejecutar pruebas controladas y guardar evidencias (sección 8).

---

### Escenario B — 3 laptops + switch / modem / hub

**Idea:** cada integrante/rol en hardware físico; red LAN aislada.

#### Topología recomendada

```text
 Laptop 1 (Attacker - Kali/Linux)
          \
           \ 
            +---- [ Switch / Hub / LAN del módem ] ----+
           /                                           \
 Laptop 2 (Victim - Ubuntu/Windows)          Laptop 3 (IDS - Ubuntu + Snort)
```

**Asignación de roles (equipo de 3):**
| Persona / Laptop | Rol | Software principal |
|------------------|-----|--------------------|
| Integrante A | Attacker | Kali o Linux + hping3, nmap, Bettercap/Ettercap |
| Integrante B | Victim | Servicios (web, SSH) + Wireshark + `ss`/`netstat` |
| Integrante C | IDS Analyst | Snort + reglas + logs + captura evidencia |

#### Configuración de red (checklist)
1. Conectar las 3 laptops al **mismo** switch/hub (o LAN del módem **sin usar Internet** si es posible).
2. Desactivar Wi‑Fi si usarán cable (evita rutas confusas).
3. IPs estáticas en la misma subred, por ejemplo:

| Host | IP | Máscara | Gateway |
|------|----|---------|---------|
| Attacker | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 (opc.) |
| Victim | 192.168.10.20 | 255.255.255.0 | 192.168.10.1 |
| IDS | 192.168.10.30 | 255.255.255.0 | 192.168.10.1 |

4. **Problema clave:** en un switch moderno, el puerto del IDS **no ve** el tráfico unicast entre Attacker y Victim (salvo que esté en el camino o haya mirroring).

**Soluciones prácticas (elige una y documéntala):**

| Solución | Cómo | Pros | Contras |
|----------|------|------|---------|
| **B1. Hub** (repetidor) | Usar hub antiguo | IDS ve todo en promiscuous | Difícil de conseguir |
| **B2. Port mirroring / SPAN** | En switch managed, mirror puerto Victim → IDS | Correcto para NIDS | Requiere switch managed |
| **B3. IDS en la Victim** | Snort en laptop víctima | Simple, evidencia clara | Menos “NIDS puro” |
| **B4. IDS como gateway** | Victim usa IDS como gateway; Attacker apunta a Victim vía IDS | Muy didáctico | Config de routing |
| **B5. Mismo segmento + ARP MITM visible** | Durante ARP spoof el tráfico pasa por Attacker; sniffea ahí **y** en IDS si se fuerza | Alineado al experimento 2 | Más caótico |

**Recomendación para el curso:** **B3 o B4** si no hay switch managed; menciona en Methodology la limitación de switches y por qué eligieron esa opción.

#### Pasos de preparación (Escenario B)
1. Acordar IPs, roles y cableado.
2. Probar `ping` entre las tres máquinas.
3. En IDS: instalar Snort; poner interfaz en modo promiscuo si aplica (`ip link set eth0 promisc on`).
4. En Victim: abrir un servicio (ej. `python3 -m http.server 80` o nginx) para SYN flood / nmap.
5. En Attacker: preparar herramientas del enunciado.
6. Sincronizar reloj (útil para correlacionar logs) y plan de pruebas.
7. Grabar evidencia (pantallas + logs) mientras se prueba cada ataque.

---

## 5. Instalación y configuración de Snort (defensivo)

> Ajusta nombres de paquetes según la versión de Ubuntu. Snort 2.x usa `snort.conf`; Snort 3 usa `snort.lua`.

### 5.1 Instalación rápida (Ubuntu, paquete)

```bash
sudo apt update
sudo apt install -y snort
```

Durante la instalación puede pedir la red a proteger (`HOME_NET`). Ejemplo: `192.168.56.0/24` (Escenario A) o `192.168.10.0/24` (Escenario B).

Verifica:

```bash
snort -V
```

### 5.2 Variables importantes
En configuración, define:
- `HOME_NET`: red interna a proteger
- `EXTERNAL_NET`: normalmente `!$HOME_NET` o `any` (según política)

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
# Cambia eth0 por tu interfaz (ip a)
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

Otras opciones útiles:
- `-A fast` / logs en `/var/log/snort/`
- `-k none` (a veces en labs con checksums raros en VMs)
- Probar regla contra un PCAP: `snort -c snort.conf -r captura.pcap -A console`

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
1. Levantar red y verificar `ping` Attacker ↔ Victim ↔ IDS.
2. Arrancar Snort con reglas cargadas.
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
1. Documentar ARP table antes (`ip neigh` / `arp -a`).
2. Ejecutar ARP poisoning en LAN de lab (Bettercap/Ettercap).
3. Preparar `etter.dns` (o equivalente) con dominio de **prueba** controlado.
4. Desde Victim, resolver/navegar al dominio de prueba; observar redirección.
5. Evidencia: PCAP ARP + DNS, tablas ARP alteradas, alertas Snort relacionadas.
6. Restaurar red (parar spoof, flush ARP).

### Fase 4 — Traceroute
1. `traceroute` / `tracert` hacia Victim o gateway del lab.
2. Validar alertas TTL/UDP.
3. Guardar salida del comando + alerta.

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
| 1:00–3:00 | Integrante C | Topología (Escenario A o B), HOME_NET, instalación Snort |
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
| **Methodology** | Topología (A o B), IPs, tools, reglas, procedimiento reproducible | Faltan detalles de red/Snort | No se entiende el lab | /10 |
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
3. **Un escenario bien hecho** > dos escenarios a medias. Documenta limitaciones (switch sin SPAN, etc.).
4. **Seguridad:** red aislada; dominios de prueba; no credenciales reales; detén floods al obtener evidencia.
5. **Coherencia video ↔ PDF:** mismas IPs, mismas reglas, mismas figuras.
6. **Cita referencias del PDF:** Kurose/Ross [1], Snort [4], Bettercap [6], Ettercap [7], Kali [8], etc.
7. **Menciona Suricata [5]** en Introduction/Conclusions como alternativa IDS moderna (aunque implementen Snort).
8. Si algo no se puede detectar bien con regla simple (ARP/DNS), **explícalo con rigor** y apoya con Wireshark; eso suma en Discussion.
9. Haz una **prueba de carga del PDF y del link** el día anterior a la entrega.
10. En Methodology incluye diagrama de red (draw.io/Excalidraw) del escenario elegido.

---

## 13. Diagrama rápido para pegar en Methodology

### Escenario A (VirtualBox)
```text
[Kali Attacker 192.168.56.10]
            |
            v
[Ubuntu IDS/GW 192.168.56.30]  <-- Snort -i ethX + local_hw2.rules
            |
            v
[Ubuntu Victim 192.168.56.20]  (HTTP/SSH; ss/netstat; Wireshark)
```

### Escenario B (3 laptops + switch)
```text
[Attacker .10] ----\
                    +---- [Switch/Hub] ---- [IDS .30 Snort]
[Victim .20] ------/
```
Con nota: “IDS installed on victim / gateway / SPAN port because unmanaged switch does not mirror traffic.”

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

1. **Kickoff:** elegir Escenario A o B; asignar roles; crear chat de evidencias.
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

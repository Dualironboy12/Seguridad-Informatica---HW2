# Setup — Escenario B: Práctica (3 laptops)

Un integrante / un rol / una laptop. El IDS **es el gateway** entre dos segmentos, usando Ethernet + Wi‑Fi de la laptop C. Snort ve el unicast **sin** hub, SPAN ni switch managed.

Documento hermano (1 laptop / 2 VMs): [`../Simulacion/README.md`](../Simulacion/README.md)  
Guía de las pruebas: [`../../Guia_HW2_IDS_Snort_LIS4062.md`](../../Guia_HW2_IDS_Snort_LIS4062.md)

---

## Cómo usar este documento

1. Completa el **[setup común](#setup-común-obligatorio-antes-de-cualquier-actividad)** una sola vez (roles, cableado, IPs, forwarding, Snort).
2. Elige ritmo:
   - **[Opción 1](#opción-1--setup-completo-antes-de-las-3-actividades)** — deja las 3 actividades listas y luego corre el plan de pruebas.
   - **[Opción 2](#opción-2--setup-y-actividad-por-bloques)** — setup de una actividad e inmediatamente esa actividad.
3. La **ejecución** (tráfico de prueba, capturas, video) está en la guía principal, sección 8.

| Actividad | Setup específico (este README) | Luego ejecuta en la guía |
|-----------|--------------------------------|--------------------------|
| 1. Reconocimiento (ICMP, traceroute, nmap) | [Setup actividad 1](#setup-específico--actividad-1-reconocimiento) | Fases 1, 4 y 5 |
| 2. TCP SYN Flood | [Setup actividad 2](#setup-específico--actividad-2-syn-flood) | Fase 2 |
| 3. ARP + DNS poisoning | [Setup actividad 3](#setup-específico--actividad-3-arp--dns) | Fase 3 |

La actividad 3 es la que **cambia de segmento** (Kali se asocia al Wi‑Fi de la víctima). Las 1 y 2 usan el camino Ethernet → IDS → Wi‑Fi.

---

## Topología (fija)

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

Sustituye `eth0` / `wlan0` por los nombres reales (`ip a`). NICs modernas hacen auto-MDIX: el cable directo Attacker↔IDS no requiere crossover especial.

| Persona / laptop | Rol | Software principal |
|------------------|-----|--------------------|
| Integrante A | Attacker | Kali: hping3, nmap, Bettercap/Ettercap |
| Integrante B | Victim | Ubuntu: web/SSH, Wireshark, `ss`/`netstat` |
| Integrante C | IDS / gateway | Ubuntu: forwarding + Snort + reglas + logs |

- `HOME_NET`: `192.168.20.0/24` (red de la víctima)
- `EXTERNAL_NET`: `192.168.10.0/24` (o `!$HOME_NET`)

| Actividad | Cómo se cubre |
|-----------|----------------|
| 1 y 2 (ping, SYN, traceroute, nmap) | Kali envía a `192.168.20.20`; el paquete **atraviesa** el IDS; Snort alerta |
| 3 (ARP/DNS) | En el L2 de la víctima el gateway a suplantar es el IDS `192.168.20.1`. Kali debe estar **en ese L2** (Wi‑Fi del Victim **durante el setup/prueba de la Fase 3**) |

En Methodology: (1) el IDS está **en línea** como gateway, por eso ve unicast sin SPAN; (2) ARP es de enlace, por eso la actividad 3 se prepara en el Wi‑Fi de la víctima.

---

## Mapa rápido: qué setup es de quién

| Paso de setup | ¿Común? | Act. 1 | Act. 2 | Act. 3 |
|---------------|---------|--------|--------|--------|
| Roles, IPs, cable Attacker–IDS, Victim+IDS en Wi‑Fi lab | Sí | — | — | — |
| AP **sin WAN/Internet** | Sí | — | — | — |
| IPs estáticas (sin DHCP del AP en el lab) | Sí | — | — | — |
| `ip_forward=1` en IDS | Sí | crítico | crítico | no se usa el path Ethernet |
| `ping` Attacker↔IDS, Victim↔IDS, **Attacker↔Victim** | Sí | crítico | crítico | path distinto |
| Snort en laptop IDS | Sí | — | — | — |
| Wireshark en Victim | Sí | recomendado | recomendado | obligatorio |
| Relojes sincronizados (evidencia) | Sí | — | — | — |
| Herramientas recon en Kali (`ping`, `traceroute`, `nmap`) | — | **Sí** | — | — |
| Reglas ICMP / traceroute / nmap | — | **Sí** | — | — |
| HTTP en Victim | — | opcional | **Sí** | útil |
| `hping3` (o equivalente) en Kali | — | — | **Sí** | — |
| Reglas SYN flood | — | — | **Sí** | — |
| Kali asociada al **Wi‑Fi de la víctima** | — | no (sigue en Ethernet) | no | **Sí** |
| Bettercap/Ettercap + `etter.dns` de prueba | — | — | — | **Sí** |
| Snort escuchando **wlan0** (segmento Victim) | recomendado ya | eth o wlan | eth o wlan | **wlan0** |
| Reglas ARP / DNS | — | — | — | **Sí** |

---

## Setup común (obligatorio antes de cualquier actividad)

### C1. Acordar roles y nombres de interfaces

En un papel o chat del equipo, anota:

| Rol | Hostname | Interfaz(es) | IP |
|-----|----------|--------------|-----|
| Attacker | | Ethernet hacia IDS | 192.168.10.10/24, GW 192.168.10.1 |
| IDS | | Ethernet (segmento Attacker) | 192.168.10.1/24 |
| IDS | | Wi‑Fi (segmento Victim) | 192.168.20.1/24 |
| Victim | | Wi‑Fi | 192.168.20.20/24, GW 192.168.20.1 |

```bash
ip -br a
```

Si el IDS solo tiene Wi‑Fi, este escenario **no aplica** tal cual: hace falta Ethernet + Wi‑Fi en la laptop C (o un USB-Ethernet). No añadas un cuarto dispositivo ni un hub para “cumplir” el enunciado.

### C2. Aislar la red

1. AP del equipo: **desconecta WAN/Internet** (sin uplink a campus ni a la red doméstica en uso).
2. Nadie debe usar DHCP del AP para las IPs del lab: estáticas.
3. Cable Ethernet **directo** Attacker ↔ IDS.
4. Victim e IDS asociados al **mismo** SSID del lab (segmento `192.168.20.0/24`).

### C3. IPs estáticas

Ejemplos netplan (Ubuntu). Ajusta nombres (`eth0`, `wlan0`, `enp…`).

**IDS — `/etc/netplan/01-hw2.yaml`:**

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.10.1/24
  wifis:
    wlan0:
      dhcp4: false
      addresses:
        - 192.168.20.1/24
      access-points:
        "SSID_DEL_LAB":
          password: "..."
```

**Victim:**

```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: false
      addresses:
        - 192.168.20.20/24
      routes:
        - to: default
          via: 192.168.20.1
      nameservers:
        addresses: [192.168.20.1]
      access-points:
        "SSID_DEL_LAB":
          password: "..."
```

**Attacker (Kali), Ethernet hacia el IDS:**

```bash
sudo nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.10.10/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.1
sudo nmcli con up "Wired connection 1"
```

```bash
sudo chmod 600 /etc/netplan/*.yaml   # en Ubuntu
sudo netplan apply
```

Desactiva `ufw` o permite forwarding/tráfico del lab en el IDS y la víctima si los `ping` mueren en el firewall:

```bash
sudo ufw status
# En el lab, si bloquea el forwarding:
# sudo ufw disable
```

### C4. Forwarding en el IDS (integrante C)

Sin esto, las actividades 1 y 2 **no** cruzan el sensor.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-hw2-forward.conf
```

Si Attacker pinguea al IDS y Victim al IDS, pero **Attacker ↛ Victim**, prueba relajar reverse-path filter en el lab:

```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.eth0.rp_filter=0
sudo sysctl -w net.ipv4.conf.wlan0.rp_filter=0
```

No hace falta NAT/masquerade: las rutas de Attacker y Victim ya apuntan al IDS como gateway.

### C5. Verificar los tres `ping`

1. Attacker → IDS `192.168.10.1`
2. Victim → IDS `192.168.20.1`
3. **Attacker → Victim `192.168.20.20`** (enrutado)

Si el tercero falla, Snort no verá SYN flood ni nmap de las actividades 1–2. No sigas.

Opcional: en el IDS, `sudo tcpdump -ni eth0 icmp` y `sudo tcpdump -ni wlan0 icmp` mientras haces el ping 3, para confirmar que el flujo cruza ambas NICs.

### C6. Instalar Snort solo en la laptop IDS

```bash
sudo apt update
sudo apt install -y snort
snort -V
```

`HOME_NET` del instalador: `192.168.20.0/24`.

```bash
sudo mkdir -p /etc/snort/rules
sudo nano /etc/snort/rules/local_hw2.rules
```

Incluye el archivo desde `snort.conf` / `snort.lua` (Snort 2 ejemplo):

```text
include $RULE_PATH/local_hw2.rules
```

Variables:

- `HOME_NET`: `192.168.20.0/24`
- `EXTERNAL_NET`: `192.168.10.0/24`

Arranque (cambia `eth0` por la NIC donde confirmes el PCAP; en B el tráfico entra por una y sale por otra):

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

Si una sola interfaz no ve el flujo, usa la otra (`wlan0`) o una instancia por NIC. Opción de lab: `-k none`.

### C7. Victim: Wireshark (y SO actualizado)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireshark
sudo usermod -aG wireshark "$USER"
```

En IDS y Attacker, también `sudo apt update && sudo apt upgrade -y` una vez (Attacker vía el medio que usen **antes** de aislar WAN, o con paquetes ya en Kali).

### C8. Attacker: paquetes base

En Kali, instala las herramientas citadas en la sección 6 de la guía (sin playbook ofensivo aquí):

```bash
sudo apt update
sudo apt install -y nmap traceroute hping3 ettercap-graphical
```

Bettercap: documentación oficial de Kali/Bettercap si no está preinstalado.

Deja el Wi‑Fi de Kali **listo para asociarse** al AP de la víctima (contraseña, SSID), pero **no lo uses todavía** si vas a hacer primero las actividades 1 y 2 por Ethernet.

### C9. Sincronizar reloj y evidencia

Acuerda zona horaria / NTP antes de desconectar WAN (o sincroniza manualmente). Las capturas de tres laptops se alinean peor si los relojes divergen.

---

## Setup específico — Actividad 1: Reconocimiento

Kali permanece en **Ethernet** (`192.168.10.10`). El destino es la víctima `192.168.20.20` **a través del IDS**.

### S1.1 Confirmar path enrutado

Repite el `ping` Attacker → Victim. En el IDS, confirma con `tcpdump` que ICMP pasa por `eth0` y `wlan0`.

Traceroute en B suele ser de **2 hops** (IDS, luego Victim). Es el comportamiento esperado.

### S1.2 Dónde poner Snort

Para recon y SYN flood, elige la interfaz donde el `tcpdump` del paso anterior mostró el flujo (a menudo `eth0` de entrada o `wlan0` de salida). Anótalo para Methodology y para no cambiarlo a mitad de la demo.

### S1.3 Reglas en el IDS

SIDs 1001001–1001002, 1005001–1005002, 1006001–1006004 (guía 7.1, 7.5, 7.6):

```text
alert icmp any any -> $HOME_NET any ( \
  msg:"HW2 ICMP Echo Request to HOME_NET"; \
  itype:8; \
  classtype:misc-activity; \
  sid:1001001; rev:1; )

alert icmp any any -> $HOME_NET any ( \
  msg:"HW2 Possible ICMP Flood"; \
  itype:8; \
  threshold: type both, track by_src, count 50, seconds 10; \
  classtype:denial-of-service; \
  sid:1001002; rev:1; )

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

alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap SYN scan"; \
  flags:S,12; \
  threshold: type both, track by_src, count 20, seconds 5; \
  classtype:network-scan; \
  sid:1006001; rev:1; )

alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap NULL scan"; \
  flags:0; \
  classtype:network-scan; \
  sid:1006002; rev:1; )

alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap FIN scan"; \
  flags:F,12; \
  classtype:network-scan; \
  sid:1006003; rev:1; )

alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible Nmap XMAS scan"; \
  flags:FPU,12; \
  classtype:network-scan; \
  sid:1006004; rev:1; )
```

Reinicia Snort. Ajusta el rango UDP de traceroute según el PCAP real.

### S1.4 Kali

`command -v ping traceroute nmap`

### S1.5 Opcional: HTTP en Victim

Adelanta [S2.1](#s21-servicio-objetivo-en-la-víctima) si quieres que nmap liste el puerto 80.

### S1.6 Verificación de setup (actividad 1)

- [ ] `ping` Attacker → Victim **enrutado**
- [ ] Snort en la NIC del flujo, SIDs de recon cargados
- [ ] Wireshark en Victim
- [ ] Kali sigue en `192.168.10.10` (no en el Wi‑Fi de la víctima)

**Siguiente:** guía, **Fases 1, 4 y 5**. Destino: `192.168.20.20`.

---

## Setup específico — Actividad 2: SYN Flood

Misma topología que la actividad 1 (Ethernet → gateway → Victim).

### S2.1 Servicio objetivo en la víctima

```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
ss -tlnp | grep ':80'
```

O: `sudo python3 -m http.server 80`

Desde Kali (setup, no el flood): un GET HTTP a `http://192.168.20.20/` debe funcionar. Si no, el flood tampoco será tráfico útil para el IDS.

### S2.2 Monitoreo en Victim

Terminal lista para la Fase 2:

```bash
ss -ant | grep SYN-RECV | wc -l
```

### S2.3 Herramienta en Kali

```bash
command -v hping3
```

Tráfico **solo** hacia `192.168.20.20` en esta red aislada. Cómo generar el flood: guía sección 6 + documentación de la herramienta, no este README.

### S2.4 Reglas en el IDS

```text
alert tcp any any -> $HOME_NET any ( \
  msg:"HW2 Possible TCP SYN Flood"; \
  flags:S,12; \
  threshold: type both, track by_dst, count 100, seconds 5; \
  classtype:attempted-dos; \
  sid:1002001; rev:1; )

alert tcp any any -> $HOME_NET 80 ( \
  msg:"HW2 SYN Flood targeting TCP/80"; \
  flags:S,12; \
  threshold: type both, track by_dst, count 50, seconds 3; \
  classtype:attempted-dos; \
  sid:1002002; rev:1; )
```

Reinicia Snort en la **misma** interfaz que usaste en la actividad 1.

### S2.5 Verificación de setup (actividad 2)

- [ ] HTTP alcanza `.20` desde Kali (camino enrutado)
- [ ] SIDs 100200x cargados
- [ ] `ss` listo en Victim
- [ ] Forwarding sigue en `1`

**Siguiente:** guía, **Fase 2**. Detén el test al tener evidencia.

---

## Setup específico — Actividad 3: ARP + DNS

ARP no cruza routers. Por eso Kali **abandona el Ethernet del segmento 10** y se asocia al Wi‑Fi de la víctima. El gateway a suplantar es el IDS `192.168.20.1`.

Haz este setup **justo antes** de la Fase 3 si vas por la opción 2 (así no rompes las actividades 1–2). Si vas por la opción 1, déjalo para el final del setup y ejecuta las Fases 1–2 **antes** de cambiar a Kali al Wi‑Fi, o vuelve Kali al Ethernet entre bloques.

### S3.1 Mover a Kali al L2 de la víctima

1. Anota cómo volver al Ethernet (`192.168.10.10`) para no perder el camino de las actividades 1–2.
2. Asocia el Wi‑Fi de Kali al **mismo AP** que Victim e IDS.
3. IP estática de Kali en ese segmento, por ejemplo `192.168.20.10/24`, GW `192.168.20.1` (no uses `.20` ni `.1`).
4. Verifica:
   - Kali → Victim: `ping 192.168.20.20`
   - Kali → gateway IDS: `ping 192.168.20.1`
5. En Victim e IDS (wlan): guarda `ip neigh` **antes**.

El cable Ethernet Attacker–IDS puede quedar puesto; Kali no debe usarlo como ruta por defecto en esta fase.

### S3.2 Snort en `wlan0`

El spoof ocurre en el segmento Victim. En el IDS:

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i wlan0
```

(ajusta el nombre de la Wi‑Fi).

### S3.3 Herramientas e `etter.dns`

En Kali: Bettercap y/o Ettercap instalados (`ettercap -v` / `bettercap -v`).

Prepara `etter.dns` (típicamente `/etc/ettercap/etter.dns`) con un **dominio de prueba** del equipo → IP controlada (Victim o IDS). No redirecciones cuentas reales.

Victim: DNS = `192.168.20.1` (paso C3).

### S3.4 Reglas ARP / DNS en el IDS

```text
alert arp any any -> any any ( \
  msg:"HW2 ARP frame observed (baseline/policy)"; \
  sid:1003001; rev:1; )

alert tcp any any -> $HOME_NET 80 ( \
  msg:"HW2 HTTP after suspected MITM path - review ARP evidence"; \
  flow:to_server,established; \
  content:"GET "; \
  classtype:policy-violation; \
  sid:1003002; rev:1; )

alert udp any 53 -> $HOME_NET any ( \
  msg:"HW2 DNS response to client - monitor for spoofing"; \
  threshold: type both, track by_src, count 20, seconds 10; \
  classtype:bad-unknown; \
  sid:1004001; rev:1; )

alert udp $HOME_NET any -> any 53 ( \
  msg:"HW2 DNS query from HOME_NET"; \
  classtype:misc-activity; \
  sid:1004002; rev:1; )
```

Si `alert arp` no carga, documenta el build y usa Wireshark (guía 7.3).

### S3.5 HTTP en Victim

Si no lo instalaste en la actividad 2, haz [S2.1](#s21-servicio-objetivo-en-la-víctima) ahora.

### S3.6 Verificación de setup (actividad 3)

- [ ] Kali en `192.168.20.0/24` (Wi‑Fi), no solo en `.10`
- [ ] `ping` Kali ↔ Victim y Kali ↔ `192.168.20.1`
- [ ] Snort en **wlan0**
- [ ] `etter.dns` (o equivalente) de prueba
- [ ] Wireshark en Victim (`arp or dns`)
- [ ] Tablas ARP guardadas **antes**

**Siguiente:** guía, **Fase 3**. Al terminar: parar spoof, flush ARP. Kali puede volver al Ethernet (`192.168.10.10`) si faltan repeticiones de las actividades 1–2.

---

## Opción 1 — Setup completo antes de las 3 actividades

Orden que **no** rompe el path enrutado:

1. **C1–C9** (red de dos segmentos + Snort + paquetes).
2. **S1** (reglas recon) + **S2** (HTTP, hping3, reglas SYN) **con Kali en Ethernet**.
3. Checklist pre-Fase 1/2:
   - [ ] Tres `ping`, incluido Attacker ↔ Victim
   - [ ] `ip_forward=1`
   - [ ] Snort en la NIC del flujo enrutado
   - [ ] HTTP en `.20`
   - [ ] nmap, traceroute, hping3, Ettercap/Bettercap instalados
   - [ ] Wireshark en Victim
4. **Ejecuta Fases 0, 1, 2, 4 y 5** (reconocimiento + SYN) **antes** de mover Kali.
5. **S3** (Kali al Wi‑Fi, Snort en wlan0, `etter.dns`, reglas ARP/DNS).
6. **Ejecuta Fase 3** y luego **Fase 6** (empaquetado).

Así el “setup completo” incluye un **punto de corte**: no asocies Kali al Wi‑Fi hasta haber terminado (o grabado) las pruebas que necesitan el gateway Ethernet.

Si el video debe mostrar las 3 actividades en un solo take, ensaya ese corte (desasociar/asociar Wi‑Fi) para que dure segundos.

---

## Opción 2 — Setup y actividad por bloques

```text
C1–C9 común (Kali en Ethernet, forwarding OK)
    → S1 recon           → guía Fases 1, 4, 5  → evidencia actividad 1
    → S2 SYN             → guía Fase 2         → evidencia actividad 2
    → S3 ARP/DNS (Kali→Wi‑Fi) → guía Fase 3    → evidencia actividad 3
    → restaurar Kali a Ethernet si hace falta
    → guía Fase 6
```

No mezcles: si Kali ya está en Wi‑Fi, un nmap “como en la actividad 1” **ya no atraviesa** el IDS como gateway; sería otro experimento (mismo L2). Para repetir 1 o 2, vuelve Kali a `192.168.10.10`.

---

## Problemas frecuentes (práctica)

| Síntoma | Qué revisar |
|---------|-------------|
| Attacker pinguea IDS, Victim pinguea IDS, pero Attacker ↛ Victim | `ip_forward=1`; `rp_filter`; `ufw`; rutas (GW `.1` en cada segmento) |
| Snort en `eth0` no alerta nmap | El flujo sale por `wlan0`; prueba esa NIC o dos instancias |
| Kali en Wi‑Fi y SYN flood “no pasa por el IDS” | Esperado: ya no hay dos hops; vuelve a Ethernet para act. 1–2 |
| ARP no altera al gateway | Kali no está en el SSID de la víctima; IPs distintas de segmento |
| El AP asigna 192.168.0.0/24 por DHCP | IPs estáticas; desactiva DHCP del AP si puedes |
| Wi‑Fi del IDS pierde la IP al reasociar | netplan/NM: vuelve a aplicar la estática `.1` |
| Relojes desalineados en el video | Sincroniza antes de cortar WAN |

---

## Evidencias que el setup debe dejar listas

Tres laptops → tres fuentes. Acuerda prefijo de archivos (`act1_`, `act2_`, `act3_`) y quién captura qué:

| Rol | Típico por actividad |
|-----|----------------------|
| IDS | Consola Snort + `/var/log/snort/` + `local_hw2.rules` |
| Victim | Wireshark `.pcap`, `ss`/`ip neigh` |
| Attacker | Screenshot de la herramienta (sin datos ajenos al lab) |

Ética: AP sin Internet; solo IPs del equipo; detén floods al tener evidencia; restaura ARP al cerrar la actividad 3.

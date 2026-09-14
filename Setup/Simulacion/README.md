# Setup — Escenario A: Simulación (1 laptop, 2 VMs)

Laboratorio completo en **un solo host** con VirtualBox. No hay VM de IDS ni de gateway: Snort corre en la víctima y el **host** (`192.168.56.1`) es el tercer nodo L2 (gateway lógico para ARP/DNS).

Documento hermano (3 laptops): `[../Practica/README.md](../Practica/README.md)`  
Guía de las pruebas: `[../../Guia_HW2_IDS_Snort_LIS4062.md](../../Guia_HW2_IDS_Snort_LIS4062.md)`

---

## Cómo usar este documento

1. Completa el **[setup común](#setup-común-obligatorio-antes-de-cualquier-actividad)** una sola vez.
2. Elige ritmo:
  - **[Opción 1](#opción-1--setup-completo-antes-de-las-3-actividades)** — deja todo listo y luego corre las 3 actividades.
  - **[Opción 2](#opción-2--setup-y-actividad-por-bloques)** — setup de una actividad e inmediatamente esa actividad.
3. La **ejecución** de cada prueba (comandos de tráfico, capturas, video) está en la guía principal, sección 8. Aquí solo se deja el entorno listo.


| Actividad                                  | Setup específico (este README)                                     | Luego ejecuta en la guía |
| ------------------------------------------ | ------------------------------------------------------------------ | ------------------------ |
| 1. Reconocimiento (ICMP, traceroute, nmap) | [Setup actividad 1](#setup-específico--actividad-1-reconocimiento) | Fases 1, 4 y 5           |
| 2. TCP SYN Flood                           | [Setup actividad 2](#setup-específico--actividad-2-syn-flood)      | Fase 2                   |
| 3. ARP + DNS poisoning                     | [Setup actividad 3](#setup-específico--actividad-3-arp--dns)       | Fase 3                   |


---



## Topología (fija)

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


| Nodo             | SO                                    | IP                        | Rol                                         |
| ---------------- | ------------------------------------- | ------------------------- | ------------------------------------------- |
| VM1 Attacker     | Kali Linux                            | 192.168.56.10/24          | Genera el tráfico de prueba                 |
| VM2 Victim + IDS | Ubuntu Desktop (recomendado) o Server | 192.168.56.20/24, GW `.1` | Servicios, Snort, Wireshark, `ss`/`netstat` |
| Host (no es VM)  | Linux del equipo                      | 192.168.56.1/24           | Gateway que se suplanta en la actividad 3   |


- `HOME_NET`: `192.168.56.0/24` (Corresponde a la red generada por VirtualBox)
- `EXTERNAL_NET`: `any` o `!$HOME_NET`
- Sensor: NIC **Host-Only** de VM2 (`snort -i <iface>`)

**Por qué 2 VMs bastan**


| Actividad                   | Cómo se cubre en esta topología                        |
| --------------------------- | ------------------------------------------------------ |
| 1. Ping / traceroute / nmap | Kali → `.20`; Snort en VM2 ve el tráfico local         |
| 2. SYN Flood                | Servicio en VM2; Kali inunda `.20`; `ss` en la víctima |
| 3. ARP / DNS                | Envenenar ARP de VM2 respecto al gateway **host** `.1` |


Traceroute en A es de **1 hop**. Sigue generando probes; documenta esa limitación en Methodology.

---



## Recursos mínimos

- Host: 8 GB RAM (16 GB más holgado)
- Kali: 2–4 GB RAM, 2 CPU
- Ubuntu Victim+Snort: 2 GB RAM, 2 CPU
- Disco: ~30–40 GB libres (2 ISOs + 2 VMs)

---



## Mapa rápido: qué setup es de quién


| Paso de setup                                                  | ¿Común?                 | Act. 1                       | Act. 2      | Act. 3                |
| -------------------------------------------------------------- | ----------------------- | ---------------------------- | ----------- | --------------------- |
| VirtualBox, red Host-Only, 2 VMs, IPs, `ping`                  | Sí                      | —                            | —           | —                     |
| `apt update` / upgrade en ambas VMs                            | Sí                      | —                            | —           | —                     |
| Snort en VM2 + `local_hw2.rules` + arranque en consola         | Sí                      | —                            | —           | —                     |
| Wireshark en VM2                                               | Sí (evidencia de todas) | recomendado                  | recomendado | obligatorio           |
| Herramientas de recon en Kali (`ping`, `traceroute`, `nmap`)   | —                       | **Sí**                       | —           | —                     |
| Reglas ICMP / traceroute / nmap                                | —                       | **Sí**                       | —           | —                     |
| HTTP (nginx o `python3 -m http.server 80`) en VM2              | —                       | opcional (nmap ve el puerto) | **Sí**      | útil (HTTP tras MITM) |
| `hping3` (u otra herramienta de SYN citada en la guía) en Kali | —                       | —                            | **Sí**      | —                     |
| Reglas SYN flood                                               | —                       | —                            | **Sí**      | —                     |
| Promiscuous **Allow All** en Host-Only de VM2                  | recomendado ya en común | —                            | —           | **Sí**                |
| Bettercap / Ettercap + `etter.dns` de prueba                   | —                       | —                            | —           | **Sí**                |
| Comprobar `ping` VM2 ↔ host `.1`                               | Sí (baseline)           | —                            | —           | **crítico**           |
| Reglas ARP / DNS                                               | —                       | —                            | —           | **Sí**                |


---



## Setup común (obligatorio antes de cualquier actividad)



### C1. Host: VirtualBox y red Host-Only

1. Instala [VirtualBox](https://www.virtualbox.org/).
2. Descarga las ISO:
  - [Kali Linux](https://www.kali.org/)
  - Ubuntu Desktop o Server (LTS).
3. Crea **una** red **Host-Only** (`vboxnet0`, `192.168.56.0/24`). El host debe quedar con `192.168.56.1`.
  - VirtualBox → Tools → Network Manager → Host-only Networks → Create.
  - IPv4: `192.168.56.1/24`.
  - DHCP del Host-Only: desactívalo (este lab usa IPs estáticas).
4. **No uses Internal Network:** ahí no existe el tercer nodo (el host) para la actividad 3.

Comprueba en el host:

```bash
ip -4 addr show
# Debe aparecer vboxnet0 (o similar) con 192.168.56.1
```



### C2. Crear exactamente 2 VMs

Para **cada** VM:

1. Adaptador 1: **Host-Only**, misma red `vboxnet0` (tráfico del lab).
2. Adaptador 2 (opcional pero recomendado): **NAT**, solo para `apt`. Las IPs y las pruebas de esta guía van por Host-Only.
3. En VM2, adaptador Host-Only → Advanced → Promiscuous Mode: **Allow All** (necesario para ver ARP; no estorba en las otras actividades).

Instala el SO en ambas VMs y arráncalas.

### C3. IPs estáticas en la NIC Host-Only

Identifica la interfaz Host-Only con `ip a` (suele ser la segunda, p. ej. `eth1` / `enp0s8`).

**VM1 Kali — ejemplo con NetworkManager:**

```bash
# Sustituye NOMBRE_CONEXION e IFACE por lo que muestre: nmcli -p device
sudo nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.56.10/24 \
  ipv4.gateway 192.168.56.1 \
  ipv4.dns 192.168.56.1
sudo nmcli con up "Wired connection 1"
```

**VM2 Ubuntu — ejemplo netplan** (`/etc/netplan/01-hw2.yaml`; ajusta el nombre de la NIC Host-Only):

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.20/24
      routes:
        - to: default
          via: 192.168.56.1
          metric: 200
      nameservers:
        addresses: [192.168.56.1]
```

```bash
sudo chmod 600 /etc/netplan/01-hw2.yaml
sudo netplan apply
```

Si existe NIC NAT, déjala en DHCP para instalar paquetes **antes** del plan de pruebas. Durante las pruebas, el tráfico Attacker ↔ Victim no debe ir por NAT.

En VM2, gateway y DNS de la NIC Host-Only = `192.168.56.1`.

### C4. Actualizar sistemas (vía NAT)

En **ambas** VMs antes de continuar con el setup:

```bash
sudo apt update && sudo apt upgrade -y
```



### C5. Verificar L3 del lab

Desde Kali: `ping -c 3 192.168.56.20`  
Desde Ubuntu: `ping -c 3 192.168.56.10` y `ping -c 3 192.168.56.1`

Importante: Si alguno falla, **no pases a Snort ni a las actividades**.

### C6. Instalar Snort solo en VM2 - Ubuntu (Victima)

```bash
sudo apt update
sudo apt install -y snort
snort -V
```

Cuando el instalador pregunte la red a proteger (`HOME_NET`), usa `192.168.56.0/24`.

Ajusta nombres según tu versión: Snort 2.x usa `snort.conf`; Snort 3 usa `snort.lua`.

Crea el archivo de reglas del equipo (puede quedar casi vacío hasta que actives reglas por actividad):

```bash
sudo mkdir -p /etc/snort/rules
sudo nano /etc/snort/rules/local_hw2.rules
```

En la config principal (Snort 2, ejemplo):

```text
include $RULE_PATH/local_hw2.rules
```

Define:

- `HOME_NET`: `192.168.56.0/24`
- `EXTERNAL_NET`: `any` o `!$HOME_NET`

Arranque típico (cambia `eth0` por la NIC Host-Only de VM2, `ip a`):

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

Opciones útiles en VMs: `-k none` si hay checksums raros; logs en `/var/log/snort/` con `-A fast`.

### C7. Wireshark en VM2 - Ubuntu (Victima)

```bash
sudo apt install -y wireshark
# Añade tu usuario al grupo si quieres capturar sin sudo (cierra sesión después):
sudo usermod -aG wireshark "$USER"
```



### C8. Herramientas en VM1 - Kali

Instala lo que pide el enunciado **solo en la VM1, Attacker**.

Paquetes frecuentes en Kali (ajusta si ya vienen preinstalados):

```bash
sudo apt update
sudo apt install -y nmap traceroute hping3 ettercap-graphical
```

Bettercap suele venir en Kali; si no: instálalo desde la documentación oficial de Kali/Bettercap.

Hasta aquí el entorno **arranca**. Aún faltan reglas y servicios especificos por actividad.

### C9. Baseline mínimo (Fase 0 parcial)

1. Snort en consola en VM2, con `local_hw2.rules` incluido.
2. Un ping Kali → Victim y, si HTTP ya está, un GET breve.
3. Confirma que no hay una ráfaga absurda de alertas (o documéntalas).

---



## Setup específico — Actividad 1: Reconocimiento

Necesario para Ping/ICMP, traceroute y nmap. **No** hace falta el servidor HTTP ni Ettercap.

### S1.1 Reglas en `local_hw2.rules` (VM2)

A continuacion se muestran y documentan las reglas necesarias para realizar las detecciones y alertas relevantes a la actividad:


| SID              | Propósito                                    |
| ---------------- | -------------------------------------------- |
| 1001001, 1001002 | ICMP echo / posible flood                    |
| 1005001, 1005002 | traceroute (ICMP TTL bajo / UDP 33434–33534) |
| 1006001–1006004  | nmap SYN / NULL / FIN / XMAS                 |


```text
# Echo Request hacia HOME_NET
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

Reinicia Snort para recargar reglas. Ajusta umbrales y el rango UDP de traceroute según tu versión y el PCAP.

### S1.2 Kali

Confirma que `ping`, `traceroute` y `nmap` responden (`command -v ping traceroute nmap`).

### S1.3 Opcional: HTTP en VM2

Si quieres que nmap muestre un servicio real en el puerto 80, adelanta el [paso S2.1](#s21-servicio-objetivo-en-vm2). No es obligatorio para detectar el scan.

### S1.4 Verificación de setup (actividad 1)

- [ ] `ping` Kali ↔ Ubuntu
- [ ] Snort escuchando la NIC Host-Only
- [ ] SIDs 1001xxx / 1005xxx / 1006xxx cargados (Snort no debe quejarse al arrancar)
- [ ] Wireshark listo en VM2 (filtro `icmp` o la interfaz Host-Only)

**Siguiente:** guía principal, **Fases 1, 4 y 5** (sección 8). Guarda evidencia y vuelve aquí si vas por la opción 2.

---



## Setup específico — Actividad 2: SYN Flood

### S2.1 Servicio objetivo en VM2

Necesitas un puerto TCP escuchando (80 / 443 / 22). Ejemplo mínimo:

```bash
sudo apt install -y nginx
sudo systemctl enable --now nginx
ss -tlnp | grep ':80'
```

Alternativa sin nginx:

```bash
sudo python3 -m http.server 80
```



### S2.2 Monitoreo en la víctima

Ten preparada una terminal para, **durante la prueba** (guía, Fase 2):

```bash
ss -ant | grep SYN-RECV | wc -l
# o
ss -ant
```



### S2.3 Herramienta de generación en Kali

Instala al menos una de las citadas en el enunciado para SYN flood (`hping3`, Scapy, etc.). Comprobar que el binario existe basta para el setup:

```bash
command -v hping3
```

Referirse a la guia de actividad para revisar el playbook de ataque y la informacion que se debe recopilar para los entregables.

### S2.4 Reglas SYN flood (VM2)

SIDs 1002001 y 1002002 (guía, sección 7.2):

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

Reinicia Snort. Empieza sin `threshold` si la alerta no dispara; luego ajústalo.

### S2.5 Verificación de setup (actividad 2)

- [ ] Puerto 80 (o el elegido) en `LISTEN` en VM2
- [ ] `hping3` (o equivalente) en Kali
- [ ] SIDs 100200x cargados
- [ ] Terminal lista para `ss -ant`

---

## Setup específico — Actividad 3: ARP + DNS

Esta actividad usa el **host** `.1` **como gateway**. Kali ya está en el mismo L2 Host-Only (no hay que cambiar de red, a diferencia del escenario B).

### S3.1 Red y visibilidad

1. Promiscuous **Allow All** en el adaptador Host-Only de las maquinas virtuales, especialmente la VM2 - Ubuntu - Victima.
2. `ping -c 3 192.168.56.1` desde VM2 **y** desde Kali.
3. En VM2, anota la tabla ARP **antes** de la prueba: `ip neigh`.
4. En el host, también puedes guardar `ip neigh` de `vboxnet0`.

### S3.2 Herramientas MITM en Kali (instalación)

Bettercap y/o Ettercap, como indica el enunciado. Verifica que arrancan (p. ej. `ettercap -v` / `bettercap -v`). El procedimiento de poisoning está en la guía de actividad 3, a alto nivel y en la documentación oficial.

### S3.3 Dominio de prueba (Ettercap)

Prepara `etter.dns` (o el equivalente en Bettercap) con un **dominio de prueba del lab** redirigido a una IP que controle el equipo (p. ej. la de VM2 o el host). No uses cuentas ni sitios reales de producción.

Ruta típica: `/etc/ettercap/etter.dns` (confirma con `dpkg -L ettercap-common` o similar).

### S3.4 DNS en la víctima

VM2 debe resolver por el camino del lab (DNS de la NIC Host-Only = `192.168.56.1`). Si el resolver de systemd ignora ese DNS, anótalo: en el experimento de redirección la evidencia clave es PCAP + tablas ARP, no “que Facebook real se caiga”.

### S3.5 Reglas ARP / DNS (VM2)

Snort clásico puede tener limitaciones con ARP según el build. Incluye al menos las de la guía, secciones 7.3–7.4, y planea evidencia cruzada con Wireshark.

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

Si `alert arp` no carga en tu build, déjalo documentado y apóyate en PCAP + logs de la herramienta (la guía lo contempla).

### S3.6 HTTP (si aún no lo tienes)

El SID 1003002 y la demo de redirección se entienden mejor con un servicio web en VM2 ([S2.1](#s21-servicio-objetivo-en-vm2)).

### S3.7 Verificación de setup (actividad 3)

- [ ] Tres nodos L2 vivos: `.10`, `.20`, `.1`
- [ ] Promiscuous Allow All en VM2
- [ ] Bettercap/Ettercap instalados; `etter.dns` (o equivalente) con dominio de **prueba**
- [ ] Wireshark en VM2 (filtro `arp or dns`)
- [ ] Snapshot de `ip neigh` **antes**
- [ ] Snort recargado; si ARP nativo falla, plan B de correlación listo

---

## Setup completo antes de las 3 actividades

Orden recomendado:

1. **C1–C9** (común).
2. **S1.1–S1.2** (reglas recon + herramientas Kali).
3. **S2.1–S2.4** (HTTP + hping3 + reglas SYN). El HTTP cubre también nmap y la actividad 3.
4. **S3.1–S3.5** (visibilidad ARP, Ettercap/`etter.dns`, reglas ARP/DNS).
5. Checklist final:

- [ ] `ping` Attacker ↔ Victim y Victim ↔ host `.1`
- [ ] Snort en VM2 sobre NIC Host-Only, **todas** las reglas `1001xxx`–`1006xxx` cargadas
- [ ] nginx (o HTTP) en `.20:80`
- [ ] Wireshark en VM2
- [ ] Kali: nmap, traceroute, hping3, Bettercap/Ettercap
- [ ] `etter.dns` de prueba listo
- [ ] Promiscuous Allow All

---

## Problemas frecuentes (simulación)


| Síntoma                     | Qué revisar                                                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------- |
| Las VMs no se hacen `ping`  | Ambas en la **misma** Host-Only; IPs `/24`; firewall (`ufw`/`firewalld`)                                |
| VM2 no llega a `.1`         | `vboxnet0` arriba en el **host**; no uses Internal Network                                              |
| `apt` no funciona           | Activa el adaptador NAT; no mezcles esa ruta con las pruebas                                            |
| Snort no ve paquetes        | `-i` es la NIC Host-Only, no la NAT; promiscuous en VM2                                                 |
| Checksums inválidos         | `snort ... -k none` en el lab virtualizado                                                              |
| `alert arp` no carga        | Build sin decoder ARP; usa Wireshark + discusión (guía 7.3)                                             |
| Traceroute de 1 hop         | Esperado en A; documentar en Methodology                                                                |
| NAT “roba” el default route | Métrica: Host-Only puede no ser el default; el lab solo exige L2/L3 Host-Only entre `.10`, `.20` y `.1` |


---



## Evidencias que el setup debe dejar listas

Para cada actividad, la guía pide: texto de regla(s), screenshot de alerta Snort, captura de la herramienta en Kali, PCAP/filtro Wireshark, y una frase de por qué matcheó. Revisar la guia

Ética: red aislada (Host-Only); dominios de prueba; detén floods al obtener evidencia.

Subir evidencias de preparacion del setup con fines de documentarlo en el documento de entrega final en el repo de Github en /Setup/Evidencias en la carpeta que corresponda.
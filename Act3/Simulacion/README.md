# Actividad 3 — ARP spoofing y DNS poisoning (simulación)

Envenenamiento de la tabla ARP de la víctima respecto al gateway del lab (el **host** `192.168.56.1`) y, sobre ese MITM, una respuesta DNS falsa a un **dominio de prueba**. Snort complementa; Wireshark y `ip neigh` son la evidencia fuerte.

Setup de esta actividad: [`../../Setup/Simulacion/README.md`](../../Setup/Simulacion/README.md) (apartado *Setup específico — Actividad 3*).  
Guía del curso: [`../../Guia_HW2_IDS_Snort_LIS4062.md`](../../Guia_HW2_IDS_Snort_LIS4062.md) (concepto: sección 3.4; herramientas: sección 6).

---

## Índice

1. [Resumen](#resumen)
2. [Estado de partida](#estado-de-partida)
3. [Antes de generar tráfico](#antes-de-generar-tráfico)
4. [Baseline ARP](#1-baseline-arp)
5. [ARP spoofing](#2-arp-spoofing)
6. [DNS poisoning](#3-dns-poisoning)
7. [Restaurar la red](#4-restaurar-la-red)
8. [Qué llevar al reporte y al video](#qué-llevar-al-reporte-y-al-video)
9. [Checklist de evidencias](#checklist-de-evidencias)

---

## Resumen

ARP asocia IP ↔ MAC en el enlace. Si la víctima cree que `192.168.56.1` es la MAC de Kali, el tráfico hacia el gateway pasa por el atacante. El DNS spoof de LAN **se apoya** en ese MITM: se responde una IP que controla el equipo para un nombre de prueba (no un sitio real con cuentas).

| Qué haces | Para qué | Por qué importa en la entrega |
|-----------|----------|-------------------------------|
| Guardar `ip neigh` **antes** | Baseline: `.1` → MAC real de `vboxnet0` | Sin “antes” el “después” no demuestra envenenamiento |
| ARP spoof Kali–víctima–gateway `.1` | Tabla ARP alterada + frames ARP en PCAP | Rúbrica: análisis ARP (Snort puede no decodificar ARP; el curso acepta Wireshark + herramienta) |
| Consulta DNS del dominio de prueba desde Ubuntu | Query + respuesta falsa en PCAP; SIDs `100400x` si aplican | Rúbrica: DNS poisoning; el enunciado pide redirección tipo “nombre → otra IP” |
| Restaurar (parar spoof, flush ARP) | Lab usable otra vez; honestidad metodológica | Conclusions / Methodology: el experimento es reversible y aislado |

En Results sé explícitos: **ARP se demuestra con PCAP + `ip neigh` + logs de Ettercap/Bettercap**; Snort aporta si `alert arp` carga o si ves HTTP/DNS post-MITM (`1003002`, `1004001`/`1004002`).

Todo en Host-Only. Kali ya está en el mismo L2 que Ubuntu y el host; no hay que cambiar de red (eso es del escenario de 3 laptops).

---

## Estado de partida

Se asume el **setup común de simulación** y el específico de esta actividad:

- Promiscuous **Allow All** en el Host-Only de VM2.
- `ping` Kali ↔ Ubuntu y Ubuntu ↔ host `.1`.
- Bettercap y/o Ettercap en Kali; `etter.dns` (o equivalente) con un **dominio de prueba** → IP del equipo (p. ej. `.20` o `.1`).
- Reglas ARP/DNS del setup (`1003001`, `1003002`, `1004001`, `1004002`) en `local_hw2.rules`, sabiendo que `alert arp` puede fallar según el build.
- HTTP en VM2 ayuda a la regla `1003002` y a ver un GET tras el MITM; si no está, el PCAP ARP/DNS sigue siendo válido.

No se copia aquí `etter.dns` ni la instalación. Si `.1` no responde, no empieces el spoof: arregla el Host-Only en el setup.

| Nodo | IP | Papel |
|------|-----|--------|
| Kali | 192.168.56.10 | Herramienta MITM en el mismo L2 |
| Ubuntu | 192.168.56.20 | Víctima: ARP, DNS, Wireshark |
| Host | 192.168.56.1 | Gateway lógico a suplantar |

---

## Antes de generar tráfico

En VM2:

1. Snort en consola, NIC Host-Only. Si al cargar `1003001` Snort rechaza `alert arp`, comenta esa línea, recarga y pasa al **plan B** (PCAP + herramienta), como dice el setup.
2. Wireshark en Host-Only. Tendrás dos momentos de captura (o un PCAP continuo con marcas de tiempo): ARP y DNS.
3. Anota MAC de Kali y MAC de `vboxnet0` en el host (`ip link` / `ip neigh`). Te sirven para leer el PCAP.

Carpeta sugerida: `Act3/Evidencias/<tu_nombre>/`.

**Evidencia ahora:** reglas usadas (o nota “ARP no soportado en este build”), interfaz en promiscuo ya resuelta en setup.

---

## 1. Baseline ARP

En **Ubuntu**:

```bash
ip neigh
# equivalente clásico:
arp -a
```

Identifica la línea de `192.168.56.1`. Guarda screenshot **completo**, no recortes la MAC.

Opcional en el **host** (Linux con `vboxnet0`):

```bash
ip neigh
```

**Evidencia:** `ip neigh` / `arp -a` de VM2 (y del host si lo capturas). Esta figura es obligatoria: es el “antes”.

Wireshark puede quedar ya grabando con filtro `arp`.

---

## 2. ARP spoofing

Objetivo: que en la víctima `192.168.56.1` quede asociado a la **MAC de Kali**, no a la de `vboxnet0`.

### En Kali

Con Bettercap o Ettercap (las que cita el enunciado), realiza el envenenamiento ARP **en esta LAN**, suplantando el gateway `192.168.56.1` frente a `192.168.56.20`. El procedimiento de la herramienta está en su documentación oficial y en el material del curso; no se detalla aquí un playbook de MITM.

Condiciones del lab:

- Solo Host-Only; objetivos `.20` y `.1`.
- Deja visible la UI o el log de la herramienta (video + screenshot).
- No atacas el NAT ni otras VMs.

### En Ubuntu, mientras corre

1. Repite `ip neigh` y compara la MAC de `.1` con el baseline.
2. Wireshark filtro `arp`: replies/announces que no coinciden con el baseline.
3. Consola Snort: `1003001` si cargó; si no, no insistas en esa SID.

### Qué debe coincidir

| Pieza | Qué buscar |
|-------|------------|
| `ip neigh` en VM2 | MAC de `.1` **distinta** a la del baseline (la de Kali) |
| Wireshark `arp` | Tráfico ARP anómalo (replies duplicados / MAC inesperada) |
| Bettercap/Ettercap | Log o pantalla indicando spoof activo |
| Snort | `1003001` **o** explicación honesta de que no hay decoder ARP |

**Evidencia:** `ip neigh` **después**, PCAP ARP, screenshot de la herramienta, alerta Snort o párrafo de limitación.

---

## 3. DNS poisoning

El spoof DNS de este homework **sigue** al ARP: la víctima debe resolver por un camino que el atacante puede responder.

El nombre a consultar es el que configuraste en `etter.dns` (o el módulo DNS de Bettercap) durante el setup: un dominio **de prueba** (p. ej. algo tipo `lab-hw2.test`), nunca credenciales de un sitio real. La IP de respuesta debe ser una del lab (`.20` o `.1`).

### En Ubuntu (víctima)

1. Wireshark: `dns` (o `arp or dns` si sigues el mismo PCAP).
2. Resuelve el nombre de prueba, por ejemplo:

```bash
getent hosts lab-hw2.test
# o
nslookup lab-hw2.test
# o
dig lab-hw2.test
```

Sustituye el nombre por el de tu `etter.dns`. Si `systemd-resolved` ignora el DNS `192.168.56.1`, anótalo: la evidencia sigue siendo el PCAP (query + answer), no “que el navegador abra Internet”.

3. Si HTTP está en `.20` y redirigiste a esa IP, un GET al nombre de prueba puede disparar `1003002` y se ve bien en video. No es obligatorio si el DNS en Wireshark ya muestra la IP falsa.

### Qué debe coincidir

| Pieza | Qué buscar |
|-------|------------|
| Wireshark DNS | Query del nombre de prueba + **respuesta A** a la IP del lab (no la “real” de Internet; en lab aislado ni siquiera hay resolución pública) |
| Snort | `1004002` (query desde HOME_NET) y/o `1004001` (respuestas hacia el cliente) |
| Víctima | Salida de `dig`/`nslookup` con la IP controlada |

**Evidencia:** PCAP DNS (query + answer), screenshot de la resolución en Ubuntu, alerta Snort si existe, recorte de `etter.dns` (solo el dominio de prueba).

En Discussion: una regla que cuenta paquetes UDP/53 **no prueba** por sí sola que la respuesta sea falsa; por eso el PCAP y el MITM son el núcleo, y Snort es apoyo.

---

## 4. Restaurar la red

1. Detén Bettercap/Ettercap en Kali.
2. En Ubuntu (y en el host si alteraste vecindario):

```bash
ip neigh flush dev <nic-host-only>
```

(Usa el nombre real de la NIC Host-Only, el mismo que en el setup.)

3. `ping -c 2 192.168.56.1` desde VM2. `ip neigh` debe volver a la MAC del baseline (puede tardar un instante).
4. Para Wireshark y Snort.

**Evidencia:** `ip neigh` restaurado y ping a `.1` OK. Una figura “después de restaurar” cierra Methodology.

---

## Qué llevar al reporte y al video

- **Methodology:** gateway = host `.1`; sensor en la víctima; ARP es L2 (Kali ya estaba en Host-Only); dominio de prueba, no Facebook real si no hay Internet en el lab — el enunciado habla de redirección; en simulación aislada se demuestra con el nombre de prueba.
- **Results:** antes/después ARP; PCAP ARP; PCAP DNS; qué pudo y qué no pudo hacer Snort.
- **Video:** integrante en el rol “víctima” muestra `ip neigh` y Wireshark; Kali muestra la herramienta (guía, sección 9, bloque ARP/DNS).

Nombres de archivo útiles:

```text
Act3/Evidencias/<nombre>/
  reglas_arp_dns.txt
  arp_antes_ipneigh.png
  arp_despues_ipneigh.png
  arp_wireshark.pcap
  arp_kali_herramienta.png
  snort_arp_o_nota_limitacion.png
  etter_dns_prueba.txt
  dns_resolucion_ubuntu.png
  dns_wireshark.pcap
  dns_snort.png
  arp_restaurado.png
```

---

## Checklist de evidencias

**Preparación**

- [ ] `ping` VM2 ↔ host `.1` OK **antes** del spoof
- [ ] Dominio de prueba definido (no cuentas reales)
- [ ] Snort arriba; si `alert arp` falló, está documentado

**ARP**

- [ ] `ip neigh` / `arp -a` **antes** (MAC de `.1`)
- [ ] `ip neigh` **durante** el spoof (MAC de `.1` = Kali)
- [ ] PCAP o screenshot Wireshark `arp`
- [ ] Screenshot de Bettercap/Ettercap
- [ ] Alerta `1003001` **o** párrafo de limitación de Snort + correlación PCAP

**DNS**

- [ ] Recorte de `etter.dns` (o config Bettercap) con el nombre de prueba
- [ ] Resolución desde Ubuntu (salida `dig`/`nslookup`/`getent`)
- [ ] PCAP DNS: query + respuesta a IP del lab
- [ ] Alerta `1004001` y/o `1004002` si disparó; si no, explicación con el PCAP
- [ ] Opcional: GET HTTP y `1003002`

**Cierre**

- [ ] Spoof detenido; `ip neigh` restaurado; `ping` a `.1` OK
- [ ] En Results: por qué ARP no se “demuestra solo con una SID”
- [ ] Archivos en `Act3/Evidencias/<nombre>/`
- [ ] Todo el tráfico quedó en `192.168.56.0/24`

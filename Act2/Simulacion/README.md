# Actividad 2 — TCP SYN Flood (simulación)

Denegación de servicio de prueba: muchos TCP SYN hacia un puerto de la víctima (`192.168.56.20`), sin completar el handshake. Snort alerta por umbral; en Ubuntu se correlaciona con conexiones *half-open* (`SYN-RECV`).

Setup de esta actividad: `[../../Setup/Simulacion/README.md](../../Setup/Simulacion/README.md)` (apartado *Setup específico — Actividad 2*).

---

## Índice

1. [Resumen](#resumen)
2. [Ética de esta actividad](#ética-de-esta-actividad)
3. [Qué pide el enunciado (HW2, procedimiento 1)](#qué-pide-el-enunciado-hw2-procedimiento-1)
4. [hping3, Scapy y RUDY: para qué sirven y dónde leer el uso](#hping3-scapy-y-rudy-para-qué-sirven-y-dónde-leer-el-uso)
5. [Estado de partida](#estado-de-partida)
6. [Antes de generar tráfico](#antes-de-generar-tráfico)
7. [Baseline del servicio](#1-baseline-del-servicio)
8. [Prueba SYN flood](#2-prueba-syn-flood)
9. [Parar y contrastar](#3-parar-y-contrastar)
10. [Qué llevar al reporte y al video](#qué-llevar-al-reporte-y-al-video)
11. [Checklist de evidencias](#checklist-de-evidencias)

---

## Resumen

El handshake normal es SYN → SYN/ACK → ACK. En un SYN flood el origen envía **muchos SYN** y **no cierra** el tres vías: la víctima acumula estados `SYN-RECV` / `SYN_RECEIVED`. El IDS no “cura” el ataque; **detecta y alerta**.


| Qué haces                                  | Para qué                                                            | Por qué importa en la entrega                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Dejar HTTP (u otro TCP) en `LISTEN` en VM2 | El flood tiene un puerto objetivo real (regla `1002002` mira el 80) | Methodology reproducible; nmap/ARP de otras actividades también se benefician del servicio             |
| Generar SYN masivo **solo** a `.20`        | Disparar `1002001` / `1002002`                                      | Rúbrica: detección SYN Flood / DoS (peso alto en el bloque técnico)                                    |
| Contar `SYN-RECV` con `ss`                 | Segunda evidencia, independiente de Snort                           | El PDF del curso pide esta verificación auxiliar; Results más sólido si alerta + `ss` + PCAP coinciden |


Esta es de las demos que más se ven en el **video** (guion: integrante que explica half-open). Prioriza capturas claras y **corta el test** en cuanto tengas evidencia: no hace falta tumbar la VM.

---

## Ética de esta actividad

Esta práctica existe para **diseñar y demostrar un IDS** (Snort detecta y alerta). No es un ejercicio de tumbar servicios ajenos ni de ensayar DoS fuera del laboratorio del curso.

- **Solo** la red Host-Only `192.168.56.0/24`. Destino de prueba: `192.168.56.20`. Prohibido campus, Internet, otras VMs del host o equipos que no sean del equipo.
- Instalar hping3/Scapy (y documentar RUDY) no autoriza usarlos contra nada que no sea esa víctima.
- Genera el mínimo de tráfico para obtener alerta Snort + `SYN-RECV`/`SYN_RECEIVED` + PCAP. **Para en cuanto tengas evidencia**; no dejes la herramienta corriendo ni busques colgar Ubuntu.
- No subas al reporte ni al video IPs, cuentas o redes que no sean del lab. No redistribuyas recetas de DoS “para otros entornos”.
- Al terminar: detén la herramienta, comprueba que `ss` vuelve a un baseline sano, y deja el HTTP de VM2 en un estado usable.

Si un integrante no tiene claro el aislamiento (NAT vs Host-Only), no genera tráfico hasta que el `ping` Kali ↔ Ubuntu sea solo por `.10` / `.20`.

---

## Qué pide el enunciado (HW2, procedimiento 1)

En `HW2 LIS4062 Autumn 26.pdf` esta actividad es **“1. TCP SYN Flood Attack”**. El PDF describe el tres vías, el estado *half-open*, la cola *backlog* y pide:

1. **Identificar** ese tráfico con Snort (reglas del setup `1002001` / `1002002`).
2. **Corroborar** en la víctima con Wireshark y con `netstat` si hay muchas conexiones en `SYN_RECEIVED`.
3. Usar **alguna** de las herramientas que lista para generar el tráfico de prueba en el lab aislado.

Instalación y `command -v` / `import scapy`: setup **C8.2** y **S2.3**. Abajo, para qué sirve cada una y **dónde leer el uso** (este README no incluye invocaciones de flood).

El PDF da el comando de **detección** en la víctima (no el de generación):

```bash
netstat -n -p tcp
```

Busca muchas líneas en estado `SYN_RECEIVED`. En Ubuntu moderno el equivalente es `ss -ant` y `SYN-RECV` (ambos son válidos en Results).

---

## hping3, Scapy y RUDY: para qué sirven y dónde leer el uso

El PDF (procedimiento 1) también nombra Engage Packet Builder, LOIC y DDoSIM. Para este lab bastan las tres de abajo; las otras siguen en las referencias [2], [10] si el equipo ya las conoce.

| Herramienta | Para qué sirve (en el homework) | Dónde leer el uso | Relación con las reglas Snort de esta actividad |
|-------------|---------------------------------|-------------------|--------------------------------------------------|
| **hping3** | Constructor de paquetes en CLI. Permite tráfico TCP (entre otros) hacia un host/puerto; el enunciado lo cita para SYN flood. | En Kali: `man hping3`, `hping3 -h`. Paquete: [hping3 en Kali](https://www.kali.org/tools/hping3/). Curso: HW1 si ya lo usaron. | Adecuada para disparar `1002001` / `1002002` (`flags:S`) si generas SYN hacia `.20:80`. |
| **Scapy** | Librería Python para ensamblar y enviar paquetes (IP/TCP/ICMP, etc.). Misma idea que hping3, con scripts. | Manual: [scapy.readthedocs.io](https://scapy.readthedocs.io/). Uso interactivo: `scapy` tras instalar `python3-scapy`. Tutorial de envío TCP: sección *Usage* / *TCP* en ese manual. | Adecuada para SYN flood del procedimiento 1. Cita Scapy en Bibliography si la usas. |
| **RUDY** (R-U-Dead-Yet?) | Clase de DoS **HTTP lento** (POST muy grande / cuerpo a cuentagotas), no un flood de SYN. El PDF la lista junto a otras herramientas de DoS. | Referencia del enunciado [11]: [Invicti — RUDY attack](https://www.invicti.com/learn/rudy-attack). No hay `man rudy` ni paquete apt (ver setup C8.2). | **No** sustituye el procedimiento 1: no apunta a half-open TCP ni a `flags:S`. Úsala solo como contraste en Discussion (“otro DoS”) si el equipo quiere citar [11]. |

Otras citadas en el PDF (documentación, no instaladas en el setup):

- Engage Packet Builder [2]: [engagesecurity.com — Packet Builder](http://www.engagesecurity.com/products/engagepacketbuilder)
- DDoSIM [10]: [ddosim.live](https://ddosim.live/)

Para alinear evidencia con SIDs `100200x`, elige **hping3 o Scapy**. Consulta el `man` / ReadTheDocs / HW1 para la sintaxis; aquí no se reproduce.

---

## Estado de partida

Se asume el **setup común de simulación** (VMs, IPs, Snort en VM2, Wireshark) y **C8.2 / S2.3**: `hping3 -v` OK, `import scapy` OK, RUDY descargado desde GitHub.

Además, para *esta* actividad el setup específico pide:

- Puerto TCP escuchando en VM2 (nginx o `python3 -m http.server 80`).
- Reglas `1002001` y `1002002` en `local_hw2.rules`.
- Una terminal en Ubuntu lista para `ss -ant`.

Si el 80 no está en `LISTEN` o las reglas no cargan, referirse al setup antes de continuar.


| Nodo   | IP            | Rol en esta prueba                  |
| ------ | ------------- | ----------------------------------- |
| Kali   | 192.168.56.10 | Genera SYN hacia `.20:80`           |
| Ubuntu | 192.168.56.20 | Servicio + Snort + `ss` + Wireshark |


---

## Antes de generar tráfico

En VM2:

1. Confirma el servicio: `ss -tlnp | grep ':80'` (o el puerto que hayas elegido; si no es 80, la regla `1002002` no aplicará tal cual).
2. Recarga Snort con las reglas SYN flood del setup. Arráncalo en consola sobre la NIC Host-Only.
3. Wireshark en esa NIC. Filtro útil después: TCP SYN sin ACK hacia el puerto 80.
4. Segunda terminal: no lances aún el conteo en bucle; ten el comando copiado.

Carpeta sugerida: `Act2/Simulacion/Evidencias/<tu_nombre>/`.

**Evidencia ahora:** `ss -tlnp` mostrando `LISTEN` en 80, snippet de las reglas `100200x`, Snort sin error de parseo.

---

## 1. Baseline del servicio

Objetivo: demostrar que, sin flood, no hay cola de half-open.

En Ubuntu:

```bash
ss -ant | grep SYN-RECV | wc -l
```

Lo esperable es `0` (o un número despreciable). Abre también `http://192.168.56.20/` desde Kali (navegador o `curl`) **una vez**: tráfico legítimo, handshake completo. Snort no debería disparar flood por un solo GET.

**Evidencia:** screenshot del conteo en ~0 y, si puedes, un GET de control. En Results: *antes = servicio sano*.

---

## 2. Prueba SYN flood

Tres ventanas a la vez: **Snort**, `ss`, **Wireshark** (Ubuntu) y la herramienta en **Kali**.

### En Ubuntu, al mismo tiempo

1. Wireshark capturando.
2. Snort en consola.
3. Monitoreo (repite durante la prueba). El enunciado pide `netstat`; en Ubuntu usa los dos si quieres capturas paralelas:

```bash
ss -ant | grep SYN-RECV | wc -l
ss -ant
netstat -n -p tcp
```

Si `netstat` no está: `sudo apt install -y net-tools` (vía NAT, como en el setup). Buscas muchos estados `SYN-RECV` (`ss`) o `SYN_RECEIVED` (`netstat`).

### En Kali — según documentación oficial

Elige **hping3, Scapy o RUDY** (ya verificados en el setup). La sintaxis de envío está en:

- hping3: `man hping3` y `hping3 -h`
- Scapy: [scapy.readthedocs.io](https://scapy.readthedocs.io/) (envío de paquetes TCP)
- RUDY: [Repo de RUDY en GitHub](https://github.com/darkweak/rudy)

Condiciones fijas del lab (coinciden con el procedimiento 1 del PDF y con la [ética](#ética-de-esta-actividad)):

- Destino: `192.168.56.20`, puerto **80** (el servicio que dejaste en `LISTEN`). No el NAT ni el host `.1`.
- Efecto buscado: muchos SYN **sin** el ACK final del tres vías (half-open en la víctima).
- Duración: corta. En cuanto veas alerta `100200x` **y** suba `SYN-RECV` / `SYN_RECEIVED`, **para** (`Ctrl+C` o cierra Scapy).
- No hace falta randomizar IPs ni colgar Ubuntu: el PDF habla de llenar el *backlog*; para el homework basta evidencia, no un DoS real.

Filtro Wireshark de apoyo (víctima): `tcp.flags.syn == 1 and tcp.flags.ack == 0 and ip.dst == 192.168.56.20`.

Si Snort no alerta pero `ss`/`netstat` sí muestran half-open: baja o quita temporalmente el `threshold` de la regla (el setup lo sugiere), recarga Snort y repite un burst breve. Documenta el umbral final en Results.

Si `ss` no sube: el tráfico no llega a Host-Only (interfaz equivocada, firewall) o la herramienta no está enviando SYN puros. Revisa Wireshark antes de insistir.

### Qué debe coincidir


| Pieza            | Qué buscar                                                                |
| ---------------- | ------------------------------------------------------------------------- |
| Snort            | `HW2 Possible TCP SYN Flood` (`1002001`) y/o targeting TCP/80 (`1002002`) |
| `ss` / `netstat` | Conteo alto de `SYN-RECV`                                                 |
| Wireshark        | Muchos SYN `.10` → `.20:80` sin ACK de cierre del cliente                 |
| Kali             | Pantalla de la herramienta (sin datos ajenos al lab)                      |


**Evidencia (durante la prueba, no después de parar todo):** screenshot de alerta Snort, screenshot de `ss -ant` con half-open, PCAP o filtro TCP, screenshot de Kali. Anota hora aproximada para alinear figuras.

---

## 3. Parar y contrastar

1. Detén la herramienta en Kali.
2. Espera unos segundos y vuelve a `ss -ant | grep SYN-RECV | wc -l`. El número debe bajar (timeouts).
3. Detén la captura de Wireshark y guarda el PCAP.
4. Copia o recorta el fragmento relevante de `/var/log/snort/` si usaste `-A fast` además de consola.

**Evidencia:** un “después” de `ss` cerca de cero refuerza que el pico fue del test, no del estado permanente de la VM.

---

## Qué llevar al reporte y al video

- **Introduction / Methodology:** handshake vs half-open; umbral `count`/`seconds` de tus SIDs; sensor en la víctima.
- **Results:** tabla *herramienta | SID | ¿alerta? | SYN-RECV máx. | figura*. Explica `flags:S` (SYN puro) y por qué un GET normal no basta para el threshold.
- **Video:** pantalla partida o cortes rápidos: Kali generando, Snort alertando, `ss` en Ubuntu. Explica el handshake en inglés (guía, sección 9).

Nombres de archivo útiles:

```text
Act2/Evidencias/<nombre>/
  reglas_synflood.txt
  listen_80.png
  baseline_synrecv.png
  flood_snort.png
  flood_ss_synrecv.png
  flood_wireshark.pcap
  flood_kali.png
  after_synrecv.png
```

---

## Checklist de evidencias

**Preparación**

- [ ] Puerto 80 (o el documentado) en `LISTEN` en VM2
- [ ] Texto de reglas `1002001` y/o `1002002`
- [ ] Snort en NIC Host-Only
- [ ] hping3, Scapy y/o RUDY verificados en el setup; 
- [ ] Leída la sección de ética (solo Host-Only, parar al tener evidencia)

**Durante el test**

- [ ] Screenshot alerta Snort `100200x`
- [ ] Screenshot `ss -ant` y/o `netstat -n -p tcp` (como pide el PDF) con muchos `SYN-RECV` / `SYN_RECEIVED`
- [ ] PCAP o screenshot Wireshark (SYN hacia `.20:80`)
- [ ] Screenshot de la herramienta en Kali (solo lab)

**Cierre**

- [ ] Conteo `SYN-RECV` tras detener el test (contraste)
- [ ] Frase en Results: por qué matcheó (SYN + umbral + destino `$HOME_NET`)
- [ ] Archivos en `Act2/Evidencias/<nombre>/`
- [ ] El flood se cortó al tener evidencia; no se dejó corriendo
- [ ] Tráfico solo a `192.168.56.20`
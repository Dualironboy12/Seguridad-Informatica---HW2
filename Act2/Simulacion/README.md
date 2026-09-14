# Actividad 2 — TCP SYN Flood (simulación)

Denegación de servicio de prueba: muchos TCP SYN hacia un puerto de la víctima (`192.168.56.20`), sin completar el handshake. Snort alerta por umbral; en Ubuntu se correlaciona con conexiones *half-open* (`SYN-RECV`).

Setup de esta actividad: [`../../Setup/Simulacion/README.md`](../../Setup/Simulacion/README.md) (apartado *Setup específico — Actividad 2*).  
Guía del curso: [`../../Guia_HW2_IDS_Snort_LIS4062.md`](../../Guia_HW2_IDS_Snort_LIS4062.md) (concepto SYN flood: sección 3.3; herramientas: sección 6).

---

## Índice

1. [Resumen](#resumen)
2. [Estado de partida](#estado-de-partida)
3. [Antes de generar tráfico](#antes-de-generar-tráfico)
4. [Baseline del servicio](#1-baseline-del-servicio)
5. [Prueba SYN flood](#2-prueba-syn-flood)
6. [Parar y contrastar](#3-parar-y-contrastar)
7. [Qué llevar al reporte y al video](#qué-llevar-al-reporte-y-al-video)
8. [Checklist de evidencias](#checklist-de-evidencias)

---

## Resumen

El handshake normal es SYN → SYN/ACK → ACK. En un SYN flood el origen envía **muchos SYN** y **no cierra** el tres vías: la víctima acumula estados `SYN-RECV` / `SYN_RECEIVED`. El IDS no “cura” el ataque; **detecta y alerta**.

| Qué haces | Para qué | Por qué importa en la entrega |
|-----------|----------|-------------------------------|
| Dejar HTTP (u otro TCP) en `LISTEN` en VM2 | El flood tiene un puerto objetivo real (regla `1002002` mira el 80) | Methodology reproducible; nmap/ARP de otras actividades también se benefician del servicio |
| Generar SYN masivo **solo** a `.20` | Disparar `1002001` / `1002002` | Rúbrica: detección SYN Flood / DoS (peso alto en el bloque técnico) |
| Contar `SYN-RECV` con `ss` | Segunda evidencia, independiente de Snort | El PDF del curso pide esta verificación auxiliar; Results más sólido si alerta + `ss` + PCAP coinciden |

Esta es de las demos que más se ven en el **video** (guion: integrante que explica half-open). Prioriza capturas claras y **corta el test** en cuanto tengas evidencia: no hace falta tumbar la VM.

Ética: únicamente `192.168.56.20` en Host-Only. No uses redes del host fuera del lab.

---

## Estado de partida

Se asume el **setup común de simulación** (VMs, IPs, Snort en VM2, Wireshark, `hping3` u otra herramienta citada en el enunciado ya instalada en Kali).

Además, para *esta* actividad el setup específico pide:

- Puerto TCP escuchando en VM2 (nginx o `python3 -m http.server 80`).
- Reglas `1002001` y `1002002` en `local_hw2.rules`.
- Una terminal en Ubuntu lista para `ss -ant`.

Si el 80 no está en `LISTEN` o las reglas no cargan, completa esos pasos del setup y vuelve aquí. No se reitera la instalación.

| Nodo | IP | Rol en esta prueba |
|------|-----|--------------------|
| Kali | 192.168.56.10 | Genera SYN hacia `.20:80` |
| Ubuntu | 192.168.56.20 | Servicio + Snort + `ss` + Wireshark |

---

## Antes de generar tráfico

En VM2:

1. Confirma el servicio: `ss -tlnp | grep ':80'` (o el puerto que hayas elegido; si no es 80, la regla `1002002` no aplicará tal cual).
2. Recarga Snort con las reglas SYN flood del setup. Arráncalo en consola sobre la NIC Host-Only.
3. Wireshark en esa NIC. Filtro útil después: TCP SYN sin ACK hacia el puerto 80.
4. Segunda terminal: no lances aún el conteo en bucle; ten el comando copiado.

Carpeta sugerida: `Act2/Evidencias/<tu_nombre>/`.

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

Tres ventanas a la vez: **Snort**, **`ss`**, **Wireshark** (Ubuntu) y la herramienta en **Kali**.

### En Ubuntu, al mismo tiempo

1. Wireshark capturando.
2. Snort en consola.
3. Monitoreo (repite durante la prueba):

```bash
ss -ant | grep SYN-RECV | wc -l
ss -ant
```

Buscas muchos estados `SYN-RECV` (o `SYN_RECEIVED` si usas `netstat -n -p tcp`).

### En Kali

Usa **una** de las herramientas que cita el enunciado (hping3, Scapy, LOIC, RUDY, DDoSIM, Engage Packet Builder) para enviar una ráfaga de TCP SYN al **puerto 80 de `192.168.56.20`**, sin completar el handshake.

La sintaxis concreta está en la documentación de la herramienta y en el material del curso / HW1; este documento no replica un playbook ofensivo. Condiciones fijas del lab:

- Destino: `192.168.56.20` (Host-Only), no el NAT ni el host `.1`.
- Duración: corta (del orden de segundos). En cuanto veas alerta `100200x` **y** suba el conteo `SYN-RECV`, **para**.
- No hace falta randomizar IPs ni saturar CPU/RAM hasta colgar Ubuntu.

Si Snort no alerta pero `ss` sí muestra half-open: baja o quita temporalmente el `threshold` de la regla (el setup lo sugiere), recarga Snort y repite un burst breve. Documenta el umbral final en Results.

Si `ss` no sube: el tráfico no llega a Host-Only (interfaz equivocada, firewall) o la herramienta no está enviando SYN puros. Revisa Wireshark antes de insistir.

### Qué debe coincidir

| Pieza | Qué buscar |
|-------|------------|
| Snort | `HW2 Possible TCP SYN Flood` (`1002001`) y/o targeting TCP/80 (`1002002`) |
| `ss` / `netstat` | Conteo alto de `SYN-RECV` |
| Wireshark | Muchos SYN `.10` → `.20:80` sin ACK de cierre del cliente |
| Kali | Pantalla de la herramienta (sin datos ajenos al lab) |

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

**Durante el test**

- [ ] Screenshot alerta Snort `100200x`
- [ ] Screenshot `ss -ant` (o `netstat`) con muchos `SYN-RECV`
- [ ] PCAP o screenshot Wireshark (SYN hacia `.20:80`)
- [ ] Screenshot de la herramienta en Kali (solo lab)

**Cierre**

- [ ] Conteo `SYN-RECV` tras detener el test (contraste)
- [ ] Frase en Results: por qué matcheó (SYN + umbral + destino `$HOME_NET`)
- [ ] Archivos en `Act2/Evidencias/<nombre>/`
- [ ] El flood se cortó al tener evidencia; no se dejó corriendo
- [ ] Tráfico solo a `192.168.56.20`

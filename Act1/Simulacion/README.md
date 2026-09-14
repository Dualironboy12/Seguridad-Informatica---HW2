# Actividad 1 — Reconocimiento (simulación)

Ping/ICMP, traceroute y nmap desde Kali (`192.168.56.10`) hacia la víctima (`192.168.56.20`), con Snort y Wireshark en VM2.

Setup de esta actividad: [`../../Setup/Simulacion/README.md`](../../Setup/Simulacion/README.md) (apartado *Setup específico — Actividad 1*).  
Guía del curso: [`../../Guia_HW2_IDS_Snort_LIS4062.md`](../../Guia_HW2_IDS_Snort_LIS4062.md).

---

## Índice

1. [Resumen](#resumen)
2. [Estado de partida](#estado-de-partida)
3. [Antes de generar tráfico](#antes-de-generar-tráfico)
4. [Baseline](#1-baseline)
5. [Ping / ICMP](#2-ping--icmp)
6. [Traceroute](#3-traceroute)
7. [Nmap](#4-nmap)
8. [Qué llevar al reporte y al video](#qué-llevar-al-reporte-y-al-video)
9. [Checklist de evidencias](#checklist-de-evidencias)

---

## Resumen

En esta actividad el equipo **genera tráfico de reconocimiento** contra la víctima del lab y **comprueba que el IDS lo identifica**. Cubres tres tipos que pide el enunciado: eco ICMP, traceroute y escaneo de puertos (nmap).

| Qué haces | Para qué | Por qué importa en la entrega |
|-----------|----------|-------------------------------|
| Ping Kali → Ubuntu | Disparar SIDs `1001001` / `1001002` y mostrar ICMP en Wireshark | Rúbrica: detección Ping/ICMP documentada |
| Traceroute a `.20` | Probes con TTL bajo o UDP 33434–33534; SIDs `1005001` / `1005002` | Rúbrica: detección traceroute; en simulación el path es **1 hop** y hay que explicarlo en Methodology |
| Nmap a `.20` | SYN (y opcional NULL/FIN/XMAS); SIDs `1006001`–`1006004` | Rúbrica: detección nmap; en Results: qué vio el “atacante” (puertos/OS) vs qué alertó Snort |

Sin estas capturas el reporte se queda en teoría: el profesor evalúa **regla + alerta + PCAP** por cada tipo.

Trabajo **solo** en Host-Only (`192.168.56.0/24`). Destino único de las pruebas: `192.168.56.20`.

---

## Estado de partida

Se asume el **setup común de simulación** terminado (VMs, IPs, `ping` Attacker ↔ Victim, Snort instalado en VM2, `local_hw2.rules` incluido en la config, Wireshark en Ubuntu, `ping` / `traceroute` / `nmap` en Kali).

No se repite aquí cómo crear la red ni cómo instalar paquetes. Si algo de esa lista falla, vuelve al setup común **antes** de seguir.

IPs de trabajo:

| Nodo | IP |
|------|-----|
| Kali (tráfico) | 192.168.56.10 |
| Ubuntu (Snort + Wireshark) | 192.168.56.20 |
| Host (no se usa como objetivo en esta actividad) | 192.168.56.1 |

---

## Antes de generar tráfico

En VM2:

1. Asegúrate de tener en `local_hw2.rules` las reglas de reconocimiento del setup (SIDs `1001001`, `1001002`, `1005001`, `1005002`, `1006001`–`1006004`). Si el archivo aún está vacío, cópialas desde el setup de la actividad 1 y recarga Snort.
2. Arranca Snort en **consola** sobre la NIC **Host-Only** (mismo comando e interfaz que en el setup). Déjala visible: es tu evidencia de alertas.
3. Abre Wireshark en esa misma NIC. No hace falta capturar las tres pruebas en un solo archivo: un PCAP (o screenshot del filtro) por prueba suele leerse mejor en el PDF.

Carpeta sugerida: `Act1/Evidencias/<tu_nombre>/`.

**Evidencia ahora:** captura del archivo de reglas (texto o screenshot) y de Snort arrancando sin error de sintaxis.

---

## 1. Baseline

Objetivo: saber qué es “normal” antes del reconocimiento.

1. Con Snort y Wireshark ya arriba, desde Kali: `ping -c 2 192.168.56.20`.
2. Mira la consola de Snort. Un eco puede disparar `1001001`; eso es coherente. Lo que no debe aparecer es una ráfaga de SIDs de scan o flood.
3. Si hay ruido extraño (alertas de nmap sin haber escaneado), anótalo: irá a Discussion como falso positivo o umbral mal ajustado.

**Evidencia:** screenshot breve de consola en calma (o con solo el ping de control) y una frase en el reporte: *baseline = red viva, sin escaneo*.

---

## 2. Ping / ICMP

Objetivo: demostrar detección de eco ICMP hacia `HOME_NET`.

### En Ubuntu (víctima)

- Wireshark: filtro `icmp`.
- Snort: consola a la vista.

### En Kali

Envía ecos a la víctima, por ejemplo:

```bash
ping -c 8 192.168.56.20
```

Para el SID de flood (`1001002`, umbral 50 ecos / 10 s en la regla del setup), genera **más** ecos en poco tiempo (sigue siendo el `ping` del lab, solo a `.20`). Si la alerta de flood no sale, no fuerces el host: documenta el umbral y qué observaste con `1001001`.

### Qué debe coincidir

| Pieza | Qué buscar |
|-------|------------|
| Snort | `HW2 ICMP Echo Request to HOME_NET` (`1001001`); opcionalmente flood `1001002` |
| Wireshark | Echo request (type 8) `.10` → `.20` y reply type 0 |
| Kali | Salida del `ping` con replies |

**Evidencia:** screenshot de la alerta Snort, screenshot o PCAP de Wireshark (`icmp`), screenshot de la terminal de Kali. En Results: la regla mira `itype:8` hacia `$HOME_NET`.

---

## 3. Traceroute

Objetivo: probes de mapeo de ruta. En este escenario hay **un solo salto** (Kali y Ubuntu están en el mismo L2). El comando sigue siendo válido: genera ICMP y/o UDP que las reglas `100500x` pueden ver. En Methodology y Results **di explícitamente** que el path es de 1 hop; no es un fallo del lab.

### En Ubuntu

- Wireshark: `icmp or udp.port >= 33434`.
- Snort en consola.

### En Kali

```bash
traceroute 192.168.56.20
```

Si tu Kali usa UDP por defecto y no ves `1005002`, prueba la variante ICMP de `traceroute` (consulta `man traceroute` en tu versión) y ajusta el rango de puertos de la regla al que salga en el PCAP, como indica el setup.

### Qué debe coincidir

| Pieza | Qué buscar |
|-------|------------|
| Salida de traceroute | Una línea hacia `.20` (1 hop) |
| Snort | `1005001` (ICMP con TTL bajo) y/o `1005002` (UDP traceroute) |
| Wireshark | TTL pequeño y/o UDP hacia 33434–33534 |

**Evidencia:** salida completa del comando, alerta Snort, PCAP o screenshot del filtro. En Discussion: por qué 1 hop no impide detectar el *probe*.

---

## 4. Nmap

Objetivo: escaneo controlado **solo** a `192.168.56.20` y correlación con SIDs de scan.

Un servicio HTTP en la víctima no es obligatorio para que Snort vea el scan; sí ayuda a que nmap liste el puerto 80 en Results. Si ya lo tienes del setup de la actividad 2, déjalo; si no, no hace falta instalarlo para esta actividad.

### En Ubuntu

- Wireshark: `tcp.flags.syn == 1` es un buen punto de partida para el SYN scan.
- Snort en consola.
- Opcional: anota `ss -tlnp` *antes* del scan (qué hay escuchando) para contrastar con el informe de nmap.

### En Kali

Escaneo SYN hacia la víctima del lab (el tipo que el enunciado asocia a nmap “stealth”). Añade, si da tiempo, NULL / FIN / XMAS para cubrir `1006002`–`1006004`. El destino es **únicamente** `.20` en Host-Only.

Consulta la documentación de nmap del curso / Kali para la sintaxis de cada tipo de scan (`-sS`, y las variantes NULL/FIN/XMAS). Si el enunciado pide *mapping* de servicios u OS, un scan de versiones (`-sV`) o de SO (`-O`) contra `.20` basta para discutir “qué reveló nmap” en Results; no escanees otras redes.

Haz **un tipo de scan cada vez** y mira qué SID aparece. Si `1006001` no dispara, el umbral del setup (20 SYN / 5 s) puede ser alto para un scan corto: bájalo, recarga Snort y repite, o documenta el ajuste.

### Qué debe coincidir

| Pieza | Qué buscar |
|-------|------------|
| nmap | Puertos abiertos/cerrados; opcionalmente servicio/OS |
| Snort | `1006001` (SYN); si aplicaste los otros scans, `1006002`–`1006004` |
| Wireshark | Muchos SYN a puertos distintos, o flags NULL/FIN/XMAS según el scan |

**Evidencia:** salida de nmap, alerta(s) Snort por tipo de scan, PCAP o screenshot. En Results: tabla *scan → SID → ¿detectado?* y una nota de falso positivo (un nmap de administrador se vería igual).

---

## Qué llevar al reporte y al video

- **Methodology:** sensor en la víctima (no SPAN); `HOME_NET` `192.168.56.0/24`; traceroute de 1 hop.
- **Results:** una subsección por ICMP, traceroute y nmap. Cita cada figura en el texto.
- **Video (inglés):** un take corto por prueba (Kali + consola Snort). Esta actividad suele ir en el bloque de recon del guion (guía del curso, sección 9).

Nombres de archivo útiles:

```text
Act1/Evidencias/<nombre>/
  reglas_recon.txt
  baseline_snort.png
  icmp_snort.png
  icmp_wireshark.pcap
  icmp_kali.png
  traceroute_kali.png
  traceroute_snort.png
  traceroute_wireshark.pcap
  nmap_syn_kali.png
  nmap_syn_snort.png
  nmap_wireshark.pcap
```

---

## Checklist de evidencias

Marca al cerrar la actividad. Cada ítem debe poder pegarse en el PDF o mostrarse en el video.

**Común**

- [ ] Texto de las reglas usadas (SIDs `1001xxx`, `1005xxx`, `1006xxx`)
- [ ] Snort escuchando la NIC Host-Only, sin error al cargar reglas

**Ping / ICMP**

- [ ] Screenshot de alerta `1001001` (y `1001002` si se obtuvo)
- [ ] PCAP o screenshot Wireshark filtro `icmp`
- [ ] Screenshot de `ping` en Kali
- [ ] Frase: por qué matcheó (`itype:8` → `$HOME_NET`)

**Traceroute**

- [ ] Salida del comando (se ve 1 hop)
- [ ] Screenshot de alerta `1005001` y/o `1005002`
- [ ] PCAP o screenshot (`icmp or udp.port >= 33434`)
- [ ] Nota en Results/Methodology sobre el path corto

**Nmap**

- [ ] Salida de al menos un SYN scan a `192.168.56.20`
- [ ] Screenshot de alerta `1006001` (y de `1006002`–`1006004` si los corriste)
- [ ] PCAP o screenshot del scan
- [ ] Qué reveló nmap (puertos / servicio / OS) vs qué detectó Snort

**Cierre**

- [ ] Archivos en `Act1/Evidencias/<nombre>/`
- [ ] Nada de tráfico fuera de `192.168.56.0/24`

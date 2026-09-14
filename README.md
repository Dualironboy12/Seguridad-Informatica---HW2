# HW2 — IDS basado en reglas con Snort

Repositorio de laboratorio del equipo para **LIS4062 Information Security (Autumn 2026)**  
Homework No. 2: *Design of an IDS based on Rules Using SNORT*  
**Profesor:** Dr. Vicente Alarcón Aquino  
**Entrega:** lunes 21 de septiembre de 2026, 11:59 PM (Blackboard; cada integrante sube el PDF)

Este repo **no sustituye** al enunciado ni a Blackboard. Organiza el trabajo: enunciado, guía del homework, setup del lab, procedimientos de las tres actividades y carpetas de evidencia.

---

## Objetivo

Diseñar un **IDS con Snort** que **detecte y alerte** (no bloquea: no es IPS) al menos estos tipos de tráfico, en un laboratorio aislado:

| # | Tipo (enunciado) | Dónde se practica en este repo |
|---|------------------|--------------------------------|
| 1 | Ping / ICMP | [Actividad 1](Act1/Simulacion/README.md) |
| 2 | DoS / DDoS (TCP SYN Flood) | [Actividad 2](Act2/Simulacion/README.md) |
| 3 | ARP spoofing | [Actividad 3](Act3/Simulacion/README.md) |
| 4 | DNS poisoning | [Actividad 3](Act3/Simulacion/README.md) |
| 5 | Traceroute | [Actividad 1](Act1/Simulacion/README.md) |
| 6 | Nmap | [Actividad 1](Act1/Simulacion/README.md) |

**Entregables oficiales** (detalle en la [guía](Guia_HW2_IDS_Snort_LIS4062.md), secciones 2, 9–11):

1. Reporte PDF por integrante, mismo contenido: `HW2_ID_ID_ID.pdf` (portada, Abstract → Bibliography, URL del video).
2. Video en **inglés**, cámara encendida, participación de **todos**.
3. Subida individual al módulo *Self-Autonomous Learning Activities*. Sin subida individual → 0.0; no hay entregas tardías.

El trabajo de lab es en equipo (máximo 3). Elegid **un** escenario: simulación (1 laptop, 2 VMs) **o** práctica (3 laptops). No hace falta montar ambos.

---

## Cómo navegar (orden recomendado)

```text
1. Enunciado PDF          qué pide el curso
2. Guía detallada         reporte, reglas, rúbrica, video
3. Setup/README           elegir escenario A o B
4. Setup/<escenario>      armar red + Snort + herramientas
5. Act1 → Act2 → Act3     ejecutar pruebas y guardar evidencia
6. Guía §§ 9–11           empaquetar PDF + video + checklist de entrega
```

| Paso | Abrir | Para qué |
|------|--------|----------|
| 1 | [`HW2 LIS4062 Autumn 26.pdf`](HW2%20LIS4062%20Autumn%2026.pdf) | Enunciado: procedimientos 1 (SYN flood), 2 (ARP/DNS), 3 (nmap/ping/traceroute), referencias, fecha. |
| 2 | [`Guia_HW2_IDS_Snort_LIS4062.md`](Guia_HW2_IDS_Snort_LIS4062.md) | Guía completa: conceptos, topologías A/B, reglas Snort, plan de pruebas, video, rúbrica. |
| 3 | [`Setup/README.md`](Setup/README.md) | Índice de setup: escenario A vs B y si hacéis todo el setup primero o por actividad. |
| 4a | [`Setup/Simulacion/README.md`](Setup/Simulacion/README.md) | **Escenario A:** 1 laptop, VirtualBox, Kali `.10` + Ubuntu `.20`, host `.1`. |
| 4b | [`Setup/Practica/README.md`](Setup/Practica/README.md) | **Escenario B:** 3 laptops, IDS como gateway. |
| 5 | [`Act1/`](Act1/Simulacion/README.md) [`Act2/`](Act2/Simulacion/README.md) [`Act3/`](Act3/Simulacion/README.md) | Cómo **realizar** cada bloque (simulación). Evidencias en `ActN/Evidencias/<nombre>/`. |

Las actividades de simulación asumen el **setup común** de [`Setup/Simulacion/README.md`](Setup/Simulacion/README.md) ya hecho (y el específico S1 / S2 / S3 de esa actividad).

Aún no hay `ActN/Practica/README.md`. Si el equipo elige el escenario B, usad el setup de práctica y las fases de la guía principal (sección 8) hasta que existan esas guías.

---

## Mapa del repositorio

```text
GHRepo/
├── README.md                          ← este archivo
├── HW2 LIS4062 Autumn 26.pdf          ← enunciado del curso
├── Guia_HW2_IDS_Snort_LIS4062.md      ← guía detallada del homework
├── Setup/
│   ├── README.md                      ← elegir simulación o práctica
│   ├── Simulacion/README.md           ← lab en 1 laptop (2 VMs)
│   └── Practica/README.md             ← lab en 3 laptops
├── Act1/                              ← reconocimiento (ICMP, traceroute, nmap)
│   ├── Simulacion/README.md
│   └── Evidencias/                    ← capturas por integrante
├── Act2/                              ← TCP SYN Flood
│   └── Simulacion/README.md
└── Act3/                              ← ARP + DNS
    └── Simulacion/README.md
```

Poned evidencias de cada prueba en `ActN/Evidencias/<nombre>/` (Act1 ya tiene carpetas de integrantes). El setup sugiere también `Setup/Evidencias/` para screenshots de instalación.

---

## Las tres actividades

| Carpeta | Contenido | Setup asociado (simulación) |
|---------|-----------|-----------------------------|
| **Act1** | Ping/ICMP, traceroute (1 hop en VMs), nmap (`-sS`, NULL/FIN/XMAS, `-sV`/`-O`) | C1–C9 + S1 (reglas `1001xxx` / `1005xxx` / `1006xxx`) |
| **Act2** | SYN flood: HTTP en la víctima, Snort `100200x`, `ss`/`netstat`; hping3/Scapy instalados en setup; ética y enlaces de documentación | C8.2 + S2 (servicio :80 + reglas SYN) |
| **Act3** | ARP respecto al gateway host `.1` y DNS de prueba; Wireshark + `ip neigh`; restaurar la red | S3 (promiscuo, Ettercap/Bettercap, reglas ARP/DNS) |

Cada README de actividad incluye índice, resumen (qué / para qué / entrega), pasos, evidencias en el momento y checklist final.

---

## Dos ritmos de trabajo

Documentados en [`Setup/README.md`](Setup/README.md):

1. **Setup completo** (común + S1+S2+S3) y luego las tres actividades seguidas — útil para grabar el video de una sentada.
2. **Por bloques:** setup común → setup Act1 → Act1 → setup Act2 → Act2 → setup Act3 → Act3 — útil si el lab es corto.

El setup común (red, IPs, Snort, Wireshark) no se omite en ningún caso.

---

## Dónde está cada tipo de información

| Necesitás… | Archivo |
|------------|---------|
| Qué califica el profesor / secciones del PDF | [`Guia_HW2_IDS_Snort_LIS4062.md`](Guia_HW2_IDS_Snort_LIS4062.md) §§ 2, 9–11 |
| Conceptos IDS vs IPS, SYN flood, ARP/DNS | Guía §§ 3 |
| Texto de reglas Snort | Guía § 7 y setup específico S1/S2/S3 |
| Armar VMs o las 3 laptops | [`Setup/`](Setup/README.md) |
| Comandos de nmap del lab | [`Act1/Simulacion/README.md`](Act1/Simulacion/README.md) § 4 |
| Documentación hping3 / Scapy / RUDY (sin playbook de flood) | [`Act2/Simulacion/README.md`](Act2/Simulacion/README.md) |
| Evidencia ARP/DNS y restauración | [`Act3/Simulacion/README.md`](Act3/Simulacion/README.md) |
| Referencias [1–11] | PDF del enunciado y guía § 14 |

---

## Ética

Laboratorio **aislado** (Host-Only o AP sin WAN). Tráfico de prueba solo a las IPs del equipo. Dominios DNS de **prueba**. Cortad floods e MITM al tener evidencia. No se documentan en este repo procedimientos para DoS o spoofing fuera de lo que piden detección, evidencias y enlaces a documentación oficial / HW1.

---

## Criterio rápido de “está listo para entregar”

Usad la rúbrica de la guía (sección 11). Mínimo técnico: evidencia (regla + alerta Snort + PCAP o captura) de ICMP, SYN flood, ARP, DNS, traceroute y nmap; URL del video en el PDF; cada integrante sube el archivo a Blackboard.

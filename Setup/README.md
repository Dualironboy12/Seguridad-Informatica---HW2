# Setup HW2 — IDS basado en reglas con Snort

Esta carpeta concentra **solo el setup** del laboratorio. Las pruebas, las reglas justificadas y la evidencia se ejecutan con la guía principal:

[`Guia_HW2_IDS_Snort_LIS4062.md`](../Guia_HW2_IDS_Snort_LIS4062.md)

Elige **exactamente un escenario**. No hace falta montar ambos.

| Situación del equipo | Escenario | Documento de setup |
|----------------------|-----------|--------------------|
| Una sola laptop | **A — Simulación** (VirtualBox, 2 VMs) | [`Simulacion/README.md`](Simulacion/README.md) |
| Tres laptops en red aislada | **B — Práctica** (Attacker, Victim, IDS/gateway) | [`Practica/README.md`](Practica/README.md) |

---

## Las 3 actividades del laboratorio

El enunciado pide detectar seis tipos de tráfico. En el lab se agrupan en **tres actividades**, porque comparten red, sensor y evidencia:

| Actividad | Qué cubre (tabla de la guía, sección 1) | Fases de prueba (guía, sección 8) |
|-----------|-----------------------------------------|-----------------------------------|
| **1. Reconocimiento** | Ping/ICMP, traceroute, nmap | Fase 1, Fase 4, Fase 5 |
| **2. Denegación de servicio** | TCP SYN Flood (DoS) | Fase 2 |
| **3. Envenenamiento en LAN** | ARP spoofing + DNS poisoning | Fase 3 |

Cada documento de escenario indica **qué pasos de setup son comunes** y **cuáles son exclusivos** de cada actividad.

---

## Dos formas de trabajar (elige una por escenario)

Los README de `Simulacion/` y `Practica/` están escritos para que el equipo elija el ritmo:

### Opción 1 — Setup completo, luego las 3 actividades

1. Ejecuta el **setup común**.
2. Ejecuta el **setup específico de las actividades 1, 2 y 3**.
3. Verifica la red y Snort (checklist final).
4. Corre las tres actividades seguidas (Fases 0–6 de la guía principal).

Útil si vas a grabar el video en una sola sesión o si varios integrantes se turnan en el mismo entorno ya listo.

### Opción 2 — Setup de una actividad e inmediatamente la actividad

1. Ejecuta el **setup común** (una sola vez).
2. Setup de la **actividad 1** → realiza la actividad 1 → guarda evidencia.
3. Setup de la **actividad 2** → realiza la actividad 2 → guarda evidencia.
4. Setup de la **actividad 3** → realiza la actividad 3 → guarda evidencia.

Útil si quieres validar cada bloque antes de seguir, o si el tiempo de lab es corto.

En ambos casos el **setup común no se salta**: sin VMs/laptops, IPs y Snort instalado no hay actividad que ejecutar.

```text
                    ┌─────────────────────────┐
                    │   Setup común           │
                    │   (red + Snort + roles) │
                    └───────────┬─────────────┘
                                │
              ┌─────────────────┴─────────────────┐
              ▼                                   ▼
   Opción 1: setup 1+2+3              Opción 2: setup 1 → actividad 1
              │                                   │
              ▼                                   ▼
      las 3 actividades              setup 2 → actividad 2
                                                  │
                                                  ▼
                                     setup 3 → actividad 3
```

---

## Qué hay en cada carpeta

```text
Setup/
├── README.md                 ← este índice
├── Simulacion/
│   └── README.md             ← 1 laptop, 2 VMs Host-Only
└── Practica/
    └── README.md             ← 3 laptops, IDS como gateway
```

Cada README de escenario incluye:

- Topología e IPs fijas
- Setup común (máquina, red, Snort, herramientas)
- Tabla “este paso es para la actividad N”
- Ruta de **setup completo**
- Ruta de **setup incremental** (por actividad)
- Verificación y problemas frecuentes
- Enlace a la sección de la guía donde se **ejecuta** la prueba (esto no se duplica aquí)

---

## Relación con la guía original

| Necesitas… | Dónde está |
|------------|------------|
| Armar el laboratorio | Este directorio `Setup/` |
| Conceptos IDS/Snort y contenido del reporte | Guía, secciones 2–3 |
| Texto de reglas y SIDs | Guía, sección 7 (también copiado en el setup de cada actividad) |
| Cómo correr cada prueba y qué evidencias guardar | Guía, sección 8 |
| Video, rúbrica y entrega | Guía, secciones 9–11 |

---

## Ética

Trabaja **solo** en el laboratorio aislado del equipo (VMs Host-Only o red privada sin WAN). No envíes tráfico de prueba a redes del campus, a Internet ni a equipos ajenos.

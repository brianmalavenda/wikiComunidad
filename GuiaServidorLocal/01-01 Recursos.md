
tags: [wiki, homelab, servidor/hardware]
fecha: 2026-11-22
links: [[01b Equipamiento de red]], [[04 Almacenamiento]], [[06 Monitoreo]], [[Velocidad de conexión de internet]], [[Uso de memoria RAM por VMs]], [[Topología red]]

# 01a Recursos para un homelab de hasta 200 usuarios

> [!info] Alcance
> Describe **recursos y requisitos, no productos**: cada componente se define por tamaño, tipo y velocidad, y se explica **por qué**. Con eso se puede elegir cualquier hardware (nuevo o usado) que cumpla. Esta nota unifica "01 Recursos" y "01 Recursos en función de usuarios estimados". Leyenda: ✅ medido · ⚠️ estimado · ❓ por medir.

## Índice de esta nota

- [[#1. Qué significa "200 usuarios"]]
- [[#2. Requisitos por componente]]
  - [[#2.1 Placa madre y plataforma]]
  - [[#2.2 CPU]]
  - [[#2.3 Memoria RAM]]
  - [[#2.4 Almacenamiento]]
  - [[#2.5 Red del servidor y red local]]
  - [[#2.6 Conexión a internet]]
  - [[#2.7 Energía y refrigeración]]
- [[#3. Reparto de recursos por servicio (64 GB)]]
- [[#4. Niveles de escalado]]
- [[#5. Caso real de referencia]]
- [[#6. Prueba de carga]]
- [[#7. Checklist de compra]]
- [[#8. Pendientes]]
- [[#Referencias]]

---

## 1. Qué significa "200 usuarios"

"200 usuarios" son las **personas con cuenta** en las aplicaciones del homelab. Lo que dimensiona el hardware es cuántas usan el sistema **a la vez**.

| Concepto | Valor |
|---|---|
| Usuarios registrados (objetivo máximo) | 200 |
| Simultáneos hoy | ✅ ~20 |
| Simultáneos objetivo | 50 |
| Picos | 100 |
| Tipo de carga | Webs y aplicaciones consultadas desde el navegador |

> [!info] Regla
> Se dimensiona para el **pico** (100), no para el promedio, y después se verifica con una prueba de carga ([[#7. Prueba de carga]]).

---

## 2. Requisitos por componente

### 2.1 Placa madre y plataforma

| Recurso | Por qué se necesita | Nota para picos de 100 usuarios |
|---|---|---|
| **IPMI/BMC** | Gestión fuera de banda: encender/apagar, consola remota, montar ISOs, ver sensores sin monitor | No suma rendimiento pero reduce el MTTR. Sin IPMI: PiKVM/KVM IP + regleta inteligente |
| **ECC (RDIMM/UDIMM)** | Detecta y corrige errores de memoria; RDIMM permite más capacidad y suele ser más barato por GB usado | Verificar placa/CPU/QVL: muchas placas X99 no soportan RDIMM. No sustituye backups |
| **8–16 núcleos** | Alimentan VMs, servicios, BD, cifrado, compresión y backups. Importan IPC, frecuencia y VT-d | Suficiente si la carga está bien repartida; medir |
| **HBA dedicada (modo IT)** | Muchos discos, passthrough a TrueNAS/ZFS, SMART correcto, evita RAID "fake" | No obligatoria con 2–4 discos, muy recomendable para NAS ([[04 Almacenamiento]]) |
| **VT-d/IOMMU, ACS, SR-IOV** | Passthrough de HBA/NIC/GPU a VMs | Debe poder activarse |
| **Slots PCIe suficientes** | HBA, NIC 10 GbE, GPU y NVMe compiten por líneas | ≥ 2–3 slots además de M.2 |
| **Bifurcación PCIe** | Partir un x16 en varios x4 para NVMe | Si se prevén varios NVMe |
| **NIC Intel, dual o 10 GbE** | Mejor soporte en Linux/Proxmox que muchas Realtek; dual permite separar gestión | Intel dual como mínimo |
| **SATA/M.2/NVMe y arranque redundante** | El sistema debe arrancar aunque falle un disco | M.2 NVMe para sistema/VMs |
| **Formato, ventilación, consumo idle** | 24/7: debe caber, refrigerarse y no disparar la factura | Medir consumo en reposo |
| **BIOS actualizable y QVL** | Algunas CPU/RAM necesitan BIOS concreta | Dual BIOS si es posible |
| **PSU de calidad + UPS** | Un corte puede corromper VMs o ZFS | PSU con margen; UPS con apagado ordenado |
| **Compatibilidad Linux/Proxmox/TrueNAS** | Drivers de NIC, HBA y sensores | Verificar antes de comprar |
| **Sensores y control de ventiladores** | Ruido y temperatura bajo control | Lectura y PWM aceptables en Linux |
| **Documentación y reproducibilidad** | Una placa "modelo" debe poder replicarse | Misma revisión de placa, BIOS, CPU y RAM |

### 2.2 CPU

| Parámetro | Requisito | Por qué |
|---|---|---|
| Arquitectura | x86-64 con VT-x/AMD-V y EPT/NPT | Sin esto la virtualización es lenta o no arranca |
| IOMMU (VT-d/AMD-Vi) | Obligatorio si hay passthrough | Aislamiento seguro de dispositivos |
| Núcleos físicos | **8** (≤ 25 usuarios) · **12–16** (objetivo) | Cada VM necesita vCPU, el host 1–2, y los backups dan ráfagas de 2–4 |
| Frecuencia | **≥ 2,5 GHz sostenidos**, turbo ~3,3 GHz o más | Las peticiones web (PHP, Node) usan **un solo hilo**: la latencia depende de la frecuencia, no del número de núcleos |
| Hilos (SMT) | ~25–30 % extra | No contarlos al 100 % |
| AES-NI | Obligatorio | TLS y cifrado de discos por hardware |
| TDP | El más bajo que cumpla | 24/7: cada vatio sostenido ≈ 8,76 kWh/año |

> [!tip] Sobreasignación (*overcommit*)
> En CPU es aceptable **2:1 a 3:1** de vCPU por núcleo físico para cargas web. **En RAM no se sobreasigna.**

### 2.3 Memoria RAM

| Parámetro      | Requisito                                                                   | Por qué                                                                                                                                    |
| -------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Capacidad      | **32 GB** (≤ 25 usuarios) · **64 GB** (objetivo) · **128 GB** (crecimiento) | Es el recurso que primero se agota. Se suma lo de cada servicio ([[#4. Reparto de recursos por servicio (64 GB)]]) y se deja 20–30 % libre |
| Tipo           | **ECC**                                                                     | Errores silenciosos pueden corromper archivos o BD. Pesa más con ZFS                                                                       |
| Velocidad      | La que soporte la plataforma                                                | DDR4-2133 vs 3200: diferencia < 5 % en cargas web                                                                                          |
| Canales        | Ocuparlos **todos**                                                         | Con la mitad se pierde la mitad del ancho de banda                                                                                         |
| Módulos        | Idénticos                                                                   | Mezclar fuerza la velocidad más lenta                                                                                                      |
| Ranuras libres | Al menos la mitad                                                           | Permite crecer                                                                                                                             |
| Compatibilidad | RDIMM o UDIMM según manual; **no se mezclan**                               | Un módulo incompatible no arranca                                                                                                          |

### 2.4 Almacenamiento

Se separa en tres funciones con necesidades opuestas. La configuración (RAID, ZFS, aplicaciones) está en [[04 Almacenamiento]].

**Disco de sistema y VMs - rápido y de baja latencia**

| Parámetro | Requisito | Por qué |
|---|---|---|
| Tipo | **NVMe** (SSD SATA como mínimo) | Miles de lecturas/escrituras pequeñas y aleatorias: HDD ~100–200 IOPS, SATA ~50.000+, NVMe 200.000+ |
| Capacidad | **~1 TB útil** | Discos de las VMs (~300–450 GB) + 40 % libre para snapshots y crecimiento |
| Redundancia | **2 unidades en espejo** | Si falla y no hay espejo, caen todos los servicios |
| Resistencia | **≥ 0,3 DWPD** (≈ 600 TBW por TB) | Logs, BD y snapshots; con copy-on-write la escritura real se multiplica ×3 a ×5 |
| PLP | Deseable | Evita perder escrituras sincrónicas |

**Disco de datos - capacidad y fiabilidad**

| Parámetro | Requisito | Por qué |
|---|---|---|
| Tipo | **HDD CMR** (no SMR) | SMR colapsa en reconstrucciones RAID/ZFS |
| Giro | 5.400–7.200 rpm | Más rpm = más ruido, calor y consumo |
| Capacidad útil | **≥ 2× los datos actuales** | Con espejo la útil es la mitad de la bruta; ZFS no debe superar **80 %** |
| Redundancia | **2 unidades en espejo** | Tolera un disco. **No reemplaza al backup** |
| Rendimiento | ~150–250 MB/s por unidad | Ya satura 1 GbE (~117 MB/s) |
| Clase | Pensados para NAS (24/7) | Vibración y garantía |

**Destino de backups - independiente**

| Parámetro | Requisito | Por qué |
|---|---|---|
| Capacidad | **≥ 500 GB** | Varias versiones y deduplicación |
| Ubicación | **Otro disco físico, mejor otro equipo o sitio** | Un backup en el mismo disco que el dato no es un backup |

> [!warning] Tiempo de copia
> 1 TB a 117 MB/s (1 GbE) ≈ 2,4 h; a ~290 MB/s (2,5 GbE) ≈ 1 h. Ese tiempo define la ventana de backup (fuera del horario pico).

### 2.5 Red del servidor y red local

El detalle de equipos (router, switch, APs, cableado) está en [[01-02 Equipamiento de red]].

| Parámetro | Requisito | Por qué |
|---|---|---|
| Puertos del servidor | **2 × Ethernet ≥ 1 GbE** (ideal 2,5 GbE) | 1 GbE ≈ 940 Mbps útiles; 2,5 GbE ayuda en copias internas; el segundo puerto separa gestión o da redundancia |
| Switch | **Administrable** con VLAN 802.1Q, trunk y ≥ 1 puerto 2,5 GbE | Sin VLAN no se puede aislar redes |
| Router | VPN Multi-WAN, Gigabit, VLAN 802.1Q | Reemplaza la gestión de paquetes del router del ISP, que queda como puente |
| Módulo SFP | Fibra a RJ45 o adaptador RJ45 | Para conectar fibra del ISP al router |
| Cableado | Cat 5e mínimo; **Cat 6** para 2,5 GbE | Cables malos generan errores |

> [!note] Tráfico interno
> Si el proxy y la aplicación están en el **mismo servidor y la misma red virtual**, el tráfico entre ellos no sale por el cable. El enlace físico dimensiona solo el tráfico hacia afuera y hacia el almacenamiento.

### 2.6 Conexión a internet

| Parámetro | Requisito | Por qué |
|---|---|---|
| Velocidad | **500 Mbps–1 Gbps** para 50–100 simultáneos | Navegación ≈ 1–3 Mbps por usuario activo: 100 × 2 ≈ 200 Mbps; + 5 descargas de 50 Mbps ≈ 450 Mbps ⚠️ |
| Simetría | **Subida igual a la bajada** | Lo que importa son las respuestas que salen |
| IP pública | Fija o DDNS, **sin CGNAT**; o túnel (Cloudflare Tunnel, VPN) | Con CGNAT no hay conexiones entrantes |
| Latencia y estabilidad | Fibra con pocos cortes | Mide más la experiencia que el pico |
| Segunda salida | Opcional | Elimina el punto único de falla |

✅ Medido: plan contratado de 940 Mbps ([[Velocidad de conexión de internet]]). ❓ Falta confirmar la velocidad de **subida** real.

### 2.7 Energía y refrigeración

| Parámetro | Requisito | Por qué |
|---|---|---|
| Fuente | Consumo promedio al **30–60 %** de la nominal; **80 Plus Gold+** | Muy sobredimensionada trabaja al 10–15 % con menor eficiencia |
| Conectores | EPS 8 pines y SATA/PCIe suficientes | Error de compra común |
| UPS | Potencia en **W** ≥ **2× el consumo medido**; **onda sinusoidal pura**; línea interactiva u online | Fuentes con PFC activo pueden fallar con onda simulada |
| Autonomía | Para un **apagado ordenado** (5–10 min mínimo) | El UPS es para apagar bien, no para seguir horas |
| Apagado automático | Software tipo **NUT** | Sin él el corte es brusco |
| Refrigeración | Disipador según TDP, flujo frontal-trasero, filtros | CPU < 80 °C, discos 30–45 °C |
| Ruido | Ventiladores grandes y lentos | Suele estar en una vivienda |

**Consumo orientativo ⚠️:** reposo 60–90 W · carga media 100–150 W · pico 200–250 W.

```text
kWh/mes = potencia (kW) × 24 h × 30 días      Ejemplo: 0,1 kW × 720 h = 72 kWh/mes
```

> [!warning] Sugerencia
> Enchufe inteligente con medición entre la pared y el UPS, registrando una semana. Los equipos de red suman ~70–110 W más ([[01-02 Equipamiento de red#11. Energía de los equipos de red]]).

---

## 3. Reparto de recursos por servicio (64 GB)

> [!note] Propuesta. Reemplazá los valores por lo realmente asignado en tu servidor.

| Servicio | Tipo | vCPU | RAM | Por qué |
|---|---|---|---|---|
| Host del hipervisor | Host | - | 3–4 GB | Sistema base y gestión |
| Caché ZFS (ARC) | Host | - | hasta 8 GB (limitada) | Se limita a mano para no consumir la RAM de las VMs |
| Almacenamiento (NAS) | VM | 2 | 8 GB | ZFS necesita RAM para caché y metadatos |
| Nube de archivos | VM | 4 | 6–8 GB | PHP-FPM + BD + Redis. Es el servicio más pesado |
| CMS (WordPress, etc.) | VM o contenedor | 2–4 | 4 GB | Con caché de página reduce mucho la carga |
| Aplicación propia | VM | 2–4 | 5,8 GB asignados | ✅ **Medido: 1,3 GB de uso real** |
| Monitoreo (métricas y logs) | VM | 2 | 3–4 GB | Los logs crecen: definir retención |
| Servidor de backups | VM | 2 | 2–4 GB | La deduplicación consume RAM |
| Proxy inverso + DNS + salto de administración | LXC/VM | 1 c/u | 0,5–1 GB c/u | Livianos |
| **Total asignado** | | | **≈ 41–49 GB** | |
| **Libre** | | | **≈ 15–23 GB** | Margen para picos y crecimiento |

> [!warning] Corrección respecto a la nota original
> La nota original indicaba un total de 35–45 GB y 19–29 GB libres. Sumando las filas de la tabla el total da ≈ 41–49 GB, por lo que el margen libre baja a ≈ 15–23 GB (23–36 % de 64 GB: sigue dentro del 20–30 % recomendado). Además la guía de TrueNAS recomienda **16 GB** para la VM NAS ([[04 Almacenamiento#3.2 Recursos de la VM]]) frente a los 8 GB de esta tabla: si se sube a 16 GB el margen libre queda ajustado.

---

## 4. Niveles de escalado

| Nivel | Simultáneos | CPU | RAM | Disco VMs | Internet |
|---|---|---|---|---|---|
| **A: arranque** | hasta ~25 | 8 núcleos | 32 GB ECC | 500 GB–1 TB SSD | 300 Mbps simétrico |
| **B: objetivo** | 50 (picos 100) | 12–16 núcleos | **64 GB ECC** | 1 TB NVMe en espejo | 500 Mbps–1 Gbps simétrico |
| **C: crecimiento** | 100+ sostenidos | 2.º nodo o ≥ 24 núcleos | 128 GB ECC | 2 TB NVMe en espejo | 1 Gbps simétrico |

> [!warning] Los niveles B y C son estimaciones. Se confirman con la [[#7. Prueba de carga]].

---

## 5. Caso real de referencia

Configuración del autor, solo como contraste (no es una recomendación):

| Recurso | Valor |
|---|---|
| CPU | 14 núcleos / 28 hilos, 2,4–3,3 GHz, TDP 120 W |
| RAM | 32 GB ECC (en ampliación a 64 GB) |
| Disco VMs | NVMe ~1 TB (una sola unidad: la recomendación de esta nota es espejo) |
| Disco datos | 2 × HDD 4 TB en espejo |
| Fuente | 750 W (sobredimensionada) |
| Internet | 940 Mbps contratados |
| Usuarios | ✅ ~20 simultáneos |
| Consumo | ⚠️ ~100 W estimado, sin medir |

---
## 6. Checklist de compra

- [ ] El CPU soporta **virtualización, IOMMU y AES-NI**
- [ ] Placa y CPU soportan **ECC** y el tipo de módulo (RDIMM/UDIMM) coincide
- [ ] **≥ 4 ranuras de RAM** y se pueden ocupar **todos los canales**
- [ ] Discos de datos **CMR** (no SMR) y clase NAS
- [ ] Disco de VMs con **≥ 0,3 DWPD**
- [ ] **Puertos** SATA/M.2 suficientes para sistema, datos y backup
- [ ] **2 puertos de red** con chip bien soportado
- [ ] Switch **administrable con VLAN**
- [ ] Fuente con **conectores y eficiencia** adecuados
- [ ] UPS de **onda sinusoidal pura** con apagado automático
- [ ] **GPU básica temporal** si la plataforma no trae video
- [ ] Disipador y gabinete **a la medida del TDP**

> [!warning] Hardware usado
> Un solo servidor es un **punto único de falla**. Es aceptable para este tamaño, pero obliga a tener **backups probados** y un plan de reposición (pregunta 5).

---

## 7. Pendientes

- [ ] Completar la tabla de [[#4. Reparto de recursos por servicio (64 GB)]] con lo realmente asignado
- [ ] Medir el consumo real durante una semana
- [ ] Ejecutar la prueba de carga y pegar las capturas del monitoreo
- [ ] Ampliar la RAM a 64 GB y registrar el antes y después
- [ ] Confirmar la velocidad de **subida** real del enlace
- [ ] Reconciliar los Mbps de `[[01_sanmartin701]]` (300–500) con [[Velocidad de conexión de internet]] (750–1000)

## Referencias

`[[01_sanmartin701]]` · [[Velocidad de conexión de internet]] · [[Uso de memoria RAM por VMs]] · [[Topología red]]

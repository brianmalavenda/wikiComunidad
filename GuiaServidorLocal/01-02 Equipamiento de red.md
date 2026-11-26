
tags: [wiki, homelab, servidor/red, red]
fecha: 2026-11-22
links: [[01a Recursos]], [[Topología red]], [[Configuracion VLAN - GUIA]], [[Configurar ONT - Router - AccesPoint]], [[Configuracion de red]], [[Port Mirroring]]

# 01b Equipamiento de red necesario

> [!info] Alcance
> Complementa [[01-01 Recursos]]. Define **qué equipos de red hacen falta, con qué características y por qué**, cuántas bocas necesita el switch y qué cables y módulos usar. La **configuración** (VLAN, firewall, VPN) va en la pregunta 2 (pendiente). Los ejemplos de familias son orientativos: verificá siempre la ficha técnica vigente. Esta nota unifica "01 Equipamiento" y "01 Equipamiento de red" (eran versiones casi idénticas). Leyenda: ✅ medido · ⚠️ estimado · ❓ por medir.

## Índice de esta nota

- [[#1. Escenario de dimensionamiento]]
- [[#2. Mapa de equipos]]
- [[#3. Router y firewall]]
- [[#4. Switch]]
- [[#5. Cantidad de bocas del switch]]
- [[#6. Puntos de acceso (APs)]]
- [[#7. Otros equipos de administración de la LAN]]
- [[#8. Servidor de backup en la red]]
- [[#9. Cableado de cobre]]
- [[#10. Fibra óptica y módulos SFP]]
- [[#11. Energía de los equipos de red]]
- [[#12. Checklist de compra de red]]
- [[#13. Pendientes]]
- [[#Referencias]]

---

## 1. Escenario de dimensionamiento

| Dato | Valor supuesto |
|---|---|
| Superficie | **50 m² por piso × 3 pisos** = 150 m² |
| Servidores | 1 principal + 1 de backup |
| Administración | 1 PC administradora, **separada del servidor** |
| Acceso Wi-Fi | Personal de la oficina (decenas de dispositivos; los 200 usuarios entran por internet) |
| Rack | Un único punto central, con recorrido de cable a cada piso |

> Si la superficie real es de 50 m² en total, el cálculo baja a 1 AP y el switch puede ser más chico.

## 2. Mapa de equipos

```text
Internet ─► [ONT / módem del ISP, en modo bridge]
                │
           [ROUTER / FIREWALL propio]  ◄── VPN, VLAN, firewall, DHCP, DNS
                │  (trunk 802.1Q)
           [SWITCH CORE administrable, PoE]
     ┌──────┬──────┬────────┬─────────┬────────┐
 [Servidor] [Servidor] [PC admin] [AP piso 1] [AP piso 2] [AP piso 3]
 principal   backup
```

> El diagrama gráfico original está en la imagen `Pasted image 20261120092438.png` de la nota "01 Equipamiento" (no se puede incluir desde el repositorio). Reinsertala aquí si querés conservarla.

Equipos mínimos: **router, switch, 3 APs, UPS, cableado estructurado**. Todo lo demás es opcional.

---

## 3. Router y firewall

### Por qué un router propio, independiente del ISP

| Motivo | Explicación |
|---|---|
| **Control** | El equipo del ISP suele tener firmware cerrado, sin VLAN ni firewall por zona, y el proveedor puede cambiarlo |
| **VLAN y segmentación** | Sin un router que enrute entre VLAN con reglas, no hay aislamiento |
| **VPN propia** | Acceso remoto cifrado sin depender de terceros |
| **Visibilidad** | Logs y reglas auditables |
| **Reemplazo simple** | Si el ISP cambia el equipo, tu configuración no se pierde |

El equipo del ISP queda en **modo *bridge* (solo ONT)** y tu router obtiene la IP pública, normalmente con PPPoE y, si el ISP lo exige, **etiqueta VLAN en el puerto WAN** (ver [[Configuracion de red]]).

### Requisitos

| Característica | Requisito | Por qué |
|---|---|---|
| Rendimiento NAT/firewall | ≥ **900 Mbps reales** con reglas activas (si el plan supera 1 Gbps, chequear la ficha) | Un router a la mitad de la velocidad contratada es el cuello de botella |
| Puertos | ≥ 1 WAN + 4 LAN; ideal **2,5 GbE o SFP+** hacia el switch | Ese enlace concentra el tráfico entre VLAN |
| VLAN 802.1Q e inter-VLAN | Obligatorio | Base de la segmentación |
| Firewall con estado por VLAN | Reglas por origen/destino/puerto, *default deny* | Define quién habla con quién |
| Servidor VPN | **WireGuard** (preferido) y/u OpenVPN/IPsec, ≥ 100–300 Mbps | Más liviano; el rendimiento VPN suele ser mucho menor que el de NAT |
| DHCP por VLAN con reservas | Obligatorio | IP fija para servidores y equipos de red |
| DNS personalizado / DDNS | Obligatorio | Apuntar a AdGuard y mantener el dominio si cambia la IP |
| Doble WAN (failover) | Deseable | Evita quedar aislado |
| Gestión local | HTTPS/SSH desde la VLAN de gestión, **sin depender de la nube** | Si falla internet seguís administrando |
| Respaldo de configuración | Exportar/importar | Reponer el equipo en minutos |
| Actualizaciones de seguridad | Fabricante con historial de parches | Un router desactualizado es la puerta de entrada |
| Syslog | Exportable al monitoreo | Base para [[06 Monitoreo]] |

### Dos caminos

| Opción | Ventajas | Desventajas | Cuándo |
|---|---|---|---|
| **A. Router/firewall dedicado** (pequeña empresa) | Bajo consumo (5–15 W), simple, estable | Menos flexible | **Recomendado para empezar** |
| **B. Router por software** (pfSense, OPNsense, OpenWrt) en mini PC con 2–4 puertos | Flexibilidad total, IDS/IPS, alto rendimiento | Más mantenimiento y consumo (10–25 W) | Si querés IDS/IPS y control fino |

> [!warning] No virtualizar el router dentro del servidor principal
> Si el router vive en una VM del servidor, el servidor depende de la red para arrancar y la red depende del servidor: una caída o reinicio deja todo sin internet y sin forma de administrar. El router va en **hardware propio**.

**Recomendación:** opción A. Familias que suelen cumplirlo (⚠️ verificar): routers VPN/Omada de TP-Link (el que usa el autor), MikroTik, UniFi. Para la opción B: OPNsense o pfSense sobre un mini PC con chips de red bien soportados.

---

## 4. Switch

Un switch no administrable no distingue VLAN ni permite aislar puertos, aplicar ACL ni espejar tráfico: para esta arquitectura es **obligatorio uno administrable**.

| Característica | Requisito | Por qué |
|---|---|---|
| VLAN 802.1Q (access/trunk, PVID) | Obligatorio | Núcleo de la segmentación ([[Configuracion VLAN - GUIA]]) |
| STP/RSTP | Obligatorio | Evita bucles |
| ACL | Deseable | Filtros por puerto |
| Port mirroring | Recomendado | Capturar tráfico con Wireshark ([[Port Mirroring]]) |
| DHCP snooping + inspección ARP | Recomendado | Bloquea DHCP falsos y suplantación ARP |
| LACP | Deseable | Agregar enlaces |
| Gestión segura | HTTPS/SSH, VLAN de gestión propia, SNMP/syslog | La administración **no** va en la red de usuarios |
| PoE+ (802.3at, 30 W/puerto) | Para los APs | Evita fuentes y tomas en el techo |
| Presupuesto PoE total | ≥ suma de APs × 1,3 (⚠️ 3 × ~15 W ≈ 45 W → **≥ 60–90 W**) | Evita que se apaguen APs |
| Conmutación | *Non-blocking* (puertos × velocidad × 2) | Sin cuello de botella interno |
| Uplinks | ≥ 1–2 **SFP+ (10G)** | Segundo switch, backup o fibra futura |
| Velocidad | 1 GbE de acceso; **2,5 GbE** en servidores | 2,5 GbE reduce la copia de 1 TB de ~2,4 h a ~1 h |
| Ventilación | Sin ventilador en zona habitada | Menos ruido (limita el PoE) |

| Topología | Cuándo conviene |
|---|---|
| **Un switch core de 24 puertos con PoE** (recomendado) | Cada AP con cable directo desde el rack (≤ 90 m) |
| Core + switch chico (8 PoE) por piso | Muchos equipos por piso o para reducir cables |
| Dos switches core | Si se necesita redundancia (más complejo) |

**Recomendación:** **1 switch administrable de 24 puertos**, **≥ 8 PoE+**, **≥ 6 de 2,5 GbE**, **2 SFP+**.

---

## 5. Cantidad de bocas del switch

⚠️ Estimación para el escenario (150 m², 3 pisos, 2 servidores).

| Dispositivo | Cant. | Velocidad | PoE | VLAN |
|---|---|---|---|---|
| Uplink al router | 1 | 2,5 G / SFP+ | No | Trunk |
| Servidor principal (datos) | 1 | 2,5 G | No | Trunk |
| Servidor principal (gestión remota BMC) | 1 | 1 G | No | Administración |
| Servidor de backup (datos) | 1 | 2,5 G | No | Trunk (backup) |
| Servidor de backup (gestión remota BMC) | 1 | 1 G | No | Administración |
| PC administradora | 1 | 1 G | No | Administración |
| AP piso 1, 2 y 3 | 3 | 1 G | **Sí (PoE+)** | Gestión + VLAN por SSID |
| UPS (tarjeta de red/NUT) | 1 | 1 G | No | Administración |
| KVM por IP / enchufe inteligente | 1 | 1 G | No | Administración |
| Puerto de espejo para análisis | 1 | 1 G | No | Libre (mirror) |
| Equipos opcionales (impresora, IoT, cámara) | 2 | 1 G | A veces | Según caso |
| **Subtotal en uso** | **14** | | | |
| **Reserva 25–40 %** | **+6** | | | |
| **Total recomendado** | **≈ 20 → switch de 24 puertos** | | | |

Sumá además **2 puertos SFP+** (no cuentan en los 24 de cobre).

> [!question] Respuesta a tu anotación: ¿por qué el servidor de backup tiene un puerto de datos y otro de gestión remota?
> *(Pregunta que dejaste marcada en la nota original; la respuesta es una incorporación de esta síntesis, no estaba en tus notas.)*
> Son **dos interfaces con funciones y VLAN distintas**. El **BMC/IPMI** es un controlador independiente con su propia NIC e IP: funciona aunque el servidor esté apagado o colgado, y permite encenderlo, ver consola y montar ISOs, por eso va en la **VLAN de administración** (solo accesible desde la PC administradora). El **puerto de datos** recibe las copias y va en la **VLAN de backups**, que solo acepta conexiones del servidor principal. Si compartieran puerto o VLAN, un atacante que llegue a la red de backups alcanzaría también la gestión del equipo, o al revés. Si el servidor de backup no tiene BMC, ese puerto se elimina (-1 boca).

---

## 6. Puntos de acceso (APs)

| Criterio | Cálculo | Resultado |
|---|---|---|
| Cobertura | Un AP de interior cubre ~60–100 m² efectivos con paredes | 50 m² por piso → **1 AP por piso** |
| Capacidad | ~25–40 clientes simultáneos por AP | ≤ 30 dispositivos por piso alcanza |
| Pisos | Una losa de hormigón atenúa 15–25 dB | **1 AP por piso, en el techo, al centro** |

⚠️ **Total: 3 APs.** Si un piso supera 40 dispositivos o tiene muros gruesos, sumá uno.

| Característica | Requisito | Por qué |
|---|---|---|
| Estándar | Wi-Fi 6 (802.11ax) o superior | Mejor manejo de muchos clientes |
| Alimentación | **PoE** (802.3af/at) | Un cable de datos y energía |
| VLAN por SSID | Obligatorio | Separa *Hogar/Oficina* e *Invitados* |
| Aislamiento de clientes | En la red de invitados | Los invitados no se ven entre sí |
| Seguridad | WPA3 (o WPA2/WPA3); gestión sin nube obligatoria | Cifrado actual |
| Conexión | **Cableada (sin mesh inalámbrico)** | El mesh pierde velocidad en cada salto |
| Gestión | Controlador local (VM/contenedor) o APs autónomos | El controlador consume 1–2 GB de RAM si es VM |
| Canales | 2,4 GHz: 1/6/11; 5 GHz: 40 MHz sin solapamiento; **bajar la potencia** | Evita interferencia entre pisos |

---

## 7. Otros equipos de administración de la LAN

| Equipo | Función | Conexión |
|---|---|---|
| **Router del ISP** | En modo puente (solo ONT) | - |
| **Router administrable** | Control de la red | WAN + LAN |
| **Switch administrable** | Gestiona las VLAN (ver lista abajo) | Trunk |
| **Servidor productivo** | Hospeda las webs y herramientas de gestión | 2,5 G + BMC |
| **PC administradora** (separada del servidor) | Única estación que gestiona Proxmox, switch y router, desde la VLAN de administración | 1 puerto 1 G |
| **Servidor de backup** | Copias independientes del principal | 2,5 G + BMC |
| **BMC/IPMI o KVM por IP** | Controlar el servidor aunque se cuelgue | 1 puerto c/u |
| **UPS con tarjeta de red o NUT** | Apagado ordenado y alertas | 1 puerto |
| **Enchufe inteligente con medición** | Medir consumo y reiniciar equipos | Wi-Fi o cable |
| **Analizador de tráfico** (opcional) | Detección de intrusos en el puerto espejo | 1 puerto espejo |
| **Controlador de APs** | Gestión de los 3 APs | VM/contenedor |
| **Salto de administración (*jump server*)** | Puente seguro hacia las VMs | VM |
| **DNS interno (AdGuard) y NGINX** | Filtrado/resolución local y proxy inverso (ubicados hoy en la VLAN ADMIN) | VM/contenedor |
| **Notebook de emergencia con consola** | Rescate si se pierde el acceso | Directo al equipo |

**VLAN planificadas (6):** 1) INVITADOS · 2) HOGAR · 3) DMZ · 4) ADMIN · 5) APPS · 6) BACKUP.

> [!question] Punto a revisar en la pregunta 2
> Tus notas ubican AdGuard y NGINX en la VLAN ADMIN. AdGuard (DNS interno) encaja ahí; **NGINX**, si publica servicios hacia internet, suele ubicarse en la **DMZ**, para que un compromiso del proxy no tenga alcance directo a la red de administración. Conviene decidirlo al diseñar las reglas de firewall.

> [!tip] Regla
> Todo equipo de red y toda gestión (BMC, switch, router, APs) va en la **VLAN de administración**, nunca en la de usuarios.

---

## 8. Servidor de backup en la red

| Aspecto | Requisito | Por qué |
|---|---|---|
| Ubicación física | **Otro cuarto, mejor otro piso**, con toma y UPS propios | Un incendio, inundación o sobrecarga en el rack no debe afectar a ambos |
| VLAN | Aislada (VLAN de backups), solo recibe conexiones del principal | Si el principal se compromete, el atacante no llega a las copias |
| Enlace | 2,5 GbE | La copia nocturna de ~1 TB baja a ~1 h |
| Recursos ⚠️ | 4 núcleos, 8–16 GB de RAM, discos ≥ 2× los datos protegidos | Menos CPU que el principal; la RAM sirve para deduplicar |
| Copia externa (3-2-1) | Una tercera copia fuera del sitio | Se desarrolla en la pregunta 5 (pendiente) |

---

## 9. Cableado de cobre

| Categoría | Velocidad y distancia | Uso |
|---|---|---|
| Cat 5e | 1 / 2,5 GbE hasta 100 m | Mínimo aceptable |
| **Cat 6** | 1 / 2,5 / 5 GbE hasta 100 m; 10 GbE hasta ~55 m | **Recomendado** para tendidos fijos |
| Cat 6A | 10 GbE hasta 100 m | Si se quiere 10 GbE en cobre |
| Cat 7/8 | No aportan en este escenario | No hace falta |

**Calidad:** cobre puro (no CCA), 23 AWG, en tendidos fijos (el CCA tiene más resistencia y falla con PoE) · U/UTP alcanza en un hogar; F/UTP solo cerca de fuentes de ruido y con buena tierra · LSZH si va por conductos comunes · máximo **90 m fijos + 10 m de patch = 100 m**.

| Concepto ⚠️ (medí el recorrido real) | Cantidad | Detalle |
|---|---|---|
| Tendido fijo rack → AP | 3 cables | Longitud medida + 15 % + 3 m de reserva |
| Tomas (keystone) en cielorraso | 3 | Una por AP |
| Patch panel | 1 de 24 puertos | Ordena y protege el switch |
| Patch cords cortos (0,3–2 m) | ~12 | Router, switch, 2 servidores (datos + BMC), PC admin, UPS, KVM |
| Patch cord ONT → router | 1 | Cat 5e/6 |
| Rack o gabinete pequeño | 1 | Ventilación y organizador |
| Tester de cables | 1 | Verificar cada tendido |
| Etiquetas | Ambos extremos | Evita errores al diagnosticar |

---

## 10. Fibra óptica y módulos SFP

En un hogar de 3 pisos con tendidos < 90 m **alcanza el cobre Cat 6**. La fibra se justifica si: el tendido supera 90–100 m o va entre edificios · querés **10 GbE** sin los problemas térmicos del cobre 10G · necesitás **aislamiento eléctrico** · planificás un switch por piso con enlaces de 10 Gb.

| Tipo de fibra | Distancia 10G | Longitud de onda | Uso |
|---|---|---|---|
| Multimodo OM3/OM4 | 300 m / 400 m | 850 nm | Interno, dentro de un edificio |
| Monomodo OS2 | 10 km o más | 1310 nm | Larga distancia y futuro (25/100 G) |

| Opción | Velocidad | Alcance | Notas |
|---|---|---|---|
| **DAC** (cobre directo con SFP+) | 10 G | ≤ 3–5 m | El más barato y de menor consumo en el mismo rack |
| **AOC** (óptico activo) | 10 G | 3–30 m | Sin módulos separados |
| SFP+ 10GBASE-SR (multimodo, LC) | 10 G | ≤ 300 m (OM3) | Estándar interno |
| SFP+ 10GBASE-LR (monomodo) | 10 G | ≤ 10 km | Largas distancias |
| SFP 1000BASE-SX/LX | 1 G | 550 m / 10 km | Puertos SFP de 1 G |
| SFP+ RJ45 (10GBASE-T) | 10 G | 30–80 m | Se calienta (~2–3 W), exige espacio entre módulos |

**Cuidados:** revisar la lista de módulos soportados (algunos switches solo aceptan "codificados") · ambos extremos del mismo tipo y longitud de onda · radio de curvatura mínimo ~30 mm · tapones de polvo y limpieza de conectores.

---

## 11. Energía de los equipos de red

Los equipos de red van **detrás del mismo UPS** que el servidor: si el router o el switch se apagan, el servidor deja de ser accesible aunque siga encendido.

| Equipo | Consumo orientativo ⚠️ |
|---|---|
| ONT / módem | 5–10 W |
| Router | 5–15 W |
| Switch 24 puertos (sin PoE) | 15–40 W |
| 3 APs por PoE | ~45 W |
| **Total de red** | **~70–110 W** |

Sumalo al consumo del servidor ([[01-01 Recursos#3.7 Energía y refrigeración]]) para dimensionar el UPS.

---

## 12. Checklist de compra de red

- [ ] **Router** propio con VLAN, firewall por zona, VPN WireGuard y **≥ 900 Mbps de NAT real**
- [ ] El equipo del ISP en **modo bridge**
- [ ] **Switch administrable de 24 puertos**, ≥ 8 PoE+, ≥ 6 de 2,5 GbE, 2 SFP+
- [ ] **Presupuesto PoE** ≥ 60–90 W
- [ ] **3 APs** Wi-Fi 6 con PoE y VLAN por SSID
- [ ] **UPS** de onda sinusoidal pura que cubra servidor + red
- [ ] **Cableado Cat 6** de cobre puro hacia cada AP, tomas y patch panel
- [ ] **Patch cords** cortos para el rack
- [ ] **DAC o módulos SFP+** solo si hay enlace de 10 G
- [ ] **Rack/gabinete**, tester y etiquetas
- [ ] **Servidor de backup** en otro cuarto, con su propia alimentación

## 13. Pendientes

- [ ] Medir el recorrido real de cada cable (rack → AP)
- [ ] Confirmar el **rendimiento NAT y VPN** del router elegido en su ficha técnica
- [ ] Confirmar que el firmware del router soporta **WireGuard**
- [ ] Registrar el consumo real del conjunto de equipos de red
- [ ] Decidir si se justifica fibra en la instalación
- [ ] Pasar la configuración de VLAN y firewall a la pregunta 2 (y decidir ubicación de NGINX)

## Referencias

[[01-01 Recursos]] · [[Topología red]] · [[Configuracion VLAN - GUIA]] · [[Configuracion de red]] · [[Port Mirroring]]

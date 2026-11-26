---
tags: [wiki, homelab, indice]
fecha: 2026-11-22
---

# 00 Índice - Wiki guía de Homelab

> [!info] Cómo usar esta wiki
> Guía para armar un homelab sobre Proxmox VE que soporte hasta **200 usuarios registrados** (≈ 50 simultáneos, picos de 100) consultando webs y aplicaciones alojadas en el servidor. Cada nota tiene su propio índice para navegarla. Leyenda usada en toda la wiki: ✅ medido · ⚠️ estimado · ❓ por medir.

## Índice de esta nota

- [[#Mapa de la wiki]]
- [[#Pregunta 1 - Equipamiento]]
- [[#Pregunta 2 - Red segura]]
- [[#Pregunta 3 - Sistema operativo e hipervisor]]
- [[#Pregunta 4 - Almacenamiento]]
- [[#Pregunta 5 - Backups]]
- [[#Pregunta 6 - Monitoreo]]
- [[#Estado de la wiki]]

## Mapa de la wiki

| Orden | Nota | Pregunta que responde |
|---|---|---|
| 00 | [[00 Indice]] | Esta nota |
| 01a | [[01-01 Recursos]] | ¿Qué recursos (CPU, RAM, disco, internet) necesito y cómo los calculo? |
| 01b | [[01-02 Equipamiento de red]] | ¿Qué equipos de red necesito (router, switch, APs, cableado, UPS)? |
| 03 | [[03 Hipervisor]] | ¿Qué sistema operativo uso y por qué Proxmox? |
| 04 | [[04 Almacenamiento]] | ¿Qué almacenamiento necesito y con qué aplicación lo configuro? |
| 06 | [[06 Monitoreo]] | ¿Cómo monitoreo equipos y aplicaciones? |

## Pregunta 1 - Equipamiento

**¿Qué equipamiento necesito para un homelab que escale hasta 200 usuarios consultando webs dentro del servidor?**

1. [[01-01 Recursos]] - define los requisitos por componente con su porqué, y el método para convertir "usuarios" en RAM/CPU/ancho de banda/disco.
2. [[01-02 Equipamiento de red]] - router, switch, APs, cableado, fibra, UPS y servidor de backup en la red.

## Pregunta 2 - Red segura

**¿Qué conexión de red tengo que hacer para que sea seguro? (VLAN, VPN, etc.)**

- ❓ **Sin nota todavía.** Hay material de partida en [[01-02 Equipamiento de red]] (las 6 VLAN, WireGuard, modo bridge del ISP). Las notas enlazadas del vault (`[[Topología red]]`, `[[Configuracion VLAN - GUIA]]`, `[[Configuracion de red]]`, `[[Port Mirroring]]`) no estaban en la carpeta revisada.

## Pregunta 3 - Sistema operativo e hipervisor

**¿Qué sistema operativo uso? Recomendado Proxmox como hipervisor, ¿por qué?**

- [[03 Hipervisor]]

## Pregunta 4 - Almacenamiento

**¿Qué almacenamiento tengo que tener y con qué aplicación lo configuro?**

- [[04 Almacenamiento]] (unifica las antiguas "04 Almacenamiento" y "04 Guia almacenamiento")

## Pregunta 5 - Backups

**¿Cómo gestiono los backups? ¿Con qué aplicación o programa?**

- ❓ **Sin nota todavía.** Material de partida: servidor de backup en la red ([[01-02 Equipamiento de red#8. Servidor de backup en la red]]), disco de backups ([[01-01 Recursos#3.4 Almacenamiento]]) y regla 3-2-1 ([[04 Almacenamiento#8. Mantenimiento y buenas prácticas]]). Las notas del vault `[[Homelab - Backups con PBS]]` no estaban en la carpeta.

## Pregunta 6 - Monitoreo

**¿Cómo realizo el monitoreo de los equipos y de las aplicaciones que corren dentro?**

- [[06 Monitoreo]]

## Estado de la wiki

| Pregunta | Estado | Qué falta |
|---|---|---|
| 1 Equipamiento | 🟢 Casi completa | Mediciones reales (consumo, subida de internet, prueba de carga) |
| 2 Red segura | 🔴 Sin escribir | Toda la nota: VLAN, firewall, WireGuard, DNS |
| 3 Hipervisor | 🟡 Base lista | Verificar versión vigente de Proxmox y scripts |
| 4 Almacenamiento | 🟢 Completa | Alinear con el diseño final de discos |
| 5 Backups | 🔴 Sin escribir | Toda la nota: PBS, 3-2-1, pruebas de restauración |
| 6 Monitoreo | 🟡 Parcial | Alertas, retención, qué monitorear en las apps |

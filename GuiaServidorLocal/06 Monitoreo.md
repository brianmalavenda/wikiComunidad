
tags: [wiki, homelab, monitoreo, grafana, proxmox]
fecha: 2026-11-22
links: [[01a Recursos]], [[03 Hipervisor]], [[04 Almacenamiento]], [[01b Equipamiento de red]]

# 06 Monitoreo y métricas

> [!info] Alcance
> Responde a "¿cómo monitoreo los equipos y las aplicaciones que corren dentro?". Para sostener 50–100 usuarios simultáneos hay que vigilar el hipervisor, los contenedores/VMs y las aplicaciones. **Esta nota combina tu nota "06 Monitoreo" (corta y con un comando cortado) con los umbrales y métodos de medición que ya estaban en [[01-01 Recursos]], [[04 Almacenamiento]] y [[03 Hipervisor]]**; las secciones marcadas ❓ son huecos a completar.

## Índice de esta nota

- [[#1. Qué se monitorea y por qué]]
- [[#2. Métricas nativas de Proxmox hacia InfluxDB y Grafana]]
- [[#3. Grafana y Prometheus en un LXC]]
- [[#4. ProxMenux Monitor]]
- [[#5. Umbrales y métodos de medición]]
- [[#6. Logs y retención]]
- [[#7. Pendientes]]

---

## 1. Qué se monitorea y por qué

| Capa | Qué mirar | Para qué |
|---|---|---|
| Host Proxmox | CPU, RAM disponible, I/O de disco, red, temperatura | Detectar cuellos de botella antes de que afecten a usuarios |
| VMs y LXC | Uso por contenedor/VM | Ajustar reparto de recursos ([[01-01 Recursos#4. Reparto de recursos por servicio (64 GB)]]) |
| Almacenamiento | Estado del pool, SMART, % de ocupación | Ver [[04 Almacenamiento#8. Mantenimiento y buenas prácticas]] |
| Aplicaciones | Workers PHP, tiempos de respuesta, errores HTTP | Validar el dimensionamiento |
| Red | Tráfico de interfaz, logs del router | Detectar saturación o incidentes |

## 2. Métricas nativas de Proxmox hacia InfluxDB y Grafana

Proxmox trae un sistema de métricas externo configurable desde la interfaz:

1. En Proxmox: **Datacenter → Metric Server**.
2. **Add** y elegir el destino, por ejemplo una instancia local o externa de **InfluxDB**.
3. Visualizar en **Grafana** (CPU, I/O de disco, RAM y red por contenedor/VM).

> [!note] Aclaración
> El *Metric Server* nativo envía los datos a InfluxDB (o Graphite). Si preferís **Prometheus**, necesita un exportador adicional (el exportador de Proxmox para Prometheus), cosa que tu nota original no detalla. Verificá en la documentación oficial qué destinos soporta tu versión.

## 3. Grafana y Prometheus en un LXC

Para una visualización avanzada orientada a administradores:

- Desplegar un **contenedor LXC ligero** en Proxmox con **Grafana** y **Prometheus**, o usar los scripts de la comunidad para instalar los dashboards de monitoreo de Proxmox.
- Recursos orientativos para la VM/contenedor de monitoreo: **2 vCPU y 3–4 GB de RAM** ([[01-01 Recursos#4. Reparto de recursos por servicio (64 GB)]]).

> [!warning] Comando incompleto en la nota original
> El comando del script de instalación de Grafana/Prometheus de *community-scripts* quedó **cortado** en tu nota original (terminaba en `.../ct/gr`). Buscá el script correcto en el repositorio de *community-scripts* y revisalo antes de ejecutarlo (ver advertencia en [[03 Hipervisor#3. Primer ingreso y configuración post-instalación]]).

## 4. ProxMenux Monitor

Dashboard web que se instala junto con ProxMenux ([[03 Hipervisor#4. ProxMenux - menú interactivo y dashboard web]]): gráficas en tiempo real de CPU, RAM, discos, tráfico de red y estado de VMs/LXC, más terminal WebSocket. Es la opción más rápida para empezar; Grafana/Prometheus es la opción para historial y alertas.

## 5. Umbrales y métodos de medición

Estos son los valores de aceptación definidos para la prueba de carga ([[01-01 Recursos#7. Prueba de carga]]) y que conviene usar como alertas permanentes:

| Métrica | Umbral aceptable | Si se pasa, el cuello de botella es... |
|---|---|---|
| CPU del host | < 70 % sostenido | CPU |
| RAM disponible (*available*, no "usada") | > 20 % libre | RAM |
| Latencia de disco | < 10 ms | Disco |
| Tráfico de red | < 70 % del enlace | Red o internet |
| Errores HTTP | < 1 % | Aplicación o configuración |
| Tiempo de respuesta p95 | < 1,5 s | Cualquiera de los anteriores |
| Ocupación del pool ZFS | < 80 % | Capacidad (rendimiento cae por encima) |

| Dato | Cómo medirlo |
|---|---|
| Tiempo de respuesta (R) | `$request_time` en el log de nginx |
| Workers PHP ocupados | `pm.status_path` de PHP-FPM ("active processes", "max children reached") |
| RAM por worker | `smem -P php-fpm -k` bajo carga |
| IOPS y latencia de disco | `iostat -x 5` (`r/s`, `w/s`, `await`) |
| Ancho de banda | Monitoreo de la interfaz o del router |

## 6. Logs y retención

- Estimación ⚠️: logs web ≈ **250 MB/día** sin comprimir (~10× menos comprimidos); Prometheus ≈ **57 MB/día** (~1,7 GB por 30 días con 5.000 series a 15 s). **Definir retención** antes de que los logs llenen el disco.
- El router y el switch deben enviar **syslog** al sistema de monitoreo ([[01-02 Equipamiento de red]]).
- Los logs críticos del host no deben vivir solo dentro de una VM del mismo host: usar un **segundo equipo syslog** o copia periódica ([[04 Almacenamiento#4. Carpeta compartida para logs y administración]]).

## 7. Pendientes

- [ ] Elegir y documentar el stack (¿InfluxDB + Grafana o Prometheus + Grafana?)
- [ ] Completar el comando/script de instalación de Grafana/Prometheus
- [ ] ❓ Definir **alertas** (correo/Telegram) para los umbrales de la sección 5
- [ ] ❓ Monitoreo de las aplicaciones (WordPress, Nextcloud, app propia)
- [ ] ❓ Monitoreo de disponibilidad externa (¿el sitio responde desde internet?)
- [ ] Definir retención de logs y métricas
- [ ] Capturas de los dashboards durante la prueba de carga

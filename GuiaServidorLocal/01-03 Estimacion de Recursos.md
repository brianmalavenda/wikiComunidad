
## 1. Método - del usuario al recurso

Convierte "cuántos usuarios" en **RAM, CPU, ancho de banda y disco**. Los valores típicos son un punto de partida ⚠️ que hay que reemplazar por mediciones propias ❓.

### 1.1 Niveles y perfiles de usuario

| Nivel | Definición | Ejemplo | Dimensiona |
|---|---|---|---|
| **Registrados** | Personas con cuenta | 200 | Disco (cuotas), base de datos |
| **Simultáneos (N)** | Con sesión abierta y navegando | 50 (pico 100) | Todo lo demás |
| **Peticiones en curso (L)** | Peticiones que el servidor procesa en este instante | ~5 | RAM de workers y CPU |

Un usuario simultáneo no consume recursos todo el tiempo: lee, piensa y después hace clic. Esa pausa es el **tiempo de reflexión (Z)** y hace que 100 usuarios generen pocas peticiones en curso.

| Perfil | Z (s entre clics) | Peticiones dinámicas por página (d) ⚠️ | Ejemplo |
|---|---|---|---|
| **Lector** | 15–30 | 1 (CMS) a 5–10 (app web) | Lee noticias o documentos |
| **Editor** | 5–10 | 3–15 | Edita, sube archivos, usa formularios |
| **Cliente de sincronización** | 30–60, **continuo** | 1–2 | App de escritorio/móvil de la nube de archivos |

> [!warning] La sincronización cuenta aparte
> Un cliente de sincronización consulta al servidor cada 30–60 s **las 24 h**, aunque el usuario no esté. 50 dispositivos son ~1–2 peticiones/s permanentes.

### 1.2 Usuarios a peticiones y workers

Ley de respuesta interactiva (ley de Little):

```text
X = N / (R + Z)      páginas por segundo
λ = X × d            peticiones dinámicas por segundo
L = λ × R            peticiones en curso (workers ocupados)
```

| Símbolo | Significa | Cómo se obtiene |
|---|---|---|
| **N** | Usuarios simultáneos | Objetivo de diseño (pico) |
| **Z** | Segundos entre clics | Tabla de perfiles o analítica |
| **R** | Tiempo de respuesta de una petición dinámica (s) | ❓ Registro del servidor web |
| **d** | Peticiones dinámicas por página | ❓ Pestaña "Red" del navegador |

⚠️ **Ejemplo (pico, solo lectores con CMS):** N = 100, Z = 10 s, R = 0,5 s, d = 1 → X = 9,5 páginas/s · λ = 9,5 pet/s · **L = 4,8 en curso**.

**Perfiles mezclados** (se suma cada perfil):

| Perfil | N | Z | R | X = N/(R+Z) |
|---|---|---|---|---|
| Lectores | 70 | 20 | 0,5 | 3,4 |
| Editores | 20 | 8 | 0,5 | 2,4 |
| Sincronización | 10 | 45 | 0,5 | 0,2 |
| **Total** | **100** | | | **6,0 páginas/s** |

> [!warning] Corrección respecto a la nota original
> La nota original calculaba el caso "Editores = 60, Lectores = 20, Sincronización = 20" como 17,89 páginas/s. Con la misma fórmula da **≈ 8,5 páginas/s** (7,06 + 0,98 + 0,44). Lo que sí crece mucho con más editores es **d** (peticiones dinámicas por página): λ = X × d, así que conviene recalcular con el d real de cada perfil.

Las peticiones **estáticas** (imágenes, CSS, JS) las sirve el servidor web o la CDN y consumen mucho menos; solo las **dinámicas** (PHP, Node, API) pasan por los workers.

**Cuántos workers configurar.** Con llegadas aleatorias (Poisson), el 99 % del tiempo las peticiones en curso no superan:

```text
workers ≈ L + 2,33 × √L
```

| L (en curso) | Workers recomendados | Margen |
|---|---|---|
| 4,8 | 10 | ×2,1 |
| 20 | 30 | ×1,5 |
| 50 | 67 | ×1,3 |

⚠️ Regla práctica: margen **×2 a ×3** en sistemas chicos, **×1,5** en grandes (cuando el servidor se carga, R sube y L también).

> [!note] Si te quedás corto
> Al agotarse los workers las peticiones hacen cola (lentitud, errores 502/504). Si además falta RAM, el sistema usa *swap* y todo se frena. Fijá el máximo de workers según la RAM disponible y no más.

### 1.3 RAM

```text
RAM de una VM = SO base + (workers × RSS por worker) + buffers de BD
              + caché de aplicación (Redis, OPcache) + reserva de caché de archivos (20–30 %)
```

| Componente ⚠️ | Típico | Depende de |
|---|---|---|
| SO base (Linux) | 0,3–0,5 GB | Servicios instalados |
| Worker PHP (WordPress) | 40–100 MB | Plugins, tamaño de página |
| Worker PHP (Nextcloud) | 80–200 MB | Apps activas, vistas previas |
| Proceso Node o Python | 50–150 MB | La aplicación |
| OPcache (PHP, compartida) | 128–256 MB | Cantidad de código |
| Buffer de base de datos | 25–50 % del tamaño de la BD (mín. 0,5–1 GB) | Conjunto de datos activo |
| Conexión a la BD | 1–10 MB c/u | Buffers por conexión |
| Redis | 128–512 MB | Caché configurado |
| nginx | 10–50 MB | Conexiones abiertas |

**Ejemplo A - VM de CMS (WordPress + BD):** 15 workers × 80 MB = 1,2 GB; + SO 0,5 + OPcache 0,25 + MariaDB 1,0 + Redis 0,25 + nginx 0,1 = **3,3 GB** → +25 % reserva ≈ 4,1 → **asignar 4–5 GB**.

**Ejemplo B - VM de nube de archivos (Nextcloud):** 20 workers × 120 MB = 2,4 GB; + SO 0,5 + OPcache 0,25 + BD 1,5 + Redis 0,5 + vistas previas 1,0 = **6,2 GB** → +25 % ≈ 7,7 → **asignar 8 GB**.

Cómo medir el RSS real **bajo carga** (no en reposo) ❓:

```bash
# Memoria proporcional real (PSS), evita contar dos veces lo compartido
smem -P php-fpm -k

# Aproximación rápida con RSS (sobreestima)
ps --no-headers -o rss -C php-fpm8.2 | awk '{s+=$1;n++} END{print s/n/1024 " MB promedio, " n " procesos"}'
```

> [!note] "Usada" no es "necesaria"
> Linux llena la RAM libre con caché de archivos. Para saber si falta memoria mirá **"disponible" (*available*)** y la actividad de *swap* ([[Uso de memoria RAM por VMs]]).

### 1.4 CPU

```text
núcleos ocupados   = λ × S              (S = segundos de CPU por petición)
núcleos necesarios = núcleos ocupados / utilización objetivo (0,6–0,7)
S = (CPU % de la VM × núcleos de la VM / 100) / peticiones por segundo
```

S **no** es R: R incluye esperas (BD, disco, red); S es solo tiempo de CPU real.

| Petición ⚠️ | S (s de CPU) |
|---|---|
| WordPress **sin** caché de página | 0,2–0,5 |
| WordPress **con** caché de página | 0,01–0,03 |
| Nextcloud (interfaz o WebDAV) | 0,1–0,4 |
| API ligera (Node, Python) | 0,01–0,1 |
| Archivo estático | < 0,001 |

Ejemplo (pico, λ = 9,5 pet/s, utilización 65 %): WordPress sin caché (S = 0,3) → 2,9 núcleos ocupados → **4,4 núcleos**; con caché (S = 0,02) → 0,19 → **0,3 núcleos**.

> [!tip] La caché de página cambia la CPU en un factor de 10 a 15
> Es la optimización con mejor relación esfuerzo/resultado antes de comprar más hardware.

Carga fija que se suma siempre ⚠️: host del hipervisor 0,5–1 · ZFS (checksum/compresión) 0,5–1 · monitoreo 0,3–0,5 · nube con sincronización 0,5–1 · **backups (ráfaga nocturna) 2–4**, que no coinciden con el pico. Los hilos (SMT) aportan ~25–30 %, no un núcleo completo: se dimensiona en **núcleos físicos**.

### 1.5 Ancho de banda

```text
Mbps web   = X × peso de página (MB) × (1 − acierto CDN) × 8
Mbps total = Mbps web + Σ (transferencias grandes simultáneas × Mbps de cada una)
```

| Dato ⚠️ | Típico | Cómo medirlo ❓ |
|---|---|---|
| Peso de página (CMS sin optimizar) | 1–3 MB | DevTools → Red |
| Peso de página optimizada | 0,3–1 MB | Ídem |
| Acierto de CDN | 60–90 % (estático) | Panel de la CDN |
| Descarga o subida grande | 20–100 Mbps c/u | Conexión del cliente |

**Ejemplo (pico):** web 9,5 × 2 MB × 0,3 × 8 = **46 Mbps** con CDN (152 sin CDN) + 5 transferencias × 50 Mbps = 250 → **≈ 300 Mbps**; con margen ×1,5 → **~450 Mbps → contratar 500 Mbps–1 Gbps simétricos**.

Lo que sale del servidor consume la **subida** del ISP: es el valor limitante. Conexiones: 100 usuarios ≈ 600 conexiones, muy por debajo de lo que soportan nginx y un router de pequeña empresa.

### 1.6 Disco

```text
IOPS = λ × (lecturas sin caché + escrituras por petición)
```

⚠️ Ejemplo: 9,5 pet/s × (1 lectura + 2 escrituras) = **~29 IOPS**.

| Disco | IOPS aprox. | ¿Alcanza para ~30 IOPS? |
|---|---|---|
| HDD 7.200 rpm | 100–200 | Justo: un backup o sincronización lo satura |
| SSD SATA | 50.000+ | Sí, con enorme margen |
| NVMe | 200.000+ | Sí, con enorme margen |

Por eso las **VMs van en SSD/NVMe**: el problema del HDD no es el promedio sino cualquier ráfaga (backup, reconstrucción, escaneo).

```text
Logs/día       = pet/s × bytes por línea × 86.400 = 9,5 × 300 B × 86.400 ≈ 250 MB/día (sin comprimir; ~10× menos comprimido)
Prometheus/día = series × (86.400 / intervalo) × ~2 B = 5.000 × (86.400/15) × 2 B ≈ 57 MB/día → ~1,7 GB por 30 días
Capacidad      = Σ (usuarios × cuota promedio) × (1 + crecimiento anual)^años × (1 / 0,8)
```

El último factor deja el sistema de archivos en **80 % de ocupación máxima**.

### 1.7 Base de datos

```text
conexiones máximas ≥ workers totales + 20 % + conexiones de administración y monitoreo
```

Cada conexión consume 1–10 MB. Más workers implica más conexiones y más RAM: el límite de workers se define **una sola vez**, mirando la RAM total.

### 1.8 Calculadora

```python
def dimensionar(N=100, Z=10, R=0.5, d=1,        # usuarios, reflexión, respuesta, pet. dinámicas por página
                S_cpu=0.3, rss_mb=100,            # CPU por petición, RAM por worker
                peso_pagina_mb=2, hit_cdn=0.7,    # peso y acierto de CDN
                descargas=5, mbps_descarga=50,    # transferencias grandes simultáneas
                io_por_peticion=3,                # operaciones de disco por petición
                margen=2.5, util=0.65):           # margen de workers y utilización objetivo de CPU
    X = N / (R + Z)
    lam = X * d
    L = lam * R
    workers = max(2, round(L * margen))
    return {
        "paginas_por_s": round(X, 1),
        "peticiones_dinamicas_por_s": round(lam, 1),
        "en_curso_L": round(L, 1),
        "workers": workers,
        "ram_workers_gb": round(workers * rss_mb / 1024, 2),
        "nucleos_cpu": round(lam * S_cpu / util, 1),
        "mbps_web": round(X * peso_pagina_mb * (1 - hit_cdn) * 8),
        "mbps_total": round(X * peso_pagina_mb * (1 - hit_cdn) * 8 + descargas * mbps_descarga),
        "iops_disco": round(lam * io_por_peticion),
    }

print(dimensionar())
# ≈ 9,5 páginas/s, 12 workers, ~1,2 GB para workers, ~4,4 núcleos, ~296 Mbps, ~29 IOPS
```

Ejecutala una vez por servicio (CMS, nube, aplicación propia) con valores medidos.

### 1.9 Cómo medir cada dato

| Dato | Herramienta | Comando o lugar |
|---|---|---|
| **R** | Log de nginx | Agregar `$request_time` al `log_format`; `awk '{s+=$NF;n++} END{print s/n}' access.log` |
| **L** | Estado de PHP-FPM | `pm.status_path` → "active processes" y "max children reached" |
| **RSS por worker** | smem | `smem -P php-fpm -k` bajo carga |
| **S** | Monitoreo + prueba de carga | Fórmula de [[#2.4 CPU]] |
| **Ancho de banda** | Monitoreo de interfaz/router | Pico de Mbps durante la prueba |
| **IOPS y latencia** | iostat | `iostat -x 5` (`r/s`, `w/s`, `await`) |
| **Acierto de CDN** | Panel de la CDN | Solicitudes en caché vs. totales |
| **Peso de página** | Navegador | DevTools → Red |

> [!note] Proceso recomendado
> 1) Calcular con valores típicos y definir hardware inicial. 2) Ejecutar la [[#7. Prueba de carga]]. 3) Medir R, RSS, S, Mbps e IOPS reales. 4) Reemplazarlos en la calculadora y ajustar. 5) Repetir al cambiar aplicaciones o usuarios.


## 2. Prueba de carga

❓ **Hoy no hay mediciones bajo carga.** Se ejecuta **desde otra máquina** de la red local con **k6** (gratuita), contra un sitio propio y en horario controlado.

```javascript
// prueba.js: simula usuarios navegando un sitio propio
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 20 },   // carga actual
    { duration: '5m', target: 50 },   // objetivo
    { duration: '5m', target: 100 },  // pico
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.01'],      // menos de 1 % de errores
    http_req_duration: ['p(95)<1500'],   // 95 % responde en < 1,5 s
  },
};

export default function () {
  http.get('https://TU-SITIO.ejemplo.com/');
  sleep(Math.random() * 10 + 5);  // 5–15 s entre clics
}
```

Ejecutar: `k6 run prueba.js`. Qué registrar mientras corre (ver [[06 Monitoreo]]):

| Métrica | Umbral aceptable | Si se pasa, el cuello de botella es... |
|---|---|---|
| CPU del host | < 70 % sostenido | CPU |
| RAM disponible | > 20 % libre | RAM |
| Latencia de disco | < 10 ms | Disco |
| Tráfico de red | < 70 % del enlace | Red o internet |
| Errores HTTP | < 1 % | Aplicación o configuración |
| Tiempo de respuesta p95 | < 1,5 s | Cualquiera de los anteriores |

---


---

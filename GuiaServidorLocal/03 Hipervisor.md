
tags: [wiki, homelab, proxmox, hipervisor]
fecha: 2026-11-22
links: [[01a Recursos]], [[04 Almacenamiento]], [[06 Monitoreo]]

# 01 Hipervisor - Proxmox VE

> [!info] Alcance
> Responde a "¿qué sistema operativo uso?": **Proxmox VE** como hipervisor bare-metal, por qué elegirlo, cómo instalarlo, el ajuste post-instalación y la herramienta ProxMenux. El almacenamiento (ZFS, TrueNAS) está en [[04 Almacenamiento]] y el monitoreo en [[06 Monitoreo]].

## Índice de esta nota

- [[#1. Por qué elegir Proxmox VE]]
- [[#2. Instalación de Proxmox VE desde cero]]
  - [[#2.1 Requisitos previos y medio de arranque]]
  - [[#2.2 Proceso de instalación]]
- [[#3. Primer ingreso y configuración post-instalación]]
- [[#4. ProxMenux - menú interactivo y dashboard web]]
- [[#5. Puntos a verificar antes de publicar]]

---

## 1. Por qué elegir Proxmox VE

Para un homelab que busca escalar (cargas web de hasta 200 usuarios, ver [[01-01 Recursos]]), un hipervisor **bare-metal de tipo 1** ofrece ventajas críticas:

- **Base sólida en Debian:** hereda estabilidad, ecosistema de paquetes y seguridad, y permite usar herramientas estándar (`nftables`, `systemd`, `iproute2`).
- **Dualidad nativa KVM + LXC:** VMs completas (KVM) para aislamiento total, y contenedores LXC ligeros (una fracción de RAM/CPU), ideales para microservicios web.
- **Almacenamiento avanzado:** soporte nativo de ZFS, LVM y Ceph.
- **Respaldos y alta disponibilidad:** integración nativa con Proxmox Backup Server (PBS) y snapshots.

## 2. Instalación de Proxmox VE desde cero

### 2.1 Requisitos previos y medio de arranque

- **Hardware mínimo:** CPU con virtualización por hardware (Intel VT-x / AMD-V), **16 GB de RAM** para empezar y al menos dos discos (para ZFS o separar SO/datos). Para el objetivo de 200 usuarios la guía de recursos propone **64 GB ECC** ([[01-01 Recursos#3.3 Memoria RAM]]).
- **Descarga:** ISO oficial *Proxmox VE Installer*.
- **USB booteable:** con **Ventoy** o **Rufus** (modo `DD`).

### 2.2 Proceso de instalación

1. Arrancar desde el pendrive y elegir **Install Proxmox VE**.
2. **Disco de destino (*Target Harddisk*):** con dos discos iguales, **ZFS (RAID 1)** para tolerancia a fallos (detalle en [[04 Almacenamiento#2. Proxmox VE sobre ZFS RAID 1 (espejo)]]); con un solo disco, `ext4` o `XFS` con LVM.
3. **Localización:** país, zona horaria y teclado.
4. **Credenciales:** contraseña segura de `root` y correo para alertas.
5. **Red:** interfaz de gestión (ej. `eno1`), **hostname** como FQDN (ej. `pve01.homelab.local`), e **IP estática** en la subred de administración, con gateway y DNS.

## 3. Primer ingreso y configuración post-instalación

Abrir en el navegador `https://<IP_DE_PROXMOX>:8006` e ingresar como `root`.

**Script de post-instalación (PVE Helper-Scripts de la comunidad).** Sirve para desactivar el aviso de suscripción, configurar los repositorios `no-subscription`, actualizar fuentes y aplicar ajustes menores:

1. Entrar por SSH al nodo como `root`.
2. Ejecutar el script interactivo (la URL se toma de tu nota original; verificá que siga vigente en el repositorio de *community-scripts*):

    ```bash
    bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/misc/post-install.sh)"
    ```

3. Seguir las opciones en pantalla: deshabilitar el aviso de suscripción, agregar/actualizar los repositorios correctos y, si se desea, habilitar actualizaciones automáticas de seguridad.

> [!warning] Revisá los scripts antes de ejecutarlos
> Ejecutar un script descargado directamente en `bash` como `root` da control total del nodo. Descargalo primero (`wget -qLO script.sh URL`), leelo y recién después ejecutalo.

## 4. ProxMenux - menú interactivo y dashboard web

**ProxMenux** simplifica la administración de Proxmox VE con un menú visual en consola y un panel web (**ProxMenux Monitor**).

**Características:**

- **Menú por terminal (`menu`):** post-instalación, GPU passthrough, discos/almacenamiento, redes y scripts de la comunidad sin recordar comandos.
- **Asistentes de backup y restauración:** creación de respaldos y almacenamiento externo.
- **ProxMenux Monitor:** se instala junto con ProxMenux; gráficas en tiempo real de CPU, RAM, discos, red y estado de VMs/LXC, con terminal WebSocket integrada (ver también [[06 Monitoreo]]).

**Instalación:**

1. En la terminal del nodo, como `root`:

    ```bash
    bash -c "$(wget -qLO - https://raw.githubusercontent.com/MacRimi/ProxMenux/main/install_proxmenux.sh)"
    ```

2. El instalador configura las dependencias (`dialog`, `jq`, `git`, `python3`, etc.).
3. Iniciar el menú con `menu` (en el primer inicio permite elegir idioma, incluido Español).

> [!warning] Aplica la misma precaución de la sección 3: revisá el script antes de ejecutarlo.

## 5. Puntos a verificar antes de publicar

- [ ] **Versión de Debian/Proxmox:** la nota original dice *Debian 12 (Bookworm)*. Verificá la versión de Proxmox VE vigente y la base de Debian que usa en su documentación oficial, y actualizá el texto y los repositorios del script de post-instalación.
- [ ] Confirmar que las URLs de los scripts de *community-scripts* y *ProxMenux* siguen vigentes.
- [ ] Agregar los pasos de seguridad básicos del nodo (2FA, acceso solo por VPN/VLAN de administración), hoy solo en [[04 Almacenamiento#5. Seguridad - checklist mínimo]].

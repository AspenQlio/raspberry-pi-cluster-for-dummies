# Raspberry Pi Cluster for Dummies

> Objetivo: Desacoplar un HomeLab saturado que corría en una sola Raspberry Pi (1GB RAM) hacia una arquitectura de dos nodos, separando los servicios de misión crítica (DNS/Seguridad) de los servicios pesados (Multimedia/Almacenamiento).

## 1. El Problema Original

El servidor principal (Raspberry Pi 3, 1GB RAM) estaba al borde del colapso. Los síntomas eran claros:
- Load average constante por encima de `14.00`
- Uso de I/O Wait rondando el `66%`
- Saturación total del espacio ZRAM (955MB al 100%) y uso masivo del Swapfile en tarjeta MicroSD (muy lenta).

El cuello de botella fue provocado por intentar correr **Frigate NVR** (procesamiento IA por CPU) y **Jellyfin** junto a la infraestructura base (**Pi-hole**, **Home Assistant**, **Vaultwarden**). La escasez de memoria RAM física forzaba al kernel a utilizar la MicroSD como RAM (Swap), provocando que los procesos de sistema compitieran por tiempo de I/O.

## 2. Arquitectura de Desacoplamiento

Para evitar conflictos y asegurar el funcionamiento de la red (DNS local), se dividió la carga de trabajo en dos nodos, comunicados a través de la red VPN de **Tailscale** (`100.x.x.x`).

```text
[Internet]
    |
[Router] (Asigna IPs 192.168.100.x)
    |
    |---> Nodo 1 (El Cerebro) --- IP: 192.168.100.75 | Tailscale: 100.84.189.38
    |       • Hardware: Raspberry Pi (1GB RAM) + MicroSD 60GB
    |       • Servicios: Pi-hole, Unbound, Home Assistant, Vaultwarden, Homepage, Tailscale Exit Node
    |       • Objetivo: Alta disponibilidad (Misión crítica). Nunca debe apagarse.
    |
    |---> Nodo 2 (El Músculo) --- IP: 192.168.100.80 | Tailscale: 100.66.109.86
            • Hardware: Raspberry Pi (1GB RAM) + Hub USB Externo + SSD 1TB
            • Servicios: Jellyfin, Transmission, Subliminal (Robot de subtítulos)
            • Objetivo: Procesamiento multimedia y descargas 24/7.
```

## 3. Errores durante la migración y cómo se solucionaron

### Error 1: Dependencia del SSD para el arranque del Nodo 1
Al intentar retirar el SSD de 1TB del Nodo 1 para dárselo al Nodo 2, nos dimos cuenta de que el sistema operativo completo corría desde la partición `/dev/sda2` del SSD, y la MicroSD solo actuaba como bootloader. 
**Solución:** Se montó la partición root de la MicroSD (`/dev/mmcblk0p2`), se utilizó `rsync` para copiar la data reciente (Docker, Vaultwarden, Pi-hole, Tailscale state), y se editó `/boot/firmware/cmdline.txt` para restaurar `root=PARTUUID=...` apuntando de nuevo a la MicroSD. Esto permitió desconectar el SSD de forma segura.

### Error 2: Jellyfin sin permisos para leer películas
En el Nodo 2, las películas se encontraban en el SSD bajo el directorio `/home/aspen`. Al ejecutar Jellyfin mediante Docker con PUID/PGID `1000`, este no tenía permisos para atravesar `/home/aspen` (que tenía permisos restrictivos `700`).
**Solución:** Se crearon las rutas públicas `/srv/media/movies` y `/srv/media/series` con permisos `755`, y se movió el catálogo a esta nueva jerarquía, permitiendo que el contenedor de Jellyfin leyera el volumen mapeado sin problemas de ACLs.

### Error 3: Descargas "desaparecidas" en Transmission
El usuario descargaba torrents y Transmission los guardaba temporalmente con la extensión `.part` dentro de un subdirectorio `incomplete`. Jellyfin ignoraba estos archivos por diseño, provocando confusión.
**Solución:** Transmission se configuró para rutear descargas directamente a `/movies` y `/series`. Se estableció la regla de que Jellyfin solo detectará el archivo de video una vez que Transmission finalice la descarga al 100% y elimine la extensión `.part`.

### Error 4: Peligro de "Undervoltage" en el Nodo 2
Conectar el SSD de 1TB directamente al puerto USB de la Raspberry Pi 2 excede la capacidad de corriente de la placa (~1.2A), lo que causaría reinicios aleatorios o corrupción de datos.
**Solución:** Se incluyó un Hub USB con alimentación externa independiente. El SSD extrae la energía directamente del tomacorriente, dejando que la Raspberry utilice el 100% de su propia fuente para procesar.

## 4. Subtítulos automáticos sin APIs (Zero-Clicks)
Se descartó el uso de extensiones nativas de Jellyfin u OpenSubtitles que requieren registro de usuarios y limitan la cuota de descargas.
En su lugar, se instaló `subliminal` nativo en el entorno host de la Raspberry Pi 2. Un job en `cron` ejecuta el siguiente script cada hora en punto:

```bash
#!/bin/bash
subliminal download -l es /srv/media/movies
subliminal download -l es /srv/media/series
```
Escanea el directorio en busca de nuevos archivos de video y descarga el `.srt` adyacente sin intervención del usuario.

## 5. Respaldo (Backups) automatizado
El archivo de contraseñas de Vaultwarden y las configuraciones de Home Assistant son críticos. Un script en `cron` (`backup.sh`) corre a las 3:00 AM en el Nodo 1:
1. Genera un `.tar.gz` de la configuración.
2. Utiliza `scp` para enviar una copia encriptada vía Tailscale (`100.66.109.86`) hacia el SSD de 1TB en el Nodo 2.
3. Utiliza `rclone` para hacer upload a Google Drive.
Esto asegura redundancia física (SSD secundario) y externa (Nube).

# Z890M Intel Core Ultra 5 245K with 4x Linksys MX5300

This covers a full home sever setup from scratch, including the network aspect. The firewall in this setup will be virtualized (gasp!).

## System specifications

- Intel Core Ultra 5 245K
- ASUS Prime Z890M-PLUS Wi-Fi
- Silicon Power Value Gaming DDR5 32GB 6000MT/s CL30 (SP032GXLWU60AFDEAE)
- ID-COOLING IS-67-XT
- Thermalright LGA1851 contact frame
- Silicon Power P34A60 256GB NVMe SSD
- Patriot P300 256GB NVMe SSD
- Western Digital Blue 4TB (WD40EZAX)
- Seagate Barracuda Compute 4TB (ST4000DM004)
- Toshiba Surveillance S300 4TB (HDWT140)
- 2x Western Digital Blue 3TB (???)
- Silverstone Extreme 500 Bronze
- [Sleeved 6x SATA Cable](https://www.amazon.com/dp/B09Q359C7Z?th=1)
- [Intel i226-V PCIe NIC](https://www.amazon.com/Network-2-5GBase-T-Adapter-Express-Ethernet/dp/B0CYB1GSHS)
- Jonsbo N4 MicroATX NAS Case

## Network specifications

- Xfinity as ISP - 150mpbs plan + VoIP
- Arris T25 for modem and VoIP
- 4x Linksys MX5300
  - OpenWrt flashed to each, A/B partition
- 802.11s mesh to connect all points
- BATMAN-adv to handle VLAN traffic over 802.11s mesh

remember: dkms, headers, secure boot, r8125, intel npu
https://www.corsair.com/us/en/explorer/diy-builder/memory/is-ddr5-ecc-memory/

## LXC config files

```text
arch: amd64
cmode: shell
console: 0
cores: 14
dev0: /dev/accel/accel0
features: fuse=1,nesting=1
hostname: pvesebringalpine
memory: 24576
mp0: /media/NVR,mp=/media/NVR
net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:92:8A:71,ip=dhcp,ip6=dhcp,type=veth
ostype: alpine
rootfs: local-zfs:subvol-100-disk-0,mountoptions=noatime,size=40G
swap: 0
tty: 0
lxc.apparmor.profile: unconfined
lxc.cap.drop: 
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
```

## Docker compose files

### Frigate NVR

```yaml
services:
  frigate:
    container_name: frigate
    privileged: true
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:0.15.0
    shm_size: "768mb"
    devices:
      - /dev/dri:/dev/dri
      - /dev/accel:/dev/accel
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/frigate/config:/config
      - /media/NVR/frigate:/media/frigate
      - /media/NVR/cache:/tmp/cache
    ports:
      - 5000:5000
      - 1984:1984
    env_file:
      - stack.env
```

#### stack.env

```text
FRIGATE_RTSP_PASSWORD=[REDACTED]
```

### Home Assistant

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2025.2
    volumes:
      - /opt/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123"]
      interval: 30s
      timeout: 10s
      retries: 6
    restart: unless-stopped
    privileged: true
    network_mode: host
```

### Immich

```yaml
name: immich
services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    extends:
      file: hwaccel.transcoding.yml
      service: quicksync
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - '2283:2283'
    depends_on:
      - redis
      - database
    restart: unless-stopped
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-openvino
    extends:
      file: hwaccel.ml.yml
      service: openvino
    volumes:
      - /opt/immich/model-cache:/cache
    env_file:
      - .env
    restart: unless-stopped
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/redis:6.2-alpine@sha256:905c4ee67b8e0aa955331960d2aa745781e6bd89afc44a8584bfd13bc890f0ae
    healthcheck:
      test: redis-cli ping || exit 1
    restart: unless-stopped

  database:
    container_name: immich_postgres
    image: docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:90724186f0a3517cf6914295b5ab410db9ce23190a2d9d0b9dd6463e3fa298f0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    healthcheck:
      test: >-
        pg_isready --dbname="$${POSTGRES_DB}" --username="$${POSTGRES_USER}" || exit 1;
        Chksum="$$(psql --dbname="$${POSTGRES_DB}" --username="$${POSTGRES_USER}" --tuples-only --no-align
        --command='SELECT COALESCE(SUM(checksum_failures), 0) FROM pg_stat_database')";
        echo "checksum failure count is $$Chksum";
        [ "$$Chksum" = '0' ] || exit 1
      interval: 5m
      start_interval: 30s
      start_period: 5m
    command: >-
      postgres
      -c shared_preload_libraries=vectors.so
      -c 'search_path="$$user", public, vectors'
      -c logging_collector=on
      -c max_wal_size=2GB
      -c shared_buffers=512MB
      -c wal_compression=on
    restart: unless-stopped
```

### .env

```text
UPLOAD_LOCATION=/media/NVR/immich
DB_DATA_LOCATION=/opt/immich/database
TZ=America/Los_Angeles
IMMICH_VERSION=release
DB_PASSWORD=[REDACTED]
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

### ZwaveJS

```yaml
services:
    zwave-js-ui:
        container_name: zwave-js-ui
        image: zwavejs/zwave-js-ui:9
        restart: unless-stopped
        tty: true
        stop_signal: SIGINT
        environment:
            - SESSION_SECRET=mysupersecretkey
            - TZ=America/Los_Angeles
        devices:
            - '/dev/serial/by-id/usb-Zooz_800_Z-Wave_Stick_533D004242-if00:/dev/zwave'
        volumes:
            - /opt/zwavejs/config:/usr/src/app/store
        ports:
            - '8091:8091'
            - '3000:3000'
```

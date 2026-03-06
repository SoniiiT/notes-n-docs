---
tags: [lxd, beispiele, praxis, homelab, nextcloud, nginx, pihole]
aliases: ["LXD Praxis", "LXD Use Cases"]
---

# LXD Praktische Beispiele

> [!ABSTRACT]
> Vollständige, praxiserprobte Setups für typische LXD-Anwendungsfälle im Homelab und in der Produktion: von einfachen Webservern über Nextcloud und Pi-hole bis hin zu Entwicklungsumgebungen und Docker-in-LXD.

---

## Beispiel 1: Einfacher Webserver (nginx)

Schnellstes Setup für einen produktiven Webserver:

```bash
# 1. Container erstellen mit Ressourcenlimits
lxc launch ubuntu:24.04 nginx-web \
    -c limits.cpu=1 \
    -c limits.memory=512MiB \
    -c boot.autostart=true

# 2. Port 80 und 443 nach außen forwarden
lxc config device add nginx-web http proxy \
    listen=tcp:0.0.0.0:80 \
    connect=tcp:127.0.0.1:80
lxc config device add nginx-web https proxy \
    listen=tcp:0.0.0.0:443 \
    connect=tcp:127.0.0.1:443

# 3. nginx installieren
lxc exec nginx-web -- bash -c "
    apt update && apt install -y nginx
    systemctl enable nginx
    systemctl start nginx
"

# 4. Testen
curl http://$(lxc list nginx-web --format=csv -c 4 | cut -d' ' -f1)
```

### Mit SSL via Certbot

```bash
lxc exec nginx-web -- bash -c "
    apt install -y certbot python3-certbot-nginx
    certbot --nginx -d meine-domain.de -d www.meine-domain.de \
        --email admin@meine-domain.de --agree-tos --non-interactive
"
```

---

## Beispiel 2: Pi-hole (DNS-Werbeblocker)

Pi-hole als Container für das gesamte Heimnetz:

```bash
# 1. Container mit macvlan (eigene IP im LAN — empfohlen für Pi-hole)
lxc launch ubuntu:22.04 pihole \
    -c limits.cpu=1 \
    -c limits.memory=256MiB \
    -c boot.autostart=true

# macvlan für direkte LAN-IP
lxc config device add pihole eth0 nic \
    nictype=macvlan \
    parent=eth0    # physisches Interface des Hosts anpassen

# Feste IP im Container setzen
lxc exec pihole -- bash -c "
    cat > /etc/netplan/99-static.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.1.53/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
EOF
    netplan apply
"

# 2. Pi-hole installieren
lxc exec pihole -- bash -c "
    curl -sSL https://install.pi-hole.net | bash /dev/stdin --unattended
"

# 3. Admin-Passwort setzen
lxc exec pihole -- pihole -a -p meinPasswort

# Pi-hole Web-UI: http://192.168.1.53/admin
```

> [!TIP]
> Router-DNS auf 192.168.1.53 setzen → alle Geräte im Netz nutzen Pi-hole automatisch.

---

## Beispiel 3: Nextcloud

Vollständige Nextcloud-Instanz mit PostgreSQL:

```bash
# ── Netz vorbereiten ─────────────────────────────────────

lxc network create nextcloud-net \
    ipv4.address=10.100.0.1/24 \
    ipv4.nat=true \
    ipv6.address=none

# ── Datenbank-Container ──────────────────────────────────

lxc launch ubuntu:24.04 nc-db \
    -c limits.cpu=2 \
    -c limits.memory=1GiB \
    -c boot.autostart=true \
    -c boot.autostart.priority=10

lxc config device add nc-db eth0 nic network=nextcloud-net
lxc config device set nc-db eth0 ipv4.address 10.100.0.10

lxc exec nc-db -- bash -c "
    apt update && apt install -y postgresql
    systemctl enable postgresql

    sudo -u postgres psql -c \"CREATE USER nextcloud WITH PASSWORD 'nc_password';\"
    sudo -u postgres psql -c \"CREATE DATABASE nextcloud OWNER nextcloud;\"
    sudo -u postgres psql -c \"GRANT ALL PRIVILEGES ON DATABASE nextcloud TO nextcloud;\"
"

# ── Nextcloud-Container ──────────────────────────────────

lxc launch ubuntu:24.04 nc-app \
    -c limits.cpu=2 \
    -c limits.memory=2GiB \
    -c boot.autostart=true \
    -c boot.autostart.priority=5 \
    -c boot.autostart.delay=15

lxc config device add nc-app eth0 nic network=nextcloud-net
lxc config device set nc-app eth0 ipv4.address 10.100.0.20

# Port 443 auf Host forwarden
lxc config device add nc-app https proxy \
    listen=tcp:0.0.0.0:443 \
    connect=tcp:127.0.0.1:443
lxc config device add nc-app http proxy \
    listen=tcp:0.0.0.0:80 \
    connect=tcp:127.0.0.1:80

lxc exec nc-app -- bash -c "
    apt update && apt install -y \
        apache2 libapache2-mod-php \
        php php-gd php-json php-pgsql php-curl php-mbstring \
        php-intl php-bcmath php-gmp php-imagick php-zip php-xml

    # Nextcloud herunterladen
    wget https://download.nextcloud.com/server/releases/latest.tar.bz2 -O /tmp/nc.tar.bz2
    tar -xjf /tmp/nc.tar.bz2 -C /var/www/html/
    chown -R www-data:www-data /var/www/html/nextcloud

    # Apache konfigurieren
    a2enmod rewrite headers env dir mime
    systemctl restart apache2
"
```

---

## Beispiel 4: LEMP-Stack (nginx + PHP + MariaDB)

Typischer Webserver-Stack für WordPress oder andere PHP-Apps:

```bash
# Alle Services in einem Container (Legacy-Stil)
lxc launch ubuntu:24.04 lemp-stack \
    -c limits.cpu=2 \
    -c limits.memory=2GiB \
    -c boot.autostart=true

lxc config device add lemp-stack http proxy \
    listen=tcp:0.0.0.0:80 \
    connect=tcp:127.0.0.1:80
lxc config device add lemp-stack https proxy \
    listen=tcp:0.0.0.0:443 \
    connect=tcp:127.0.0.1:443

lxc exec lemp-stack -- bash << 'EOF'
# System
apt update && apt upgrade -y

# MariaDB
apt install -y mariadb-server
mysql_secure_installation --use-default

# nginx + PHP
apt install -y nginx php8.3-fpm php8.3-mysql php8.3-xml \
    php8.3-mbstring php8.3-curl php8.3-zip php8.3-gd

# WordPress
wget https://wordpress.org/latest.tar.gz -O /tmp/wp.tar.gz
tar -xzf /tmp/wp.tar.gz -C /var/www/html/
chown -R www-data:www-data /var/www/html/wordpress

# Datenbank erstellen
mysql -u root -e "
    CREATE DATABASE wordpress;
    CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'wp_password';
    GRANT ALL ON wordpress.* TO 'wp_user'@'localhost';
"

systemctl enable nginx php8.3-fpm mariadb
systemctl start nginx php8.3-fpm mariadb
EOF
```

---

## Beispiel 5: Entwicklungsumgebung

Isolierte Entwicklungsumgebung mit Zugriff auf Host-Projektordner:

```bash
# 1. Entwicklungs-Container mit großzügigen Ressourcen
lxc launch ubuntu:24.04 dev-env \
    -c limits.cpu=4 \
    -c limits.memory=8GiB \
    -c security.nesting=true  # für Docker-in-LXD

# 2. Projektordner vom Host einhängen
lxc config device add dev-env projects disk \
    source=$HOME/projects \
    path=/home/dev/projects

# 3. Entwicklungs-Tools installieren
lxc exec dev-env -- bash -c "
    apt update && apt install -y \
        git curl wget build-essential \
        python3 python3-pip python3-venv \
        nodejs npm \
        golang-go \
        docker.io docker-compose

    # Benutzer anlegen
    useradd -m -s /bin/bash -G sudo,docker dev
    echo 'dev:dev123' | chpasswd

    # SSH aktivieren
    apt install -y openssh-server
    systemctl enable ssh
"

# 4. SSH-Port forwarden
lxc config device add dev-env ssh proxy \
    listen=tcp:0.0.0.0:2222 \
    connect=tcp:127.0.0.1:22

# Verbinden
ssh dev@localhost -p 2222
# Projekte unter /home/dev/projects (= $HOME/projects auf Host)
```

---

## Beispiel 6: Docker-in-LXD

Docker innerhalb eines LXD-Containers betreiben:

```bash
# Container mit Nesting-Support
lxc launch ubuntu:24.04 docker-host \
    -c security.nesting=true \
    -c limits.cpu=4 \
    -c limits.memory=8GiB

# Docker installieren
lxc exec docker-host -- bash -c "
    apt update
    curl -fsSL https://get.docker.com | sh
    systemctl enable docker
    systemctl start docker
    usermod -aG docker ubuntu
"

# Testen
lxc exec docker-host -- docker run hello-world
lxc exec docker-host -- docker run -d -p 8080:80 nginx

# Port auf Host forwarden
lxc config device add docker-host web proxy \
    listen=tcp:0.0.0.0:8080 \
    connect=tcp:127.0.0.1:8080
```

> [!NOTE]
> `security.nesting=true` aktiviert User-Namespace-Nesting, was Docker-in-LXD ermöglicht. Performance-Overhead ist minimal.

---

## Beispiel 7: Gitea (Git-Server)

Selbst gehosteter Git-Server als Alternative zu GitHub:

```bash
# Container erstellen
lxc launch ubuntu:24.04 gitea \
    -c limits.cpu=2 \
    -c limits.memory=1GiB \
    -c boot.autostart=true

# Ports forwarden
lxc config device add gitea web proxy \
    listen=tcp:0.0.0.0:3000 \
    connect=tcp:127.0.0.1:3000
lxc config device add gitea ssh proxy \
    listen=tcp:0.0.0.0:2222 \
    connect=tcp:127.0.0.1:22

# Gitea installieren
lxc exec gitea -- bash -c "
    apt update && apt install -y git sqlite3

    # Gitea herunterladen
    wget -O /usr/local/bin/gitea \
        https://dl.gitea.com/gitea/1.22/gitea-1.22-linux-amd64
    chmod +x /usr/local/bin/gitea

    # Gitea-Benutzer anlegen
    adduser --system --shell /bin/bash --gecos 'Git Version Control' \
        --group --disabled-password --home /home/git git

    # Systemd-Service
    cat > /etc/systemd/system/gitea.service << 'SEOF'
[Unit]
Description=Gitea
After=network.target

[Service]
User=git
WorkingDirectory=/home/git
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always

[Install]
WantedBy=multi-user.target
SEOF

    mkdir -p /etc/gitea /var/lib/gitea
    chown git:git /etc/gitea /var/lib/gitea
    systemctl enable gitea
    systemctl start gitea
"
# Web-UI: http://host:3000
# Erstkonfiguration im Browser abschließen
```

---

## Beispiel 8: Monitoring-Stack

Prometheus + Grafana für das gesamte Homelab:

```bash
# Gemeinsames Monitoring-Netz
lxc network create monitoring-net \
    ipv4.address=10.200.0.1/24 \
    ipv4.nat=true

# Prometheus
lxc launch ubuntu:24.04 prometheus \
    -c limits.cpu=1 \
    -c limits.memory=512MiB \
    -c boot.autostart=true \
    -c boot.autostart.priority=10

lxc config device add prometheus eth0 nic network=monitoring-net
lxc config device set prometheus eth0 ipv4.address 10.200.0.10

lxc exec prometheus -- bash -c "
    apt update && apt install -y prometheus
    systemctl enable prometheus && systemctl start prometheus
"

# Grafana
lxc launch ubuntu:24.04 grafana \
    -c limits.cpu=1 \
    -c limits.memory=512MiB \
    -c boot.autostart=true \
    -c boot.autostart.priority=5

lxc config device add grafana eth0 nic network=monitoring-net
lxc config device set grafana eth0 ipv4.address 10.200.0.20
lxc config device add grafana web proxy \
    listen=tcp:0.0.0.0:3000 \
    connect=tcp:127.0.0.1:3000

lxc exec grafana -- bash -c "
    apt install -y apt-transport-https software-properties-common
    wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
    echo 'deb https://packages.grafana.com/oss/deb stable main' \
        > /etc/apt/sources.list.d/grafana.list
    apt update && apt install -y grafana
    systemctl enable grafana-server && systemctl start grafana-server
"
# Grafana: http://host:3000 (admin/admin)
```

---

## Container-Inventar pflegen

```bash
# Alle Container mit Beschreibung versehen
lxc config set nginx-web description "nginx Webserver für meine-domain.de"
lxc config set pihole description "Pi-hole DNS-Server für Heimnetz"
lxc config set nc-app description "Nextcloud App-Server"
lxc config set nc-db description "Nextcloud PostgreSQL Datenbank"

# Übersicht aller Container
lxc list -c n,s,4,description

# Als YAML exportieren (Dokumentation)
for ct in $(lxc list --format=csv -c n); do
    echo "--- $ct ---"
    lxc config show $ct | grep -E "description|limits|boot"
done
```

---

## Verwandte Notizen

- [[LXD Installation und Setup]] — Basis-Installation
- [[LXD Best Practices]] — Ressourcenlimits und Sicherheit
- [[LXD Netzwerk]] — Netzwerk-Konfiguration
- [[Profile und Konfiguration]] — Wiederverwendbare Konfigurationen
- [[LXD Troubleshooting]] — Wenn etwas nicht funktioniert

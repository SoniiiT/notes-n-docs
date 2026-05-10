---
tags: [lxd, sicherheit, security, apparmor, seccomp, idmap, privileged]
aliases: ["LXD Security", "LXD Absicherung"]
---

# LXD Sicherheit

> [!ABSTRACT]
> LXD bietet mehrere Sicherheitsschichten: UID/GID-Remapping für unprivilegierte Container, AppArmor-Profile, Seccomp-Filter und feingranulare Capabilities. Dieser Artikel erklärt das Sicherheitsmodell und zeigt, wie Container sicher konfiguriert werden.

---

## Sicherheitsmodell im Überblick

```
┌─────────────────────────────────────────────────────────┐
│                      Host-System                        │
│                                                         │
│  AppArmor-Policy (pro Container)                        │
│  Seccomp-Filter (Syscall-Whitelist)                     │
│  Capabilities (begrenzt)                               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Unprivilegierter Container            │   │
│  │                                                 │   │
│  │  Container-UID 0 (root)                        │   │
│  │         ↕ UID-Remapping                        │   │
│  │  Host-UID 100000 (kein echter Root!)           │   │
│  │                                                 │   │
│  │  Kernel-Exploit im Container                   │   │
│  │  → max. Host-UID 100000 (kein Root-Zugriff)   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Privileged vs. Unprivileged

### Unprivileged (Standard — empfohlen)

```bash
# Status prüfen
lxc config get mycontainer security.privileged
# Ausgabe: false = unprivileged (sicher)

# Explizit setzen
lxc config set mycontainer security.privileged false
```

**Wie UID-Remapping funktioniert:**

```
Container UID 0 (root)    → Host UID 100000
Container UID 1000 (user) → Host UID 101000
Container UID 65535       → Host UID 165535

# Auf dem Host sehen Container-Prozesse so aus:
ps aux | grep nginx
# lxd      100033  ...  nginx: worker process

# UID-Mapping einsehen
cat /proc/$(lxc exec mycontainer -- cat /proc/1/status | grep -i pid | head -1 | awk '{print $2}')/status | grep Uid
```

### Privileged Container (nur wenn nötig)

```bash
# Privilegierten Container erstellen
lxc config set mycontainer security.privileged true

# WARNUNG: Container-Root = Host-Root
# Nur für spezielle Anwendungsfälle:
# - NFS-Server
# - FUSE-Filesystems
# - Bestimmte Network-Namespacing-Anforderungen
```

> [!DANGER]
> Privilegierte Container sind ein Sicherheitsrisiko. Ein Kernel-Exploit im Container ermöglicht Host-Root-Zugriff. Immer zuerst prüfen ob unprivilegiert ausreicht.

---

## UID/GID-Mapping anpassen

### Isoliertes Mapping (security.idmap.isolated)

```bash
# Jeder Container bekommt eigenen UID-Bereich
lxc config set mycontainer security.idmap.isolated true

# Container A: UIDs 100000-165535
# Container B: UIDs 200000-265535
# → Keine UID-Überschneidung zwischen Containern
```

### Explizites Mapping

```bash
# Eigenes UID/GID-Mapping definieren
lxc config set mycontainer raw.idmap "uid 1000 1000
gid 1000 1000"
# Container-UID 1000 → Host-UID 1000 (Identity-Mapping für einen User)
```

---

## AppArmor

LXD generiert automatisch AppArmor-Profile für jeden Container:

```bash
# AppArmor-Status prüfen
sudo aa-status | grep lxd

# Profil eines Containers anzeigen
cat /var/snap/lxd/common/lxd/security/apparmor/profiles/lxd-mycontainer

# AppArmor für Container deaktivieren (nicht empfohlen)
lxc config set mycontainer raw.apparmor "mount,"
```

**Was AppArmor in LXD-Containern blockiert:**
- Bestimmte `/proc` und `/sys` Zugriffe
- Mount-Operationen (je nach Konfiguration)
- Gefährliche Kernel-Interfaces

---

## Seccomp-Filter

Seccomp blockiert gefährliche System Calls auf Kernel-Ebene:

```bash
# Seccomp-Profil anzeigen
lxc config get mycontainer raw.seccomp

# Beispiel: zusätzlichen Syscall verbieten
lxc config set mycontainer raw.seccomp "
2
blacklist
[all]
syscall_name keyctl action errno 1
"

# Seccomp komplett deaktivieren (für spezielle Anwendungsfälle)
lxc config set mycontainer security.syscalls.deny_default false
```

**Standard-blockierte Syscalls in LXD:**
- `keyctl`, `bpf`, `perf_event_open`
- Bestimmte `ioctl`-Varianten
- Kernel-Module laden

---

## Linux Capabilities

```bash
# Bestimmte Capabilities erlauben
lxc config set mycontainer linux.kernel_modules wireguard

# Capabilities-Profil anpassen
lxc config set mycontainer raw.lxc "lxc.cap.drop=sys_admin"

# Nesting erlauben (Docker in LXD, LXD in LXD)
lxc config set mycontainer security.nesting true

# CAP_SYS_ADMIN für Container erlauben
lxc config set mycontainer security.privileged false
lxc config set mycontainer raw.lxc "lxc.cap.drop="
# Leer = alle Capabilities erlaubt (fast privileged)
```

---

## Netzwerk-Sicherheit

```bash
# MAC-Spoofing verhindern
lxc config device set mycontainer eth0 security.mac_filtering true

# ARP-Spoofing verhindern
lxc config device set mycontainer eth0 security.ipv4_filtering true

# IPv6 NDP-Spoofing verhindern
lxc config device set mycontainer eth0 security.ipv6_filtering true

# Alle drei auf einmal via Profil
lxc profile device set default eth0 security.mac_filtering=true
lxc profile device set default eth0 security.ipv4_filtering=true
```

---

## Zugriffskontrolle für LXD

### API-Authentifizierung

```bash
# Verfügbare Auth-Methoden anzeigen
lxc config get core.https_address

# TLS-Client-Zertifikate verwalten
lxc config trust list
lxc config trust show <fingerprint>
lxc config trust revoke <fingerprint>

# Zertifikat hinzufügen
lxc config trust add myclient.crt

# Mit Namen und Einschränkungen
lxc config trust add myclient.crt --name "CI Server" --restricted
```

### RBAC mit OpenFGA (LXD 6.x)

```bash
# OIDC für externe Authentifizierung
lxc config set oidc.issuer https://auth.example.com
lxc config set oidc.client.id lxd-client
lxc config set oidc.claim.groups groups
```

---

## Sichere Konfigurationsvorlage

```bash
# Sicheres Profil erstellen
lxc profile create secure

lxc profile set secure security.privileged false
lxc profile set secure security.idmap.isolated true
lxc profile device set secure eth0 security.mac_filtering true
lxc profile device set secure eth0 security.ipv4_filtering true
lxc profile device set secure eth0 security.ipv6_filtering true

# Container mit sicherem Profil
lxc launch ubuntu:24.04 secure-container -p default -p secure
```

---

## Security Checkliste

```
Vor dem Deployment prüfen:

Container:
  [ ] security.privileged = false (unprivileged)
  [ ] security.idmap.isolated = true (für sensitive Workloads)
  [ ] limits.cpu und limits.memory gesetzt
  [ ] boot.autostart.delay gesetzt (kein paralleler Massenstart)

Netzwerk:
  [ ] MAC-Filtering aktiviert
  [ ] IP-Filtering aktiviert
  [ ] Nicht benötigte Ports nicht weitergeleitet

LXD-Daemon:
  [ ] core.https_address nur bei Bedarf aktiv
  [ ] Trust-Zertifikate regelmäßig rotieren
  [ ] Unix-Socket-Gruppe korrekt (lxd)

Host:
  [ ] AppArmor aktiv (aa-status)
  [ ] LXD-Gruppe nur vertrauenswürdigen Benutzern
  [ ] LXD-Snap regelmäßig aktualisieren
```

---

## Verwandte Notizen

- [[Was ist LXC-LXD]] — Unprivileged Containers erklärt
- [[Profile und Konfiguration]] — security.*-Schlüssel
- [[LXD Troubleshooting]] — Sicherheitsfehler diagnostizieren
- [[LXD Cluster Betrieb]] — Cluster-Sicherheit

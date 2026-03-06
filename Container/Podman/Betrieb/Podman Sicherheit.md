---
tags: [podman, sicherheit, selinux, seccomp, rootless, capabilities, apparmor, hardening, secrets]
aliases: ["Podman Security", "Podman Hardening", "Container Sicherheit"]
---

# Podman Sicherheit

> [!ABSTRACT]
> Podman bietet Defense-in-Depth durch mehrere Sicherheitsebenen: Rootless-Betrieb (kein Daemon-Root), User Namespaces (Container-Root ist kein Host-Root), Linux Capabilities (minimaler Capability-Set), SELinux/AppArmor (MAC-Kontext) und Seccomp (Syscall-Filter). Alle Ebenen greifen zusammen.

---

## Sicherheitsmodell — Defense in Depth

```
Ebene 1: Kein Daemon      → keine dauerhaft laufende Root-Angriffsfläche
Ebene 2: Rootless         → Container-Root ≠ Host-Root
Ebene 3: User Namespaces  → UID-Isolation zwischen Container und Host
Ebene 4: Capabilities     → minimaler Kernel-Rechte-Satz
Ebene 5: Seccomp          → Syscall-Whitelist
Ebene 6: SELinux/AppArmor → Mandatory Access Control
Ebene 7: Read-Only FS     → kein Container-Prozess kann Image-Dateien ändern
```

Jede Ebene schützt auch wenn eine andere umgangen wird.

---

## Rootless als erste Sicherheitslinie

Der wichtigste Sicherheitsgewinn von Podman gegenüber Docker:

```bash
# Angriffsszenario mit Docker
docker run -v /:/host --rm -it alpine chroot /host sh
# → Root-Shell auf dem Host! (kein sudo nötig für docker-Gruppe)

# Bei Podman rootless:
podman run -v /:/host --rm -it alpine chroot /host sh
# Container-Root = UID 1000 auf dem Host
# → chroot schlägt fehl oder bietet keinen Root-Zugriff auf Host-Dateien
```

Für Details zu Rootless → [[Rootless Container]]

---

## Linux Capabilities

Capabilities teilen Root-Rechte in feingranulare Einheiten auf. Podman startet Container mit einem **minimalen Default-Set**:

```bash
# Default-Capabilities anzeigen
podman run --rm alpine cat /proc/self/status | grep Cap
# CapBnd: 00000000a80425fb

# Menschenlesbar dekodieren
podman run --rm alpine sh -c "apk add libcap -q && capsh --decode=00000000a80425fb"
```

**Podman Default-Capabilities (rootless):**
- `CHOWN` — Dateieigentümer ändern
- `DAC_OVERRIDE` — Dateiberechtigungen überschreiben
- `FOWNER` — Dateioperationen auf nicht-eigenen Dateien
- `NET_BIND_SERVICE` — Ports < 1024 binden
- `SETUID`, `SETGID` — UID/GID wechseln
- Kein `CAP_SYS_ADMIN`, kein `CAP_NET_ADMIN`, kein `CAP_SYS_PTRACE`

### Capabilities verwalten

```bash
# Einzelne Capability hinzufügen
podman run --cap-add CAP_NET_ADMIN myapp

# Capability entfernen
podman run --cap-drop CAP_CHOWN nginx

# Alle droppen, nur benötigte hinzufügen (sicherste Option)
podman run \
  --cap-drop all \
  --cap-add CAP_NET_BIND_SERVICE \
  nginx:alpine

# Privilege Escalation verhindern
podman run --security-opt no-new-privileges myapp
```

> [!TIP] Minimale Capabilities
> Das Prinzip "least privilege" anwenden: Mit `--cap-drop all` starten und nur die tatsächlich benötigten Capabilities hinzufügen. Die meisten Anwendungen brauchen keine erweiterten Capabilities.

---

## SELinux-Integration (RHEL/Fedora/CentOS)

Podman setzt automatisch SELinux-Kontext-Labels auf Container-Prozesse und -Dateien.

### Container-Prozess-Kontext

```bash
# SELinux-Label eines Container-Prozesses anzeigen
podman inspect --format "{{.ProcessLabel}}" container-name
# system_u:system_r:container_t:s0:c100,c200

# Kontext auf dem Host anzeigen
ps -eZ | grep nginx
# unconfined_u:system_r:container_t:s0:c100,c200  ← MCS-Labels
```

### Volume-Labels — kritisch!

```bash
# FALSCH — SELinux blockiert Zugriff:
podman run -v /daten:/data nginx

# RICHTIG:
podman run -v /daten:/data:z nginx   # :z = shared (mehrere Container)
podman run -v /daten:/data:Z nginx   # :Z = private (nur dieser Container)
```

```bash
# SELinux-Fehler diagnostizieren
sudo ausearch -m avc -ts recent
sudo ausearch -m avc -ts recent | audit2why
sudo ausearch -m avc -ts recent | audit2allow   # Erlaubnis generieren (mit Vorsicht!)

# SELinux-Kontext manuell setzen
sudo chcon -Rt container_file_t /daten/
```

> [!WARNING] SELinux nie dauerhaft deaktivieren
> ```bash
> sudo setenforce 0   # Nur für temporäres Debugging!
> ```
> In Produktion immer `:z` oder `:Z` nutzen statt SELinux zu deaktivieren.

---

## AppArmor (Ubuntu/Debian)

```bash
# AppArmor-Profil eines Containers anzeigen
podman inspect --format "{{.AppArmorProfile}}" container-name
# containers-default

# Custom Profil laden
sudo apparmor_parser -r -W /etc/apparmor.d/containers.nginx

# Profil beim Container setzen
podman run --security-opt apparmor=containers.nginx nginx

# AppArmor deaktivieren (nur Debugging!)
podman run --security-opt apparmor=unconfined nginx
```

---

## Seccomp — Syscall-Filter

Seccomp filtert Kernel-Syscalls, die Container aufrufen dürfen.

```bash
# Aktives Seccomp-Profil anzeigen
podman inspect --format "{{.HostConfig.SecurityOpt}}" container-name

# Standard-Profil anzeigen
cat /usr/share/containers/seccomp.json | jq '.syscalls | length'
# → Anzahl erlaubter Syscalls (~300+)

# Custom Seccomp-Profil
podman run --security-opt seccomp=/etc/containers/seccomp-strict.json myapp

# Deaktivieren (nur für Debugging!)
podman run --security-opt seccomp=unconfined myapp
```

---

## Read-Only-Container

```bash
# Container mit Read-Only Root-FS
podman run --read-only nginx

# Schreibbare Ausnahmen via tmpfs
podman run \
  --read-only \
  --tmpfs /tmp:rw,size=64m \
  --tmpfs /run:rw,size=16m \
  --tmpfs /var/cache/nginx:rw,size=32m \
  nginx:alpine

# Named Volume für Applikationsdaten
podman run \
  --read-only \
  --tmpfs /tmp:rw \
  -v app-data:/data:Z \
  myapp:latest
```

---

## Image-Sicherheit

### Nur vertrauenswürdige Quellen

```bash
# Bevorzugte Registries (nach Vertrauensniveau):
# registry.access.redhat.com  → Red Hat geprüft und signiert
# quay.io                     → Automatische Vulnerability-Scans
# ghcr.io/VERIFIZIERTE_ORG    → GitHub-Projekte mit Reviews
# docker.io/library/IMAGE     → Offizielle Docker-Images

# Unbekannte Images aus docker.io meiden
# NICHT: docker.io/irgendein-user/myapp:latest  ← gefährlich!
```

### Image-Signatur-Policy

```bash
# Policy anzeigen
cat /etc/containers/policy.json

# Alle Images aus einer Registry erlauben (Default)
# Policy für nur signierte Images:
```

```json
{
  "default": [{"type": "reject"}],
  "transports": {
    "docker": {
      "registry.access.redhat.com": [{"type": "insecureAcceptAnything"}],
      "quay.io/meinorg": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/rpm-gpg/meinorg.gpg"
        }
      ]
    }
  }
}
```

### Image-Scanning

```bash
# Trivy (empfohlen)
trivy image docker.io/library/nginx:latest

# Grype
grype docker.io/library/nginx:latest

# Podman Scout (wenn verfügbar)
podman scout cves nginx:latest
```

---

## Secrets sicher übergeben

```bash
# FALSCH — Passwort in Umgebungsvariable direkt:
podman run -e DB_PASSWORD=geheim123 myapp  # ❌ sichtbar in podman inspect!

# RICHTIG — Podman Secrets:
echo "geheim123" | podman secret create db-password -

# Als Umgebungsvariable (aus Secret)
podman run --secret db-password,type=env,target=DB_PASSWORD myapp

# Als Datei (sicherer — Secret nur im Container-FS sichtbar)
podman run --secret db-password,type=mount,target=/run/secrets/db-password myapp
# Im Container: cat /run/secrets/db-password

# Secret im Quadlet (.container-Datei)
```

```ini
[Container]
Secret=db-password,type=env,target=DB_PASSWORD
```

---

## Netzwerk-Sicherheit

```bash
# Container-Netzwerk isolieren
podman network create --internal interne-netz   # kein Internet-Zugang
podman run --network interne-netz --name db postgres:16

# Nur nötige Ports exponieren
podman run -p 127.0.0.1:5432:5432 db   # Nur lokal erreichbar

# Kein Host-Networking außer wenn absolut nötig
# VORSICHT: --network host deaktiviert alle Netzwerk-Isolation
```

---

## Sicherheits-Checkliste

```bash
# Empfohlenes Template für sichere Container:
podman run \
  --read-only \                          # Read-Only Root-FS
  --cap-drop all \                       # Alle Capabilities droppen
  --cap-add CAP_NET_BIND_SERVICE \       # Nur was nötig ist
  --security-opt no-new-privileges \     # Kein Privilege Escalation
  --security-opt seccomp=/etc/containers/seccomp.json \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \  # tmpfs ohne exec
  -v /daten:/data:ro,Z \                # Read-Only Volume, private SELinux
  --user 1000:1000 \                    # Als nicht-root User
  --network intern-netz \               # Isoliertes Netzwerk
  myapp:latest
```

---

## Verwandte Notizen

- [[Rootless Container]] — Rootless als Sicherheitsbasis
- [[Podman Networking]] — Netzwerk-Isolation
- [[Container und Images]] — Image-Vertrauensstrategie
- [[Podman Troubleshooting]] — SELinux-Fehler debuggen

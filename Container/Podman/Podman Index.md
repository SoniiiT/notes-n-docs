---
tags: [moc, index, podman, container, rootless, daemonless]
aliases: ["Podman Übersicht", "Podman Index"]
---

# 🦭 Podman Index

> [!ABSTRACT]
> Podman ist ein daemonloser, rootless-fähiger Container-Manager von Red Hat und der Open-Source-Community. Im Gegensatz zu Docker benötigt Podman keinen zentralen Daemon-Prozess — jeder Container läuft als direkter Kindprozess des aufrufenden Benutzers. Vollständig OCI-konform und weitgehend CLI-kompatibel mit Docker.

---

## 📚 Grundlagen

Einstieg: Was Podman ist, wie es intern aufgebaut ist und wie es sich von Docker unterscheidet.

| Thema | Notiz |
|-------|-------|
| Einführung und Konzept | [[Was ist Podman]] |
| Interne Architektur | [[Podman Architektur]] |
| Vergleich mit Docker | [[Podman vs Docker]] |

---

## ⚙️ Installation

Podman auf verschiedenen Plattformen einrichten.

| Thema | Notiz |
|-------|-------|
| Installation (Linux, macOS, Windows) | [[Podman Installation]] |

---

## 🧩 Konzepte

Kernkonzepte: Container-Lifecycle, Images, Pods, Netzwerk, Storage und Rootless-Betrieb.

### 📦 Images

| Thema | Notiz |
|-------|-------|
| Container und Images | [[Container und Images]] |
| Containerfile-Referenz | [[Containerfile]] |
| Containerfile-Beispiele (Node, Python, Go, Java, Nginx) | [[Containerfile Beispiele]] |
| Multi-Stage Builds | [[Multi-Stage Builds]] |
| Registries & Repositories (Skopeo) | [[Podman Registries]] |

### 🏗️ Container

| Thema | Notiz |
|-------|-------|
| Container-Lebenszyklus | [[Container Lifecycle]] |
| Pods (Kubernetes-Kompatibilität) | [[Pods]] |
| Netzwerk-Stack (Netavark, Pasta) | [[Podman Networking]] |
| Volumes und Storage | [[Volumes und Storage]] |
| Rootless Container | [[Rootless Container]] |

---

## 🔧 Betrieb

Podman im täglichen Einsatz — CLI, Compose, systemd-Integration, Sicherheit und Fehlersuche.

| Thema | Notiz |
|-------|-------|
| CLI-Referenz & Cheatsheet | [[Podman CLI Cheatsheet]] |
| Compose-Stacks verwalten | [[Podman Compose]] |
| systemd-Integration & Quadlet | [[Systemd Integration]] |
| Security & Hardening | [[Podman Sicherheit]] |
| Monitoring & Logging | [[Podman Monitoring und Logging]] |
| Best Practices & Checkliste | [[Podman Best Practices]] |
| CI/CD-Pipelines & Automation | [[Podman CI-CD Integration]] |
| Fehlersuche & Debugging | [[Podman Troubleshooting]] |
| Grafische Oberfläche (GUI) | [[Podman Desktop]] |
| Praxisbeispiele & Use Cases | [[Podman Praktische Beispiele]] |

---

## Verwandte Notizen

- [[Docker vs Podman und LXC]] — Architekturvergleich: Docker, Podman und LXC auf einen Blick
- [[LXC vs LXD vs Docker]] — Systemcontainer vs. Anwendungscontainer im Vergleich

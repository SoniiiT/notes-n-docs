---
tags: [virtualization, esxi, vsphere, vmware, vcenter, cluster, ha, drs, vmotion]
aliases: ["vCenter", "vCenter Server", "VCSA", "vSphere vCenter"]
---

# vCenter Server

> [!ABSTRACT]
> vCenter Server ist die zentrale Verwaltungsplattform für vSphere-Umgebungen. Ohne vCenter sind erweiterte Features wie HA, DRS und vMotion nicht verfügbar. Seit vSphere 6.5 wird vCenter ausschließlich als **VCSA** (vCenter Server Appliance) — eine Linux-basierte OVA — ausgeliefert. Die Windows-basierte Variante wurde eingestellt.

---

## Was ist vCenter Server?

vCenter Server ist die **Managementebene der vSphere-Plattform**. Es verwaltet mehrere ESXi-Hosts, fasst sie zu Clustern zusammen und aktiviert Cluster-Features wie vMotion, HA und DRS. Ohne vCenter kann jeder ESXi-Host nur einzeln über den Host Client verwaltet werden — Cluster-Funktionen stehen nicht zur Verfügung.

| Funktion | Ohne vCenter | Mit vCenter |
|---------|-------------|------------|
| Einzelne VM verwalten | ✓ | ✓ |
| Mehrere Hosts zentral verwalten | ✗ | ✓ |
| vMotion (Live Migration) | ✗ | ✓ |
| High Availability (HA) | ✗ | ✓ |
| Distributed Resource Scheduler (DRS) | ✗ | ✓ |
| Distributed vSwitch (VDS) | ✗ | ✓ |
| Rollen und Berechtigungen zentral | ✗ | ✓ |
| Content Library | ✗ | ✓ |
| vSphere Lifecycle Manager | ✗ | ✓ |

vCenter wird als **VCSA** (vCenter Server Appliance) deployed — eine vorkonfigurierte Linux-VM im OVA-Format. Die VCSA enthält eine integrierte **PostgreSQL-Datenbank** für kleinere Umgebungen (bis ~2000 Hosts / 35000 VMs) oder unterstützt eine externe Oracle-Datenbank für größere Deployments.

---

## VCSA Deployment

### Voraussetzungen

| Ressource | Tiny (≤10 Hosts) | Small (≤100 Hosts) | Medium (≤400 Hosts) |
|-----------|-----------------|------------------|-------------------|
| **vCPU** | 2 | 4 | 8 |
| **RAM** | 14 GB | 20 GB | 32 GB |
| **Storage** | 415 GB | 480 GB | 700 GB |

> [!NOTE] VCSA muss auf ESXi laufen
> Die VCSA wird als VM auf einem ESXi-Host deployed. Das bedeutet: Bevor vCenter deployed werden kann, muss mindestens ein ESXi-Host bereits installiert und erreichbar sein.

### Installation via VCSA Installer

```
1. VCSA ISO herunterladen (Broadcom Portal)
   → ISO mounten

2. Installer starten:
   Windows: vcsa-ui-installer\win32\installer.exe
   macOS:   vcsa-ui-installer/mac/Installer.app
   Linux:   vcsa-ui-installer/lin64/installer

3. "Installieren" wählen

Stufe 1: Appliance deployen
   ESXi-Verbindung:
     ESXi-Host:     192.168.1.10
     HTTPS-Port:    443
     Benutzername:  root
     Passwort:      (ESXi root Passwort)
   → Zertifikat akzeptieren

   Appliance-Einstellungen:
     VM-Name:       vCenter
     Passwort:      (sicheres Passwort für root/administrator@vsphere.local)
     Größe:         Tiny (für Lab/KMU)
     Datastore:     datastore1

   Netzwerk:
     IP-Adresse:    192.168.1.20
     Subnetz:       255.255.255.0
     Gateway:       192.168.1.1
     DNS:           192.168.1.1
     Hostname:      vcenter.firma.local

   → Fertig stellen (Deployment ~10–15 Minuten)

Stufe 2: Appliance konfigurieren
   NTP-Server:    pool.ntp.org
   SSH:           Aktivieren (optional)
   SSO-Konfiguration:
     SSO Domain:  vsphere.local
     Passwort:    (Administrator-Passwort)
   → Fertig stellen (~15 Minuten)
```

### vSphere Client öffnen

```
Browser → https://vcenter.firma.local (oder IP)
→ "vSphere Client starten"
→ Login: administrator@vsphere.local
→ Passwort: (beim Deployment gesetzt)
```

---

## Infrastruktur-Hierarchie

vCenter organisiert die Infrastruktur in einer Baumstruktur:

```
vCenter Server
└── Datacenter: "Rechenzentrum-Berlin"
    ├── Cluster: "Prod-Cluster"
    │   ├── Host: esxi01.firma.local
    │   ├── Host: esxi02.firma.local
    │   └── Host: esxi03.firma.local
    ├── Cluster: "Dev-Cluster"
    │   └── Host: esxi04.firma.local
    └── Standalone Host: esxi05.firma.local (kein Cluster)
```

### Datacenter und Cluster erstellen

#### GUI

```
vSphere Client → Menü → Hosts und Cluster

Datacenter erstellen:
→ vCenter-Objekt → Rechtsklick → "Neues Datacenter"
→ Name: Rechenzentrum-Berlin → OK

Cluster erstellen:
→ Datacenter → Rechtsklick → "Neuer Cluster"
→ Name: Prod-Cluster
→ DRS aktivieren: ✓
→ HA aktivieren: ✓
→ EVC aktivieren: ✓ (AMD oder Intel Modus wählen)
→ vSAN: ○ (optional)
→ Fertig stellen

Host zum Cluster hinzufügen:
→ Cluster → Rechtsklick → "Host hinzufügen"
→ Hostname/IP: 192.168.1.10
→ Verbindung: root / Passwort
→ Fingerabdruck bestätigen
→ Lizenz zuweisen
→ Fertig stellen
```

---

## High Availability (HA) konfigurieren

### GUI

```
Cluster → Konfigurieren → vSphere-Verfügbarkeit → Bearbeiten

vSphere HA:
  ✓ vSphere HA aktivieren

  Host-Überwachung:
    ✓ Host-Überwachung aktivieren

  Zugangskontrolle:
    ⦿ Clusterressourcen reservieren
    → Hosts für Ausfallschutz: 1
    (reserviert 1/N der Ressourcen als Reserve)

  VM-Neustartreihenfolge:
    ✓ VM-Neustart-Priorität
    → Kritische VMs: Hoch
    → Standard-VMs: Mittel

  VM-Überwachung:
    ✓ VM-Überwachung aktivieren
    (Neustart wenn VMware-Tools-Heartbeat ausbleibt)

→ OK
```

---

## Distributed Resource Scheduler (DRS) konfigurieren

### GUI

```
Cluster → Konfigurieren → vSphere DRS → Bearbeiten

DRS:
  ✓ vSphere DRS aktivieren

  DRS-Automatisierung:
    ⦿ Vollautomatisch (empfohlen für Produktion)
    Migrationsschwelle: 3 (Mitte — Balance zwischen Stabilität und Optimierung)

  Predictive DRS:
    ✓ Aktivieren (wenn vROps integriert)

  VM-Automatisierungsgrad (individual per VM möglich):
    → Standard: Cluster-Standard verwenden

→ OK
```

---

## Content Library

Die Content Library speichert zentral **VM-Templates, ISOs und OVA/OVF-Dateien**.

### GUI

```
Menü → Content Libraries → Erstellen

Name: "Firm-Library"
Typ:
  ⦿ Lokale Inhaltsbibliothek
  (oder: Abonnierte Bibliothek von URL für öffentliche Templates)

Storage: datastore1
→ Fertig stellen

Template veröffentlichen:
→ VM → Rechtsklick → Template → Als Template in Bibliothek klonen
→ Template-Name, Bibliothek auswählen → OK

VM aus Template erstellen:
→ Content Library → Template → Rechtsklick → Neue VM aus diesem Template
→ VM-Name, Ziel-Datacenter/Cluster, Storage, Netzwerk → Fertig stellen
```

---

## Benutzer und Rollen

### Globale vCenter-Rollen

| Rolle | Beschreibung |
|-------|-------------|
| **Administrator** | Vollzugriff |
| **Read-Only** | Nur lesen, keine Änderungen |
| **No Access** | Kein Zugriff (Standard für neue Objekte) |
| **Virtual Machine Power User** | VMs starten/stoppen/konfigurieren |
| **Virtual Machine User** | VMs nur starten/stoppen/konsole |
| **Datacenter Administrator** | Verwaltet ein Datacenter vollständig |

### GUI — Benutzer Berechtigung vergeben

```
vCenter/Datacenter/Cluster/Host → Rechtsklick → Berechtigungen hinzufügen
→ Benutzer/Gruppe: FIRMA\vSphere-Admins (AD-Gruppe)
→ Rolle: Administrator
→ ✓ Auf untergeordnete Objekte übertragen
→ OK
```

---

## Verwandte Notizen

- [[ESXi Grundlagen]]
- [[ESXi Installation]]
- [[ESXi Netzwerk und Storage]]
- [[ESXi Troubleshooting]]

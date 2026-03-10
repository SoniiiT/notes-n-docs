---
tags:
  - virtualization
  - proxmox
  - gpu
  - passthrough
  - vfio
  - iommu
  - gaming
  - ai-ml
aliases:
  - Proxmox GPU Passthrough
  - PVE VFIO
  - Proxmox PCI Passthrough
---

# Proxmox GPU Passthrough

> [!ABSTRACT] TL;DR
> Mit **PCI-Passthrough (VFIO)** kann eine physische GPU direkt und exklusiv an eine VM durchgereicht werden — die VM nutzt die GPU ohne Virtualisierungsschicht mit nahezu nativer Performance. Voraussetzungen: IOMMU im BIOS aktivieren, VFIO-Treiber binden, GPU aus dem Host-System entfernen. NVIDIA erfordert zusätzliche Patches (`vendor-id`); AMD-GPUs funktionieren in der Regel problemloser.

---

## Anwendungsfälle

| Use Case | Beschreibung |
|---|---|
| **Gaming-VM** | Windows-VM mit voller GPU-Performance für Spiele |
| **AI / ML Workloads** | CUDA/ROCm in einer isolierten VM (PyTorch, TensorFlow) |
| **CAD / 3D-Rendering** | Workstation-VMs mit dedizierter GPU |
| **Video-Transcoding** | Hardware-Encoding in einer VM (NVENC, AMF, QuickSync) |
| **Remote-Workstation** | GPU-beschleunigte Remote-Desktop-VMs |

---

## Voraussetzungen

### Hardware

| Anforderung | Details |
|---|---|
| **CPU** | Intel VT-d **oder** AMD-Vi (IOMMU) aktiviert im BIOS |
| **Mainboard** | IOMMU-Unterstützung (die meisten Server- und modernen Desktop-Boards) |
| **GPU** | Dedizierte GPU (NVIDIA, AMD, Intel Arc) — keine integrierte iGPU |
| **Separate IOMMU-Gruppe** | GPU muss alleine in einer IOMMU-Gruppe sein (sonst weitere Geräte mitreichen) |

> [!IMPORTANT] IOMMU-Gruppen prüfen
> Wenn die GPU nicht alleine in ihrer IOMMU-Gruppe ist, müssen **alle Geräte in der Gruppe** an die VM durchgereicht werden — oder es wird ACS Override Patch verwendet (instabil, nicht empfohlen für Produktion).

---

## Schritt 1: IOMMU aktivieren

### BIOS-Einstellungen

| CPU-Hersteller | BIOS-Einstellung |
|---|---|
| **Intel** | `Intel VT-d` aktivieren |
| **AMD** | `AMD-Vi` oder `IOMMU` aktivieren (je nach Board) |

### Kernel-Parameter setzen

```bash
# GRUB-Konfiguration öffnen
nano /etc/default/grub

# Intel:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# AMD:
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"

# GRUB aktualisieren und neu starten
update-grub
reboot

# IOMMU-Aktivierung prüfen
dmesg | grep -e DMAR -e IOMMU -e AMD-Vi | head -20
# Erwartete Ausgabe (Intel): "DMAR: IOMMU enabled"
# Erwartete Ausgabe (AMD): "AMD-Vi: IOMMU found"
```

---

## Schritt 2: IOMMU-Gruppen prüfen

```bash
# Skript: Alle IOMMU-Gruppen und ihre Geräte anzeigen
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done | sort -V
```

```bash
# Als Einzeiler (direkter Ausführen):
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done | sort -V | grep -A2 -B2 "VGA\|3D\|Display"
```

Beispielausgabe — GPU alleine in Gruppe 16 (ideal):
```
IOMMU Group 16 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [RTX 3080] [10de:2206]
IOMMU Group 16 01:00.1 Audio device [0403]: NVIDIA Corporation GA102 High Definition Audio [10de:1aef]
```

> [!NOTE] GPU-Audio mitreichen
> Die HDMI/DP-Audiokomponente der GPU (`01:00.1`) ist Teil der IOMMU-Gruppe und muss ebenfalls an die VM durchgereicht werden — sonst funktioniert Audio über die GPU nicht.

---

## Schritt 3: VFIO-Treiber laden

Der Host-Kernel darf die GPU **nicht** selbst nutzen — VFIO übernimmt die Kontrolle:

```bash
# Kernel-Module in /etc/modules hinzufügen
echo "vfio" >> /etc/modules
echo "vfio_iommu_type1" >> /etc/modules
echo "vfio_pci" >> /etc/modules
echo "vfio_virqfd" >> /etc/modules

# GPU vom Host-Treiber trennen (Vendor-ID und Device-ID der GPU)
# IDs aus lspci ermitteln:
lspci -nn | grep -E "VGA|3D|Audio" | grep -i nvidia
# Beispiel: [10de:2206] (GPU) und [10de:1aef] (Audio)

# VFIO für diese IDs konfigurieren
echo "options vfio-pci ids=10de:2206,10de:1aef" > /etc/modprobe.d/vfio.conf

# Sicherstellen dass VFIO VOR dem NVIDIA/AMD-Treiber geladen wird
echo "softdep nouveau pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
echo "softdep nvidia pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
echo "softdep amdgpu pre: vfio-pci" >> /etc/modprobe.d/vfio.conf
echo "softdep radeon pre: vfio-pci" >> /etc/modprobe.d/vfio.conf

# Initramfs aktualisieren
update-initramfs -u -k all

# Neustart
reboot

# Prüfen: GPU sollte jetzt unter vfio-pci laufen
lspci -nnk | grep -A3 "VGA\|3D"
# Erwartet: "Kernel driver in use: vfio-pci"
```

---

## Schritt 4: Host-Grafiktreiber blacklisten

Wenn der Proxmox-Host keinen Monitor benötigt (Headless-Server), müssen die GPU-Treiber geblockt werden:

```bash
# NVIDIA blacklisten
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidiafb" >> /etc/modprobe.d/blacklist.conf

# AMD blacklisten
echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
echo "blacklist amdgpu" >> /etc/modprobe.d/blacklist.conf

# Initramfs und GRUB aktualisieren
update-initramfs -u -k all
update-grub
reboot
```

---

## Schritt 5: VM für GPU-Passthrough konfigurieren

### VM-Grundkonfiguration (wichtig!)

```bash
# VM erstellen / konfigurieren mit korrekten Einstellungen für Passthrough
qm set 100 --machine q35           # q35 Chipsatz (Pflicht für PCIe)
qm set 100 --bios ovmf             # UEFI (Pflicht für moderne GPUs)
qm set 100 --cpu host              # Host-CPU-Features durchreichen

# Hyper-V Enlightenments für Windows-VMs aktivieren (bessere Performance)
qm set 100 --cpu host,hv_vendor_id=AuthenticAMD    # NVIDIA-Workaround (siehe unten)
```

### GPU an VM durchreichen

```bash
# PCI-Adresse der GPU ermitteln
lspci | grep -E "VGA|3D"
# Beispiel: 01:00.0 VGA ... (PCI-Adresse: 0000:01:00.0)

# GPU via Web-UI:
# VM → Hardware → Add → PCI Device
# Device: 0000:01:00.0
# ✅ All Functions (GPU + Audio zusammen)
# ✅ Primary GPU (wenn keine andere Display im System)
# ✅ PCI-Express
# ✅ ROM-Bar

# GPU via CLI hinzufügen:
qm set 100 --hostpci0 0000:01:00,allFunctions=1,pcie=1,rombar=1,x-vga=1

# Erklärung der Parameter:
# allFunctions=1  → GPU + Audio-Device (01:00.0 + 01:00.1)
# pcie=1          → PCIe statt PCI emulation
# rombar=1        → ROM-Bar aktivieren (für vBIOS)
# x-vga=1         → Als primäre VGA markieren
```

### Display-Konfiguration für GPU-Passthrough-VMs

```bash
# VirtIO-Display deaktivieren — GPU übernimmt vollständig
qm set 100 --vga none

# Oder: Serial-Console als Notfall-Zugang behalten
qm set 100 --vga none
qm set 100 --serial0 socket
```

---

## NVIDIA-spezifische Konfiguration

NVIDIA-Treiber erkennen Virtualisierungsumgebungen und verweigern die Arbeit (Error 43). Lösung: Hypervisor-Fingerprint verschleiern.

### Vendor-ID Patch (NVIDIA Error 43)

```bash
# In der VM-Konfiguration /etc/pve/qemu-server/100.conf:
# args: -cpu host,hv_vendor_id=AuthenticAMD,kvm=off
# ODER:

qm set 100 --args "-cpu host,hv_vendor_id=AuthenticAMD,kvm=off"

# Alternativ: Hyper-V Enlightenments mit custom vendor-id
qm set 100 --cpu "host,hidden=1,flags=+pcid"
```

```bash
# Vollständige /etc/pve/qemu-server/100.conf Beispiel für NVIDIA:
# agent: enabled=1
# args: -cpu host,hv_vendor_id=GenuineIntel
# bios: ovmf
# boot: order=scsi0
# cores: 8
# cpu: host,hidden=1
# hostpci0: 0000:01:00,allFunctions=1,pcie=1,rombar=1,x-vga=1
# machine: q35
# memory: 16384
# name: gaming-vm
# net0: virtio,bridge=vmbr0
# ostype: win11
# scsi0: local-lvm:vm-100-disk-0,size=100G
# scsihw: virtio-scsi-single
# serial0: socket
# sockets: 1
# vga: none
```

### vBIOS ROM dumpen (falls nötig)

Manche NVIDIA-GPUs benötigen einen ROM-Dump des vBIOS:

```bash
# ROM-Dump der GPU (GPU muss noch unter nvidia-Treiber laufen — vor VFIO-Bindung)
# GPU PCI-Adresse: 01:00.0

# 1. Temporär VFIO deaktivieren und nvidia-Treiber laden
# 2. ROM dumpen:
echo 1 > /sys/bus/pci/devices/0000:01:00.0/rom
cat /sys/bus/pci/devices/0000:01:00.0/rom > /usr/share/kvm/nvidia-rtx3080.rom
echo 0 > /sys/bus/pci/devices/0000:01:00.0/rom

# 3. ROM in VM-Config referenzieren:
qm set 100 --hostpci0 0000:01:00,allFunctions=1,pcie=1,romfile=nvidia-rtx3080.rom,x-vga=1
```

---

## AMD-spezifische Konfiguration

AMD-GPUs sind in der Regel einfacher für Passthrough — kein Error-43-Problem, kein vBIOS-Dump nötig bei den meisten Karten.

```bash
# AMD GPU durchreichen (RDNA2/RDNA3 — RX 6000/7000 Serie)
qm set 100 --hostpci0 0000:03:00,allFunctions=1,pcie=1,x-vga=1

# AMD Reset-Bug (ältere GPUs vor RDNA2)
# Manche AMD-GPUs können nach VM-Neustart nicht zurückgesetzt werden
# Lösung: vendor-reset Kernel-Modul
apt install pve-headers-$(uname -r)
# vendor-reset aus den Quellen bauen oder DKMS-Paket:
# https://github.com/gnif/vendor-reset

# AMD GPU ohne Reset-Problem (RDNA2+) — kein Workaround nötig
```

---

## Intel Arc GPU Passthrough

```bash
# Intel Arc (Alchemist) ab PVE 8.x mit Kernel 6.2+
# SR-IOV für Intel Arc aktivieren (mehrere VMs teilen eine GPU)

# i915 SR-IOV Patch laden
echo "options i915 enable_guc=3" > /etc/modprobe.d/i915.conf
update-initramfs -u

# Virtuelle Funktionen erstellen
echo 7 > /sys/class/drm/card0/device/sriov_numvfs

# VF an VM durchreichen
lspci | grep "Arc"
qm set 100 --hostpci0 0000:00:02.1,pcie=1
```

---

## SR-IOV (GPU teilen zwischen mehreren VMs)

SR-IOV ermöglicht das Aufteilen einer physischen GPU in virtuelle Funktionen (VFs) für mehrere VMs gleichzeitig:

| GPU-Typ | SR-IOV-Unterstützung | Bemerkung |
|---|---|---|
| **NVIDIA vGPU** | ✅ | Lizenzpflichtig (GRID-Lizenz) — Enterprise-Feature |
| **AMD MxGPU** | ✅ | Lizenzpflichtig (für Instinct/Radeon Pro) |
| **Intel Arc** | ✅ | Open-Source, kostenlos (Kernel 6.2+) |
| **Intel UHD (iGPU)** | ✅ | SR-IOV via GVT-g / i915 (ältere Gens) |

```bash
# NVIDIA vGPU (erfordert Enterprise-Lizenz und vGPU-Treiber)
# 1. vGPU-Treiber installieren (von NVIDIA-Lizenzserver)
# 2. mdev-Gerät erstellen
nvidia-smi vgpu -c 100 -m MIG-4g.40gb   # 4 GPU-Abschnitte, 40 GB

# 3. mdev-UUID ermitteln und in VM einbinden
ls /sys/bus/mdev/devices/
qm set 100 --hostpci0 0000:01:00,mdev=nvidia-256
```

---

## Troubleshooting

### Error 43 (NVIDIA)

```
Symptom: Geräteverwaltung zeigt "Code 43" für NVIDIA GPU in Windows-VM
```

```bash
# Lösung 1: Hypervisor-Kennung verschleiern
qm set 100 --cpu "host,hidden=1"
# Und in args:
qm set 100 --args "-cpu host,hv_vendor_id=GenuineIntel"

# Lösung 2: KVM-Flag deaktivieren
qm set 100 --args "-cpu host,kvm=off"
```

### GPU nicht in VFIO (noch Host-Treiber aktiv)

```bash
# Aktuellen Treiber prüfen
lspci -nnk | grep -A3 "01:00.0"
# "Kernel driver in use: nvidia" → VFIO nicht aktiv

# VFIO-Bindung prüfen
cat /etc/modprobe.d/vfio.conf
# Muss enthalten: options vfio-pci ids=10de:xxxx,...

# Initramfs neu erstellen und neu starten
update-initramfs -u -k all
reboot

# Nach Neustart prüfen:
lspci -nnk | grep -A3 "01:00.0"
# "Kernel driver in use: vfio-pci" → Korrekt
```

### Schwarzer Bildschirm in der VM

```bash
# Ursache 1: Falsches Display-Setting
# vga muss "none" sein wenn GPU Passthrough aktiv
qm config 100 | grep vga
# Setzen: qm set 100 --vga none

# Ursache 2: UEFI / Secure Boot Problem
# Windows: Secure Boot temporär deaktivieren zum Test
# OVMF-BIOS in VM → Del drücken → Secure Boot deaktivieren

# Ursache 3: IOMMU-Gruppe nicht vollständig durchgereicht
# Alle Geräte der IOMMU-Gruppe prüfen und alle hinzufügen:
for d in /sys/kernel/iommu_groups/16/devices/*; do lspci -nns "${d##*/}"; done
# Alle IDs müssen in hostpci0 mit allFunctions=1 oder einzeln als hostpci1, hostpci2 stehen
```

### VM startet nicht nach GPU-Hinzufügen

```bash
# Fehlerlog der VM prüfen
cat /var/log/pve/qemu-server/100.log | tail -30

# Häufige Fehler:
# "vfio: Failed to open /dev/vfio/16: No such file or directory"
# → IOMMU nicht aktiv oder VFIO nicht geladen
dmesg | grep vfio
dmesg | grep IOMMU

# "romfile '/usr/share/kvm/...' not found"
# → ROM-Datei fehlt oder falscher Pfad
ls /usr/share/kvm/
```

---

## Verwandte Notizen

- [[Was ist Proxmox]]
- [[Proxmox Installation]]
- [[Proxmox VM und Container Verwaltung]]
- [[Proxmox Best Practices]]
- [[Proxmox Troubleshooting]]

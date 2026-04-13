# proxmox-android-vm

Ansible Playbook zur automatisierten Erstellung einer Android-VM auf einem Proxmox VE Node.

Die VM basiert auf **Bliss OS 16.x** und erstellt eine **normale Bliss-OS-x86-VM auf KVM/Proxmox**. Ein dauerhaftes und verlässliches **Pixel-1-Spoofing ist in dieser Umgebung nicht Teil des Playbooks**.

---

## Voraussetzungen

### Auf dem Ansible-Host
- Ansible >= 2.14
- `community.proxmox` Collection:
  ```bash
  ansible-galaxy collection install community.proxmox
  ```
- SSH-Zugang zum Proxmox Node als `root`

### Auf dem Proxmox Node
- Proxmox VE 8.x oder 9.x
- Bliss OS ISO wurde **manuell** heruntergeladen und in den ISO-Storage des Nodes gelegt
- `jq` und `python3-proxmoxer` werden automatisch installiert, falls nicht vorhanden

---

## Variablen

Alle Variablen können in der Inventory-Datei oder per `-e` auf der CLI überschrieben werden.

| Variable | Standard | Beschreibung |
|---|---|---|
| `proxmox_host` | – | IP/Hostname des PVE Node |
| `proxmox_node` | `pve` | Node-Name in der PVE-Weboberfläche |
| `proxmox_user` | `root@pam` | API-Benutzer |
| `proxmox_password` | *(Prompt)* | Wird bei Ausführung abgefragt |
| `vm_name` | `android-bliss` | VM-Name in Proxmox |
| `vm_cores` | `2` | Anzahl vCPUs |
| `vm_memory` | `8192` | RAM in MB |
| `vm_disk_size` | `128G` | Disk-Größe |
| `vm_storage` | `data` | Proxmox Storage für die VM-Disk |
| `vm_bridge` | `vmbr0` | Netzwerk-Bridge |
| `vm_iso_storage` | `local` | Proxmox Storage-Name für ISO-Ablage |
| `vm_start_after_create` | `true` | VM nach Erstellung starten |
| `blissos_iso_filename` | `Bliss-v16.9.7-x86_64-OFFICIAL-gapps-20241011.iso` | Dateiname des manuell abgelegten Bliss-OS-ISOs |

---

## Inventory

Kopiere `inventory/hosts.yml.example` nach `inventory/hosts.yml` und passe die Werte an:

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
```

---

## Ausführung

### Standard (Passwort wird interaktiv abgefragt)
```bash
ansible-playbook playbook.yml -i inventory/hosts.yml
```

---

## Was das Playbook tut

1. **Verifikation** – Prüft ob der Zielhost ein echter Proxmox VE Node ist
2. **Dependencies** – Installiert `jq` und `python3-proxmoxer` via apt
3. **ISO-Pfad ermitteln** – Liest den Storage-Pfad dynamisch via `pvesm` vom Node
4. **ISO verifizieren** – Prüft ob das angegebene Bliss-OS-ISO im Storage vorhanden ist
5. **VM erstellen** – Erstellt eine UEFI/OVMF-VM mit VirtIO-Disk und hängt das ISO ein
6. **VM starten** – Optional via `vm_start_after_create`

---

## Wichtige Hinweise

- Das Playbook **patched das ISO nicht**. Frühere Ansätze über `virt-customize` wurden entfernt, weil das verwendete Bliss-OS-16-ISO ein `/system.efs` enthält und kein beschreibbares `system.sfs` oder `system.img`.
- Die bereitgestellte VM ist daher eine **normale Bliss-OS-x86-KVM-VM** ohne eingebautes Pixel-1-Spoofing.
- Auch mit manuellen Änderungen an `build.prop` ist ein verlässliches Bestehen von SafetyNet / Play Integrity in einer Proxmox-KVM-VM in der Regel **nicht realistisch**, weil Hypervisor-, Firmware- und Attestation-Merkmale weiterhin sichtbar bleiben.
- Die VM-ID wird automatisch als nächste freie ID von Proxmox vergeben.
- Das Passwort wird bei jeder Ausführung interaktiv abgefragt und nie gespeichert.

---

## Optionale Nacharbeiten in der VM

Wenn Root-Hiding oder App-Kompatibilität verbessert werden soll, sind manuelle Schritte **nach der Installation innerhalb der VM** erforderlich.

### Magisk + Shamiko (optional)

1. **Bliss OS auf die VM-Disk installieren** – nicht nur im Live-Modus booten.
2. **ADB in Bliss OS aktivieren** – Entwickleroptionen freischalten und USB-Debugging einschalten.
3. **Magisk APK installieren** – z. B. per `adb install`.
4. **`initrd.img` mit Magisk patchen** – auf Android-x86/Bliss OS typischerweise nicht `boot.img`, sondern `initrd.img`.
5. **Gepatchtes `initrd.img` zurückspielen und neu booten**.
6. **Zygisk in Magisk aktivieren** und erneut booten.
7. **Shamiko als Magisk-Modul installieren**.
8. **DenyList pflegen** – Ziel-Apps und relevante Prozesse aufnehmen; `Enforce DenyList` für Shamiko typischerweise deaktiviert lassen.
9. **Optional Play-Integrity-Fix testen** – mit der Erwartung, dass KVM/Proxmox-VMs trotzdem oft an starker Geräteattestierung scheitern.

### Erwartungsmanagement

- **Root vor einfachen Prüfungen verstecken:** oft möglich
- **Banking-Apps / starke Integritätsprüfungen zuverlässig bestehen:** eher unwahrscheinlich
- **Pixel-1-Spoofing auf Attestation-Niveau:** in KVM/Proxmox praktisch nicht zuverlässig erreichbar

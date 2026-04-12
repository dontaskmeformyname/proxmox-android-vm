# proxmox-android-vm

Ansible Playbook zur automatisierten Erstellung einer Android-VM auf einem Proxmox VE Node.

Die VM basiert auf **Bliss OS 16.x** und gibt sich gegenüber Google-Diensten als **Google Pixel 1 (sailfish, Android 10)** aus. Primärer Anwendungsfall: Syncthing-Synchronisation von Fotos → Google Fotos (unbegrenzter Original-Speicher via Pixel-1-Berechtigung).

---

## Voraussetzungen

### Auf dem Ansible-Host
- Ansible >= 2.14
- Community General Collection:
  ```bash
  ansible-galaxy collection install community.general
  ```
- SSH-Zugang zum Proxmox Node als `root`

### Auf dem Proxmox Node
- Proxmox VE 8.x oder 9.x
- Internetverbindung (für ISO-Download)
- `libguestfs-tools` und `wget` werden automatisch installiert, falls nicht vorhanden

---

## Variablen

Alle Variablen können in der Inventory-Datei oder per `-e` auf der CLI überschrieben werden.

| Variable | Standard | Beschreibung |
|---|---|---|
| `proxmox_host` | – | IP/Hostname des PVE Node |
| `proxmox_node` | `pve` | Node-Name in der PVE-Weboberfläche |
| `proxmox_user` | `root@pam` | API-Benutzer |
| `proxmox_password` | – | Passwort (empfohlen: ansible-vault) |
| `vm_name` | `android-pixel` | VM-Name in Proxmox |
| `vm_hostname` | `pixel1-android` | Hostname/Netzwerkname der VM |
| `vm_cores` | `2` | Anzahl vCPUs |
| `vm_memory` | `8192` | RAM in MB |
| `vm_disk_size` | `128G` | Disk-Größe |
| `vm_storage` | `data` | Proxmox Storage für die VM-Disk |
| `vm_bridge` | `vmbr0` | Netzwerk-Bridge |
| `vm_iso_storage` | `local` | Proxmox Storage-Name für ISO-Downloads |
| `vm_network_mode` | `dhcp` | Netzwerkmodus: `dhcp` oder `static` |
| `vm_ip` | `""` | IP mit Prefix, z.B. `192.168.1.50/24` (nur bei `static`) |
| `vm_gateway` | `""` | Standard-Gateway (nur bei `static`) |
| `vm_dns` | `""` | DNS-Server (nur bei `static`) |
| `vm_start_after_create` | `true` | VM nach Erstellung starten |
| `blissos_iso_url` | *(Bliss OS 16 URL)* | Download-URL des Bliss OS ISO |

---

## Inventory

Kopiere `inventory/hosts.yml.example` nach `inventory/hosts.yml` und passe die Werte an:

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
```

---

## Ausführung

### Mit Passwort-Prompt (empfohlen)
```bash
ansible-playbook playbook.yml -i inventory/hosts.yml -k
```

### Mit ansible-vault (Passwort verschlüsselt)
```bash
ansible-vault encrypt_string 'deinPasswort' --name 'proxmox_password'
# Ausgabe in hosts.yml einfügen, dann:
ansible-playbook playbook.yml -i inventory/hosts.yml --ask-vault-pass
```

### Einzelnen Host per CLI angeben
```bash
ansible-playbook playbook.yml -i inventory/hosts.yml \
  -e proxmox_host=192.168.1.10 \
  -e proxmox_password=secret \
  -k
```

### Statische IP-Konfiguration
```bash
ansible-playbook playbook.yml -i inventory/hosts.yml \
  -e vm_network_mode=static \
  -e vm_ip=192.168.1.50/24 \
  -e vm_gateway=192.168.1.1 \
  -e vm_dns=192.168.1.1 \
  -k
```

---

## Was das Playbook tut

1. **Verifikation** – Prüft ob der Zielhost ein echter Proxmox VE Node ist
2. **Dependencies** – Installiert `libguestfs-tools` und `wget` via apt
3. **ISO-Pfad ermitteln** – Liest den Storage-Pfad dynamisch via `pvesm` vom Node
4. **ISO Download** – Lädt Bliss OS 16.x herunter
5. **ISO patchen** – Setzt Pixel-1-Identität via `virt-customize`:
   - `ro.product.model=Pixel`
   - `ro.product.device=sailfish`
   - `ro.build.fingerprint=google/sailfish/sailfish:10/...`
   - Netzwerkkonfiguration (DHCP oder statisch)
6. **VM erstellen** – UEFI/OVMF, VirtIO-Disk, gepatchtes ISO eingehängt
7. **VM starten** – Optional via `vm_start_after_create`

---

## Hinweise

- Das Device-Spoofing (Pixel 1 Fingerprint) ist bereits im gepatchten ISO enthalten – keine manuellen ADB-Schritte nach der Installation nötig.
- Die VM-ID wird automatisch als nächste freie ID von Proxmox vergeben.
- Das Passwort sollte in produktiven Umgebungen immer via `ansible-vault` verschlüsselt werden.

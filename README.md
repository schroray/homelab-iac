
# Homelab Infrastructure as Code (IaC)

Automatisierung und Verwaltung meiner Homelab-Infrastruktur mit **Ansible**, **Proxmox VE** und **GitHub**.

## 1. Projektziele

Dieses Repository dient als zentrale Grundlage für die deklarative Verwaltung und Automatisierung meines Homelabs.

### Schwerpunkte

- **Day-0 - Provisioning:** Virtuelle Maschinen auf Proxmox bereitstellen
- **Day-1 - Configuration:** Betriebssysteme, Benutzer, Software und Dienste konfigurieren
- **Day-2 - Operations:** Patching, Wartung und Zustandsprüfungen automatisieren
- **Infrastructure as Code:** Konfigurationen versionieren und reproduzierbar verwalten
- **Security:** Least Privilege, sichere Secret-Verwaltung und nachvollziehbare Änderungen
- **Learning:** Praktische Anwendung von Ansible, orientiert am Red-Hat-Kurs AU094

**Primäre Virtualisierungsplattform:** Proxmox VE

**Primäres Git-Repository:** GitHub

---

## 2. Architektur

```text
Windows 11
└── Ubuntu WSL2
    ├── Visual Studio Code
    ├── Git / GitHub
    ├── Python Virtual Environment
    └── Ansible Control Node
         │
         ├── HTTPS / Proxmox API (TCP 8006)
         │    └── proxmox-01
         │         ├── Cloud-Init-Template (VMID 8000)
         │         └── vm-test-01 (VMID 200, geplant)
         │
         └── SSH (TCP 22)
              └── Linux-VMs
                   ├── Benutzer und SSH
                   ├── Software und Pakete
                   ├── Dienste
                   ├── Patching
                   └── Compliance-Prüfungen
```

### Infrastruktur

| System | Adresse | Funktion |
|---|---|---|
| Ubuntu WSL2 | Lokal | Ansible Control Node |
| proxmox-01 | 192.168.0.101 | Primärer Proxmox-Host |
| proxmox-02 | 192.168.0.110 | Weiterer Proxmox-Host |
| vm-test-01 | DHCP | Geplante Test-VM |
| GitHub | Remote | Versionsverwaltung |

**Provisionierungsziel:** Neue VMs werden aktuell ausschliesslich auf `proxmox-01` erstellt.

Das Ubuntu-Cloud-Init-Template mit VMID `8000` liegt auf `proxmox-01`.

---

## 3. Technologie-Stack

| Komponente | Version / Technologie |
|---|---|
| Control Node | Ubuntu unter WSL2 |
| Python | 3.12.3 |
| Ansible Core | 2.21.5 |
| Ansible Lint | 26.10.1 (definiert) |
| Proxmox Collection | community.proxmox 2.0.0 |
| Proxmoxer | 2.3.0 (definiert) |
| Requests | 2.34.2 (definiert) |
| IDE | Visual Studio Code |
| Versionsverwaltung | Git / GitHub |

Die in `requirements.txt` und `requirements.yml` definierten Versionen sollen reproduzierbare Installationen ermöglichen.

---

## 4. Repository-Struktur

```text
homelab-iac/
├── .env                     # Lokale Secrets (nicht versionieren)
├── .env.example             # Beispiel für Umgebungsvariablen
├── .gitignore
├── .vscode/
│   ├── extensions.json
│   └── settings.json
├── ansible.cfg
├── inventory/
│   └── hosts.yml
├── playbooks/
│   ├── 01-proxmox-info.yml
│   ├── 02-proxmox-health.yml
│   └── 03-proxmox-vm-plan.yml
├── vars/
│   ├── proxmox.yml
│   └── vms.yml
├── requirements.txt
├── requirements.yml
└── README.md
```

### Verantwortlichkeiten

| Verzeichnis / Datei | Funktion |
|---|---|
| `inventory/` | Verwaltete Systeme und Hostgruppen |
| `playbooks/` | Automatisierungsabläufe |
| `vars/` | Infrastruktur- und VM-Konfigurationen |
| `requirements.txt` | Python-Abhängigkeiten |
| `requirements.yml` | Ansible Collections |
| `.env` | Lokale API-Zugangsdaten |
| `ansible.cfg` | Projektweite Ansible-Konfiguration |

---

## 5. Ansible Control Node

### Bestehende Installation

Ansible ist unter Ubuntu WSL2 in einer Python Virtual Environment installiert:

```bash
~/.venvs/ansible
```

Die Ansible-CLI ist über Symlinks in `~/.local/bin` verfügbar.

Damit ist die manuelle Aktivierung der Virtual Environment für normale Ansible-Befehle nicht erforderlich.

### Installation überprüfen

```bash
ansible --version
ansible-playbook --version
ansible-galaxy --version
ansible-inventory --version
```

### Virtual Environment bei Bedarf aktivieren

```bash
source ~/.venvs/ansible/bin/activate
```

Die Aktivierung wird beispielsweise für direkte Python- oder pip-Operationen innerhalb der Umgebung benötigt.

### Installation auf einem neuen Control Node

```bash
python3 -m venv ~/.venvs/ansible

source ~/.venvs/ansible/bin/activate

python -m pip install -r requirements.txt

ansible-galaxy collection install -r requirements.yml
```

Anschliessend können die Ansible-Befehle entweder über die aktivierte Virtual Environment oder über passende Symlinks bereitgestellt werden.

---

## 6. Proxmox-Authentifizierung

Die Proxmox-API wird für Clusterabfragen, Health Checks und spätere VM-Provisionierungen verwendet.

### Umgebungsvariablen

Die Zugangsdaten werden lokal in `.env` verwaltet.

Die API-Playbooks erwarten unter anderem:

```text
PROXMOX_USER
```

Weitere erforderliche Variablennamen ergeben sich aus `.env.example` und den bestehenden Playbooks.

**API-Tokens, Passwörter und private Schlüssel dürfen nicht im Git-Repository gespeichert werden.**

### Lokale .env laden

```bash
cd ~/git/homelab-iac

set -a
source .env
set +a
```

Dadurch werden die Variablen in die aktuelle Shell-Umgebung exportiert.

**Hinweis:** `source` führt den Dateiinhalt als Shell-Code aus. Deshalb ausschliesslich eine eigene, vertrauenswürdige `.env` verwenden.

### Authentifizierung prüfen

```bash
test -n "$PROXMOX_USER" \
  && echo "Proxmox API user configured"
```

### Sicherheitsanforderungen

- Dedizierte Proxmox-API-Identität verwenden
- Least-Privilege-Berechtigungen konfigurieren
- TLS-Zertifikatsprüfung aktivieren
- Secrets nicht in Git speichern
- Sensible Tasks mit `no_log: true` schützen
- Langfristig Ansible Vault oder Secret Management verwenden

---

## 7. Ansible Inventory

Datei: `inventory/hosts.yml`

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local

  children:
    proxmox_nodes:
      hosts:
        proxmox-01:
          ansible_host: 192.168.0.101

        proxmox-02:
          ansible_host: 192.168.0.110
```

### Inventory überprüfen

```bash
ansible-inventory --graph
```

Das Inventory definiert Hosts und Hostgruppen.

Die Existenz eines Hosts im Inventory bedeutet nicht automatisch, dass eine funktionierende SSH- oder API-Verbindung besteht.

---

## 8. Bestehende Playbooks

| Playbook | Funktion | Status |
|---|---|---|
| `01-proxmox-info.yml` | Proxmox-Clusterinformationen abfragen | API-Test offen |
| `02-proxmox-health.yml` | Proxmox-Health-Checks | API-Test offen |
| `03-proxmox-vm-plan.yml` | VM-Definitionen lokal validieren | Erfolgreich getestet |

### Proxmox-Informationen

```bash
ansible-playbook playbooks/01-proxmox-info.yml
```

### Proxmox Health Check

```bash
ansible-playbook playbooks/02-proxmox-health.yml
```

### VM-Plan validieren

```bash
ansible-playbook \
  playbooks/03-proxmox-vm-plan.yml \
  --syntax-check

ansible-playbook \
  playbooks/03-proxmox-vm-plan.yml
```

Das Playbook prüft:

- Vorhandensein von VM-Definitionen
- Eindeutige VMIDs und Namen innerhalb der Definitionen
- Erforderliche VM-Parameter
- CPU-, RAM- und Disk-Angaben
- Zielnode, Storage und Netzwerk-Bridge

**Letztes bestätigtes Ergebnis:**

```text
PLAY RECAP

localhost : ok=4 changed=0 unreachable=0 failed=0
```

Dieses Playbook verändert keine Proxmox-Ressourcen.

Die tatsächliche Existenz von VMIDs, Templates, Storages und Bridges im Proxmox-Cluster wird noch nicht überprüft.

---

## 9. VM-Definitionen

Datei: `vars/vms.yml`

```yaml
---
proxmox_vms:
  - name: vm-test-01
    vmid: 200
    target_node: proxmox-01
    template_vmid: 8000

    cpu:
      cores: 2

    memory:
      mb: 4096

    disk:
      storage: nvme01-thin
      size_gb: 32

    network:
      bridge: vmbr0
      vlan: null

    cloud_init:
      ipconfig0: dhcp
      user: ansible
```

### Noch zu validierende Voraussetzungen

- VMID `200` ist clusterweit frei
- Template `8000` ist auf `proxmox-01` verfügbar
- Storage `nvme01-thin` ist auf `proxmox-01` vorhanden
- Netzwerk-Bridge `vmbr0` existiert
- Cloud-Init ist korrekt vorbereitet
- SSH-Public-Key ist für den Benutzer `ansible` konfiguriert
- Disk-Grösse kann mit dem Template umgesetzt werden

---

## 10. Automatisierungsphasen

### Day-0 – VM-Provisionierung

Geplante Playbooks:

```text
04-proxmox-vm-preflight.yml
05-proxmox-vm-deploy.yml
```

Ziele:

1. Proxmox-API-Verbindung validieren
2. VMID und VM-Namen clusterweit prüfen
3. Template, Storage und Netzwerk überprüfen
4. VM aus Cloud-Init-Template klonen
5. CPU, RAM, Disk und Netzwerk konfigurieren
6. VM starten und Verfügbarkeit überprüfen

### Day-1 – Betriebssystem-Konfiguration

Geplantes Playbook:

```text
06-linux-baseline.yml
```

Ziele:

- Benutzer und SSH konfigurieren
- Standardpakete installieren
- Dienste verwalten
- Betriebssystem-Basiskonfiguration anwenden
- Sicherheitsrelevante Einstellungen kontrollieren

### Day-2 – Betrieb und Wartung

Ziele:

- Betriebssystem-Updates installieren
- Reboots kontrollieren
- Dienste überprüfen
- Konfigurationsabweichungen erkennen
- Idempotenz nachweisen
- Wartungsergebnisse dokumentieren

---

## 11. Qualitäts- und Sicherheitsregeln

### Idempotenz

Playbooks sollen wiederholt ausführbar sein, ohne unnötige Änderungen auszulösen.

Bei bereits erreichtem Soll-Zustand soll ein erneuter Lauf idealerweise ergeben:

```text
changed=0
failed=0
```

### Syntaxprüfung

```bash
ansible-playbook \
  playbooks/03-proxmox-vm-plan.yml \
  --syntax-check
```

### Ansible Lint

```bash
ansible-lint playbooks/
```

### Check Mode

```bash
ansible-playbook \
  playbooks/03-proxmox-vm-plan.yml \
  --check --diff
```

Der Check Mode ersetzt keine API-Preflight-Prüfungen. Seine Aussagekraft hängt von der Unterstützung durch die verwendeten Module ab.

### Git und Secrets

Vor jedem Commit:

```bash
git status
git diff --check
git diff --cached
```

Prüfen, ob `.env` ignoriert wird:

```bash
git check-ignore .env
```

Falls `.env` bereits früher versioniert wurde, reicht `.gitignore` allein nicht aus. In diesem Fall müssen die Datei aus dem Git-Index entfernt und gegebenenfalls kompromittierte Secrets rotiert werden.

---

## 12. Lern- und Entwicklungsplan

| Phase | Thema | Status |
|---|---|---|
| 01 | WSL2 und Ansible installieren | Abgeschlossen |
| 02 | Git-Repository und Inventory | Vorhanden |
| 03 | Proxmox-API-Informationen | Playbook vorhanden, Test offen |
| 04 | Proxmox Health Checks | Playbook vorhanden, Test offen |
| 05 | VM-Plan validieren | Abgeschlossen |
| 06 | API-Preflight | Geplant |
| 07 | VM provisionieren | Geplant |
| 08 | Linux-Baseline | Geplant |
| 09 | Patching und Compliance | Geplant |
| 10 | Roles und Skalierung | Geplant |

---

## 13. Nächster Meilenstein

### Ziel

**VM `vm-test-01` auf `proxmox-01` vollständig automatisiert bereitstellen.**

### Nächste Aufgaben

1. Proxmox-API-Zugangsdaten aus `.env` laden
2. Bestehende API-Playbooks erfolgreich ausführen
3. API-Preflight für VMID, Template, Storage und Bridge implementieren
4. Provisionierungs-Playbook erstellen
5. Test-VM kontrolliert bereitstellen
6. SSH-Verbindung zur neuen VM herstellen
7. Erstes idempotentes Linux-Konfigurations-Playbook ausführen

---

## 14. Referenzen

### Ansible

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Getting Started](https://docs.ansible.com/projects/ansible/latest/getting_started/index.html)
- [Ansible Best Practices](https://docs.ansible.com/projects/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Ansible Community Proxmox Collection](https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/index.html)

### Proxmox

- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
- [Proxmox API Viewer](https://pve.proxmox.com/pve-docs/api-viewer/)

### Red Hat

- [Red Hat Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible)

---

## Projektstatus

**Stand:** 15. Oktober 2026

- Ansible-Control-Node funktionsfähig
- Ansible CLI ohne manuelle venv-Aktivierung verfügbar
- Git-Repository und Inventory vorhanden
- Proxmox-Collection installiert
- VM-Plan erfolgreich validiert
- Proxmox-API-Zugangsdaten müssen vor Ausführung geladen werden
- API-Preflight und VM-Provisionierung als nächste Schritte geplant

**Aktueller Fokus:** Proxmox VE, Ansible und Infrastructure as Code.

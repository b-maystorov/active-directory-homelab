# Active Directory Homelab

In diesem Projekt baue ich eine kleine Active-Directory-Umgebung mit Windows Server und Windows 11 in VirtualBox auf.

Das Ziel ist, Active Directory nicht nur theoretisch zu lernen, sondern typische Aufgaben eines Systemadministrators praktisch umzusetzen.

## Umgebung

| System | Aufgabe | IP-Adresse |
|---|---|---|
| AD-SRV01 | Domain Controller und DNS-Server | 192.168.100.10 |
| WIN11-CLIENT | Windows 11 Client | 192.168.100.20 |
| WIN11-02 | Windows 11 Client | 192.168.100.30 |

**Domain:** `adlab.local`  
**Netzwerk:** `192.168.100.0/24`

## Bisher umgesetzt

- Windows Server installiert und konfiguriert
- Active Directory Domain Services eingerichtet
- DNS eingerichtet und getestet
- Domain `adlab.local` erstellt
- Windows-11-Clients zur Domain hinzugefügt
- Benutzer, Gruppen und OUs erstellt
- Anmeldung mit Domain-Benutzern getestet
- Gruppenmitgliedschaften geprüft
- SMB-Netzwerkfreigabe erstellt
- Share- und NTFS-Berechtigungen eingerichtet
- Zugriffsrechte mit verschiedenen Benutzern getestet
- Fehler bei DNS, Domain Join und Berechtigungen untersucht

## Nächste Schritte

Als Nächstes wird die Umgebung um weitere typische Administrationsaufgaben erweitert:

- Group Policies (GPO)
- Netzlaufwerke
- Benutzer- und Passwortverwaltung
- PowerShell für Active Directory
- weitere Datei- und Gruppenberechtigungen
- typische Helpdesk- und Admin-Tickets

## Dokumentation

Die einzelnen Schritte und Erklärungen befinden sich im Ordner [`docs`](docs/).

Screenshots werden verwendet, um wichtige Konfigurationen und Tests zu zeigen.
# 01 - Lab Overview

## Ziel des Labs

In diesem Homelab baue ich eine kleine Windows-Domain mit Active Directory Domain Services auf.

Das Ziel ist, typische Aufgaben aus der Systemadministration praktisch zu lernen und zu testen.

Dabei geht es nicht nur um die Installation von Active Directory, sondern auch um die tägliche Verwaltung von Benutzern, Gruppen, Clients, Berechtigungen und später Group Policies.

## Aufbau der Umgebung

Die Umgebung läuft vollständig in Oracle VirtualBox.

Verwendete Systeme:

| System | Rolle | IP-Adresse |
|---|---|---|
| AD-SRV01 | Domain Controller und DNS-Server | 192.168.100.10 |
| WIN11-CLIENT | Windows 11 Domain Client | 192.168.100.20 |
| WIN11-02 | Windows 11 Domain Client | 192.168.100.30 |

Die Domain heißt:

`adlab.local`

Das Lab-Netzwerk ist:

`192.168.100.0/24`

Gateway:

`192.168.100.1`

DNS-Server:

`192.168.100.10`

## Grundidee

Der Server `AD-SRV01` übernimmt die zentrale Verwaltung der Umgebung.

Er stellt aktuell folgende Dienste bereit:

- Active Directory Domain Services
- DNS
- Benutzer- und Gruppenverwaltung
- Domain-Authentifizierung
- zentrale Datei- und Zugriffsrechte

Die Windows-11-Clients sind Mitglieder der Domain `adlab.local`.

Dadurch können Domain-Benutzer auf den Clients angemeldet werden, ohne dass diese Benutzer lokal auf jedem PC einzeln erstellt werden müssen.

## Beispiel

Ein Benutzer wie:

`ADLAB\max.musterman`

wurde im Active Directory erstellt.

Dieser Benutzer kann sich anschließend an einem domain-joined Windows-11-Client anmelden.

Der Benutzer ist außerdem Mitglied der Gruppe:

`IT`

Über diese Gruppe können später Berechtigungen vergeben werden, zum Beispiel für Netzwerkfreigaben.

## Was in diesem Lab gelernt wird

Das Lab behandelt unter anderem:

- Aufbau einer Windows-Domain
- Benutzer- und Gruppenverwaltung
- Organizational Units
- DNS in Active Directory
- Domain Join von Clients
- Domain-Anmeldung
- SMB-Freigaben
- NTFS-Berechtigungen
- gruppenbasierte Zugriffssteuerung
- Group Policies
- Troubleshooting
- PowerShell-Administration

Die Umgebung wird Schritt für Schritt erweitert.
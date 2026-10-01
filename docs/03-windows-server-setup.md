# 03 - Windows Server vorbereiten

## Windows Server installieren

Für den Domain Controller wurde Windows Server 2025 in VirtualBox installiert.

Der Server wurde später als zentrale Maschine für Active Directory und DNS verwendet.

Nach der Installation wurde der Server zunächst normal gestartet und über den Server Manager verwaltet.

![Server Manager nach der Installation](../images/server-manager-initial.png)

## Servername

Der Server wurde in:

`AD-SRV01`

umbenannt.

Ein klarer Servername hilft später bei der Verwaltung und bei der Fehlersuche.

## Feste IP-Adresse

Für den Server wurde eine feste IPv4-Adresse vergeben.

Konfiguration:

| Einstellung | Wert |
|---|---|
| IP-Adresse | 192.168.100.10 |
| Subnetzmaske | 255.255.255.0 |
| Gateway | 192.168.100.1 |

Ein Domain Controller sollte eine feste IP-Adresse besitzen, damit die Clients ihn zuverlässig erreichen können.

## Netzwerk testen

Nach der IP-Konfiguration wurde geprüft, ob der Server:

- das Gateway erreicht
- Zugriff auf das Internet hat
- DNS-Namen auflösen kann

Verwendete Tests:

    ping 192.168.100.1
    ping 8.8.8.8
    ping google.com

Alle Tests waren erfolgreich.

![Netzwerk- und Internettest des Servers](../images/server-network-test.png)

## Vorbereitung für Active Directory

Bevor Active Directory Domain Services installiert wurde, wurde der Server vollständig aktualisiert.

Das war wichtig, damit das System auf einem sauberen und aktuellen Stand ist.

Danach war `AD-SRV01` bereit für die Installation von:

- Active Directory Domain Services
- DNS Server

Im nächsten Schritt wurde der Server zum Domain Controller aufgebaut.
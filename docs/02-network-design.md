# 02 - Netzwerkaufbau

## Virtuelles Netzwerk

Für das Active-Directory-Lab wurde in VirtualBox ein eigenes NAT-Netzwerk erstellt.

Name des Netzwerks:

`AD-Lab`

Netzwerk:

`192.168.100.0/24`

Gateway:

`192.168.100.1`

Alle Systeme im Lab befinden sich im gleichen virtuellen Netzwerk und können miteinander kommunizieren.

Gleichzeitig ermöglicht das NAT-Netzwerk den virtuellen Maschinen den Zugriff auf das Internet.

## IP-Adressen

Für die wichtigsten Systeme wurden feste IP-Adressen verwendet.

| System | IP-Adresse | Aufgabe |
|---|---|---|
| AD-SRV01 | 192.168.100.10 | Domain Controller und DNS |
| WIN11-CLIENT | 192.168.100.20 | Domain Client |
| WIN11-02 | 192.168.100.30 | Domain Client |
| Gateway | 192.168.100.1 | VirtualBox NAT Gateway |

Die Subnetzmaske lautet:

`255.255.255.0`

bzw.

`/24`

## Warum feste IP-Adressen?

Ein Domain Controller sollte eine feste IP-Adresse besitzen.

Die Clients müssen den Domain Controller und besonders den DNS-Server zuverlässig erreichen können.

Würde sich die IP-Adresse des Servers ständig ändern, könnten die Clients den Domain Controller möglicherweise nicht mehr finden.

Deshalb verwendet `AD-SRV01` dauerhaft:

`192.168.100.10`

## DNS-Konfiguration der Clients

Die Windows-Clients verwenden nicht direkt einen öffentlichen DNS-Server wie `8.8.8.8`.

Als DNS-Server wird der Domain Controller eingetragen:

`192.168.100.10`

Das ist wichtig, weil Active Directory DNS verwendet, um Domain Controller und andere Domain-Dienste zu finden.

Der grundlegende Aufbau sieht damit so aus:

    Internet
       |
    VirtualBox NAT
       |
    192.168.100.1
       |
       +-----------------------------+
       |              |              |
    AD-SRV01     WIN11-CLIENT     WIN11-02
    .100.10         .100.20        .100.30
       |
     DNS + AD DS

## Verbindung testen

Nach der Netzwerkkonfiguration wurde die Verbindung vom Windows-Client zum Domain Controller getestet.

Dafür wurde unter anderem folgender Befehl verwendet:

    ping AD-SRV01.adlab.local

Der Client konnte den Namen über DNS auflösen und den Server unter `192.168.100.10` erreichen.

![Verbindung zwischen Client und Domain Controller](../images/network-client-dc-ping.png)

Damit war bestätigt, dass die grundlegende Netzwerkverbindung zwischen Client, DNS-Server und Domain Controller funktioniert.
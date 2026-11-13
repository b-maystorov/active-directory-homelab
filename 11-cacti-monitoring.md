# 11 – Cacti im Homelab: Ubuntu Server überwachen

> **Projektstatus: 13.11.2026**  
> **Erfolgreich getestet:** CPU, Arbeitsspeicher, Systemlast, Betriebszeit, Poller-Laufzeit und Netzwerkverkehr.  
> **Noch offen:** Das Diagramm für die Ping-Latenz zeigt weiterhin `nan`. Die Überwachung des Windows-Domänencontrollers ist für ein späteres Kapitel geplant.

## 1. Ziel

Nach der Installation von Cacti aus [Kapitel 10](10-cacti-installation.md) möchte ich echte Messwerte meines Ubuntu Servers sammeln und in Diagrammen darstellen. Dabei lerne ich, wie **SNMP**, der **Cacti-Poller** und **RRD-Dateien** zusammenarbeiten. Ich dokumentiere auch die Fehler, die bei der Einrichtung aufgetreten sind.

Meine Umgebung:

| System | Rolle | Adresse |
|---|---|---|
| Ubuntu Server (`ubuntutest`) | Cacti-Webserver, Poller und SNMP-Agent | `192.168.100.40` |
| Windows Server 2022 (`AD-SRV01`) | Domänencontroller und DNS, späteres Monitoring-Ziel | `192.168.100.10` |
| Ubuntu-Laptop | Zugriff auf die Cacti-Weboberfläche | `http://127.0.0.1:8080/cacti/` |
| VirtualBox NAT-Netzwerk `AD-Lab` | Virtuelles Labornetz | `192.168.100.0/24` |

Der Browserzugriff erfolgt über eine **VirtualBox-Portweiterleitung** (`127.0.0.1:8080` auf dem Laptop → `192.168.100.40:80` in der VM). Die SNMP-Abfrage von Cacti findet dagegen **innerhalb der Ubuntu-Server-VM** statt und verwendet deshalb `127.0.0.1:161`. Das sind zwei verschiedene Verbindungen.

## 2. So funktioniert Cacti

```text
Ubuntu Server (192.168.100.40)
 ├─ snmpd (SNMP-Agent, UDP 161)
 │    └─ liefert z. B. CPU-, RAM- und Netzwerkschnittstellen-Zähler
 ├─ Cacti-Poller (regelmäßig über cron)
 │    └─ liest Messwerte über SNMP von 127.0.0.1
 ├─ RRD-Dateien (/var/lib/cacti/rra/)
 │    └─ speichern Messwerte mit Zeitstempeln
 └─ Apache / Cacti-Weboberfläche
      └─ stellt die Zeitreihen als Diagramme dar
```

- **SNMP** (*Simple Network Management Protocol*) liefert Geräte- und Systeminformationen.
- **OID** (*Object Identifier*) bezeichnet einen bestimmten Messwert, beispielsweise Speichernutzung.
- **Poller** ist der Prozess, der die Werte regelmäßig abfragt.
- **RRD** (*Round Robin Database*) speichert Messwerte über einen längeren Zeitraum in festen Zeitrastern.
- **Graph Template** ist eine Diagrammvorlage. Das Zuweisen einer Vorlage und das Sammeln gültiger Werte sind zwei unterschiedliche Schritte.

## 3. SNMP auf Ubuntu Server einrichten

Zuerst habe ich geprüft, welche SNMP-Pakete installiert sind:

```bash
dpkg -l | grep -E 'snmpd|snmp'
```

`snmp` (die Kommandozeilenprogramme) war bereits vorhanden, aber der Dienst **`snmpd`** fehlte. Deshalb habe ich den Agenten installiert:

```bash
sudo apt update
sudo apt install snmpd
sudo systemctl status snmpd
```

Der Status war `active (running)` und der Dienst war für den automatischen Start aktiviert.

![Aktiver SNMP-Dienst auf dem Ubuntu Server](../images/cacti/11-01-snmp-dienst.png)

### 3.1 Konfiguration für die lokale Testumgebung

Die Konfigurationsdatei wird mit Nano bearbeitet:

```bash
sudo nano /etc/snmp/snmpd.conf
```

Die relevante Einstellung für den Agenten war:

```conf
agentaddress 127.0.0.1,[::1]
```

In diesem ersten Laborschritt bleibt der Agent nur über die lokale Schleifenschnittstelle erreichbar. Die SNMPv2c-Leseregel habe ich auf IPv4-Localhost beschränkt:

```conf
rocommunity public 127.0.0.1
#rocommunity6 public default -V systemonly
```

Die ursprünglich eingeschränkte Regel mit `-V systemonly` erlaubte nur einen kleinen Teil der OIDs. Für CPU-, Speicher- und Netzwerkschnittstellen-Messwerte musste Cacti mehr lesen können.

![SNMP-Konfiguration mit lokaler Community](../images/cacti/11-02-snmp-konfiguration.png)

**Sicherheit:** `public` ist eine bekannte und unverschlüsselte SNMPv2c-Community. Für dieses auf `127.0.0.1` begrenzte Übungslabor ist sie nur eine Übergangslösung. Für entfernte Geräte würde ich SNMPv3 mit Authentifizierung und Verschlüsselung sowie passende Firewall-Regeln verwenden. Die Community nicht einfach im Netzwerk freigeben.

Nach dem Speichern in Nano (`Strg+O`, `Enter`, `Strg+X`) habe ich SNMP neugestartet:

```bash
sudo systemctl restart snmpd
sudo systemctl status snmpd
```

### 3.2 SNMP testen

Zunächst habe ich Systeminformationen abgefragt:

```bash
snmpwalk -v2c -c public 127.0.0.1 1.3.6.1.2.1.1
```

Anschließend die Speicherwerte:

```bash
snmpwalk -v2c -c public 127.0.0.1 1.3.6.1.4.1.2021.4
```

Die Ausgabe enthielt numerische Werte für RAM und Swap. Damit war nachgewiesen, dass der Agent Informationen bereitstellt, die Cacti abfragen kann. Wichtig: Der erfolgreiche `snmpwalk`-Test beweist noch nicht, dass Cacti auch ein funktionierendes Diagramm erzeugt hat.

## 4. Ubuntu Server als Zielsystem in Cacti anlegen

In der deutschen Oberfläche bin ich zu **Konsole → Verwaltung → Zielsysteme** gegangen und habe oben rechts das **Pluszeichen (+)** gewählt.

Für mein Zielsystem habe ich diese Werte verwendet:

| Feld in Cacti | Einstellung |
|---|---|
| Beschreibung | `Ubuntu-Server-Cacti` |
| Zielsystemname | `127.0.0.1` |
| Zielsystem Schablone | `Generic SNMP Device` |
| Poller Association | `Main Poller` |
| SNMP Version | `Version 2` |
| SNMP Community String | `public` |
| SNMP Port | `161` |

Nach **Erzeugen** konnte Cacti den Rechner `ubuntutest` erkennen. Danach habe ich die Einstellungen mit **Speichern** bestätigt.

## 5. Diagrammvorlagen hinzufügen

In den Geräteeinstellungen unter **Zugeordnete Graph Templates** habe ich folgende Vorlagen ergänzt:

- `Net-SNMP - CPU Utilization` für die Prozessorauslastung
- `Net-SNMP - Memory Usage` für Arbeitsspeicher
- `Net-SNMP - Load Average` für die Systemlast

Die Vorlagen für `Device - Uptime`, `Device - Polling Time` und `Unix - Ping Latency` waren bereits vorhanden. Die Datenabfrage `SNMP - Interface Statistics` erkannte zwei Schnittstellen.

![Cacti-Zielsystem mit zugeordneten CPU-, RAM- und Load-Vorlagen](../images/cacti/11-03-zielsystem-und-vorlagen.png)

Die Graphen wurden angelegt, hatten am Anfang aber teilweise noch keine RRD-Dateien oder zeigten `nan`. Das bedeutet **fehlende bzw. unbekannte Messwerte** und ist nicht dasselbe wie ein Messwert von null.

![Neue Diagramme unmittelbar nach dem Hinzufügen – teilweise noch ohne Daten](../images/cacti/11-04-erste-diagramme.png)

### 5.1 Graphen in der Oberfläche sichtbar machen

Unter **Verwaltung → Diagramm-Bäume** existierte bereits der veröffentlichte `Default Tree`. Zunächst war er leer. Nach **Baum bearbeiten** habe ich den Eintrag `Ubuntu-Server-Cacti` aus **Verfügbare Geräte** in den Baum gezogen, gespeichert und die Bearbeitung beendet.

Anschließend waren die Graphen unter **Diagramme → Default Tree → Ubuntu-Server-Cacti** sichtbar. Dafür musste ich keinen neuen Baum erstellen.

## 6. Prüfen, ob der Poller läuft

Als einige Graphen leer blieben, habe ich nicht sofort neu installiert, sondern Schritt für Schritt geprüft, wo die Datenkette unterbrochen sein könnte.

**Zeitplan ansehen:**

```bash
cat /etc/cron.d/cacti
```

Der Cacti-Poller wird in meinem System durch einen Cron-Eintrag etwa **alle fünf Minuten** mit dem Benutzer `www-data` gestartet.

**Protokoll ansehen:**

```bash
sudo tail -n 20 /var/log/cacti/cacti.log
```

Im Log standen Einträge wie `RRDsProcessed:13`, also Hinweise darauf, dass RRD-Messdaten verarbeitet wurden. Zusätzlich traten PHP-Warnungen zur Verwendung von `null` als Array-Offset auf. Diese Warnungen sollte man im Blick behalten; sie verhinderten in diesem Test nicht grundsätzlich das Sammeln der Daten.

**RRD-Dateien prüfen:**

```bash
sudo ls -lh /var/lib/cacti/rra/
```

Hier erschienen Dateien für CPU, RAM, Load Average, Uptime und Polling Time. Die Dateien gehörten `www-data`.

![Cron-Konfiguration, RRD-Dateien und PHP-Hinweise](../images/cacti/11-05-poller-und-rrd-dateien.png)

### 6.1 Direkte Messwerte aus RRD lesen

Mit `rrdtool` habe ich die neuesten gespeicherten Messwerte geprüft:

```bash
sudo rrdtool lastupdate /var/lib/cacti/rra/ubuntu-server-cacti_sscpusystem_5.rrd
sudo rrdtool lastupdate /var/lib/cacti/rra/ubuntu-server-cacti_mem_total_10.rrd
```

Die Dateien enthielten gültige Werte, darunter `0` für die System-CPU-Messung und `3480320` für den Gesamt-Speicherzähler. `0` ist ein gültiger Messwert, `nan` nicht.

Weitere Datenpunkte lassen sich so prüfen:

```bash
sudo rrdtool fetch /var/lib/cacti/rra/ubuntu-server-cacti_mem_total_10.rrd AVERAGE --start -1800
```

Das Ergebnis enthielt anfangs `nan`, danach mehrere gültige Speicherwerte. Zusätzlich habe ich mit `date` und `timedatectl` bestätigt, dass die VM-Uhr synchronisiert war (`System clock synchronized: yes`). Die VM verwendet UTC; das ist kein Problem, solange Datum und Uhrzeit korrekt sind.

### 6.2 Warum die Graphen erst später vollständig waren

Neue RRD-Zeitreihen haben noch keine längere Historie. Manche Diagramme benötigen mehrere Poller-Durchläufe, bevor genügend Werte für eine Kurve vorliegen. Nach dem Weiterlaufen des Servers über Nacht waren CPU-, RAM-, Load-, Uptime- und Polling-Time-Graphen mit historischen Daten sichtbar.

![Cacti-Dashboard mit tatsächlichen Messwerten nach der Datensammlung](../images/cacti/11-06-messwerte-nach-einem-tag.png)

**Wichtig:** Ein fehlendes RRD unmittelbar nach dem Erzeugen kann vorübergehend sein. Wenn ein Diagramm dauerhaft `nan` zeigt, sollte man SNMP, Poller-Log, RRD-Datei und Zeitstempel einzeln überprüfen, statt einfach weiter zu warten.

## 7. Netzwerkverkehr über `enp0s3` überwachen

### 7.1 Netzwerkschnittstelle identifizieren

Im Ubuntu-Server-Terminal:

```bash
ip -br addr
ip -s link
```

Die Ergebnisse zeigten `enp0s3` als aktive VirtualBox-Netzwerkschnittstelle mit `192.168.100.40/24`; `lo` ist nur die lokale Loopback-Schnittstelle. Unter `ip -s link` waren RX- und TX-Bytezähler sichtbar, ohne Paketfehler.

![IP-Adresse und Netzwerkzähler der Schnittstelle enp0s3](../images/cacti/11-07-netzwerk-interface-enp0s3.png)

### 7.2 Netzwerkgrafik erstellen

In Cacti:

1. **Konsole → Erzeugen → Neue Diagramme** öffnen.
2. Als Gerät `Ubuntu-Server-Cacti` auswählen.
3. In **Datenabfrage [SNMP - Interface Statistics]** die Zeile `enp0s3` markieren; `lo` nicht auswählen.
4. Als Diagrammtyp **`In/Out Bits (64-bit)`** wählen.
5. Auf **Erzeugen** klicken.

![Neue Netzwerk-Grafik: enp0s3 und In/Out Bits (64-bit) ausgewählt](../images/cacti/11-08-netzwerk-diagramm-erstellen.png)

**Wieso 64 Bit?** Für Netzwerk-Bytezähler sind 64-Bit-Counter geeignet, weil sie bei größerem Datenverkehr seltener überlaufen als 32-Bit-Counter. Cacti berechnet aus Zählerständen zu verschiedenen Zeitpunkten die Übertragungsrate.

### 7.3 SNMP-Netzwerkzähler separat prüfen

```bash
snmpwalk -v2c -c public 127.0.0.1 1.3.6.1.2.1.31.1.1.1.6
```

Dieser OID-Baum liefert 64-Bit-Zähler (`Counter64`) für empfangene Bytes je Schnittstelle. In meinem Test wurden Werte für beide Interfaces zurückgegeben.

Der Dateiname des Netzwerk-RRD war:

```text
/var/lib/cacti/rra/ubuntu-server-cacti_traffic_in_14.rrd
```

Ich habe den Dateinamen zuerst ohne Unterstrich vor `14` eingegeben. Die Meldung `No such file or directory` war deshalb nur ein Tippfehler. Mit dem korrekten Dateinamen funktionierte die Abfrage:

```bash
sudo rrdtool lastupdate /var/lib/cacti/rra/ubuntu-server-cacti_traffic_in_14.rrd
```

Die Datei enthielt `traffic_in` und `traffic_out` mit numerischen Werten. Anschließend erschien die neue Netzwerkgrafik zunächst leer, nach weiteren Poller-Durchläufen aber mit tatsächlichen RX-/TX-Raten.

![Fertiges Netzwerkdiagramm mit ein- und ausgehendem Datenverkehr](../images/cacti/11-09-netzwerk-monitoring-ergebnis.png)

In diesem Screenshot wurden ungefähr **4,22 kbit/s eingehend** und **65,50 kbit/s ausgehend** angezeigt. Das sind Momentaufnahmen, keine konstanten Geschwindigkeiten.

## 8. Ergebnis und offene Punkte

| Überwachung | Stand am 13.11.2026 |
|---|---|
| CPU-Auslastung | Funktioniert |
| Arbeitsspeicher | Funktioniert |
| Load Average | Funktioniert |
| Uptime | Funktioniert |
| Polling Time | Funktioniert |
| Netzwerkverkehr `enp0s3` | Funktioniert, RX und TX sichtbar |
| Ping Latency | Noch keine gültigen Werte (`nan`) |
| Windows Server `AD-SRV01` | Noch nicht in Cacti eingerichtet |

**Was ich gelernt habe:** Ein vorhandenes Diagramm bedeutet nicht automatisch, dass die Messwertsammlung funktioniert. Bei Problemen prüfe ich zuerst den SNMP-Agenten, dann den Poller, die RRD-Dateien und zuletzt die Darstellung. Außerdem habe ich gelernt, zwischen Bytezählern und Übertragungsraten sowie zwischen `0` und `nan` zu unterscheiden.

**Nächste Schritte:** Die Ursache für die leere Ping-Latenz-Grafik untersuchen, anschließend `AD-SRV01` über eine geeignete und abgesicherte Monitoring-Methode ergänzen. Dieses Kapitel beschreibt bewusst nur die nachweislich durchgeführten Arbeiten und Tests.

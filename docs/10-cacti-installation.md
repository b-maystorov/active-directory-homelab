# 10 – Cacti im Active-Directory-Homelab installieren

> **Stand: 10.11.2026** · Installation und Anmeldung erfolgreich getestet.  
> **Als Nächstes:** Kapitel 11 – Geräte mit SNMP überwachen und Diagramme erstellen (noch nicht durchgeführt).

## Ziel des Projekts

Ich habe mein bestehendes Active-Directory-Homelab um einen Ubuntu Server mit **Cacti** erweitert. Damit möchte ich später meine Linux- und Windows-Systeme überwachen und zum Beispiel CPU-Auslastung, Arbeitsspeicher und Netzwerkverkehr in Diagrammen sehen.

**Cacti** ist ein kostenloses Open-Source-Monitoring-Tool. Es kann Messwerte, häufig über **SNMP (Simple Network Management Protocol)**, von Geräten abfragen. Mit **RRDtool** lassen sich daraus Verlaufsgrafiken erstellen. In diesem Kapitel geht es erst einmal um **Installation, Netzwerkverbindung, Webzugriff und Fehlerbehebung**. Die eigentliche Überwachung folgt in Kapitel 11.

## 1. Meine Laborumgebung

```text
Ubuntu-Laptop mit Brave
http://127.0.0.1:8080/cacti/
             |
      VirtualBox-Portweiterleitung
      127.0.0.1:8080 → 192.168.100.40:80
             |
       NAT-Netzwerk „AD-Lab“
          192.168.100.0/24
             |
    +--------+--------------------+
    |                             |
AD-SRV01                       Ubuntu-Server
192.168.100.10                 192.168.100.40
AD DS + DNS                    Cacti + Apache + MySQL
```

| Komponente | Konfiguration |
|---|---|
| Virtualisierung | Oracle VirtualBox, NAT-Netzwerk `AD-Lab` |
| Netzwerk | `192.168.100.0/24` |
| Domänencontroller | Windows Server, `192.168.100.10`, AD DS und DNS |
| Ubuntu Server | Ubuntu 26.04.1 LTS (`resolute`), `192.168.100.40` |
| VM-Ressourcen | 2 vCPUs, 4 GB RAM, 20 GB virtuelle Festplatte |
| Monitoring | Cacti 1.2.30 |
| Webserver | Apache2 (intern Port 80) |
| Datenbank | MySQL 8.4, Datenbank `cacti` |

**Hinweis:** Die Adressen und Namen gehören zu **meinem** Homelab. Bei einer anderen Umgebung muss man Netz, freie IP-Adresse und Gateway überprüfen.

## 2. Ubuntu Server in das AD-Lab-Netzwerk einbinden

Ich hatte bereits eine funktionierende Ubuntu-Server-VM und musste keine neue installieren. In VirtualBox habe ich bei der ausgeschalteten VM unter **Ändern → Netzwerk → Adapter 1** das **NAT-Netzwerk `AD-Lab`** ausgewählt und das virtuelle Netzwerkkabel aktiviert.

In den VirtualBox-Einstellungen unter **Netzwerk → NAT-Netzwerke** war für `AD-Lab` der Bereich `192.168.100.0/24` eingestellt. **DHCP war deaktiviert.** Deshalb konnte Ubuntu mit einer DHCP-Konfiguration keine IPv4-Adresse beziehen.

![VirtualBox: AD-Lab mit deaktiviertem DHCP](../images/cacti/01-dhcp-disabled.png)

### Netzwerkschnittstelle überprüfen

```bash
ip link show
ip -4 addr
sudo cat /etc/netplan/*.yaml
```

Die tatsächliche Schnittstelle hieß bei mir **`enp0s3`**. In einer alten Netplan-Konfiguration standen aber noch `ens3` und `ens4`. Die Karte war zwar eingeschaltet (`UP`), hatte aber noch keine IPv4-Adresse.

### Feste IP-Adresse mit Netplan einrichten

Zuerst habe ich die Konfigurationsdatei gesucht:

```bash
ls -l /etc/netplan/
```

Meine Datei war `00-installer-config.yaml`. Für zukünftige Änderungen empfiehlt sich eine Sicherung **außerhalb** von `/etc/netplan/`:

```bash
sudo cp /etc/netplan/00-installer-config.yaml /root/netplan-before-cacti.yaml
sudo nano /etc/netplan/00-installer-config.yaml
```

Beispiel für meine Konfiguration:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.100.40/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 192.168.100.10
```

![Netplan: Statische IP-Adresse und AD-DNS](../images/cacti/02-netplan-static-ip.png)

**Einfach erklärt:** `dhcp4: false` bedeutet, dass Ubuntu die IPv4-Adresse manuell erhält. `/24` entspricht der Subnetzmaske `255.255.255.0`. `via: 192.168.100.1` ist das Standard-Gateway. Als DNS-Server wird mein Domänencontroller `192.168.100.10` verwendet. Für öffentliche Domainnamen muss dessen DNS-Auflösung ebenfalls funktionieren.

Bei YAML sind die **Einrückungen wichtig**: Leerzeichen statt Tabulatoren. In `nano` speichert man mit **Strg+O**, Enter und beendet mit **Strg+X**.

Anschließend habe ich die Konfiguration aktiviert und überprüft:

```bash
sudo netplan generate
sudo netplan apply
ip -4 addr show enp0s3
```

![Ubuntu Server: IPv4-Adresse 192.168.100.40](../images/cacti/03-static-ip-verified.png)

**Tipp für spätere Arbeiten:** Bei Netzwerkänderungen über SSH ist `sudo netplan try` oft sicherer. Wenn die Verbindung verloren geht, kann die Änderung automatisch zurückgenommen werden.

## 3. Netzwerk und DNS testen

```bash
ip route
ping -c 4 192.168.100.10
ping -c 4 1.1.1.1
ping -c 4 google.com
```

| Test | Bedeutung | Ergebnis in meinem Lab |
|---|---|---|
| `192.168.100.10` | Verbindung zum Domänencontroller | Erfolgreich |
| `1.1.1.1` | Internetverbindung ohne DNS-Namen | Erfolgreich |
| `google.com` | Internetzugriff mit Namensauflösung | Erfolgreich |

![Ping zum Windows-Domänencontroller](../images/cacti/04-ad-connectivity.png)

Damit konnte Ubuntu den AD-Server und das Internet erreichen. **Wichtig:** Ein erfolgreicher Ping bestätigt die Erreichbarkeit über ICMP, aber nicht automatisch alle anderen Dienste oder Ports.

## 4. Ubuntu aktualisieren

```bash
sudo apt update
sudo apt upgrade
lsb_release -a
```

Meine Version war **Ubuntu 26.04.1 LTS** (`resolute`). Bei der Aktualisierung erschien eine **GRUB-Abfrage** zum Installationsziel des Bootloaders. Bei **dieser BIOS-basierten VirtualBox-VM** wurde die gesamte virtuelle Festplatte `/dev/sda` gewählt – nicht eine einzelne Partition. Auf anderen Systemen muss das Bootverfahren und die Festplattenaufteilung vorher geprüft werden.

Nach einem Kernel-Update war ein Neustart sinnvoll:

```bash
sudo reboot
```

## 5. Cacti installieren

Zuerst habe ich geprüft, ob das Paket in den Ubuntu-Repositories verfügbar ist:

```bash
apt-cache policy cacti
```

In meinem Fall wurde `1.2.30+ds1-1ubuntu2` aus `resolute/universe` angeboten. Danach:

```bash
sudo apt install cacti
```

Während der Installation habe ich zwei wichtige Optionen ausgewählt:

1. **`apache2`** als Webserver, damit Cacti über einen Browser erreichbar ist.
2. **„Yes“** bei `dbconfig-common`, damit die benötigte Datenbank automatisch eingerichtet wird.

![Cacti-Installation: Apache2 auswählen](../images/cacti/05-webserver-choice.png)

### Wofür sind die Komponenten da?

| Komponente | Aufgabe |
|---|---|
| **Cacti** | Weboberfläche zum Einrichten und Anzeigen der Überwachung |
| **Apache2** | Liefert die Webseite aus |
| **PHP** | Führt die Cacti-Anwendung auf dem Server aus |
| **MySQL** | Speichert Benutzer, Geräte und Einstellungen |
| **RRDtool** | Speichert Messreihen und erstellt Verlaufsgrafiken |
| **SNMP** | Wird später verwendet, um Messwerte von Geräten abzurufen |

### Apache und Cacti lokal prüfen

```bash
sudo systemctl status apache2
curl -I http://localhost/cacti/
curl -I http://192.168.100.40/cacti/
```

Apache zeigte **`active (running)`**. Beide `curl`-Tests lieferten **`HTTP/1.1 200 OK`**. Mit **Q** kann man die `systemctl`-Statusansicht verlassen.

Eine Warnung mit `AH00558` bezüglich `ServerName` erschien ebenfalls. Sie war für den späteren Browserfehler nicht verantwortlich.

## 6. Cacti vom Ubuntu-Laptop aus öffnen

Mein Ubuntu Server läuft im VirtualBox-NAT-Netzwerk. Der Laptop kann die interne VM-IP nicht unbedingt direkt aufrufen. Deshalb habe ich eine **Portweiterleitung** eingerichtet.

Unter **VirtualBox → Netzwerk → NAT-Netzwerke → AD-Lab → Port-Weiterleitung → IPv4** habe ich folgende Regel erstellt:

| Feld | Wert |
|---|---|
| Name | `Cacti-Web` |
| Protokoll | `TCP` |
| Host-IP | `127.0.0.1` |
| Host-Port | `8080` |
| Gast-IP | `192.168.100.40` |
| Gast-Port | `80` |

![Portweiterleitung für Cacti](../images/cacti/06-port-forwarding.png)

**Bedeutung:** Wenn ich auf meinem Laptop `127.0.0.1:8080` aufrufe, leitet VirtualBox die Anfrage an Port 80 des Ubuntu Servers weiter.

Von meinem **Laptop-Terminal** aus getestet:

```bash
curl -I http://127.0.0.1:8080/cacti/
```

Ergebnis: **`HTTP/1.1 200 OK`**. Im Brave-Browser lautet die Adresse:

```text
http://127.0.0.1:8080/cacti/
```

### Fehler: 404 Not Found im Browser

Am Anfang zeigte Brave eine **404-Fehlermeldung**, obwohl `curl` erfolgreich war. Die zuerst verwendete Browseradresse enthielt zusätzliche Zeichen hinter dem Cacti-Pfad. Mit der **genauen Adresse ohne Zusatz** in einem privaten Browserfenster erschien dann die Anmeldeseite.

**Gelernt:** Erst die vollständige URL kontrollieren und mit `curl` gegenprüfen, bevor man Apache oder das Netzwerk verändert.

**Sicherheit:** Die Host-IP `127.0.0.1` macht diese Portweiterleitung nur vom Laptop selbst erreichbar. Für das Homelab reicht das. Cacti läuft hier über **HTTP ohne TLS** und sollte nicht öffentlich im Internet freigegeben werden.

## 7. Login-Problem beheben

Der Standardversuch mit `admin` als Benutzer und Passwort funktionierte nicht. Auf der Webseite erschien:

> Zugriff verweigert, Benutzerkonto deaktiviert.

Ich habe deshalb den Kontostatus in der Datenbank überprüft. **Die folgenden Änderungen waren eine Wiederherstellung in meinem eigenen Testsystem, keine allgemeine Standardinstallation.**

### MySQL statt MariaDB

Der erste Befehl mit `mariadb` funktionierte nicht (`command not found`). Danach habe ich die installierten Pakete kontrolliert:

```bash
dpkg -l | grep -E 'mariadb|mysql'
sudo mysql -e "SHOW DATABASES;"
```

Installiert waren **MySQL Server und Client 8.4**. In der Liste war die Datenbank **`cacti`** zu sehen.

### Administrator-Konto lesen

```bash
sudo mysql -e "SELECT id, username, enabled, locked, failed_attempts FROM cacti.user_auth WHERE username='admin';"
```

Das Konto hatte zunächst `enabled = on`, `locked = 1` und `failed_attempts = 0`. Die Ursache der Sperre ließ sich daraus **nicht sicher bestimmen**. Danach habe ich das Konto entsperrt:

```bash
sudo mysql -e "UPDATE cacti.user_auth SET locked=0 WHERE id=1 AND username='admin';"
```

Der erneute Check zeigte `locked = 0`, aber die Anmeldung funktionierte noch nicht. Daher musste ich zusätzlich das Passwort zurücksetzen.

### Passwort sicher zurücksetzen

**Wichtig:** Für eine echte produktive Umgebung zuerst ein Datenbank-Backup erstellen und vorzugsweise offizielle Wiederherstellungsfunktionen nutzen. In meiner Installation gab es unter `/usr/share/cacti/cli/` kein `reset_password.php`.

Im **gleichen Ubuntu-Server-Terminal** habe ich ein neues, einzigartiges Passwort unsichtbar eingegeben:

```bash
read -rs -p "New Cacti password: " CACTI_PASS; echo
```

Danach habe ich mit PHP einen Passwort-Hash erzeugt, ohne das Klartextpasswort im Befehl anzugeben:

```bash
CACTI_HASH=$(printf '%s' "$CACTI_PASS" | php -r 'echo password_hash(stream_get_contents(STDIN), PASSWORD_DEFAULT);')
```

Die Spalte `password` in `cacti.user_auth` hatte den Typ `varchar(256)`:

```bash
sudo mysql -e "SHOW COLUMNS FROM cacti.user_auth LIKE 'password';"
```

Den neuen Hash habe ich für **mein** Admin-Konto gespeichert:

```bash
printf "UPDATE user_auth SET password='%s' WHERE username='admin' AND id=1;\n" "$CACTI_HASH" | sudo mysql cacti
```

Danach habe ich nur den Hash-Anfang und den Kontostatus überprüft (nicht das Passwort selbst):

```bash
sudo mysql -e "SELECT username, enabled, locked, LEFT(password, 4) AS hash_prefix FROM cacti.user_auth WHERE username='admin';"
```

Die temporären Shell-Variablen habe ich anschließend entfernt:

```bash
unset CACTI_PASS CACTI_HASH
```

**Ergebnis:** Die Anmeldung mit dem neuen Passwort hat funktioniert. Ich konnte die **Cacti-Konsole, Version 1.2.30**, im Browser öffnen.

![Erfolgreiche Anmeldung in der Cacti-Konsole](../images/cacti/07-cacti-console.png)

**Wichtig für GitHub:** Niemals echte Passwörter, Datenbank-Dumps mit Zugangsdaten, vollständige Passwort-Hashes oder Session-Cookies hochladen.

## 8. Was ich gelernt habe

| Begriff | Kurze Erklärung |
|---|---|
| NAT-Netzwerk | Die VMs können untereinander kommunizieren und über NAT ins Internet gehen |
| DHCP | Verteilt Netzwerkeinstellungen automatisch; war hier deaktiviert |
| Statische IP | Manuell festgelegte IP-Adresse, sinnvoll für Server |
| Netplan | Werkzeug zur Netzwerkkonfiguration unter Ubuntu mit YAML-Dateien |
| Standard-Gateway | Nächster Router für Ziele außerhalb des eigenen Netzes |
| DNS | Übersetzt Namen wie `google.com` in IP-Adressen |
| Apache2 | Webserver, der die Cacti-Oberfläche bereitstellt |
| MySQL | Datenbanksystem für Benutzer und Cacti-Einstellungen |
| Portweiterleitung | Verbindet einen Host-Port mit einem Port in der VM |
| Passwort-Hash | Einwegwert zum sicheren Prüfen eines Passworts |
| HTTP 200 | Die HTTP-Anfrage war erfolgreich; heißt nicht automatisch, dass man eingeloggt ist |
| HTTP 404 | Die angeforderte Seite oder Ressource wurde nicht gefunden |

## 9. Ergebnis und Ausblick

**Erledigt:** Ubuntu-Netzwerk eingerichtet, Verbindung zum AD-Server und Internet geprüft, Cacti installiert, Apache und MySQL getestet, Portweiterleitung eingerichtet und Administrator-Anmeldung wiederhergestellt.

**Noch offen:** SNMP konfigurieren, Geräte in Cacti anlegen, Polling testen und erste CPU-, RAM- und Netzwerkdiagramme erstellen. Das dokumentiere ich später als **Kapitel 11 – Cacti-Monitoring**.

### Quellen und weiterführende Dokumentation

- [Cacti – Projektseite](https://www.cacti.net/)
- [Cacti – Dokumentation](https://docs.cacti.net/)
- [Netplan – Dokumentation](https://netplan.readthedocs.io/)
- [VirtualBox – Benutzerhandbuch](https://www.virtualbox.org/manual/)

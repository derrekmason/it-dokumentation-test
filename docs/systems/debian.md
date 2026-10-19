# Debian Linux – Administration

## Zweck und Geltungsbereich

Diese Dokumentation beschreibt die grundlegende Administration eines
Debian-Servers: Installation, Paketverwaltung, Benutzer und Rechte,
Netzwerk, systemd, Protokollierung, Sicherheit und Datensicherung.

**Behandelt wird:**

- Einsatzzweck von Debian
- Installation und Erstkonfiguration
- Paketverwaltung mit APT
- Benutzer, Gruppen und Dateiberechtigungen
- Netzwerkkonfiguration und Namensauflösung
- Dienste und systemd
- Protokollierung und Fehlersuche
- Updates, Firewall und grundlegende Sicherheitsmaßnahmen
- Backup und Wiederherstellung

**Nicht behandelt wird:** Der Vergleich mit anderen Distributionen,
Desktop-Installationen, Container- und Cloud-spezifische Themen.

> **Versions- und Konfigurationsabhängigkeit:** Debian-Befehle sind über
> mehrere Releases hinweg stabil. Der Pfad der Netzwerkkonfiguration und der
> Programmnamen einzelner Sicherheitswerkzeuge haben sich zwischen den
> Releases geändert (siehe [Netzwerkkonfiguration](#netzwerkkonfiguration-und-namensauflösung)
> und [Firewall](#firewall-und-grundlegende-absicherung)). Für Produktivsysteme
> ist immer die Dokumentation der konkret installierten Version maßgeblich.

Alle Beispieldaten (Hostnamen, Adressen, Konten) sind fiktiv.

---

## Grundlagen

### Einsatzzweck

Debian ist eine freie Linux-Distribution, die ausschließlich freie Software
verwendet. Im Serverbereich wird Debian häufig eingesetzt für:

- Web- und Anwendungsserver (Apache/nginx, ownCloud-artige Dienste, Docker)
- Datei- und Freigabeserver (Samba, NFS)
- Infrastrukturdienste (DNS, DHCP, Mail, Proxys)
- Containersysteme und Cloud-Images
- Router oder Firewall (mit Linux-Kern-Funktionen)

Debian veröffentlicht stabile Releases inklusive Unterstützung über mehrere
Jahre; die genauen Zeiträume stehen auf der [Release-Übersicht](https://www.debian.org/releases/).

### Paketverwaltung mit APT

Debian verwendet **dpkg** als Basis der Paketverwaltung (Pakete `.deb`) und
**APT** als Werkzeug davor, das Abhängigkeiten auflöst, Quellen liest und
Pakete herunterlädt.

| Befehl | Zweck |
| --- | --- |
| `apt update` | Paketlisten aus den konfigurierten Quellen aktualisieren (keine Installation) |
| `apt upgrade` | Pakete aus den aktualisierten Listen aktualisieren |
| `apt full-upgrade` | wie `upgrade`, erlaubt aber das Entfernen oder Installieren weiterer Pakete |
| `apt install <paket>` | Paket und seine Abhängigkeiten installieren |
| `apt remove <paket>` | Paket entfernen, Konfigurationsdateien behalten |
| `apt purge <paket>` | Paket samt Konfigurationsdateien entfernen |
| `apt autoremove` | nicht mehr benötigte Pakete entfernen |
| `apt list --upgradable` | verfügbare Aktualisierungen anzeigen |
| `apt-cache policy <paket>` | Installierte und verfügbare Version anzeigen |
| `apt-cache show <paket>` | Paketbeschreibung und Metadaten anzeigen |
| `dpkg -l` | Liste installierter Pakete |
| `dpkg -L <paket>` | Dateien eines installierten Pakets auflisten |
| `dpkg -S <datei>` | Paket finden, das eine Datei enthält |

Zusätzlich werden häufig nicht offizielle, aber auf Debian abgestimmte
Archive eingebunden. Sie ersetzen keine Debian-Pakete, ergänzen sie aber.

---

## Vorgehen und Beispiele

### 1. Installation und Erstkonfiguration

Voraussetzungen:

1. Ein Installationsmedium, auf dem der Netboot-Installer liegt.
2. Eine Startparameterliste für den zu installierenden Kernel.
3. Angaben zu Netzwerk, Partitionierung und Benutzerkonto.

Ablauf (Netboot-Installer, vereinfacht):

| Schritt | Inhalt |
| --- | --- |
| 1 | Sprache, Ort und Tastaturbelegung |
| 2 | Netzwerk-Hostname und Domain |
| 3 | primärer Benutzername und Kennwort, eventuell sudo-Berechtigung |
| 4 | Partitionierung (manuell oder mit `partman`, subarchitekturspezifisch) |
| 5 | Paketauswahl und Installationsmedium |
| 6 | GRUB-Bootloader installieren |
| 7 | Neuinstallation des Systems und Abschluss |

Nach der Installation:

```bash
# Root-Passwort und lokale Benutzer anlegen bzw. ändern
passwd

# sudo-Berechtigung prüfen (erwartet: NOPASSWD nur, wenn bewusst so konfiguriert)
sudo -l -U benutzername

# System aktualisieren
sudo apt update
sudo apt full-upgrade

# Wichtige Pakete für die erste Konfiguration
sudo apt install sudo ufw rsync
```

> **Hinweis:** Auf einem Produktivserver wird `apt full-upgrade` in der Regel
> bewusst geplant, weil dabei Pakete entfernt werden können.

### 2. Benutzer, Gruppen und Dateiberechtigungen

Grundsatz: **root** wird nur für Aufgaben verwendet, die Root-Rechte wirklich
erfordern. Für alles andere ein normales Benutzerkonto mit `sudo`.

```bash
# Benutzer mit Heimatverzeichnis anlegen (z. B. aus dem Skelett /etc/skel)
sudo useradd -m -s /bin/bash anna
# bzw. adduser (interaktiv, Debian-üblich)
sudo adduser anna

# Gruppe anlegen und Benutzer hinzufügen
sudo groupadd ablage
sudo usermod -aG ablage anna

# Zugehörigkeiten prüfen
groups anna
id anna
```

Rechte setzen:

```bash
# Besitzer und Gruppe setzen
sudo chown anna:ablage /srv/ablage

# Rechte rekursiv setzen: Verzeichnisse rx, Dateien r
sudo find /srv/ablage -type d -exec chmod 750 {} \;
sudo find /srv/ablage -type f -exec chmod 640 {} \;

# Verzeichnis für andere lesbar und ausführbar machen
chmod o+rx /srv/ablage
```

Übersicht der Rechte:

| Rechte | Bedeutung |
| --- | --- |
| `4` | lesen (`r`) |
| `2` | schreiben (`w`) |
| `1` | ausführen (`x`) – bei Verzeichnissen: betreten/listen |
| `7` | rwx, `6` rw-, `5` r-x, `4` r--, `0` --- |
| `750` | Besitzer `rwx`, Gruppe `r-x`, Andere `---` |

Sonderbits:

| Bit | Bedeutung |
| --- | --- |
| `4000` | Set-UID |
| `2000` | Set-GID |
| `1000` | Sticky Bit – in einem gemeinsamen Verzeichnis (`/tmp`) kann nur der Eigentümer einer Datei sie löschen |

Prüfwerkzeuge:

```bash
ls -l /srv/ablage
stat /srv/ablage
```

### 3. Netzwerkkonfiguration und Namensauflösung

> **Versionsabhängigkeit:** In älteren Debian-Releases wird die Netzkonfiguration
> über `/etc/network/interfaces` mit den Paketen `netbase`/`ifupdown`
> gesteuert. Neuere Systeme verwenden systemd-networkd und die Dateien unter
> `/etc/systemd/network/`. Welches Verfahren auf einem System aktiv ist, zeigt
> `systemctl is-active systemd-networkd` beziehungsweise der Zustand der
> Dateien.

`/etc/network/interfaces` (ältere Releases) – Beispiel:

```text
auto ens18
iface ens18 inet static
    address 192.168.10.21
    netmask 255.255.255.0
    gateway 192.168.10.1
    dns-nameservers 192.168.10.53 192.168.10.54
    dns-search contoso.example
```

`/etc/systemd/network/10-ens18.network` (systemd-networkd):

```ini
[Match]
Name=ens18

[Network]
Address=192.168.10.21/24
Gateway=192.168.10.1
DNS=192.168.10.53
DNS=192.168.10.54
Domains=contoso.example
```

Anwenden und prüfen:

```bash
sudo systemctl restart systemd-networkd   # bzw. ifdown ens18 && ifup ens18
ip -4 addr show
ip route show
```

Namensauflösung:

```bash
cat /etc/resolv.conf
resolvectl status            # bei systemd-resolved
resolvectl query files01.contoso.example
nslookup files01.contoso.example   # aus dem Paket dnsutils
```

Grundlagen zu Adressbereichen und Subnetzierung stehen in
[Private IPv4-Adressbereiche](../networking/private-ipv4.md) und
[VLSM](../networking/vlsm.md).

### 4. Dienste und systemd

systemd verwaltet Dienste über Unit-Dateien. Die wichtigsten Befehle:

| Befehl | Zweck |
| --- | --- |
| `systemctl status dienst` | Zustand und Meldungen des Dienstes anzeigen |
| `systemctl start dienst` | Dienst starten |
| `systemctl stop dienst` | Dienst beenden |
| `systemctl restart dienst` | Dienst neu starten |
| `systemctl reload dienst` | Konfiguration neu laden, ohne den Dienst zu stoppen |
| `systemctl enable dienst` | Dienst beim Systemstart automatisch starten |
| `systemctl disable dienst` | Autostart entfernen |
| `systemctl enable --now dienst` | Autostart setzen und sofort starten |
| `systemctl is-enabled dienst` | Abfrage des Autostart-Status |
| `systemctl list-units --type=service` | laufende Dienste auflisten |
| `systemctl list-unit-files --type=service` | verfügbare Unit-Dateien auflisten |
| `journalctl -u dienst` | Protokoll eines Dienstes anzeigen |
| `systemctl cat dienst` | Unit-Datei eines Dienstes anzeigen |
| `systemctl daemon-reload` | Unit-Dateien nach Änderungen neu einlesen |
| `systemctl set-default multi-user.target` | Standard-Startziel setzen |

Beispiel – SSH-Server:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
sudo ss -tlnp | grep ':22'
```

Eigene Unit-Datei unter `/etc/systemd/system/` ablegen, danach
`sudo systemctl daemon-reload` ausführen.

### 5. Protokollierung und Fehlersuche

Debian protokolliert über **systemd-journald** und stellt die Daten über
`journalctl` bereit. Das klassische Syslog-Verzeichnis `/var/log/` gibt es
weiterhin, wird aber je nach Version und Installation nur teilweise befüllt.

| Befehl | Zweck |
| --- | --- |
| `journalctl -b` | Protokoll des aktuellen Boots |
| `journalctl -p err` | Fehler und höher |
| `journalctl -u nginx` | Protokoll einer Unit |
| `journalctl --since "2026-01-15 10:00" --until "2026-01-15 12:00"` | Zeitfenster |
| `journalctl -k` | Kernel-Meldungen |
| `journalctl -f` | Protokoll laufend verfolgen |
| `systemd-analyze blame` | Zeitverbrauch beim Systemstart |

Weitere Werkzeuge:

```bash
df -h                      # Belegung der Dateisysteme
free -h                    # Arbeitsspeicher
uptime                     # Laufzeit und Last
dmesg | tail -n 50         # Kernel-Ringpuffer
lsblk -f                   # Blockgeräte und Dateisysteme
ss -tulpn                  # offene TCP-/UDP-Sockets mit Prozess
```

Netzwerkdiagnose (Paket `dnsutils` und `iputils-ping`):

```bash
ping -c 3 192.168.10.1
traceroute 192.168.10.1
resolvectl query files01.contoso.example
```

### 6. Updates, Firewall und grundlegende Absicherung

Sicherheitsaktualisierungen:

```bash
# Nur Sicherheitsupdates aus einem zusätzlichen Repository
sudo apt update
apt list --upgradable | grep -i security
sudo apt upgrade
```

Die Konfiguration des Paketmanagements liegt in
`/etc/apt/sources.list` und `/etc/apt/sources.list.d/*.list` bzw. `*.sources`.
Je nach Release wird der klassische Einzeilen- oder das neue
`deb822`-Format verwendet.

**Firewall:** Debian stellt nicht genau eine, sondern mehrere Firewall-Frameworks
bereit. `nftables` (der moderne Nachfolger von `iptables`) ist in aktuellen
Releases verfügbar, `ufw` und `firewalld` sind zusätzliche Pakete und nicht Teil
der Standardinstallation. Welche Variante auf einem System verwendet wird,
richtet sich nach der Installation – nicht nach einer Vorgabe von Debian.

```bash
# ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.10.0/24 to any port 445 proto tcp
sudo ufw enable
sudo ufw status verbose
```

```bash
# nftables (Regeldatei /etc/nftables.conf)
sudo nft list ruleset
```

Grundlegende Absicherung:

1. Ein **eigenes Administrationskonto** statt `root` per SSH verwenden.
2. **Schlüsselbasierte Authentifizierung** statt Passwörter einrichten:

   ```bash
   sudo install -d -m 700 -o anna -g anna /home/anna/.ssh
   # öffentlichen Schlüssel in /home/anna/.ssh/authorized_keys ablegen
   sudo chmod 600 /home/anna/.ssh/authorized_keys
   ```

   Erst wenn die Schlüsselanmeldung funktioniert, Passwortanmeldung in
   `/etc/ssh/sshd_config` einschränken und `sudo systemctl reload ssh`
   ausführen – und eine zweite Sitzung offen lassen.
3. `PermitRootLogin` bewusst setzen (empfohlen: `no`).
4. SSH auf einen nicht standardmäßigen Port legen ist nur eine
   Zusatzmaßnahme und ersetzt keine Absicherung der Anmeldung.
5. Fail2ban oder eine vergleichbare Automatik für Brute-Force-Schutz
   einrichten.
6. Unbenötigte Dienste deaktivieren: `sudo systemctl disable --now dienst`.
7. Automatische Sicherheitsupdates mit `unattended-upgrades` erwägen.
8. Offene Ports regelmäßig kontrollieren: `ss -tulpn`.
9. **Zeitsynchronisation** über `systemd-timesyncd` oder chrony – fehlende
   Synchronisation führt zu Zertifikats- und Protokollproblemen.
10. **Keine Zugangsdaten im Repository.** Geheimnisse gehören in
    verschlüsselte Dateien außerhalb des Versionsverzeichnisses oder in einen
    Secret Store.

### 7. Backup und Wiederherstellung

Aufteilung der Verantwortung:

- **Backups** auf einem anderen System oder Medium abgelegt, idealerweise
  getrennt vom Produktivsystem (3-2-1 als verbreitetes Muster).
- **Restore** vor Ort testen – eine ungetestete Sicherung ist keine Sicherung.

Ein verbreitetes Werkzeug auf Debian ist `rsync`.

```bash
# Lokales Ziel (Beispiel)
sudo rsync -aHAX --delete --numeric-ids /etc/ /backup/etc/

# Auf ein entferntes Ziel (Beispiel, fiktiver Host)
sudo rsync -aHAX --delete /srv/ /home/backup/srv/
```

Für zusammenhängende Images eignen sich Werkzeuge auf Basis von LVM-Snapshots,
`tar`, `dd` oder ZFS/Btrfs – die Auswahl hängt vom Speicherlayout ab.

> **Achtung:** `--delete` löscht Dateien im Ziel, die in der Quelle nicht mehr
> vorhanden sind. Vor dem ersten Lauf das Zielverzeichnis prüfen; die Option
> gehört nur in sorgfältig geplante, wiederkehrende Sicherungsläufe.

Wiederherstellung:

```bash
# Aus einem tar-Archiv (Beispiel)
tar -xzf /backup/etc-2026-01-15.tar.gz -C /etc
```

Nach einem Restore die Dateieigentümer und -rechte prüfen: mit `tar -p` beim
Entpacken können sie erhalten bleiben.

---

## Sicherheitshinweise und typische Fehler

| Fehler | Auswirkung | Gegenmaßnahme |
| --- | --- | --- |
| `apt update` überspringen | `install`/`upgrade` schlägt fehl, veraltete Listen | Update immer direkt vor der Installation ausführen |
| `apt install` ohne Test in Produktion | unerwartete Paketwechsel | `apt full-upgrade` zuerst in Staging testen |
| Paket `sudo` nicht installiert | Konten ohne Rechte | `sudo` aus dem Standard-Installationsmedium mitnehmen |
| `apt remove` statt `apt purge` | Konfigurationsdateien bleiben zurück | bewusst entscheiden, welche Dateien erhalten bleiben sollen |
| Alle Rechte weltweit (`chmod -R 777`) | jeder Benutzer kann alles | gezielt `chown`/`chmod` mit Zahlenwerten setzen |
| Als `root` im Netz arbeiten | Fehlbedienung trifft das ganze System | `sudo` nur punktuell verwenden |
| Firewall-Regel ohne Portfreigabe vergessen | Dienst lokal erreichbar, von außen nicht | Regeln und Erreichbarkeit gemeinsam testen |
| Shell-Skripte ohne `set -e` und ohne Prüfung | Fehler laufen unbemerkt weiter | Fehlerbehandlung und Quoting (`"$var"`) konsequent verwenden |
| Paketmanager auf einem Skript ausführen | Abhängigkeiten werden manipuliert | systemweite Updates dem Paketmanager überlassen |
| Backup auf demselben Datenträger | Verlust bei Defekt des Datenträgers | mindestens eine Kopie offline/getrennt |
| `rsync --delete` ohne Prüfung | Datenverlust im Ziel | Ziel und Quelle vorab kontrollieren |
| Netzwerk- und Firewall-Konfiguration | doppelte Zuständigkeiten | nur ein Verfahren pro System nutzen (`ifupdown` oder `systemd-networkd`) |

---

## Weiterführende Quellen

### Offizielle Debian-Dokumentation

- Debian-Projektseite: <https://www.debian.org/>
- Debian Releases (Versionen und Unterstützungszeiträume):
  <https://www.debian.org/releases/>
- Debian Handbook: <https://www.debian.org/doc/manuals/debian-handbook/>
- Installationshandbuch der aktuellen Version:
  <https://www.debian.org/releases/stable/installmanual>
- Debian Wiki: <https://wiki.debian.org/>
- Debian Reference: <https://www.debian.org/doc/manuals/reference/>
- Debian Release Notes (aktuelle Version):
  <https://www.debian.org/releases/stable/release-notes/>

### Weitere Verweise in diesem Repository

- [Windows Server – Administration](windows-server.md)
- [Private IPv4-Adressbereiche](../networking/private-ipv4.md)
- [VLSM – Variable Length Subnet Masking](../networking/vlsm.md)

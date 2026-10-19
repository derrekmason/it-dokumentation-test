# FTP, FTPS und SFTP

## Zweck und Geltungsbereich

Diese Dokumentation beschreibt FTP als Dateiübertragungsdienst, seine
Funktionsweise, die Unterscheidung der drei verbreiteten Varianten sowie die
daraus folgenden Sicherheits- und Firewall-Anforderungen.

**Behandelt wird:**

- Zweck und Grundlagen von FTP
- Steuer- und Datenverbindung
- aktiver und passiver Modus
- Netzwerk- und Firewall-Anforderungen
- Authentifizierung und Berechtigungen
- klassisches FTP vs. FTPS vs. SFTP
- typische Verbindungsprobleme
- Sicherheitsrisiken und sichere Alternativen

**Nicht behandelt wird:** die Installation konkreter FTP-Serverprodukte.
Befehlszeilen-Beispiele verwenden einen generischen Client, weil
Kommandozeilen bei FTP-Dialekten je nach Server abweichen.

Alle Beispieldaten (Hostnamen, Adressen, Konten) sind fiktiv.

---

## Grundlagen

### Zweck von FTP

FTP (File Transfer Protocol) ist ein Anwendungsprotokoll zum Austausch von
Dateien zwischen Hosts. Es ist in [RFC 959](https://www.rfc-editor.org/rfc/rfc959)
als Standard beschrieben und arbeitet oberhalb von TCP.

Historische Entwurungsziele waren:

- Übertragung ganzer Dateien
- bequeme Verwaltung entfernter Verzeichnisse
- Auswahl zwischen Binär- und ASCII-Übertragung
- zuverlässige Übertragung großer Dateien über nicht zuverlässige Leitungen

**FTP war nie für Vertraulichkeit ausgelegt.** Benutzername und Kennwort werden
im Klartext übertragen, ebenso die Dateiinhalte. Das ist eine Eigenschaft des
Protokolls, keine Fehlkonfiguration – die späteren Standards FTPS und SFTP
wurden genau deshalb entwickelt.

### Steuer- und Datenverbindung

FTP benutzt **zwei** TCP-Verbindungen pro Sitzung:

| Verbindung | Zweck | Standard-Verhalten |
| --- | --- | --- |
| **Steuerverbindung** (Control Connection) | Befehle vom Client an den Server und Antworten (Statuscodes) | TCP-Port **21**, dauerhaft gehalten |
| **Datenverbindung** (Data Connection) | Dateiinhalte und Verzeichnislisten | je nach Modus ein- oder ausgehend, Port variabel |

Zusätzlich wird eine dritte, kurzlebige Verbindung für den Kanal zwischen
Client und Server bei aktiven Transfers benötigt (siehe aktiver Modus).

Der **FTP-Antwortcode** ist dreistellig; die erste Ziffer beschreibt die
Klasse:

| Ziffer | Bedeutung | Beispiel |
| --- | --- | --- |
| 1 | vorläufige Antwort, weiteren Code erwartet | `150 File status okay, about to open data connection` |
| 2 | positiv abgeschlossen | `200 Command okay` |
| 3 | positiv, erste Kennung folgt | `331 User name okay, need password` |
| 4 | vorübergehender Fehler | `421 Service not available` |
| 5 | dauerhafter Fehler | `530 Not logged in` |

Wichtige Befehle (Syntax nach RFC 959):

| Befehl | Zweck |
| --- | --- |
| `USER` / `PASS` | Anmeldedaten übermitteln |
| `TYPE` | Übertragungsmodus (ASCII/Binary) |
| `PASV` | passiven Modus anfordern |
| `PORT` | Adresse und Port für aktiven Modus übermitteln |
| `LIST` / `NLST` | Verzeichnisinhalt auflisten |
| `RETR` | Datei vom Server holen |
| `STOR` | Datei zum Server senden |
| `DELE` | Datei löschen |
| `CWD` / `PWD` | Verzeichnis wechseln / anzeigen |
| `QUIT` | Sitzung beenden |

### Aktiver und passiver Modus

#### Passiver Modus (`PASV`, Standard)

1. Der Client öffnet die Steuerverbindung zu Port 21 und sendet `PASV`.
2. Der Server antwortet mit einem Port, den der Client für die Datenverbindung
   verwenden soll (typischerweise ein Port ≥ 1024, gern aus dem ephemeren
   Bereich).
3. **Der Client öffnet die Datenverbindung** vom Server zum angegebenen Port.

Der Server muss keine eingehende Verbindung zum Client aufbauen. Das ist der
grund, warum der passive Modus heute der Regelfall ist: Clients liegen
hinter NAT und besitzen selten eine öffentlich erreichbare Adresse.

#### Aktiver Modus (`PORT`, veraltet)

1. Der Client öffnet die Steuerverbindung und sendet `PORT` mit seiner
   eigenen Adresse und einem Port.
2. **Der Server öffnet die Datenverbindung** vom Client weg, typischerweise
   von Quellport **20**.

Der Server muss den Client erreichen können. Hinter NAT, in DMZ oder mit
Firewall ist das in aller Regel nicht möglich.

#### Vergleich

| Merkmal | Passiv (`PASV`) | Aktiv (`PORT`) |
| --- | --- | --- |
| Wer verbindet sich für Daten? | Client → Server | Server → Client |
| Typischer Quellport | ephemerer Client-Port | Port 20 des Servers |
| Hinter NAT nutzbar | ja | nein |
| Ausgangszustand der Firewall | Client muss ausgehende Verbindungen erlauben | Server muss eingehende Verbindungen erlauben |
| Empfehlung | verwenden | nur in geschützten Altumgebungen |

### Die drei Varianten: FTP, FTPS und SFTP

> **FTP, FTPS und SFTP sind drei verschiedene Protokolle.**
> FTPS ist FTP mit TLS. SFTP ist **kein FTP und kein FTPS**, sondern ein
> Dateiübertragungsprotokoll, das als Subsystem über SSH läuft.

| Merkmal | FTP | FTPS | SFTP |
| --- | --- | --- | --- |
| Grundlage | TCP, RFC 959 | TLS über TCP, dann FTP-Kommandos | SSH, dann eigenes Dateiübertragungsprotokoll |
| Standardport | 21 | 21 (explizit) bzw. 990 (implizit) | 22 (SSH) |
| Verschlüsselung | keine (Klartext) | ja, Steuer- und Datenkanal | ja, durch SSH |
| Datenzugriff | nur Server-Dateisystem | nur Server-Dateisystem | nur Server-Dateisystem |
| Moduswahl | `PASV` / `PORT` | wie FTP | **keine** – das Konzept entfällt, der Client baut die Verbindung auf |
| Protokollfehler bei Verschlüsselung | – | typisch, wenn nur ein Kanal geschützt ist | – |
| Verbreitung | sehr verbreitet (Legacy) | verbreitet | heute bevorzugt für Automatisierung und Administration |

**Klassisches FTP:** Unverschlüsselt. Zugangsdaten und Dateninhalte werden im
Klartext übertragen. Nur in geschützten, abgegrenzten Netzen vertretbar.

**FTPS (FTP über TLS):** Dasselbe FTP-Protokoll, aber der Handshake mit TLS wird
über die Option `AUTH TLS` angefordert (explizites FTPS, Standardport 21,
beschrieben in [RFC 2228](https://www.rfc-editor.org/rfc/rfc2228.html)) oder die
Verbindung wird direkt als TLS auf Port 990 aufgebaut (implizites FTPS,
[RFC 7151](https://www.rfc-editor.org/rfc/rfc7151)). Die Regeln für die
Absicherung des Datenkanals stehen in [RFC 4217](https://www.rfc-editor.org/rfc/rfc4217).

**SFTP:** Läuft als Subsystem über eine bestehende SSH-Verbindung (Port 22,
siehe [RFC 4253](https://www.rfc-editor.org/rfc/rfc4253) für SSH) und benutzt
dessen Authentifizierung, Schlüssel und Verschlüsselung. Das eigentliche
SFTP-Protokoll ist im IETF-Entwurf `draft-ietf-secsh-filexfer` beschrieben
(<https://www.ietf.org/archive/id/draft-ietf-secsh-filexfer-13.txt>).

Es gibt **kein** `PASV`/`PORT` bei SFTP: Der Client wählt einen lokalen Port
und verbindet sich zum SSH-Server. Die historische Notwendigkeit für passive
Modi entfällt damit – ein praktischer Vorteil in Firewallumgebungen.

---

## Vorgehen und Beispiele

### Verbindungsaufbau (klassisch, Klartext – nur in geschützten Netzen)

```bash
# Verbindung zu einem fiktiven Server (Hinweis: unverschluesselt)
# Option -n unterdrueckt die automatische Anmeldung aus ~/.netrc
ftp -n 192.168.10.20
```

Innerhalb der Sitzung:

```text
> open 192.168.10.20      # Verbindung zu einem Server
> user anna              # Benutzername (im Klartext)
> pass                   # Kennwort (im Klartext)
> binary                 # Binaeruebertragung waehlen
> pwd                    # aktuelles Verzeichnis
> cd /pub                # Verzeichnis wechseln
> ls                     # Inhalt anzeigen
> get bericht.pdf        # Datei holen
> put notizen.txt        # Datei senden
> bye                    # Sitzung beenden
```

> **Warnung:** Die Befehle `user` und `pass` übertragen die Zugangsdaten
> unverschlüsselt. Für produktive Daten ist dieser Weg nicht geeignet.

### Verbindungsaufbau mit SFTP

```bash
# Schlüsselbasierte Anmeldung, verschluesselte Uebertragung
sftp anna@192.168.10.20

# Nicht-interaktiver Transfer einer einzelnen Datei (fiktives Beispiel)
sftp -i ~/.ssh/id_ed25519 -b - anna@192.168.10.20 <<'EOF'
get /pub/bericht.pdf ./bericht.pdf
put ./notizen.txt /pub/notizen.txt
bye
EOF
```

Grundlagen zu SSH und Schlüsseln: <https://openssh.com/manual.html>.

### Netzwerk- und Firewall-Anforderungen

| Variante | Regel in der Firewall |
| --- | --- |
| FTP passiv | **Ausgehend** vom Client: TCP 21 (Steuer) und **TCP dynamisch** (Daten, Server wählt den Port) |
| FTP aktiv | Ausgehend TCP 21; **eingehend** TCP 20 vom Server zum Client |
| FTPS explizit | wie FTP, zusätzlich Ausgang zu Port 990 bzw. TLS auf 21 |
| FTPS implizit | Ausgehend TCP 990 |
| SFTP | **Ausgehend** TCP 22 |

Weitere Hinweise:

- Der Server kann den Datenbereich einschränken. Typisch ist ein fester
  Bereich (klassisch 50000–55000), der auch als `passive port range` genannt
  wird. Für eine genaue Einstellung ist die Dokumentation des konkreten
  Servers maßgeblich.
- Für den passiven Modus wird häufig zusätzlich eine Regel auf Basis von
  **FTP-Dienst-Ports** (Dienst-Ports 20–21) genutzt, wenn die Firewall
  Zielports dynamisch bestimmen muss.
- Eine gemeinsame Fehlerquelle: Der Server liegt in einer DMZ oder wird über
  einen Reverse Proxy veröffentlicht. Dann müssen sowohl die Firewall
  **als auch** der Proxy den passiven Modus unterstützen.
- In Container- und Cloud-Umgebungen ist zusätzlich zu prüfen, ob
  Verbindungen vom Container-Netz in das Backend zugelassen werden.

### Authentifizierung und Berechtigungen

| Verfahren | Beschreibung | Bewertung |
| --- | --- | --- |
| Benutzername und Kennwort | Standard bei FTP | Bei unverschlüsseltem FTP im Klartext |
| Anonymer Zugang | Benutzer `anonymous` (oft `ftp`) mit beliebiger E-Mail-Adresse | Nur für bewusst öffentliche Inhalte |
| Virtueller Benutzer | Serverseitig hinterlegte Zugangsdaten ohne UNIX-Konto | Gute Trennung vom System |
| TLS-Clientzertifikat | Zusätzliche Identifikation des Clients (FTPS) | Stark, aber higher Aufwand |
| SSH-Schlüssel (SFTP) | Öffentlicher Schlüssel auf dem Server hinterlegt | heute üblich und empfehlenswert |

Rechte auf Server- und App-Ebene:

1. **Dateisystemrechte** – Benutzer auf ein eigenes Heimatverzeichnis und auf
   definierte Bereiche beschränken.
2. **Jail/Chroot oder virtuelles Root** – Benutzer sehen nur einen begrenzten
   Ausschnitt des Dateisystems.
3. **Anonymous-Uploads deaktivieren**, wenn sie nicht benötigt werden.
4. **Schreibzugriff auf Systemverzeichnisse immer ausschließen.**

> **Typische Fehlkonfiguration:** Ein FTP-Benutzer landet in einem Verzeichnis,
> in dem er durch ein gemeinsames Verzeichnis (World-readable) oder durch
> unpassende Eigentümerrechte auch Systemdateien sieht.

### Typische Verbindungsprobleme

| Symptom | Mögliche Ursache | Prüfung |
| --- | --- | --- |
| Login erfolgreich, `ls` oder `get` bleibt hängen | passiver Modus, Datenverbindung wird blockiert | Firewall-Regeln für den Datenport; `tcpdump`/`Wireshark` auf dem Server |
| `425 Can't open data connection` | Datenport außerhalb des erlaubten Bereichs oder Server kann Client nicht erreichen | Konfiguration der Firewall und des `passive port range` |
| `530 Not logged in` oder `530 Login incorrect` | Anmeldedaten falsch, Konto gesperrt, oder Benutzer ohne gültiges Heimatverzeichnis | Server-Log; Benutzerrechte prüfen |
| Verbindung bricht nach dem `USER` ab | Klartext-Anmeldung, die der Server nicht mehr akzeptiert (Policy) | TLS-Modus (FTPS) oder SFTP verwenden |
| Zeitverzögerung beim Verbinden | reverse DNS-Auflösung beim Verbindungsaufbau | `UseReverseDNS no` bzw. `reverse DNS` in der Serverdokumentation |
| Listing gelingt, `get` scheitert | Berechtigung im Zielpfad oder volles Dateisystem | Rechte und `df -h` prüfen |
| Sehr langsame Übertragung | Moduswechsel, viele kleine Dateien, MTU- oder Latenzproblem | `passiven` Modus, `binary`, Netzwerkanalyse |

---

## Sicherheitshinweise und Alternativen

### Risiken von klassischem FTP

1. **Klartext-Zugangsdaten.** Benutzername und Kennwort sind für jeden mitlesbar,
   der den Datenverkehr beobachten kann.
2. **Klartext-Dateiinhalte.** Auch die Daten selbst sind mitles- und veränderbar.
3. **Übertragung nur über die Steuerverbindung.** Ist nur die Steuerverbindung
   geschützt, bleiben die Daten ungeschützt – ein sehr typischer Fehler bei
   FTPS. Der Datenkanal muss **ebenfalls** durch TLS geschützt werden
   (RFC 4217).
4. **Angriffsfläche für Brute-Force und Portscans.** Ein offen erreichbarer
   FTP-Port auf dem Internet wird systematisch abgeklopft.
5. **Missbrauch als Ablage für Schadsoftware** bei erlaubten Uploads.
6. **Fehlkonfiguration von Berechtigungen** ermöglicht Lesezugriff auf Dateien
   außerhalb des vorgesehenen Bereichs.

### Sichere Alternativen

| Anforderung | Empfehlung |
| --- | --- |
| Dateiübertragung für Administration und Skripte | **SFTP** (über SSH) |
| Dateiübertragung mit vorhandenem FTP-Protokoll, aber verschlüsselt | **FTPS** (explizit oder implizit) |
| Dateiaustausch mit externen Partnern, die FTP erwarten | FTPS, Zugang auf einzelne Verzeichnisse, Zeitfenster, Protokollierung |
| Dateiübertragung für Webanwendungen / öffentliche Bereitstellung | HTTPS-Upload oder Objektspeicher-Schnittstelle statt FTP |

Die konkrete Wahl hängt von den vorhandenen Systemen, den Anforderungen an
Kompatibilität und der Compliance-Vorgabe ab. Grundsatz: Für sensible Daten
wird eine angemessen abgesicherte Lösung verwendet – das ist praktisch immer
SFTP oder FTPS, nie unverschlüsseltes FTP.

> **Verwechslungsgefahr:** „FTP aktivieren“ oder „FTP-Konto“ bezeichnet in
> vielen Tickets SFTP. Im Zweifel ist die **Portnummer** das entscheidende
> Merkmal: 21 = FTP/FTPS, 990 = implizites FTPS, 22 = SSH/SFTP.

---

## Weiterführende Quellen

### Standards

- RFC 959 – File Transfer Protocol (STD 9):
  <https://www.rfc-editor.org/rfc/rfc959>
- RFC 2228 – Extensions to FTP for TLS Support:
  <https://www.rfc-editor.org/rfc/rfc2228.html>
- RFC 4217 – Securing FTP with TLS (BCP 97):
  <https://www.rfc-editor.org/rfc/rfc4217>
- RFC 7151 – Implicit FTPS/FTP over TLS:
  <https://www.rfc-editor.org/rfc/rfc7151>
- RFC 1123 – Requirements for Internet Hosts, Abschnitt 4.1.2.12 (FTP):
  <https://www.rfc-editor.org/rfc/rfc1123>
- RFC 4253 – SSH Transport Layer Protocol:
  <https://www.rfc-editor.org/rfc/rfc4253>
- draft-ietf-secsh-filexfer – SFTP File Transfer Protocol:
  <https://www.ietf.org/archive/id/draft-ietf-secsh-filexfer-13.txt>

### Weiterführend

- OpenSSH – Manual: <https://openssh.com/manual.html>

### Weitere Verweise in diesem Repository

- [Windows Server – Administration](../systems/windows-server.md)
- [Debian Linux – Administration](../systems/debian.md)
- [Merge-Konflikte](../git/merge-conflicts.md)

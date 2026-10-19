# Private IPv4-Adressbereiche

## Zweck und Geltungsbereich

Diese Dokumentation beschreibt, was private IPv4-Adressen sind, welche Bereiche
per RFC 1918 dafür reserviert sind und welche praktischen Konsequenzen ihre
Verwendung hat.

**Behandelt wird:**

- Definition und Zweck privater IPv4-Adressen
- die drei reservierten Bereiche nach RFC 1918
- Netzadressen und CIDR-Notation
- typische Verwendungszwecke und die Rolle von NAT
- Abgrenzung privat/öffentlich sowie häufige Fehlannahmen

**Nicht behandelt wird:** die Planung vollständiger IPv4-Netzwerke (dazu siehe
[VLSM – Variable Length Subnet Masking](vlsm.md)) und die Umstellung auf IPv6.

Alle Adressen in diesem Dokument sind fiktive Beispiele.

---

## Grundlagen

### Was bedeutet „privat“?

Nach RFC 1918 hat die IANA drei Adressblöcke des IPv4-Adressraums für den
Einsatz in *privaten Internets* reserviert – also in Netzen, die ihr Adressierungsschema
selbst festlegen.

Adressen aus diesen Bereichen sind **nicht global eindeutig**. Zwei völlig
unabhängige Unternehmen dürfen beide `192.168.1.10` verwenden, ohne dass ein
Konflikt entsteht – solange die Netze nicht direkt miteinander verbunden sind.

RFC 1918 unterscheidet drei Kategorien von Hosts:

| Kategorie | Bedarf an Verbindungen nach außen | Adressierung |
| --- | --- | --- |
| 1 | keiner | privat ausreichend |
| 2 | wenige externe Dienste (Mail, FTP, WWW) über ein Gateway | privat ausreichend |
| 3 | Netzschicht-Zugriff ins Internet | global eindeutig erforderlich |

Die Kategorien 1 und 2 gelten laut RFC 1918 als „privat“, Kategorie 3 als „public“.

### Die drei privaten Bereiche (RFC 1918)

| Bereich (CIDR) | Erste Adresse | Letzte Adresse | Präfix | Subnetzmaske | Anzahl Adressen |
| --- | --- | --- | --- | --- | --- |
| `10.0.0.0/8` | 10.0.0.0 | 10.255.255.255 | /8 | 255.0.0.0 | 16.777.216 |
| `172.16.0.0/12` | 172.16.0.0 | 172.31.255.255 | /12 | 255.240.0.0 | 1.048.576 |
| `192.168.0.0/16` | 192.168.0.0 | 192.168.255.255 | /16 | 255.255.0.0 | 65.536 |

> **Wichtig – der Bereich um `172`:**
> Der private Bereich mit `172` umfasst **ausschließlich** die Adressen von
> **`172.16.0.0` bis `172.31.255.255`**. Er beginnt also bei der *zweiten*
> Oktettgruppe 16 und endet bei 31.
> `172.15.0.0` und `172.32.0.0` liegen **außerhalb** des reservierten Bereichs und
> gehören zum öffentlichen Adressraum. Ebenso ist `172.32.x.x` kein privater Bereich.

Die Angabe `/12` bedeutet: Die ersten 12 Bit der Adresse gehören zum Netzpräfix,
also die ersten beiden Oktettgruppen (`172.16` … `172.31`). Deshalb ist die
Untergrenze `172.16.0.0` und nicht `172.0.0.0`.

### Netzadresse und Broadcast-Adresse

Jedes IPv4-Netz besitzt genau:

- eine **Netzadresse** (Netzwerkadresse) – alle Host-Bits sind `0`,
- eine **Broadcast-Adresse** – alle Host-Bits sind `1`.

Beide Adressen sind nicht als Hostadresse verwendbar. Eine weitere Adresse kann
durch Ausnahme entfallen (siehe Hinweis zu `/31` in
[VLSM](vlsm.md#sonderfaelle--31-und--32)).

Beispiel für `192.168.1.0/24`:

| Rolle | Adresse |
| --- | --- |
| Netzadresse | 192.168.1.0 |
| Erster nutzbarer Host | 192.168.1.1 |
| Letzter nutzbarer Host | 192.168.1.254 |
| Broadcast-Adresse | 192.168.1.255 |

Die CIDR-Schreibweise `/24` ist die Kurzform der Subnetzmaske `255.255.255.0`.
Beide Schreibweisen bezeichnen dasselbe Netz.

---

## Vorgehen und Beispiele

### Beispiel: LAN eines kleinen Büros

Ein fiktives Büro mit einem Router, einem Drucker, einem NAS und 12
Arbeitsplätzen nutzt `192.168.10.0/24`:

| Gerät | Adresse | Zweck |
| --- | --- | --- |
| Router (Gateway) | 192.168.10.1 | Weiterleitung, NAT, DNS-Weiterleitung |
| NAS | 192.168.10.10 | Dateiablage |
| Drucker | 192.168.10.20 | Netzwerkdruck |
| Arbeitsplatz 1 | 192.168.10.101 | statisch vergeben |
| Arbeitsplatz 2 | 192.168.10.102 | per DHCP vergeben |
| … | … | … |

Die Planung mit `192.168.0.0/16` gibt ausreichend Platz für mehrere /24-Netze
(z. B. `192.168.10.0` für Büro, `192.168.20.0` für Testumgebung).

### Beispiel: Unterschiedliche Standorte

Zwei Standorte verwenden beide `10.0.0.0/8`, um einen späteren Zusammenschluss
zu erleichtern. Sie kommunizieren über ein VPN; die privaten Adressen bleiben
dabei erhalten, weil sie nur innerhalb des Tunnels übertragen werden.

### Beispiel: Verbindung ins Internet (NAT)

Ein Host mit `192.168.10.102` ist direkt nicht aus dem Internet erreichbar.
Der Router übersetzt seine Adresse beim Verlassen des Netzes:

```text
Intern:   192.168.10.102:51000  ->  Ausgang: 203.0.113.15:62000
```

Der Router führt dazu eine Übersetzungstabelle (PAT/NAT). Antwortpakete werden anhand
der Tabelle wieder dem internen Host zugeordnet.

### Werkzeuge zur Prüfung

```bash
# IPv4-Konfiguration und Maske anzeigen (Linux)
ip -4 addr show

# Standardgateway und Routen anzeigen
ip route show

# Erreichbarkeit prüfen
ping -c 3 192.168.10.1
```

```powershell
# IPv4-Konfiguration anzeigen (Windows)
ipconfig /all

# Standardgateway prüfen
Test-Connection -TargetName 192.168.10.1 -Count 3
```

---

## Private gegenüber öffentliche Adressen

| Merkmal | private Adresse | öffentliche Adresse |
| --- | --- | --- |
| global eindeutig | nein | ja |
| Vergabe | frei wählbar durch das Unternehmen | durch ein Internet-Registry (RIPE, ARIN, …) zugeteilt |
| im Internet geroutet | nein | ja |
| direkt aus dem Internet erreichbar | nein | ja (abhängig von Firewall und NAT) |
| RFC | RFC 1918 | u. a. RFC 6890 (Sonderadressen) |

**Weitere Adressbereiche, die nicht öffentlich geroutet werden** (RFC 6890):
`127.0.0.0/8` (Localhost), `169.254.0.0/16` (Link-Local, „APIPA“),
`0.0.0.0/8`, Multicast-Bereiche u. a.

Diese Bereiche sind *kein* Ersatz für RFC-1918-Adressen. So ist
`169.254.x.x` ausschließlich zur autonomen Adresskonfiguration eines einzelnen
Links gedacht und sollte nicht für eine geplante Infrastruktur verwendet werden.

---

## Rolle von NAT

NAT (Network Address Translation) hat in diesem Zusammenhang zwei Aufgaben:

1. **Adresserhaltung (Konservierung):** Viele interne Hosts benötigen keinen
   eigenen öffentlichen Adressraum. RFC 1918 nennt ausdrücklich, dass Unternehmen
   öffentliche Adressungen nur für die Hosts benötigen, die echten
   Netzschicht-Zugriff nach außen brauchen.
2. **Adressübersetzung:** Der Router ersetzt die private Quelladresse durch eine
   öffentliche Adresse, häufig kombiniert mit Portübersetzung (PAT/NAPT).

Typische Betriebsformen:

- **1:1-NAT (Static NAT):** Ein interner Host wird dauerhaft auf eine öffentliche
  Adresse abgebildet, z. B. für einen internen Webserver.
- **PAT / NAT Overload:** Viele interne Hosts teilen sich eine öffentliche Adresse,
  unterscheidbar über die Portnummer. Dies ist der Default bei Consumer-Routern.
- **Outbound NAT:** Nur ausgehende Verbindungen werden übersetzt. Eingehende
  Verbindungen benötigen zusätzlich eine Portweiterleitung (Port Forwarding).

---

## Sicherheitshinweise und typische Fehler

> **Private Adressierung ist kein Sicherheitskonzept.**
> Der Umstand, dass ein Gerät eine `192.168.x.x`-Adresse hat, bedeutet weder, dass
> es geschützt ist, noch dass es von außen nicht erreichbar wäre. Sicherheit entsteht
> durch Firewalls, Segmentierung, Authentifizierung und Verschlüsselung.

- **Private Adressen werden nicht öffentlich geroutet.** RFC 1918 ist Best
  Current Practice (BCP 5). Provider filtern private Quell- und Zieladressen
  an ihren Grenzen; solche Pakete erreichen das Internet in der Regel nicht.
- **Adresskonflikte bei Zusammenschlüssen.** Werden zwei Netze zusammengeführt,
  die beide `192.168.1.0/24` verwenden, entstehen doppelte Adressen. RFC 1918
  empfiehlt, Subblöcke aus dem privaten Bereich möglichst zufällig zu wählen, um
  solche Konflikte zu reduzieren.
- **Routeninformationen dürfen nicht nach außen gegeben werden.** RFC 1918: Private
  Adressen sollen nicht über Verbindungen zwischen Unternehmen hinweg propagiert
  werden. Der Datenverkehr zwischen zwei Firmen muss gekapselt (z. B. per VPN)
  werden.
- **DNS-Split-Horizon (Split-Brain DNS).** Private Adressen sollten nicht in
  öffentlich sichtbare DNS-Zonen gelangen. Üblich ist die Aufteilung in eine
  öffentliche und eine interne Sicht derselben Zone.
- **Häufige Fehlannahmen:**

| Fehlannahme | Sachverhalt |
| --- | --- |
| „Alles ab `172.` ist privat“ | Nur `172.16.0.0` – `172.31.255.255`. |
| „Private Adressen sind geschützt“ | Adresswahl ist kein Schutzmechanismus. |
| „Ein privater Host ist aus dem Internet nicht erreichbar“ | Bei vorhandenem Port-Forwarding, UPnP oder fehlkonfigurierten Gateways sehr wohl möglich. |
| „`192.168.1.0/24` und `192.168.0.0/16` sind dasselbe“ | Beide enthalten `192.168.1.0/24`, sind aber unterschiedlich große Netze. |
| „Ein privates Netz kann ohne Weiteres global geroutet werden“ | Nein; genau das ist der Zweck von RFC 1918 – diese Adressen sind nicht für globale Weitergabe gedacht. |

- **Umbenennung (Renumbering).** Der Wechsel zwischen privat und öffentlich
  erfordert eine Adressänderung, angepasste DNS-Einträge und die Anpassung aller
  Konfigurationsdateien, die Hosts per IP-Adresse referenzieren. DHCP erleichtert
  diesen Vorgang erheblich.

---

## Weiterführende Quellen

- RFC 1918 – Address Allocation for Private Internets
  <https://www.rfc-editor.org/rfc/rfc1918>
- RFC 6890 – Special-Purpose IP Address Registries
  <https://www.rfc-editor.org/rfc/rfc6890>
- RFC 5735 – IPv4 Address Representation
  <https://www.rfc-editor.org/rfc/rfc5735>
- RFC 4632 – CIDR: Classless Inter-domain Routing
  <https://www.rfc-editor.org/rfc/rfc4632>
- VLSM – Variable Length Subnet Masking (in diesem Repository)
  [docs/networking/vlsm.md](vlsm.md)

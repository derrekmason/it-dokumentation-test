# VLSM – Variable Length Subnet Masking

## Zweck und Geltungsbereich

Diese Dokumentation beschreibt die Planung eines IPv4-Netzwerks mit
unterschiedlich großen Subnetzen (**VLSM**) anhand eines vollständig
durchgerechneten Beispiels.

**Behandelt wird:**

- Bedeutung von VLSM und der Bezug zu CIDR-Präfix und Subnetzmaske
- Berechnung der benötigten Subnetzgrößen
- Netz-, Host- und Broadcast-Adressen
- ein durchgerechnetes Beispiel mit `192.168.10.0/24`
- typische Fehler bei der Subnetzplanung

**Nicht behandelt wird:** Routing-Protokolle im Detail, IPv6-Subnetzierung und
allgemeine Netzwerktopologie. Zum Thema private Adressbereiche siehe
[Private IPv4-Adressbereiche](private-ipv4.md).

Alle Adressen in diesem Dokument sind fiktive Beispiele.

---

## Grundlagen

### Was bedeutet VLSM?

Bei der klassischen, noch heute in Lehrbeispielen verbreiteten Planung mit
*fester* Subnetzmaske (FLSM) wird ein Netz in gleich große Subnetze zerlegt.
Das ist einfach, passt aber oft nicht zu den tatsächlichen Anforderungen: Ein
Subnetz für zwei Drucker wird genauso groß wie eines für 200 Arbeitsplätze.

**Variable Length Subnet Masking** bedeutet, jedes Subnetz **genau so groß** zu
wählen, wie es die tatsächliche Anforderung verlangt. Voraussetzung ist
CIDR-Notation, damit beliebige Netzgrößen (nicht nur Zweierpotenzen der
historischen Klassen A/B/C) verwendet werden können.

### CIDR-Präfix, Subnetzmaske und Hostanzahl

| CIDR-Präfix | Subnetzmaske | Hostbits | Adressen gesamt | Nutzbare Hosts |
| --- | --- | --- | --- | --- |
| /24 | 255.255.255.0 | 8 | 256 | 254 |
| /25 | 255.255.255.128 | 7 | 128 | 126 |
| /26 | 255.255.255.192 | 6 | 64 | 62 |
| /27 | 255.255.255.224 | 5 | 32 | 30 |
| /28 | 255.255.255.240 | 4 | 16 | 14 |
| /29 | 255.255.255.248 | 3 | 8 | 6 |
| /30 | 255.255.255.252 | 2 | 4 | 2 |

**Formeln:**

```text
Hostbits      = 32 - CIDR-Präfix
Adressen      = 2 ^ Hostbits
Nutzbare Hosts = 2 ^ Hostbits - 2
```

Die `-2` sind die **Netzadresse** (alle Hostbits 0) und die
**Broadcast-Adresse** (alle Hostbits 1). Beide können nicht als Hostadresse
vergeben werden.

### Netz-, Host- und Broadcast-Adresse

Für ein Subnetz mit Netzadresse `N` und Maske `M`:

```text
Netzadresse      = Adresse & Maske
Broadcast        = Adresse | ~Maske
Erster Host      = Netzadresse + 1
Letzter Host     = Broadcast - 1
Nutzbarer Bereich = Netzadresse + 1  bis  Broadcast - 1
```

---

## Vorgehen: Planungsalgorithmus

1. Ausgangsnetz und dessen Gesamtgröße bestimmen.
2. Alle Anforderungen **nach absteigender Hostanzahl sortieren**.
3. Pro Anforderung die benötigte Blockgröße bestimmen:
   `nötige Adressen = benötigte Hosts + 2` (Netz- und Broadcast-Adresse).
4. Die kleinste **Zweierpotenz** wählen, die ≥ der benötigten Adressenzahl ist.
5. Daraus das CIDR-Präfix ableiten (`Adressen = 2^n` → Präfix `32 - n`).
6. Subnetze der Reihe nach vom Netzstart aus vergeben – jeweils auf einer
   Blockgrenze beginnend, damit keine Lücken entstehen.

### Warum absteigend sortieren?

Wer die größte Anforderung zuerst bedient, vergibt den großen Block am
Anfang des Adressraums. Alle nachfolgenden kleineren Blöcke passen danach
zwangsläufig ohne Umrechnung in die verbleibenden Lücken – es entsteht eine
lückenlose, überschneidungsfreie Aufteilung.

Würde man mit der kleinsten Anforderung beginnen, könnte man das Netz in
viele kleine Blöcke zerlegen, für die große Anforderung bliebe aber nur noch
ein zusammenhängender Restbereich – oder man müsste den großen Block über
die hinaus vergeben. Die absteigende Reihenfolge ist deshalb die einfachste
und sicherste Regel.

---

## Rechenbeispiel: `192.168.10.0/24`

### Ausgangsdaten

| Angabe | Wert |
| --- | --- |
| Ausgangsnetz | `192.168.10.0/24` |
| Subnetzmaske | `255.255.255.0` |
| Adressen gesamt | 2⁸ = 256 |
| Nutzbare Hosts (ohne Teilung) | 254 |

Anforderungen, **absteigend sortiert**:

| # | Abteilung | Benötigte Hosts |
| --- | --- | --- |
| 1 | Büro / Arbeitsplätze | 50 |
| 2 | Serverraum | 25 |
| 3 | Veranstaltungstechnik | 10 |
| 4 | Management (Router, Switches) | 5 |

Summe der Anforderungen: 50 + 25 + 10 + 5 = **90 Hosts**.

### Berechnung je Subnetz

#### Subnetz 1 – 50 Hosts (Büro)

```text
benötigte Adressen = 50 + 2 = 52
kleinste Zweierpotenz >= 52  ->  64  (2^6)
Präfix: 32 - 6 = /26
```

| Eigenschaft | Wert |
| --- | --- |
| Netzadresse / CIDR | `192.168.10.0/26` |
| Subnetzmaske | `255.255.255.192` |
| Erster nutzbarer Host | `192.168.10.1` |
| Letzter nutzbarer Host | `192.168.10.62` |
| Broadcast-Adresse | `192.168.10.63` |
| Nutzbare Hosts | 62 (≥ 50 ✔) |
| Blockgröße | 64 Adressen |

#### Subnetz 2 – 25 Hosts (Serverraum)

```text
benötigte Adressen = 25 + 2 = 27
kleinste Zweierpotenz >= 27  ->  32  (2^5)
Präfix: 32 - 5 = /27
```

| Eigenschaft | Wert |
| --- | --- |
| Netzadresse / CIDR | `192.168.10.64/27` |
| Subnetzmaske | `255.255.255.224` |
| Erster nutzbarer Host | `192.168.10.65` |
| Letzter nutzbarer Host | `192.168.10.94` |
| Broadcast-Adresse | `192.168.10.95` |
| Nutzbare Hosts | 30 (≥ 25 ✔) |
| Blockgröße | 32 Adressen |

#### Subnetz 3 – 10 Hosts (Veranstaltungstechnik)

```text
benötigte Adressen = 10 + 2 = 12
kleinste Zweierpotenz >= 12  ->  16  (2^4)
Präfix: 32 - 4 = /28
```

| Eigenschaft | Wert |
| --- | --- |
| Netzadresse / CIDR | `192.168.10.96/28` |
| Subnetzmaske | `255.255.255.240` |
| Erster nutzbarer Host | `192.168.10.97` |
| Letzter nutzbarer Host | `192.168.10.110` |
| Broadcast-Adresse | `192.168.10.111` |
| Nutzbare Hosts | 14 (≥ 10 ✔) |
| Blockgröße | 16 Adressen |

#### Subnetz 4 – 5 Hosts (Management)

```text
benötigte Adressen = 5 + 2 = 7
kleinste Zweierpotenz >= 7  ->  8  (2^3)
Präfix: 32 - 3 = /29
```

| Eigenschaft | Wert |
| --- | --- |
| Netzadresse / CIDR | `192.168.10.112/29` |
| Subnetzmaske | `255.255.255.248` |
| Erster nutzbarer Host | `192.168.10.113` |
| Letzter nutzbarer Host | `192.168.10.118` |
| Broadcast-Adresse | `192.168.10.119` |
| Nutzbare Hosts | 6 (≥ 5 ✔) |
| Blockgröße | 8 Adressen |

### Gesamtergebnis

| # | Subnetz | CIDR | Maske | Erster Host | Letzter Host | Broadcast | Hosts | Belegt von/bis |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Büro | `192.168.10.0/26` | 255.255.255.192 | .1 | .62 | .63 | 62 | 0–63 |
| 2 | Serverraum | `192.168.10.64/27` | 255.255.255.224 | .65 | .94 | .95 | 30 | 64–95 |
| 3 | Veranstaltungstechnik | `192.168.10.96/28` | 255.255.255.240 | .97 | .110 | .111 | 14 | 96–111 |
| 4 | Management | `192.168.10.112/29` | 255.255.255.248 | .113 | .118 | .119 | 6 | 112–119 |

### Plausibilitätsprüfung

**1. Überschneidungsfreiheit.** Die Belegungsbereiche `0–63`, `64–95`, `96–111`,
`112–119` sind paarweise disjunkt – jedes Subnetz beginnt genau am ersten
Adresswert nach dem Broadcast des Vorgängers. Keine Überschneidung.

**2. Lückenlose Aufteilung.** Die Summe der Blockgrößen beträgt

```text
64 + 32 + 16 + 8 = 120 Adressen
```

Belegt wird also durchgehend `192.168.10.0` bis `192.168.10.119` – lückenlos.

**3. Innerhalb des Ausgangsnetzes.** `192.168.10.119` < `192.168.10.255`, und alle
Präfixe (`/26`, `/27`, `/28`, `/29`) sind länger als `/24`. Damit liegt jedes
Subnetz vollständig in `192.168.10.0/24`. ✔

**4. Freier Restbereich.** Es verbleiben `256 - 120 = 136` Adressen
(`192.168.10.120` – `192.168.10.255`) für Erweiterungen, z. B. ein weiteres
`192.168.10.120/26` oder `192.168.10.128/27`.

**5. Erfüllung aller Anforderungen.** 62 ≥ 50, 30 ≥ 25, 14 ≥ 10, 6 ≥ 5. ✔

**Kontrolle mit Werkzeugen (Linux):**

```bash
# Prüfung der Netz-/Broadcast-Adressen gegen die Maske
ipcalc -s 192.168.10.112/255.255.255.248     # Adresse: 192.168.10.112
ipcalc -s 192.168.10.113/255.255.255.248     # Host:    192.168.10.113
```

---

## Sonderfälle /31 und /32

Für reine Point-to-Point-Verbindungen zwischen zwei Routern sieht
[RFC 3021](https://www.rfc-editor.org/rfc/rfc3021) die Nutzung von `/31` vor:
Es gibt dann nur zwei Adressen, die traditionell als Netz- und
Broadcast-Adresse gelten. Moderne Implementierungen (z. B. BGP, systemd-networkd,
zahlreiche Router) behandeln `/31` als echte Hostverbindung.

Für einzelne Loopback-Adressen wird `/32` verwendet (z. B. `10.0.0.1/32`).

Für normale Hostnetze bleiben diese Sonderfälle außen vor – dort gilt weiterhin
`2^Hostbits - 2` nutzbare Adressen.

---

## Sicherheitshinweise und typische Fehler

- **Die `-2` vergessen.** Wer für 30 Hosts ein `/27` (32 Adressen) plant, hat nur
  30 nutzbare – aber sobald Gateway, DHCP-Server oder ein Routerinterface
  mitgezählt werden, geht der Platz aus. Netzwerkgeräte und Infrastruktur
  zählen zur Anforderung.
- **Feste Blockgröße für alle.** Wer ein `/24` in vier × `/26` aufteilt, verschenkt
  Adressraum und bindet das Netz unnötig an eine Größe. Bei Adressknappheit
  (IPv4) ist genau das ein Problem.
- **Erste und letzte Hostadresse übersehen.** Netzwerk- und Broadcast-Adresse
  stehen nicht zur freien Verfügung; für `/29` bleiben nur 6 Adressen, von denen
  bereits Gateway und erste DHCP-Adresse belegt sein können.
- **Überschneidung durch Rundungsfehler.** Die Blockgröße muss immer eine
  Zweierpotenz sein. Eine „runde“ Größe wie 60 Adressen existiert nicht – es
  gibt nur 64 (also `/26`).
- **Subnetzgrenzen nicht beachten.** Ein Routerinterface mit `192.168.10.1/26`
  und ein Host mit `192.168.10.65/27` liegen in verschiedenen Subnetzen, auch
  wenn die Adresse optisch „benachbart“ wirkt. Der Host benötigt ein Gateway.
- **Maske verwechseln.** CIDR-Präfix und Maske sind äquivalent
  (`/26` = `255.255.255.192`), werden aber leicht vertauscht. Eine falsche Maske
  führt dazu, dass der Host die Netzgrenzen falsch berechnet und die
  Kommunikation über das angebliche Subnetz hinaus fehlschlägt.
- **Planung ohne Wachstum.** Die verbleibenden 136 Adressen im Beispiel sind
  Reserve für Erweiterungen. Netze ohne Reserve müssen später umnummeriert
  werden – das betrifft alle DNS-Einträge und alle IP-basierten
  Konfigurationen.

---

## Weiterführende Quellen

- RFC 1519 – A CIDR-based Address Assignment and Route Aggregation Strategy
  <https://www.rfc-editor.org/rfc/rfc1519>
- RFC 4632 – CIDR: Classless Inter-domain Routing
  <https://www.rfc-editor.org/rfc/rfc4632>
- RFC 3021 – Using 31-Bit Prefixes on IPv4 Point-to-Point Links
  <https://www.rfc-editor.org/rfc/rfc3021>
- RFC 1918 – Address Allocation for Private Internets
  <https://www.rfc-editor.org/rfc/rfc1918>
- `ipcalc(1)` – <https://man7.org/linux/man-pages/man1/ipcalc.1.html>
- Private IPv4-Adressbereiche (in diesem Repository)
  [docs/networking/private-ipv4.md](private-ipv4.md)

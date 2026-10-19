# IT-Dokumentation (Test-Repository)

> **Hinweis:** Dieses Repository dient ausschließlich **Test- und Demonstrationszwecken**.
> Alle Inhalte sind fiktive Beispieldokumentationen für eine spätere interne
> IT-Wissensdatenbank.

## Zweck

Dieses Repository bündelt interne IT-Dokumentationen zu den Themen

- Netzwerktechnik
- Betriebssysteme
- Dienste
- Git-Workflows

Es dient als Muster dafür, wie eine interne Wissensdatenbank aufgebaut sein kann:
kurz, praxisnah, einheitlich strukturiert und mit nachvollziehbarer Historie.

## Struktur

```text
it-dokumentation-test/
├── README.md                 # dieses Dokument
├── CONTRIBUTING.md           # Regeln für Beiträge
├── .gitignore
└── docs/
    ├── README.md             # Index aller Dokumentationen
    ├── networking/           # Netzwerktechnik
    ├── systems/              # Betriebssysteme
    ├── services/             # Dienste
    └── git/                  # Versionsverwaltung
```

## Dokumentationen

Der vollständige Index находится in [`docs/README.md`](docs/README.md).

| Bereich | Dokumentation | Status |
| --- | --- | --- |
| Netzwerktechnik | [Private IPv4-Adressbereiche](docs/networking/private-ipv4.md) | verfügbar |
| Netzwerktechnik | [VLSM – Variable Length Subnet Masking](docs/networking/vlsm.md) | verfügbar |
| Betriebssysteme | Windows Server | in Arbeit |
| Betriebssysteme | Debian Linux | in Arbeit |
| Dienste | FTP, FTPS und SFTP | in Arbeit |
| Git/GitHub | Merge-Konflikte | in Arbeit |

## Arbeitsweise

Alle fachlichen Änderungen laufen über Feature-Branches und Pull Requests.
Die Regeln dazu stehen in [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Sicherheitshinweis

Dieses Repository enthält ausschließlich fiktive Beispieldaten.
Zugangsdaten, Tokens, Schlüssel oder reale Infrastrukturinformationen dürfen
**nicht** eingecheckt werden.

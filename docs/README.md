# Dokumentationsindex

Übersicht aller Dokumentationen in diesem Repository. Alle Themen sind Muster
für eine spätere interne IT-Wissensdatenbank und ausschließlich zu Testzwecken
entstanden.

## Struktur

```text
docs/
├── README.md   # dieses Dokument
├── networking/ # Netzwerktechnik
├── systems/    # Betriebssysteme
├── services/   # Dienste
└── git/        # Versionsverwaltung und GitHub
```

## Netzwerktechnik

| Dokumentation | Inhalt |
| --- | --- |
| [Private IPv4-Adressbereiche](networking/private-ipv4.md) | RFC-1918-Bereiche `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`; Netz- und Broadcast-Adressen; CIDR-Notation; NAT und die Abgrenzung zu öffentlichen Adressen. |
| [VLSM – Variable Length Subnet Masking](networking/vlsm.md) | Subnetzplanung mit unterschiedlich großen Netzen; vollständig durchgerechnetes Beispiel `192.168.10.0/24` mit 50/25/10/5 Hosts. |

## Betriebssysteme

| Dokumentation | Inhalt |
| --- | --- |
| [Windows Server – Administration](systems/windows-server.md) | Installation, Netzwerkanbindung, DNS und DHCP, Active Directory als eine von mehreren Rollen, Benutzer und Gruppen, Updates, Firewall, Ereignisanzeige, Backup. |
| [Debian Linux – Administration](systems/debian.md) | Installation, APT-Paketverwaltung, Benutzer und Dateirechte, Netzwerkkonfiguration, systemd, `journalctl`, Firewall und Absicherung, Backup mit `rsync`. |

## Dienste

| Dokumentation | Inhalt |
| --- | --- |
| [FTP, FTPS und SFTP](services/ftp.md) | Funktionsweise von FTP, Steuer- und Datenverbindung, aktiver und passiver Modus, Firewall-Anforderungen, Abgrenzung von FTP, FTPS und SFTP sowie sichere Alternativen. |

## Git / GitHub

| Dokumentation | Inhalt |
| --- | --- |
| [Merge-Konflikte](git/merge-conflicts.md) | Ursachen von Konflikten bei `merge`, `rebase`, `cherry-pick` und `am`; Zustand im Arbeitsbaum; Konfliktmarkierungen; sicherer Acht-Schritte-Ablauf; `git merge --abort` und `git rebase --abort`; Strategien zur Konfliktvermeidung. |

## Konventionen

Jede Dokumentation folgt möglichst dieser Gliederung:

1. Zweck und Geltungsbereich
2. Grundlagen
3. Vorgehen und Beispiele
4. Wichtige Konfigurationen und Befehle
5. Sicherheitshinweise und typische Fehler
6. Weiterführende Quellen

Die Regeln dazu stehen im [CONTRIBUTING.md](../CONTRIBUTING.md).

> **Hinweis zu Konfliktmarkierungen:** Die Datei
> [Merge-Konflikte](git/merge-conflicts.md) enthält bewusst ein erklärendes
> Beispiel mit den Markierungen `<<<<<<<`, `=======` und `>>>>>>>`. Diese
> Zeichen sind dort kein Fehler, sondern Beispielinhalt.

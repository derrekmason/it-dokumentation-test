# Beiträge zu diesem Repository

## Ziel

Dieses Repository ist eine Muster-Wissensdatenbank. Beiträge sollen **kurz,
praxisnah, korrekt und einheitlich** sein – keine vollständigen Handbücher.

## Dokumentationsformat

Jede Dokumentation folgt möglichst derselben Gliederung:

1. **Zweck und Geltungsbereich** – was wird behandelt, was nicht?
2. **Grundlagen** – Fachbegriffe und Hintergrund, getrennt von konkreten Beispielen.
3. **Vorgehen und Beispiele** – konkrete, nachvollziehbare Schritte.
4. **Wichtige Konfigurationen und Befehle** – jeweils mit Zweck und Kontext erklärt.
5. **Sicherheitshinweise und typische Fehler**
6. **Weiterführende Quellen** – möglichst offizielle bzw. primäre Quellen
   (z. B. RFCs, Herstellerdokumentation, Projekt-Dokumentation).

## Formatierungsregeln

- Eine Sprache: **Deutsch**.
- Markdown verwenden, Überschriftenhierarchie einhalten (`#` bis `###`).
- Befehle in Code-Blöcken oder als Inline-Code schreiben, damit sie kopierbar bleiben.
- Allgemeine Erläuterungen nicht mit konkreten Beispielen vermischen.
- Keine erfundenen Optionen oder technischen Eigenschaften. Wenn etwas nicht
  sicher belegbar ist, wird es nicht behauptet.
- Ausschließlich **fiktive** IP-Adressen, Hostnamen, Benutzernamen und Beispieldaten.

## Branch-Namen

```
docs/kurzer-themenbereich
```

Beispiele:

- `docs/networking-ipv4-vlsm`
- `docs/windows-server-debian`
- `docs/ftp-merge-conflicts`

Keine direkten fachlichen Commits auf `main`.

## Pull Requests

1. Feature-Branch vom aktuellen `main` ableiten.
2. Änderungen committen mit aussagekräftiger Nachricht
   (z. B. `docs: VLSM-Rechenbeispiel ergänzen`).
3. Vor dem Push prüfen:
   - `git status` – sind nur die beabsichtigten Dateien geändert?
   - `git diff` – ist die Änderung plausibel?
   - `git diff --check` – sind keine unbeabsichtigten Formatierungsfehler vorhanden?
4. Branch pushen und Pull Request gegen `main` eröffnen.
5. Beschreibung enthält **Ziel**, **Änderungen** und **Prüfschritte**.
6. Reviews abwarten; keine Statusprüfungen oder Schutzmechanismen umgehen.
7. Nach dem Merge den Branch **nicht** automatisch löschen, damit der Verlauf
   nachvollziehbar bleibt.

## Qualität und Sicherheit

- Vor dem Merge prüfen, dass Verlinkungen stimmen und die Inhalte fachlich korrekt sind.
- Keine Zugangsdaten, Tokens, privaten Schlüssel oder echten Infrastrukturdaten.
- Bei Merge-Konflikten: Konflikte fachlich auflösen, nicht pauschal mit
  `ours` oder `theirs` überschreiben. Siehe
  [Merge-Konflikte](docs/git/merge-conflicts.md).

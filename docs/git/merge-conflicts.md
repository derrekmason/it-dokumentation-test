# Merge-Konflikte in Git und GitHub

## Zweck und Geltungsbereich

Diese Dokumentation beantwortet zwei Fragen, die bei der täglichen Arbeit mit
Git regelmäßig auftreten: **Wie entstehen Merge-Konflikte?** und **Was passiert,
wenn Git einen Merge-Konflikt meldet?** Anschließend wird ein sicherer Ablauf
zur Auflösung beschrieben.

**Behandelt wird:**

- Ursachen von Konflikten bei `git merge`, `git rebase` und `git am`
- Zustand und Markierungen eines Konflikts
- sicherer Ablauf zur Auflösung in acht Schritten
- Abbruch eines laufenden Merge- oder Rebase-Vorgangs
- Gegenmaßnahmen zur Vermeidung von Konflikten

**Nicht behandelt wird:** die Grundlagen von Branches, Commits oder Pull
Requests selbst. Quellen: die Git-Dokumentation und die GitHub-Dokumentation
(siehe [Quellen](#weiterführende-quellen)).

---

## A. Wie können Merge-Konflikte entstehen?

### Grundprinzip

Git betrachtet Commits nicht als Dateien, sondern als **Veränderungen gegenüber
dem gemeinsamen Vorfahren** (dem sogenannten Merge-Base). Bei der
Zusammenführung versucht Git, die Änderungen beider Seiten anzuwenden.

Kann Git nicht eindeutig entscheiden, **was an einer Stelle im Ergebnis stehen
soll**, entsteht ein Konflikt. Git stoppt die Zusammenführung dann bewusst
und übergibt die Entscheidung an den Entwickler.

> **Wichtig:** Nicht jede parallele Bearbeitung erzeugt einen Konflikt.
> Änderungen an **verschiedenen, unabhängigen Stellen** – auch in derselben
> Datei – lässt Git in der Regel automatisch zusammenführen. Beide Seiten
> fügen Zeilen an verschiedenen Stellen ein, es entsteht kein Konflikt.

### Typische Ursachen

#### 1. Zwei Branches verändern dieselben oder überlappende Zeilen

Beispiel: `main` und `feature` ändern beide Zeile 42 derselben Datei – und
zwar unterschiedlich. Es gibt keine eindeutige Lösung.

```text
main:      Eine Ortswahl mit drei Arbeitsplätzen nutzen.
feature:   Eine Ortswahl mit drei Arbeitsplätzen nutzen. Leider nur mit zwei.
```

#### 2. Eine Änderung bearbeitet einen Abschnitt, den die andere löscht

Bearbeitet `feature` einen Block, den `main` vollständig entfernt hat, kann Git
den Block nicht sinnvoll wiederherstellen. Das Ergebnis wäre eine
tote oder doppelte Information.

#### 3. Mehrere Entwickler arbeiten parallel an derselben Datei

Je länger ein Branch ohne Abgleich zurückliegt, desto größer wird die
Wahrscheinlichkeit, dass sich Änderungsbereiche überschneiden. Häufige Ursache
ist ein Branch, der über Tage oder Wochen ohne `merge` beziehungsweise `rebase`
auf `main` weiterentwickelt wurde.

#### 4. Konflikte beim `git merge`

`git merge` versucht, den aktuellen Branch (`HEAD`) mit einem anderen Branch zu
zusammenzuführen. Beide Seiten haben seit der Merge-Base dieselbe Stelle
verändert → Konflikt.

#### 5. Konflikte beim `git rebase`

`git rebase` spielt die Commits des aktuellen Branches auf einen neuen
Standpunkt ab (in der Regel den aktuellen `main`). Weil dabei jeder Commit
nacheinander angewendet wird, kann **jeder einzelne Commit** für sich genommen
einen Konflikt auslösen. `git rebase` ist deshalb häufiger von Konflikten
betroffen als `git merge`.

#### 6. Konflikte beim Anwenden einzelner Commits

- `git cherry-pick <commit>` und `git revert <commit>` wenden einen einzelnen
  Commit auf einen anderen Stand an – derselbe Konfliktmechanismus greift.
- `git am` (Abspielen eines Patch-Streams, z. B. aus einem Mailarchiv) kann
  ebenfalls scheitern, wenn der Kontext nicht mehr passt.
- `git apply` kann scheitern; ob ein Patch anwendbar ist, hängt vom
  umgebenden Kontext ab.

#### 7. Weitere Konfliktarten

| Art | Beschreibung |
| --- | --- |
| `modify/delete` | Eine Seite ändert eine Datei, die andere löscht sie. Git kann nicht entscheiden, ob die Änderung oder die Löschung gewollt ist. |
| `rename/rename`, `rename/modify`, `rename/delete` | Eine Datei wird umbenannt und parallel verändert oder gelöscht. Git erkennt Umbenennungen nur heuristisch. |
| Binärkonflikte | Bei Binärdateien (Bilder, Archive, Datenbanken) kann Git keinen Text-3-Wege-Merge durchführen. |
| `cherry-pick`/`revert`-Konflikte | Ein einzelner Commit passt nicht mehr auf den aktuellen Stand. |
| Wiederholte Konflikte beim Rebase | Nach dem Lösen schlägt derselbe Commit erneut fehl (bei `rebase --merge`/`--rebase-merges` oder verschachtelten Rebases). |
| `.gitattributes`-Konflikte | Bei `merge.conflictStyle` `zdiff3` zeigt Git zusätzlich die gemeinsame Ausgangsbasis. |

---

## B. Was passiert bei einem Merge-Konflikt?

### Warum Git nicht selbst entscheiden kann

Git führt eine **Drei-Wege-Zusammenführung** durch:

```text
     A (Merge-Base)
    /         \
   B           C
        ?
```

- `A`: letzte gemeinsame Version (Merge-Base)
- `B`: Ihre Änderung (aktueller Branch)
- `C`: Änderung des zu integrierenden Branches

Für jede Stelle im Ergebnis gilt:

| Situation in B und C | Ergebnis |
| --- | --- |
| nur eine Seite geändert | Änderung wird übernommen |
| beide Seiten gleich geändert | kein Konflikt – identisch |
| beide Seiten unterschiedlich geändert | **Konflikt** – Git stoppt |

Im letzten Fall gibt es keine technisch eindeutige Antwort. Git kann die
beiden Varianten nicht in eine „falsche“ und eine „richtige“ unterscheiden –
diese fachliche Bewertung muss der Entwickler vornehmen.

### Zustand im Arbeitsbaum

Git markiert die betroffenen Dateien und schreibt die Konfliktmarkierungen
**direkt in die Datei**. Der Merge ist damit **noch nicht abgeschlossen**:

- `MERGE_HEAD` im Verzeichnis `.git` zeigt, dass ein Merge läuft.
- Bei einem Rebase liegen `.git/rebase-merge` bzw. `.git/rebase-apply` vor.
- Ein `git commit` schließt den Merge ab, ein weiteres `git merge` würde ihn
  erneut versuchen.

### Konflikte erkennen: `git status`

```bash
git status
```

Beispielausgabe:

```text
On branch feature/konfig
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
        both modified:   docs/services/ftp.md
```

Kurzformen:

```bash
git status --short
# UU docs/services/ftp.md

git diff --name-only --diff-filter=U
# listet alle noch unaufgelösten Dateien auf

git ls-files -u
# listet die Konfliktstufen (Stufe 1/2/3) je Datei und Objekt
```

### Die Konfliktmarkierungen

Beispiel (vereinfacht, zeigt den Fall aus Abschnitt A.1):

```text
<<<<<<< HEAD:feature/konfig
Eine Ortswahl mit drei Arbeitsplätzen nutzen. Leider nur mit zwei.
=======
Eine Ortswahl mit drei Arbeitsplätzen nutzen.
>>>>>>> origin/main
```

| Markierung | Bedeutung |
| --- | --- |
| `<<<<<<<` | Beginn der Konfliktstelle, gefolgt vom Namen der **eigenen** Seite (`HEAD`, das ist der aktuell ausgecheckte Branch) |
| `=======` | Trennzeile zwischen den beiden konkurrierenden Varianten |
| `>>>>>>>` | Ende der Konfliktstelle, gefolgt vom Namen der **anderen** Seite (z. B. des zu integrierenden Branches) |

Innerhalb von `HEAD` steht die Variante des aktuellen Branches, danach die des
incoming Branches. Bei einem Rebase ist die Zuordnung identisch: `HEAD` ist
der Branch, auf den rebasiert wird.

Konflikttypen, die nicht über Textmarkierungen dargestellt werden (zum Beispiel
`rename/rename`), sind im Abschnitt
[Weitere Konfliktarten](#7-weitere-konfliktarten) beschrieben.

Mit dem Wert `zdiff3` in `merge.conflictStyle` ergänzt Git die Anzeige um die
gemeinsame Ausgangsbasis, was das Nachvollziehen erleichtert:

```bash
git config merge.conflictStyle zdiff3
```

### Warum der Entwickler entscheiden muss

Nur der Entwickler kennt die fachliche Absicht:

- Muss die Formulierung aus `main` erhalten bleiben, weil sie inzwischen
  fachlich korrigiert wurde?
- Ist die Änderung auf dem Feature-Branch veraltet und überflüssig?
- Gehören **beide** Änderungen in das Ergebnis, in welcher Form?

Eine automatisierte „Gewinnerwahl“ würde diese Information stillschweigend
verwerfen. Deshalb verlangt Git hier eine ausdrückliche Entscheidung.

> **Warnung:** Konflikte pauschal mit `--ours` oder `--theirs` aufzulösen
> entfernt den Konflikt, ohne die Fachfrage zu beantworten. Bei einem Rebase
> oder einer Aufgabe mit Ähnlichkeit im Namen ist die Zuordnung zudem leicht
> verkehrt herum gelesen. Nutzen Sie diese Optionen nur, wenn Sie den Inhalt
> beider Seiten tatsächlich geprüft haben und den Verlust bewusst in Kauf
> nehmen.

---

## Sicherer Ablauf zur Konfliktlösung

### Schritt 1 – Status prüfen

```bash
git status
git status --short
```

Prüfen: Läuft ein Merge, ein Rebase, ein `cherry-pick` oder ein `revert`?
Welche Dateien sind betroffen?

### Schritt 2 – Betroffene Dateien identifizieren

```bash
git diff --name-only --diff-filter=U
```

Bei umfangreichen Konflikten ist die Liste je Datei abzuarbeiten, nicht alles
auf einmal.

### Schritt 3 – Konkurrierende Änderungen fachlich vergleichen

```bash
git diff                                   # Konflikte im Arbeitsbaum
git log --oneline --graph --all -20        # Verlauf und Verzweigungen
git show HEAD:docs/networking/vlsm.md      # eigene Seite
git show origin/main:docs/networking/vlsm.md # andere Seite
git merge-base HEAD origin/main            # gemeinsame Basis
git show $(git merge-base HEAD origin/main):docs/networking/vlsm.md
```

Damit lässt sich für jede Konfliktstelle beantworten: Was wollte welcher
Commit erreichen, und was ist davon heute noch gültig?

> **Vor dem Abbruch oder einer Wiederherstellung:** nicht abgeschlossene
> Arbeit vorher sichern. Unaufgelöste Konflikte liegen nur im Arbeitsbaum und
> in Index-Einträgen – sind sie nicht committet, sind sie nach einem Abbruch
> oder einer Neuaktionierung verloren.

### Schritt 4 – Gewünschten Inhalt herstellen

Die Datei so bearbeiten, dass:

- alle Konfliktmarkierungen `<<<<<<<`, `=======` und `>>>>>>>` entfernt sind,
- der fachlich gewünschte Text verbleibt (notfalls beide Aspekte kombiniert),
- die umgebende Struktur konsistent bleibt.

Hilfen: Editoren mit Konfliktunterstützung (Abschnitt „merge drivers“ in
`git help merge` bzw. die Unterstützung des verwendeten Editors nutzen), oder
bei Zeilenkonflikten `git checkout --conflict=diff3 <datei>` zur erneuten
Darstellung mit mehr Kontext.

```bash
# Datei noch einmal mit zdiff3-Darstellung erzeugen (überschreibt die Datei!)
git checkout --conflict=zdiff3 docs/services/ftp.md
```

> **Achtung:** Dieser Befehl **überschreibt die Datei im Arbeitsbaum** mit dem
> Konfliktzustand aus dem Index. Bisherige manuelle Bearbeitung geht dabei
> verloren – vorher sichern oder `git diff > /tmp/konflikt.diff` ablegen.

### Schritt 5 – Ergebnis kontrollieren

```bash
# Restliche Markierungen im Arbeitsbaum aufspüren
git grep -n -e '^<<<<<<<' -e '^=======$' -e '^>>>>>>>' -- .

# Noch nicht aufgelöste Pfade (muss leer sein)
git diff --name-only --diff-filter=U
```

Der erste Befehl findet Markierungen **auch in Dateien, die Git nicht als
konfliktbehaftet führt** – das ist eine sinnvolle Zusatzkontrolle vor dem
Commit.

### Schritt 6 – Gelöste Dateien markieren

```bash
git add docs/networking/vlsm.md
```

`git add` bedeutet hier: „Diese Datei ist jetzt vollständig aufgelöst und soll
in den Commit aufgenommen werden.“ Erst danach gilt sie als aufgelöst.

Optional vorab:

```bash
git diff docs/networking/vlsm.md      # eigene Version gegen Index ansehen
```

### Schritt 7 – Vorgang abschließen

Je nach begonnenem Vorgang:

| Vorgang | Abschluss |
| --- | --- |
| `git merge` | `git commit` (öffnet den Editor mit der Merge-Nachricht) oder `git commit --no-edit` |
| `git rebase` | `git rebase --continue` (mit `GIT_EDITOR=true`, wenn kein Editor nötig ist) |
| `git cherry-pick` | `git cherry-pick --continue` |
| `git revert` | `git revert --continue` |

Bei einem Rebase folgt nach dem Abschluss dieser einen Commits in der Regel
der nächste Commit. Weitere Konflikte können auftreten – Schritt 1 beginnt
erneut.

### Schritt 8 – Verlauf und Arbeitsbaum erneut prüfen

```bash
git status
git log --oneline --graph --decorate -15
git diff --check          # keine unbeabsichtigten Formatierungsfehler
```

Zusätzlich fachlich prüfen, ob die zusammengeführte Datei noch das tut, was
sie tun soll – ein Merge, der formal sauber ist, kann inhaltlich falsch sein.

### Ablaufübersicht

```text
1. git status                                    Zustand feststellen
2. git diff --name-only --diff-filter=U          betroffene Dateien
3. HEAD / origin/main / merge-base vergleichen   fachlich verstehen
4. Konfliktdatei manuell bereinigen             Markierungen entfernen
5. git grep nach Markierungen                   Kontrolle
6. git add <datei>                              als gelöst markieren
7. git commit  |  git rebase --continue         Vorgang abschließen
8. git status | git log --graph                 erneut prüfen
```

---

## Abbruch eines laufenden Vorgangs

### `git merge --abort`

Bricht einen **laufenden Merge** ab und stellt den Zustand **vor** dem
`git merge` wieder her – inklusive aller zuvor nicht committeten Änderungen.

```bash
git merge --abort
```

### `git rebase --abort`

Bricht einen **laufenden Rebase** ab und setzt den Branch auf den Ausgangspunkt
zurück, von dem der Rebase gestartet wurde.

```bash
git rebase --abort
```

### Unterschiede und Hinweise

| Frage | `git merge --abort` | `git rebase --abort` |
| --- | --- | --- |
| Beendet | Merge | Rebase |
| Ergebnis | Zustand wie **vor** `git merge` | Branch auf Ausgangsstand **vor** dem Rebase |
| Wann sinnvoll | Ergebnis entspricht nicht den Erwartungen | Rebase sollte neu gestartet werden (z. B. nach `git pull`-Problemen) |
| Abbruch nach Abschluss | wirkungslos, der Merge ist bereits committet | wirkungslos, der Rebase ist abgeschlossen |

> **Wichtige Einschränkung:** Beide Befehle setzen den Arbeitsbaum auf einen
> früheren Zustand zurück. **Alle seit dem Merge- bzw. Rebase-Beginn
> vorgenommenen Änderungen im Arbeitsbaum gehen dabei verloren.** Deshalb:
>
> - Unaufgelöste oder zwischenzeitlich weiterbearbeitete Dateien vorher sichern
>   (`git diff > meine-aufloesung.diff` oder `git stash push -m "Konfliktstand"`),
> - Alternativ den Zustand mit `git commit` (bei einem Merge) abschließen, statt
>   ihn zu verwerfen, wenn die Auflösung fachlich bereits richtig ist.
>
> Destruktive Befehle (`git reset --hard`, `git clean -fd`) sollten nicht
> unüberlegt als Ersatz eingesetzt werden: Sie überschreiben Änderungen ohne
> Vorwarnung und lassen sich nicht rückgängig machen. Auch
> `git rebase --skip` oder `git rebase --quit` entfernen Commits aus der
> Kette – sie sollten nur mit klarer Absicht verwendet werden.

### Keine Gewalt, sondern Diagnose

```bash
git reflog                 # vorherige HEAD-Zustände
git log --oneline -5       # Commit vor dem laufenden Vorgang
git show HEAD              # Inhalt des letzten Commits prüfen
```

Der `reflog` zeigt auch nach einem Abbruch, welcher Zustand vorher aktiv war –
und ob der Vorgang schon committet war.

---

## Konflikte reduzieren

| Maßnahme | Wirkung |
| --- | --- |
| **Kleine, zusammenhängende Änderungen** machen | Weniger Zeilenbereiche pro Commit, weniger Überschneidungen. Ein Commit sollte *eine* sinnvolle Änderung enthalten, nicht fünf Tage Arbeit. |
| **Häufig synchronisieren** | Regelmäßig `origin/main` in den Branch holen (`git fetch origin` + `git rebase origin/main`), statt wochenlang auf einen Merge zu warten. Der Konflikt entsteht dann klein statt groß. |
| **Klare Zuständigkeiten** | Wer welche Datei oder welches Dokument bearbeitet, sollte bekannt sein. Vor größeren Änderungen kurz absprechen, welche Dateien betroffen sind. |
| **Früh und oft committen** | Viele kleine Commits ergeben feinere Berührungspunkte als ein einzelner RiesencCommit. |
| **Ordentliche Formatierung** | Eine einheitliche Formatierung erzeugt weniger „weißraumbedingte“ Überschneidungen. Formatter-Antworten im selben Pull Request erledigen. |
| **Sorgfältige Reviews** | Was früh im Review gesehen wird, wird vor dem Merge korrigiert – und nicht im Konflikt. |
| **Keine unnötigen Merge-Kommits** | `git rebase` auf dem eigenen Feature-Branch hält die Historie linear und reduziert Konflikte auf dem aktuellen Branch. |
| **Konflikte fachlich auflösen** | Pauschales `ours`/`theirs` erzeugt zwar grüne Merges, aber stillschweigend verlorene Änderungen. |
| **Große Textdateien aufteilen** | Lange Konfigurations- oder Dokumentationsdateien lassen sich schwerer parallel bearbeiten als mehrere kleine. |
| **`rerere` aktivieren** (optional) | Git merkt sich bereits gelöste Konflikte und schlägt sie beim nächsten Mal automatisch vor: `git config rerere.enabled true`. Die vorgeschlagenen Lösungen bleiben sichtbar und werden nicht ungeprüft übernommen. |

---

## Weiterführende Quellen

### Git-Dokumentation

- git-merge: <https://git-scm.com/docs/git-merge>
- git-rebase: <https://git-scm.com/docs/git-rebase>
- git-checkout (Optionen `--ours`, `--theirs`, `--conflict=`):
  <https://git-scm.com/docs/git-checkout>
- git-status: <https://git-scm.com/docs/git-status>
- git-add: <https://git-scm.com/docs/git-add>
- git-commit: <https://git-scm.com/docs/git-commit>
- git-diff: <https://git-scm.com/docs/git-diff>
- git-config (u. a. `merge.conflictStyle`, `rerere.enabled`):
  <https://git-scm.com/docs/git-config>
- gitattributes (Merge-Treiber, `merge=`):
  <https://git-scm.com/docs/gitattributes>

### GitHub-Dokumentation

- About pull requests:
  <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests>
- Addressing merge conflicts on GitHub:
  <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts>
- About protected branches:
  <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches>
- Using issues: <https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues>

### Weitere Verweise in diesem Repository

- [CONTRIBUTING.md – Branch-Namen, Pull Requests und Konflikt-Hinweise](../../CONTRIBUTING.md)
- [Dokumentationsindex](../README.md)

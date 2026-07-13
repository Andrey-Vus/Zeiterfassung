# Design: "M"-Button für Mittagspause im Tagesprotokoll

Datum: 2026-07-13
Betroffene Datei: `index.html` (Komponente `TasksTab`)

## Ziel

Im Tagesprotokoll (Tab "Tagesaufgaben") soll ein Button "M" eine Mittagspause
(`name: "m"`, `start: "12:00"`, `end: "12:30"`) einfügen. Das Protokoll muss dabei
stets chronologisch sortiert bleiben (`task[i].start < task[i+1].start`), und
zeitliche Überlappungen zwischen aufeinanderfolgenden Aufgaben sollen visuell
(rosa) markiert werden.

## 1. Button-Platzierung

Im Card-Header von "Protokoll" (aktuell Zeile ~714-717 in `index.html`, neben dem
"Delete Day"-Papierkorb-Button) wird ein weiterer Button mit Label "M" ergänzt.
Tooltip: "Mittagspause 12:00–12:30 einfügen".

## 2. Sortier-Invariante bei jedem Insert

Neue Hilfsfunktion `sortByStart(list)`:

```js
const sortByStart = (list) => [...list].sort((a, b) => a.start.localeCompare(b.start));
```

Da `start` im Format `HH:MM` (zero-padded) vorliegt, ist ein lexikographischer
String-Vergleich äquivalent zum chronologischen Vergleich.

- `addTask` (bestehender "Start"-Button): hängt das neue Element wie bisher an,
  ruft danach `sortByStart` auf das Ergebnis-Array auf, bevor `setTasks` /
  `triggerAutosave` aufgerufen wird. Die bestehende Logik ("letzte offene
  Aufgabe automatisch schließen, wenn neue Aufgabe gestartet wird") bleibt
  unverändert, da sie vor dem Sortieren ausgeführt wird und im
  Normalfall (Echtzeit-Erfassung) ohnehin die letzte chronologische Position
  betrifft.
- `addLunch` (neuer "M"-Button, siehe Abschnitt 3): fügt Element(e) ein und
  sortiert danach ebenfalls mit `sortByStart`.

Bearbeiten (Editieren) bestehender Start-/Endzeiten über die Inline-Inputs löst
**kein** erneutes Sortieren aus — nur echte Insert-Operationen (Start-Button,
M-Button) sortieren neu. Das entspricht der Anforderung "bei beliebigem Insert
die Zeile sortieren".

## 3. `addLunch`-Funktion (Auto-Split bei laufender Aufgabe)

```js
const addLunch = () => {
    const LUNCH_START = '12:00', LUNCH_END = '12:30';
    let updated = sortByStart(tasks);

    const openIdx = updated.findIndex(t => !t.end && t.start < LUNCH_START);
    if (openIdx !== -1) {
        const openTask = updated[openIdx];
        const closedTask = {
            ...openTask,
            end: LUNCH_START,
            durationHours: calcDuration(openTask.start, LUNCH_START)
        };
        const continuation = {
            id: Date.now().toString() + '-cont',
            name: openTask.name,
            start: LUNCH_END,
            end: '',
            durationHours: 0
        };
        updated[openIdx] = closedTask;
        updated.splice(openIdx + 1, 0, continuation);
    }

    const lunchItem = {
        id: Date.now().toString() + '-lunch',
        name: 'm',
        start: LUNCH_START,
        end: LUNCH_END,
        durationHours: calcDuration(LUNCH_START, LUNCH_END)
    };
    updated.push(lunchItem);
    updated = sortByStart(updated);

    setTasks(updated);
    triggerAutosave(updated);
};
```

Verhalten:

- **Es gibt eine offene Aufgabe, die vor 12:00 begonnen hat** (kein `end`-Wert,
  `start < "12:00"`): Diese Aufgabe wird bei "12:00" beendet (Dauer neu
  berechnet), und eine Fortsetzungsaufgabe mit demselben Namen wird ab "12:30"
  angelegt (offen, ohne Ende) — die Aufgabe wird also um die Mittagspause herum
  gesplittet.
- **Keine passende offene Aufgabe gefunden** (z.B. weil der letzte Eintrag
  bereits ein `end` hat, oder weil er erst nach 12:00 beginnt): Die
  Mittagspause wird einfach an der chronologisch passenden Stelle eingefügt,
  ohne Split. Eventuelle Überlappungen mit bestehenden (bereits
  abgeschlossenen) Aufgaben werden über das Konflikt-Highlighting (Abschnitt 4)
  sichtbar gemacht und müssen manuell korrigiert werden — kein automatischer
  Split für bereits geschlossene Aufgaben.
- IDs werden mit festen Suffixen (`-lunch`, `-cont`) statt zweier `Date.now()`-
  Aufrufe gebildet, um Kollisionen bei gleichzeitiger Erzeugung im selben
  Millisekunden-Tick zu vermeiden.

## 4. Konflikt-Highlighting (rosa)

Neuer `useMemo`, der für die aktuell (im State) vorliegende `tasks`-Reihenfolge
ein Array von Flags berechnet:

```js
const overlapFlags = useMemo(() => {
    return tasks.map((t, i) => {
        const next = tasks[i + 1];
        return !!(next && t.end && next.start && t.end > next.start);
    });
}, [tasks]);
```

Beim Rendern der Protokoll-Zeilen (aktuell Zeile ~720-729):

- Ist `overlapFlags[i]` gesetzt, wird das Ende-Input der Zeile `i` **und** das
  Start-Input der Zeile `i + 1` mit rosa Hintergrund dargestellt (z.B. inline
  `style={{ backgroundColor: '#ffd6e7' }}` zusätzlich zu den bestehenden
  Bootstrap-Klassen).
- Diese Berechnung ist rein anzeigend (kein Einfluss auf gespeicherte Daten)
  und aktualisiert sich automatisch bei jeder Änderung an `tasks`, unabhängig
  vom Sortieren-bei-Insert-Mechanismus aus Abschnitt 2.

## Out of Scope

- Kein automatisches Re-Sortieren bei Inline-Edits bestehender Zeiten.
- Kein automatischer Split für bereits abgeschlossene (nicht offene) Aufgaben,
  die zufällig mit der Mittagspause überlappen — nur visuelles Highlighting.
- Keine Schutzmaßnahme gegen mehrfaches Klicken von "M" am selben Tag
  (erzeugt mehrere Pausen-Einträge, falls der Nutzer das tut — bewusst nicht
  verhindert, da nicht angefordert).

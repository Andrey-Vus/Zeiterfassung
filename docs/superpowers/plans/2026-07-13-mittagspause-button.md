# Mittagspause-Button ("M") Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** In `TasksTab` (index.html) einen "M"-Button ergänzen, der eine Mittagspause (12:00–12:30) chronologisch korrekt ins Tagesprotokoll einfügt, dabei eine laufende offene Aufgabe automatisch splittet, das Protokoll sortiert hält und überlappende Zeiten rosa markiert.

**Architecture:** Reines Client-seitiges Feature innerhalb der bestehenden React-Komponente `TasksTab` in `index.html`. Kein Build-Tooling, kein Router, keine neuen Dateien — alle Änderungen sind lokal auf die `tasks`-State-Logik und das JSX-Rendering dieser einen Komponente begrenzt.

**Tech Stack:** React 18 (UMD, `Babel standalone` JSX im Browser), Bootstrap 5 CSS, kein Build-Schritt, kein Test-Framework.

## Global Constraints

- Datei: ausschließlich `index.html` wird verändert (einziges Source-File im Repo).
- Zeitformat überall `HH:MM` (zero-padded, 24h) — String-Vergleich ist äquivalent zu chronologischem Vergleich.
- Feste Werte: Mittagspause ist immer `start: "12:00"`, `end: "12:30"`, `name: "m"`.
- Pro Tag genau eine Mittagspause erlaubt (Button `disabled`, sobald ein Eintrag mit `name === 'm'` existiert).
- Sortiert wird nur bei echten Inserts (Start-Button, M-Button) — nicht bei Inline-Edits bestehender Zeiten.
- Konflikt-Highlighting ist rein visuell (kein Einfluss auf gespeicherte Daten) und aktualisiert sich bei jeder Änderung von `tasks`.
- **Keine automatisierte Test-Suite vorhanden** (statische Single-File React/Babel-App ohne Build-Tooling, keine package.json). Verifikation der reinen Logik (Sortierung, Split, Overlap-Berechnung) erfolgt über kleine Node-Snippets (`node -e "..."`), die dieselbe Logik isoliert ausführen. Die UI/Wiring-Verifikation (Button-Klick, Disabled-State, rosa Hervorhebung im echten Browser) erfordert eine aktive Upstash-Redis-Verbindung des Nutzers und wird daher als manuelle Checkliste am Ende jedes Tasks beschrieben, nicht durch den Agenten automatisiert.
- Spec: `docs/superpowers/specs/2026-07-13-mittagspause-button-design.md`

---

### Task 1: Sortier-Invariante — `sortByStart` Helper + Nutzung in `addTask`

**Files:**
- Modify: `index.html:620-637` (Funktion `addTask` in `TasksTab`)

**Interfaces:**
- Produces: `sortByStart(list)` — reine Funktion, Signatur `(Array<{start: string, ...}>) => Array<{...}>`, gibt eine **neue**, nach `start` aufsteigend sortierte Kopie zurück. Wird in Task 2 von `addLunch` wiederverwendet.

- [ ] **Step 1: Node-Snippet schreiben, das die aktuelle (noch nicht sortierende) `addTask`-Logik nachbildet, um den Ist-Zustand zu belegen**

Lege eine temporäre Datei `scratch_test.js` im Scratchpad an (nicht Teil des Repos) mit:

```js
// scratch_test.js
const sortByStart = (list) => [...list].sort((a, b) => a.start.localeCompare(b.start));

// Nachbildung des addTask-Kerns nach der Änderung:
function addTaskSorted(tasks, newTaskName, newTaskStart) {
    let updatedTasks = [...tasks];
    if (updatedTasks.length > 0) {
        const last = updatedTasks[updatedTasks.length - 1];
        if (!last.end) {
            updatedTasks[updatedTasks.length - 1] = { ...last, end: newTaskStart };
        }
    }
    updatedTasks.push({ id: 'new', name: newTaskName, start: newTaskStart, end: "", durationHours: 0 });
    return sortByStart(updatedTasks);
}

// Test 1: Einfügen mit früherer Startzeit als vorhandene Einträge landet vorne
const t1 = addTaskSorted(
    [{ id: 'a', name: 'A', start: '09:00', end: '10:00', durationHours: 1 }],
    'Vor A', '08:00'
);
console.assert(t1[0].name === 'Vor A' && t1[1].name === 'A', 'Test 1 FAILED', t1);

// Test 2: Normales Anhängen am Ende bleibt sortiert und schliesst die letzte offene Aufgabe
const t2 = addTaskSorted(
    [{ id: 'a', name: 'A', start: '09:00', end: '', durationHours: 0 }],
    'B', '10:00'
);
console.assert(t2[0].end === '10:00' && t2[1].name === 'B', 'Test 2 FAILED', t2);

console.log('Alle Tests durchgelaufen (siehe oben auf FAILED prüfen).');
```

Run: `node scratch_test.js`
Erwartete Ausgabe: keine `FAILED`-Zeilen, letzte Zeile `Alle Tests durchgelaufen...`

- [ ] **Step 2: `sortByStart` in `index.html` ergänzen und in `addTask` verwenden**

In `index.html`, direkt vor der bestehenden `addTask`-Definition (aktuell Zeile 620), folgende Zeile einfügen:

```js
const sortByStart = (list) => [...list].sort((a, b) => a.start.localeCompare(b.start));

const addTask = () => {
    if (!newTaskName || !newTaskStart) return;
    let updatedTasks = [...tasks];
    if (updatedTasks.length > 0) {
        const last = updatedTasks[updatedTasks.length - 1];
        if (!last.end) {
            const newLast = { ...last, end: newTaskStart };
            newLast.durationHours = calcDuration(newLast.start, newLast.end);
            updatedTasks[updatedTasks.length - 1] = newLast;
        }
    }
    const newItem = { id: Date.now().toString(), name: newTaskName, start: newTaskStart, end: "", durationHours: 0 };
    updatedTasks.push(newItem);
    updatedTasks = sortByStart(updatedTasks);
    setTasks(updatedTasks);
    triggerAutosave(updatedTasks);
    setNewTaskName("");
    setNewTaskStart("");
};
```

(Einzige Änderung gegenüber dem Original: die neue `sortByStart`-Konstante davor, und die Zeile `updatedTasks = sortByStart(updatedTasks);` nach `updatedTasks.push(newItem);`.)

- [ ] **Step 3: Manuelle Verifikation im Browser (erfordert bestehende Upstash-Verbindung des Nutzers)**

Checkliste für den Nutzer (nicht durch den Agenten automatisierbar, da eine echte DB-Verbindung nötig ist):
1. `index.html` im Browser öffnen, DB verbunden, Tab "Tagesaufgaben" öffnen.
2. Aufgabe "A" mit Start `09:00` anlegen, danach Aufgabe "B" mit Start `10:00` anlegen — Reihenfolge A, B erwartet, A.end wurde automatisch auf `10:00` gesetzt.
3. Aufgabe "C" mit Start `08:00` anlegen (bewusst vor A) — Erwartung: Zeile "C" erscheint **oben**, vor "A".

- [ ] **Step 4: Aufräumen & Commit**

```bash
rm scratch_test.js
git add index.html
git commit -m "Insert-Sortierung für Tagesprotokoll (sortByStart)"
```

---

### Task 2: `addLunch`-Funktion + "M"-Button mit Auto-Split und Once-Per-Day-Guard

**Files:**
- Modify: `index.html` — neue Funktion `addLunch` (nach `addTask`, vor `updateTask`, aktuell ab Zeile 639)
- Modify: `index.html:713-717` — Card-Header von "Protokoll" (neuer Button neben "Delete Day")

**Interfaces:**
- Consumes: `sortByStart(list)` aus Task 1, `calcDuration(s, e)` (bereits vorhanden in `TasksTab`, Signatur `(string, string) => number`), `tasks` State, `setTasks`, `triggerAutosave` (alle bereits vorhanden in `TasksTab`).
- Produces: `addLunch()` — parameterlose Funktion, von `onClick` des neuen "M"-Buttons aufgerufen.

- [ ] **Step 1: Node-Snippet für die Split-Logik schreiben und Fehlschlag/Erfolg vorab prüfen**

Scratchpad-Datei `scratch_lunch.js`:

```js
// scratch_lunch.js
const sortByStart = (list) => [...list].sort((a, b) => a.start.localeCompare(b.start));

function calcDuration(s, e) {
    const round15Min = (hhmm) => {
        const [h, m] = hhmm.split(':').map(Number);
        return Math.round((h * 60 + m) / 15) * 15 / 60;
    };
    if (!s || !e) return 0;
    let dur = round15Min(e) - round15Min(s);
    if (dur < 0) dur = 0;
    return parseFloat(dur.toFixed(2));
}

function addLunch(tasks) {
    if (tasks.some(t => t.name === 'm')) return tasks;

    const LUNCH_START = '12:00', LUNCH_END = '12:30';
    let updated = sortByStart(tasks);

    const openIdx = updated.findIndex(t => !t.end && t.start < LUNCH_START);
    if (openIdx !== -1) {
        const openTask = updated[openIdx];
        const closedTask = { ...openTask, end: LUNCH_START, durationHours: calcDuration(openTask.start, LUNCH_START) };
        const continuation = { id: 'cont', name: openTask.name, start: LUNCH_END, end: '', durationHours: 0 };
        updated[openIdx] = closedTask;
        updated.splice(openIdx + 1, 0, continuation);
    }

    updated.push({ id: 'lunch', name: 'm', start: LUNCH_START, end: LUNCH_END, durationHours: calcDuration(LUNCH_START, LUNCH_END) });
    return sortByStart(updated);
}

// Test 1: offene Aufgabe ab 09:00 wird bei 12:00 gesplittet, Fortsetzung ab 12:30
const t1 = addLunch([{ id: 'a', name: 'Projekt X', start: '09:00', end: '', durationHours: 0 }]);
console.assert(t1.length === 3, 'Test1 Länge FAILED', t1);
console.assert(t1[0].name === 'Projekt X' && t1[0].end === '12:00', 'Test1 Split-Ende FAILED', t1);
console.assert(t1[1].name === 'm' && t1[1].start === '12:00' && t1[1].end === '12:30', 'Test1 Lunch FAILED', t1);
console.assert(t1[2].name === 'Projekt X' && t1[2].start === '12:30' && t1[2].end === '', 'Test1 Fortsetzung FAILED', t1);

// Test 2: keine offene Aufgabe -> einfaches Einfügen ohne Split
const t2 = addLunch([{ id: 'a', name: 'A', start: '08:00', end: '09:00', durationHours: 1 }]);
console.assert(t2.length === 2 && t2[1].name === 'm', 'Test2 FAILED', t2);

// Test 3: bereits vorhandene Pause -> keine zweite wird hinzugefügt
const t3 = addLunch([{ id: 'a', name: 'm', start: '12:00', end: '12:30', durationHours: 0.5 }]);
console.assert(t3.length === 1, 'Test3 Guard FAILED', t3);

console.log('Alle Lunch-Tests durchgelaufen (auf FAILED prüfen).');
```

Run: `node scratch_lunch.js`
Erwartete Ausgabe: keine `FAILED`-Zeilen.

- [ ] **Step 2: `addLunch` in `index.html` ergänzen**

Nach der `addTask`-Funktion (Ende von Task 1, vor `const updateTask = ...`) einfügen:

```js
const addLunch = () => {
    if (tasks.some(t => t.name === 'm')) return;

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

- [ ] **Step 3: "M"-Button im Protokoll-Header ergänzen**

Aktueller Code (`index.html:713-717`):

```jsx
                                <div className="card-header bg-white d-flex justify-content-between align-items-center">
                                    <span className="fw-bold">Protokoll</span>
                                    <button onClick={deleteDay} className="btn btn-sm btn-outline-danger" title="Delete Day"><IconTrash /></button>
                                </div>
```

Ersetzen durch:

```jsx
                                <div className="card-header bg-white d-flex justify-content-between align-items-center">
                                    <span className="fw-bold">Protokoll</span>
                                    <div className="d-flex gap-2">
                                        <button
                                            onClick={addLunch}
                                            disabled={tasks.some(t => t.name === 'm')}
                                            className="btn btn-sm btn-outline-secondary fw-bold"
                                            title={tasks.some(t => t.name === 'm') ? "Mittagspause bereits erfasst" : "Mittagspause 12:00–12:30 einfügen"}
                                        >
                                            M
                                        </button>
                                        <button onClick={deleteDay} className="btn btn-sm btn-outline-danger" title="Delete Day"><IconTrash /></button>
                                    </div>
                                </div>
```

- [ ] **Step 4: Manuelle Verifikation im Browser (erfordert bestehende Upstash-Verbindung des Nutzers)**

Checkliste für den Nutzer:
1. Tag mit einer offenen Aufgabe anlegen (z.B. "Projekt X" ab `09:00`, kein Ende).
2. Auf "M" klicken — erwartet: "Projekt X" endet jetzt um `12:00`, neue Zeile "m" `12:00–12:30`, neue Zeile "Projekt X" ab `12:30` (offen).
3. Erneut auf "M" klicken (oder prüfen) — Button muss jetzt deaktiviert sein, Tooltip "Mittagspause bereits erfasst".
4. Tag ohne offene Aufgabe testen (alle Aufgaben haben ein Ende) — "M" fügt die Pause einfach an der richtigen chronologischen Stelle ein, ohne Split.

- [ ] **Step 5: Aufräumen & Commit**

```bash
rm scratch_lunch.js
git add index.html
git commit -m "Mittagspause-Button (M) mit Auto-Split und Once-Per-Day-Guard"
```

---

### Task 3: Konflikt-Highlighting (rosa) bei überlappenden Zeiten

**Files:**
- Modify: `index.html` — neuer `useMemo` `overlapFlags` (nach dem bestehenden `pivot`-`useMemo`, aktuell Zeilen 676-682)
- Modify: `index.html:720-729` — Rendering der Protokoll-Zeilen

**Interfaces:**
- Consumes: `tasks` State.
- Produces: `overlapFlags` — Array<boolean>, gleiche Länge wie `tasks`; `overlapFlags[i] === true` bedeutet, `tasks[i].end` überlappt mit `tasks[i+1].start`.

- [ ] **Step 1: Node-Snippet für die Overlap-Berechnung**

Scratchpad-Datei `scratch_overlap.js`:

```js
// scratch_overlap.js
function computeOverlapFlags(tasks) {
    return tasks.map((t, i) => {
        const next = tasks[i + 1];
        return !!(next && t.end && next.start && t.end > next.start);
    });
}

// Test 1: Überlappung erkannt (Ende 12:15 nach Start des nächsten 12:00)
const f1 = computeOverlapFlags([
    { end: '12:15' }, { start: '12:00', end: '13:00' }
]);
console.assert(f1[0] === true && f1[1] === false, 'Test1 FAILED', f1);

// Test 2: Keine Überlappung (Ende exakt = nächster Start ist erlaubt/kein Konflikt)
const f2 = computeOverlapFlags([
    { end: '12:00' }, { start: '12:00', end: '13:00' }
]);
console.assert(f2[0] === false, 'Test2 FAILED', f2);

// Test 3: Offene Aufgabe ohne Ende erzeugt keinen Konflikt
const f3 = computeOverlapFlags([
    { end: '' }, { start: '12:00', end: '' }
]);
console.assert(f3[0] === false, 'Test3 FAILED', f3);

console.log('Alle Overlap-Tests durchgelaufen (auf FAILED prüfen).');
```

Run: `node scratch_overlap.js`
Erwartete Ausgabe: keine `FAILED`-Zeilen.

- [ ] **Step 2: `overlapFlags`-`useMemo` in `index.html` ergänzen**

Nach dem bestehenden `pivot`-`useMemo` (aktuell Zeile 676-682) einfügen:

```js
const overlapFlags = useMemo(() => {
    return tasks.map((t, i) => {
        const next = tasks[i + 1];
        return !!(next && t.end && next.start && t.end > next.start);
    });
}, [tasks]);
```

- [ ] **Step 3: Rendering der Protokoll-Zeilen anpassen**

Aktueller Code (`index.html:720-729`):

```jsx
                                        {tasks.map((t) => (
                                            <div key={t.id} className="input-group mb-2">
                                                <input className="form-control" value={t.name} onChange={e => updateTask(t.id, 'name', e.target.value)} />
                                                <input type="time" className="form-control" style={{ maxWidth: '100px' }} value={t.start} onChange={e => updateTask(t.id, 'start', e.target.value)} />
                                                <span className="input-group-text">-</span>
                                                <input type="time" className="form-control" style={{ maxWidth: '100px' }} value={t.end} onChange={e => updateTask(t.id, 'end', e.target.value)} />
                                                <span className="input-group-text fw-bold text-primary" style={{ minWidth: '70px' }}>{t.durationHours > 0 ? t.durationHours.toFixed(2) + 'h' : ''}</span>
                                                <button onClick={() => deleteTask(t.id)} className="btn btn-outline-danger"><IconTrash /></button>
                                            </div>
                                        ))}
```

Ersetzen durch:

```jsx
                                        {tasks.map((t, i) => {
                                            const endConflict = overlapFlags[i];
                                            const startConflict = i > 0 && overlapFlags[i - 1];
                                            return (
                                                <div key={t.id} className="input-group mb-2">
                                                    <input className="form-control" value={t.name} onChange={e => updateTask(t.id, 'name', e.target.value)} />
                                                    <input type="time" className="form-control" style={{ maxWidth: '100px', backgroundColor: startConflict ? '#ffd6e7' : undefined }} value={t.start} onChange={e => updateTask(t.id, 'start', e.target.value)} />
                                                    <span className="input-group-text">-</span>
                                                    <input type="time" className="form-control" style={{ maxWidth: '100px', backgroundColor: endConflict ? '#ffd6e7' : undefined }} value={t.end} onChange={e => updateTask(t.id, 'end', e.target.value)} />
                                                    <span className="input-group-text fw-bold text-primary" style={{ minWidth: '70px' }}>{t.durationHours > 0 ? t.durationHours.toFixed(2) + 'h' : ''}</span>
                                                    <button onClick={() => deleteTask(t.id)} className="btn btn-outline-danger"><IconTrash /></button>
                                                </div>
                                            );
                                        })}
```

- [ ] **Step 4: Manuelle Verifikation im Browser (erfordert bestehende Upstash-Verbindung des Nutzers)**

Checkliste für den Nutzer:
1. Zwei Aufgaben anlegen, deren Zeiten sich überlappen (z.B. "A" `09:00–10:30`, "B" `10:00–11:00`) — per Inline-Edit die Startzeit von "B" manuell auf `10:00` setzen, obwohl "A" bis `10:30` geht.
2. Erwartet: Ende-Feld von "A" und Start-Feld von "B" sind rosa hinterlegt.
3. Startzeit von "B" auf `10:30` oder später korrigieren — rosa Markierung verschwindet bei beiden Feldern.

- [ ] **Step 5: Aufräumen & Commit**

```bash
rm scratch_overlap.js
git add index.html
git commit -m "Rosa Konflikt-Highlighting bei überlappenden Protokoll-Zeiten"
```

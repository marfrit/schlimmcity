# SchlimmCity 2030 – Design v0

Stand: 2026-10-25 · Status: Entwurf, vor M0

> Sim City mit Tagebuch: eine lokale Agenten-Gesellschaft zur Simulation von
> Wohnungspolitik (Mietpreisbremse, Mietendeckel, Vergesellschaftung) auf einem
> deterministischen ökonomischen Substrat, sichtbar gemacht im isometrischen
> Retro-Stil (eigene Sprites, keine Maxis-Assets).

---

## 1. Ziele und Nicht-Ziele

**Ziele**
- Policy-Experimente mit kontrafaktischen Zeitlinien (gleicher Seed, andere Regel).
- Emergente Dysfunktionen (Schattenmarkt, Lock-in, Umwandlung, Verfall), nicht hineingeskriptet.
- Validierung gegen gut untersuchte historische Ereignisse im Blind-Replay.
- Blog-taugliche Visualisierung: Player mit Zoom, Zeitleiste, Overlays, Split-Screen.

**Nicht-Ziele (v0)**
- Keine Gesamtvolkswirtschaft. Löhne, Zinsen, Baukosten und Zuzugsdruck sind exogen.
- Keine LLM-Einwohner in v0. Haushalte entscheiden regelbasiert (Logit).
- Kein Echtzeitbetrieb. Läufe sind Batch-Jobs.

## 2. Leitprinzipien

1. **LLMs rechnen keine Ökonomie.** Preise, Mengen und Bilanz entstehen ausschließlich im Substrat.
2. **Das Substrat ist die einzige Wahrheit** für Geld, Wohnungen, Verträge und Personen.
3. **Determinismus:** Seed + Eventlog ⇒ bitgleiche Welt. Der Log *ist* die Aufzeichnung.
4. **Geschlossene Bilanz:** Faucets und Sinks für Geld, Wohnungen und Menschen werden pro Tick exakt verbucht.
5. **Grobe Grenzen:** zwischen Schichten nur Batches pro Tick, keine feinkörnigen Aufrufe.
6. **Geschlossener Aktionsraum:** Agenten wählen aus einer validierten Menge von Aktionen; das Substrat lehnt Ungültiges ab.

## 3. Architektur

```
┌──────────────────────────┐   Batch/Tick    ┌──────────────────────────┐
│ Substrat (Python/NumPy,  │ ── events ────▶ │ Bürger (Pharo/GT,        │
│ Numba), dirac            │ ◀─ decisions ── │ Green Threads)           │
│  - State (SoA)           │                 │  - Persona, Versammlungen│
│  - Märkte, Bau, Verfall  │                 │  - mneme-Destillat       │
│  - Bilanz, Beobachter    │                 │  - vLLM-Clients (Sparks) │
└───────────┬──────────────┘                 └──────────────────────────┘
            │ lädt/aktualisiert
┌───────────▼──────────────┐                 ┌──────────────────────────┐
│ Gesetze (Luau, sandboxed,│                 │ Player (Browser, PixiJS) │
│ typgeprüft)              │                 │ spielt Eventlogs ab      │
└──────────────────────────┘                 └──────────────────────────┘
```

| Schicht | Technik | Ort | Bemerkung |
|---|---|---|---|
| Substrat | Python, NumPy, Numba | dirac (CPU) | systemd-Unit, CPUs 4-7,12-15, 32 GiB, CPUWeight=50, Nice=5 |
| Gesetze | Luau via Embedding | dirac | „Lua entscheidet, NumPy rechnet“ – Regeln deklarieren, Ausführung vektorisiert |
| Bürger | Pharo / Glamorous Toolkit | tbd | dünne Schicht, Code via Iceberg/Tonel in Git, austauschbar gegen Python-asyncio |
| LLM | vLLM, Qwen3-30B-A3B FP8 | bosch + escher | zwei unabhängige Server, fester Slot (z. B. eine Nacht/Woche), DS-4.1 pausiert |
| Player | statische Webseite, PixiJS | beliebig | lädt eine Lauf-Datei, einbettbar in Blogposts |

**Betrieb:** Bei Nachtläufen `data-busy.lock` auf hertz setzen, sonst fährt data (und damit dirac) um 03:00 herunter.

## 4. Substrat

### 4.1 State (Structure of Arrays)

| Tabelle | Felder (Auswahl) |
|---|---|
| `world` | tick, zins, lohnindex, baukosten, zuzugsdruck, seed |
| `tiles` | id, x, y, bezirk, lagewert, oepnv_zone |
| `buildings` | id, tile, baujahr, einheiten, zustand [0..1], eigentuemer, typ, reg_klasse, status (ok, gerüst, leer, baugrube, abriss) |
| `units` | id, building, qm, zimmer, leer |
| `contracts` | id, unit, household, beginn, miete, formell/informell, befristet |
| `households` | id, groesse, einkommen, arbeitsort, unit, praef_urban, praef_ruhe, bindung, risiko, umzugswunsch, citizen_id (null ⇒ regelbasiert) |
| `owners` | id, typ (klein, konzern, genossenschaft, kommunal), kapital, verschuldung, strategie |
| `developers` | id, pipeline (Projekte mit Fertigstellungstick) |
| `travel` | Reisezeitmatrix tile × tile (ÖPNV, Auto) |
| `ledger` | Zu- und Abflüsse Geld, Wohnungen, Personen pro Tick und Kategorie |

Umland als Ring von Tiles mit eigener Miet- und Pendelstruktur, damit „Wegzug“ nicht gratis ist.

### 4.2 Tick-Reihenfolge (1 Tick = 1 Monat)

1. Exogene Inputs (Zins, Löhne, Zuzugsdruck, Baukosten)
2. Gesetze (Luau) aktualisieren Parameter und Constraints
3. Gebäude: Verfall, fertige Sanierungen und Neubauten, Abriss
4. Verträge: Mietanpassungen, Kündigungen, Befristungsende
5. Haushalte: Lebensereignisse (Job, Haushaltsgröße), Umzugswunsch
6. Entscheidungen der geweckten Bürger (v0: Logit; später: Bürger-Batch)
7. Märkte: Matching Mietmarkt (Suchfriktion), Kaufmarkt, informeller Markt
8. Eigentümer- und Bauträgerentscheidungen (sanieren, verkaufen, umwandeln, bauen, leer lassen)
9. Bilanz-Invarianten prüfen (Abbruch bei Verletzung)
10. Beobachter: Ground-Truth-Log, simuliertes Statistikamt, Input für den Schlimmboten

### 4.3 Märkte

- **Mietmarkt:** Such- und Matching-Modell statt Walras; Inserate sind Teil des State (für das Statistikamt sichtbar).
- **Kaufmarkt:** Umwandlung Miete → Eigentum als Eigentümeroption.
- **Neubau:** Pipeline mit Verzögerung, abhängig von Zins, Baukosten, erwarteter Miete und Regulierung.
- **Informeller Markt:** Untermiete, Abstand, Möblierungszuschlag. Volumen exakt bekannt, für das Statistikamt unsichtbar.

### 4.4 Beobachter

- **Ground Truth:** vollständiger Eventlog.
- **Statistikamt:** sieht nur Inserate, Mietspiegel-Stichprobe, Meldedaten. Die Lücke zur Ground Truth ist eine zentrale Messgröße.
- **Später:** Social-Media-Feed mit Engagement-Selektion als dritte Wahrnehmungsebene.

## 5. Gesetze (Luau)

- Jede Policy ist ein Luau-Modul mit deklarierter Schnittstelle (lesbare State-Felder, setzbare Parameter/Constraints).
- Sandbox: kein I/O, keine Schleifen über Einzelhaushalte; Ausführung wird an vektorisierte NumPy-Operationen delegiert.
- Typcheck als Gate: Ein Gesetz, das nicht typcheckt, wird nicht geladen bzw. kommt nicht zur Abstimmung.
- Hot-swap ohne Server-Neustart.
- **Später:** Agenten (z. B. Senat) entwerfen Gesetze als Luau, Versammlung stimmt ab, Schlupflöcher werden von Vermieter-Agenten gefunden statt geskriptet.

## 6. Bürger (spätere Phase)

### 6.1 Schnittstelle Substrat ↔ Bürger

- **Out:** `events[]` = {agent_id, tick, facts[]} für geweckte Agenten
- **In:** `decisions[]` = {agent_id, action, params} aus geschlossenem Aktionsraum (umziehen, bleiben, untervermieten, klagen, sanieren, verkaufen, bauen, wählen, protestieren, …)
- Format: Arrow oder JSON-Lines, ein Batch pro Tick.
- Snapshots: Image-Snapshot und Substrat-State nur tick-genau gemeinsam; alternativ Rekonstruktion aus Seed + Log.

### 6.2 Wecken

1. **Deterministisch** bei direkter Betroffenheit (Mieterhöhung, Kündigung, Jobverlust): immer.
2. **Embedding-Triage** für Indirektes (Zeitung, Gerüchte, Aufrufe): Event einmal einbetten, Kosinus gegen Profilvektoren der Agenten. Kosten skalieren mit Events, nicht Events × Agenten.

### 6.3 Kontextbudget

- Gemeinsamer Präfix (gecacht): Weltbeschreibung, Regeln, Lage der Stadt, bei Versammlungen das Transkript.
- Eigener Teil: Persona + mneme-Destillat, **≤ 500 Tokens**. mneme muss verdichten, nicht nur finden.
- Versammlungen: simultane Runden, alle n Agenten parallel, Protokollant-Agent komprimiert nach jeder Runde; danach privates vs. öffentliches Gedächtnis getrennt schreiben (Divergenz = Messsignal).

### 6.4 Messdaten Spark (2026-10-25, escher, Qwen3-30B-A3B FP8, vLLM)

| Prompt | c=1 | c=16 | c=32 | c=64 | TTFT p95 @64 |
|---|---|---|---|---|---|
| 6k shared + 300 eigen, Cache an | 42 | 188 | 272 | 371 | 5.6 s |
| 6k shared + 2000 eigen, Cache an | 38 | 125 | 142 | 144 | 33 s |
| 6k + 300, Cache aus | 34 | 64 | 53 | 36 | 164 s |

Aggregierte Output-t/s. 50-Jahre-Lauf (~18 M Output-Tokens): ~13,5 h auf einem Spark, ~7 h auf beiden.
Rohdaten: `~/claude/schlimmcity/spark-a3b-fp8-20261025.jsonl` (noether). Offener Tuning-Punkt: `--max-num-batched-tokens` (Skript `~/sc_extra.sh` auf escher).

## 7. Player

- Statische Webseite (PixiJS), lädt Seed + Eventlog bzw. vorgerechnete Tick-Snapshots.
- Isometrische 2:1-Pixelart; Sprites aus Low-Poly-Modellen: orthografisch rendern, klein auflösen, Palette ~256 Farben, Nearest-Neighbor-Upscale. Ein Mesh je Gebäudezustand.
- Semantischer Zoom: Stadt → Kiez → Gebäude → Wohnung → Bürger (Tagebuch).
- Zeitleiste zum Scrubben, Overlays (Miete, Leerstand, Fluktuation, Schwarzmarkt, Statistikamt vs. Wahrheit).
- Split-Screen zweier Zeitlinien mit gleichem Seed.
- Zeitungsspalte „Der Schlimmbote“ synchron zum Zeitstempel.
- Assets: eigene; Cytopia-Sprites nur unter CC-BY-SA (Share-Alike beachten). Logo nicht am SimCity-Schriftzug orientieren.

## 8. Meilensteine (falsifizierbar)

| ID | Kriterium | Gilt als verfehlt, wenn … |
|---|---|---|
| M-perf | 50k Haushalte × 600 Ticks auf 4 Kernen dirac in < 1 h | Laufzeit > 1 h ⇒ Rust/bevy_ecs neu bewerten |
| M0 | 50 Sim-Jahre ohne Schock stabil, Bilanz auf den Cent geschlossen | Drift, Oszillation mit wachsender Amplitude, Bilanzleck |
| M1 | Stilisierte Fakten: Zuzugsschock ⇒ Leerstand ↓, Miete ↑; Zinsschock ⇒ Neubau ↓ mit Lag | Richtung falsch oder nur bei handverlesenen Parametern |
| M2 | Blind-Replay SF 1994 (anonymisiert): Angebot regulierte Klasse ↓, Mieten unreguliert ↑ | Richtung falsch über Seed-Verteilung; Fingerprint-Test vorher nicht bestanden ⇒ ungültig |
| M3 | Mietendeckel-Muster: Inserate ↓, Lücke Statistikamt ↔ Ground Truth | kein Inseratseinbruch oder keine Lücke |
| M4 | Logit- vs. LLM-Bürger im Aggregat vergleichen | LLM ununterscheidbar von Logit (⇒ teure Deko) oder Abweichung nicht begründbar |

Treffer werden vor jedem Lauf definiert; Aussagen nur über Verteilungen vieler Seeds.

**Fingerprint-Test:** Anonymisiertes Setup einem Modell vorlegen („Welchem realen Ereignis ähnelt das?“). Erkennt es das Ereignis, wird stärker verfremdet.

## 9. Vorab deklarierte Failure Modes

1. **Wissensleck:** Modelle spielen bekannte Debatten nach ⇒ Policies und Stadt anonymisieren.
2. **Homogenität / Mode Collapse** der Agenten ⇒ Heterogenität im State, nicht nur im Persona-Text.
3. **Erzählung statt Mechanik:** Effekte existieren nur in Monologen ⇒ nur Substrat-Größen zählen.
4. **Kalibrierungsfreiheit:** Parameter vorab aus Daten fixieren; Sensitivität ±20 % je Parameter.
5. **Schweinezyklus explodiert** im Neubau.
6. **Bilanzlecks.**
7. **Embedding-Triage übersieht Betroffenheit** ⇒ direkte Betroffenheit immer deterministisch wecken.
8. **Fehlende Opportunitätskosten:** Agenten haben unbegrenzt Zeit ⇒ Zeit/Aufwand explizit modellieren.
9. **Bus-Faktor Smalltalk:** Schicht dünn halten, Schnittstelle scharf spezifizieren, Ersatz durch Python-asyncio möglich.

## 10. Prior Art

- **Cormas** (CIRAD, Pharo): agentenbasierte Sozial-Ökosystem-Modelle, Companion Modelling. Zuerst ansehen.
- **Smalltalk-80 Blue Book:** Framework für ereignisgesteuerte Simulation.
- **Kendrick** (Pharo): DSL für Epidemiologie – Vorbild für lesbare Regeln.
- **Croquet:** deterministisch replizierte Berechnung – Vorbild für Log-basierte Reproduzierbarkeit.
- **Generative Agents / Smallville, Project Sid, AgentSociety (Tsinghua), AI Economist (Salesforce).**
- **EVE Online:** Faucets/Sinks, Monthly Economic Report.
- **Cities: Skylines II:** Einzelagenten-Sim mit ECS auf CPU; Launch-Ökonomie musste nachgepatcht werden.
- **Diamond, McQuade & Qian (2019):** SF-Mietregulierung 1994 – Referenz für M2.
- **Colburn & Aldern:** „Homelessness Is a Housing Problem“ – Validierungsidee für emergente Obdachlosigkeit.

## 11. Offene Entscheidungen

- Skala: 1 Sim-Haushalt = 100 echte? Stadtgröße 10k–50k Agenten?
- Kalibrierung: generische Stadt vs. Berlin-Daten (Mietspiegel, Zensus, IBB-Wohnungsmarktbericht).
- Bürger-Host: Pharo pur oder Glamorous Toolkit; auf welchem Rechner?
- Luau-Einbettung in Python: Binding wählen bzw. bauen.
- Fester Spark-Slot: welcher Wochentag?
- Lizenz des Projekts (Kompatibilität mit CC-BY-SA-Assets).

## 12. Nächste Schritte

1. Cormas sichten: bewährte Abstraktionen, Grenzen.
2. State-Schema als Code (NumPy-Strukturen) + Bilanz-Invarianten.
3. Minimal-Tick mit Verfall, Verträgen, Logit-Umzug, Matching ⇒ M-perf messen.
4. M0-Lauf.

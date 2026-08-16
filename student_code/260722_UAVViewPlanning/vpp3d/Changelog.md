# Changelog — vpp3d

## 2026-07-01 — Registrierungs-Overlap: flächenbasiert + Connected Set Cover [5]

### Motivation

Der bisherige Overlap-Schritt war **randbasiert**: er markierte je gewähltem Scan
das äußere `min_overlap`-Band des Footprints (normierter Bild-Radius ≥
`1 − min_overlap`) als „muss nochmal" und ergänzte greedy Brücken, bis jede
Rand-Facette doppelt gesehen wurde. Zwei Probleme:

1. **Falsche Parameter-Semantik.** 25 % *Radius* entsprechen ~44 % *Fläche*
   (`1 − 0.75²`). Fast die halbe Fläche jedes Scans musste dupliziert werden →
   sehr viele unnötige Brücken-Posen. Der Ansatz verwechselte „*Nachbarn müssen
   sich überlappen*" mit „*jeder Rand muss doppelt gesehen werden*".
2. **Nachträgliches Reparieren.** Der Cover-Schritt minimierte für Abdeckung
   allein (kachelt mit minimaler Überlappung → registrierungs-feindlich); die
   Reparatur konnte nur *hinzufügen*, nie die Basisauswahl umwählen → strukturell
   suboptimal.

### Änderung

* **`overlap.py`** komplett auf **flächenbasierten Registrierungsgraphen**
  umgestellt (Ansatz aus der großen `pipeline`): Kante zwischen zwei Scans, wenn
  sie ≥ `min_overlap` Anteil gemeinsamer Facetten teilen, **bezogen auf den
  kleineren Scan**. `min_overlap` ist damit ein echter **Flächen-Anteil** (25 %
  = 25 % gemeinsame Fläche). Gefordert ist **Zusammenhang** des Graphen;
  `ensure_connected` ergänzt nur so viele Brücken, wie zum Verbinden der
  Komponenten nötig — statt Ränder zu verdoppeln. Neuer `OverlapResult`:
  `selected/added/n_components/overlap_facets/min_overlap` (kein `margin_facets`
  / `violated_after` mehr).
* **`setcover.py`** um **`connected_set_cover_ilp`** ergänzt: optimiert Abdeckung
  *und* Zusammenhang **gemeinsam** in einem CP-SAT-Modell (Single-Commodity-Flow
  über die Overlap-Kanten aus `overlap.overlap_adjacency`). Näher am echten
  Optimum als die Zwei-Phasen-Heuristik; exakte Referenz für kleine/mittlere
  Instanzen. Zerfällt schon der Kandidaten-Graph in mehrere Komponenten, wird
  Zusammenhang nur je Komponente gefordert → bleibt lösbar, wann immer die reine
  Abdeckung lösbar ist.
* **`run.py` / `inspector.py`** — `--method` um `connected` (und `all`)
  erweitert; `ensure_overlap` → `ensure_connected` (läuft fair nach jeder
  Methode; bei `connected` 0 Brücken). Auswertungs-Achse:
  *keine* → *Reparatur (A)* → *gemeinsam (Connected ILP)*.
* **`viz.py` / `inspector.py`** — Visualisierung zeigt statt der Rand-Facetten
  jetzt die tatsächliche Registrierungs-Überlappung (Facetten, die ≥ 2 Scans
  sehen) in Orange/Gelb.

## 2026-07-01 — Lokale Pose-Verfeinerung [5b] (Local Viewpoint Optimization)

### Motivation

Die Kandidatengenerierung [2/3] erzeugt Posen auf einem **diskreten Raster**:
feste Distanz-Stützstellen × Azimut-Ringe des Kegel-Samplings. Die vom Set Cover
gewählten Posen liegen deshalb selten im Optimum ihres Footprints — sie sehen
ihren zugewiesenen Facetten-Cluster oft etwas zu nah/zu fern, mit unnötig
schrägem Einfallswinkel oder am Rand des Sichtfelds. Statt das Raster global zu
verfeinern (teuer), wird jede *gewählte* Pose nachträglich lokal auf ein
Qualitätsoptimum verschoben. Der Ansatz ist aus der großen `pipeline`
(`pipeline/refine.py`) übernommen und auf die vpp3d-Datenstrukturen portiert.

### Was geändert wurde

Neuer, **optionaler** Pipeline-Schritt [5b] zwischen Overlap-Reserve [5] und
Sequenzierung [7], aktiviert per `--refine`. Ohne das Flag ist das Verhalten
unverändert.

Jede gewählte Pose wird einzeln per **Nelder-Mead** (ableitungsfrei) auf ein
lokales Maximum einer **glatten** Qualitäts-Score verschoben:

    J = q_dist · q_incidence · q_frustum

- `q_dist`: Glockenkurve um den Soll-Abstand `d_opt` (→ 0 an `d_min`/`d_max`),
- `q_incidence`: Smoothstep von 0 (bei `θ_max`) auf 1 (frontaler Einfall),
- `q_frustum`: Glocke über die normierte Bildposition (1 in der Mitte,
  → 0 am FoV-Rand).

Die binäre Sichtbarkeit aus [4] ist als Zielfunktion ungeeignet (stückweise
konstant, keine Gradienten); die glatte Score liegt innerhalb der harten
Constraints und zieht die Pose gleichzeitig Richtung `d_opt`, frontalem Einfall
und zentriertem Footprint. Die **Orientierung ist kein freier Parameter**,
sondern wird je Schritt auf den qualitätsgewichteten Facetten-Schwerpunkt
ausgerichtet (Yaw exakt, Pitch geklemmt, Roll = 0 — 4-DoF wie [2/3]). Der
Suchradius ist auf `max_offset` (Default 0,5·`d_opt`) begrenzt; Clearance-Proxy
und optionaler Bodenfilter (`--z-min`) gehen als harte Strafterme ein.

**Abdeckungsbewusst (Default):** Jede Pose wird nur über ihre vom Set Cover
zugewiesenen Facetten (ihre Zeile in V) optimiert; eine Barriere
(`--coverage-weight`, Default 4,0) bestraft jede zugewiesene Facette, die unter
die Sichtbarkeitsgrenze rutscht. Damit wandert die Pose nicht in dichtere
Nachbarregionen ab und die globale Abdeckung bleibt erhalten. Die Ablation
`--refine-free` optimiert stattdessen über die ganze Nachbarschaft (ohne
Zuweisung) — die Posen kollabieren und die Abdeckung bricht ein.

**Abdeckungsmaximierend (`--gain-weight g`, Default 0 = aus):** Zusätzlich zu den
zugewiesenen Facetten gehen benachbarte, noch erreichbare Facetten mit Gewicht
`g` in die Score ein — die Pose nimmt so weitere Facetten mit. Der drohende
**d_max-Drift** (die Pose weicht zurück, um mehr Fläche einzufangen) wird durch
die Distanz-Glocke `q_dist` verhindert: bei `d_max` ist `q_dist = 0`, und die
Barriere der zugewiesenen Facetten schlägt zu. Das gewünschte Ziel „sichtbare
Facetten maximieren" wird so erreicht, ohne den Messabstand zu verschlechtern.

Die Score ist **rein geometrisch** (kein Raycast) — eine verschobene Pose könnte
neu verdeckt sein. Maßgeblich bleibt die Sichtbarkeit [4], die nach der
Verfeinerung erneut läuft; `run.py` gibt die neu geraycastete Abdeckung
vorher/nachher aus.

### Betroffene Dateien (nur `vpp3d`)

- **`refine.py`** (neu):
  - `pose_quality(position, facet_pos, facet_norm, cfg, weights=None, …)` —
    glatte Score `J`, optional per-Facette gewichtet (für `--gain-weight`);
    liefert auf Wunsch Orientierung + Per-Facette-Score.
  - `refine_poses(poses, facets, cfg, target_facets=…, coverage_weight=…,
    gain_weight=…, max_offset=…, z_min=…)` — Hauptfunktion (Nelder-Mead je Pose);
    liefert neue `Poses` + `RefineInfo` mit Vorher/Nachher-Metriken.
  - `RefineInfo.summary()` — Score `J`, weiche Abdeckung, Einfallswinkel,
    `|d − d_opt|`, Verschiebung.
  - `target_facets_from_visibility(V, rows)` — leitet die abdeckungsbewusste
    Facetten-Zuweisung je gewählter Pose aus der Coverage-Matrix ab.
  - `subset_poses(poses, rows)` und `refine_selected_poses(vis, poses, scene,
    selected, cfg, …)` — **gemeinsamer Einstieg für `run.py` und `inspector.py`**:
    bestimmt die Zuweisung, verschiebt die gewählten Posen und schreibt
    Position/Yaw/Pitch der `selected`-Zeilen in-place zurück. Die
    Sichtbarkeits-Neuberechnung bleibt dem Aufrufer überlassen (siehe unten).
- **`run.py`** — Schritt [5b] nach der Overlap-Reserve eingehängt
  (`_apply_refinement` → `refine_selected_poses`): verfeinert die gewählten
  Posen, rechnet die Sichtbarkeit der *Teilmenge* neu, meldet die Abdeckung
  vorher/nachher und nutzt die zurückgeschriebene Geometrie in Tracking,
  Sequenzierung und Visualisierung. Neue CLI-Flags: `--refine`, `--refine-free`,
  `--gain-weight`, `--max-offset`, `--sigma-scale`, `--coverage-weight`,
  `--z-min`.
- **`inspector.py`** — **auf denselben Stand gebracht**: identische CLI-Flags und
  derselbe `refine_selected_poses`-Aufruf nach der Overlap-Reserve. Danach wird
  die **vollständige** Coverage-Matrix neu geraycastet (`compute_visibility` über
  alle Posen), damit die interaktive Facetten-Einfärbung (Kandidaten-, Plan-,
  Übersichts- und Mosaik-Modus) die verschobenen Posen korrekt widerspiegelt;
  Route und Matrix-Log verwenden ebenfalls die verfeinerten Posen.
- **`__init__.py`** — Exporte `refine_poses`, `pose_quality`, `RefineInfo`,
  `target_facets_from_visibility`.
- **`README.md`** — Schritt [5b] in der Pipeline-Tabelle, eigener
  Beschreibungsabschnitt, CLI-Beispiele für `run.py` *und* `inspector.py` sowie
  die neuen Optionen in der Inspektor-Optionstabelle ergänzt.

### Verifikation

- `--scene box --refine` (abdeckungsbewusst): `|d − d_opt|` 0,121 → 0,040 m
  (**−66,6 %**), mittlerer Einfallswinkel 16,6° → 13,9°, erreichbare Abdeckung
  66,7 % → 66,4 % (nach erneutem Raycast **nahezu erhalten**; die kleine
  Einbuße stammt aus der rein geometrischen Score ohne Verdeckung).
- `--scene box --refine --gain-weight 0.3` (abdeckungsmaximierend): weiche
  Abdeckung +8,9 % Facetten/Pose (statt +2,1 %) bei weiter sinkendem
  `|d − d_opt|` (**−36 %**) — **kein d_max-Drift** bestätigt.
- `--scene notched_box --refine --refine-free` (Ablation): Abdeckung bricht von
  68,8 % auf 53,8 % ein — bestätigt die Notwendigkeit der abdeckungsbewussten
  Zuweisung.
- Voller Durchlauf `--scene box --refine` mit Tracking: die verfeinerten Posen
  fließen korrekt in Tracking (3 Standorte) und Sequenzierung ein.
- `inspector.py --scene notched_box --refine` (headless getestet): Verfeinerung
  läuft (`|d − d_opt|` −56 %), die Coverage-Matrix wird anschließend vollständig
  neu geraycastet (1623/2357 abgedeckt) und Route/Log nutzen die neuen Posen.

## 2026-07-01 — Vektorisierte Kandidatengenerierung (Kegel-Sampling)

### Motivation

Die Kandidatengenerierung `sample_pose_regions` (`candidates.py`) baute die rohe
Posenmenge in **drei verschachtelten Python-Schleifen** auf (`for frac`, `for az`,
`for d`). Je Iteration folgte zwar eine vektorisierte N-Facetten-Operation, doch
die Zahl der Schleifendurchläufe wächst mit `cone_fractions × n_azimuth ×
n_distance` und erzeugt entsprechend viele Teil-Arrays samt abschließendem
`np.vstack`/`np.concatenate`. Bei großen Meshes wird das spürbar langsamer als
nötig. Die große `pipeline` (`pipeline/candidates.py`) berechnet dieselbe Menge
bereits als reine Tensoroperation — dieser Ansatz wird nun nach `vpp3d`
übernommen.

### Was geändert wurde

Die Posen aller `(Richtungskombination × Distanz × Facette)`-Tripel werden als
zusammenhängender Tensor `(D, T, N, ·)` berechnet und in **C-Ordnung** zu Zeilen
abgeflacht. Die einzige verbleibende Python-Schleife läuft über die
`cone_fractions` (typisch drei Einträge); die eigentliche Arbeit je Facette ist
vollständig vektorisiert.

Entscheidend ist die **exakt erhaltene Zeilenreihenfolge** `[Richtungskombi,
Distanz, Facette]`. Sie ist nicht kosmetisch: `project_and_dedup` behält per
`np.unique(..., return_index=True)` + `np.sort` den *ersten* Repräsentanten je
Voxelzelle. Eine andere Reihenfolge hätte andere Kandidaten selektiert und damit
das Ergebnis verändert. Der **Datenoutput bleibt daher unverändert** (identische
`Poses`: `positions`, `yaws`, `pitches`, `ids`, `seed_facet`).

### Betroffene Dateien (nur `vpp3d`)

- **`candidates.py`**:
  - Neuer Helper `_cone_directions(Nn, t1, t2, cfg)` → alle Kegelrichtungen als
    `(D, N, 3)`-Tensor, in der Reihenfolge der früheren `(frac, az)`-Doppel-
    schleife (frontale Richtung + Azimut-Ringe je `cone_fraction`).
  - `sample_pose_regions` ersetzt die drei verschachtelten Schleifen durch
    Broadcasting über `(D, T, N, ·)`; Blickrichtung (distanzunabhängig) und
    Seed-Facetten werden per `np.broadcast_to` über die Distanzachse
    wiederverwendet. `seed_facet` wird aus dem read-only-View materialisiert
    (`.copy()`), damit `Poses` ein beschreibbares Array hält.

### Verifikation

- Numerischer Gleichheitstest gegen die alte Schleifenimplementierung
  (137 Facetten, `cone_fractions = 0/0.5/0.85` × 12 Azimute × 3 Distanzen →
  10 275 Posen): `positions`, `yaws`, `pitches`, `seed_facet` und `ids` sind
  **bit-genau identisch** (`np.array_equal == True`), inklusive Zeilenreihenfolge.

## 2026-07-01 — k-Coverage im Set Cover (Registrierungs-Redundanz)

### Motivation

Das Set Cover in `vpp3d` stellte bisher nur sicher, dass **jede Facette von
mindestens einem Scan** gesehen wird. Für die reine Abdeckung genügt das, für die
spätere Registrierung (ICP) ist es aber fragil: Ist Scan A die *einzige* Pose,
die eine bestimmte Facette sieht, und ist A verrauscht oder schlecht registriert,
existiert keine zweite, unabhängige Messung als Absicherung.

**k-Coverage** fordert stattdessen, dass jede Facette von **mindestens `k` Scans**
gesehen wird. Mit `k = 2` erhält jede Facette redundante Messungen, was die
Robustheit der Registrierung erhöht (die billige, ungerichtete Variante des
Registrierungs-Constraints; `overlap.py` bleibt die schärfere, rand-gezielte).

### Was geändert wurde

k-Coverage ist als **Option mit Default `k = 1` eingeführt** — bei `k = 1` ist
das Verhalten identisch zum bisherigen Stand (reine Vollabdeckung), sodass alle
bestehenden Aufrufe und Ergebnisse unverändert bleiben.

Der Kniff ist die **Bedarfs-Kappung** (übernommen aus der großen `pipeline`):
Kann eine Facette physikalisch nur von `a < k` Posen gesehen werden, wird ihr
Bedarf auf `a` gekürzt, statt das Problem unlösbar zu machen. Die
„Vollständigkeit relativ zur *erreichbaren* Fläche" bleibt damit gewahrt.

### Betroffene Dateien (nur `vpp3d`)

- **`setcover.py`** — Kern der Änderung:
  - Neuer Helper `_demand(V, k)` → Bedarf je Facette = `min(k, verfügbare
    Abdeckung)`. Die Kappung macht nicht `k`-fach abdeckbare Facetten lösbar.
  - `greedy_set_cover(V, k=1, trace=None)`: Das Defizit ist nun ein
    **Integer-Zähler** je Facette, der pro gewählter Pose um 1 verringert wird
    (statt eines Bool-Flags). Der Gewinn einer Pose zählt nur Facetten mit noch
    offenem Bedarf (`deficit > 0`). Der `trace`-Mechanismus für `stepviz.py`
    bleibt kompatibel (Facette gilt als „neu", solange ihr Bedarf offen ist).
  - `ilp_set_cover(V, k=1, ...)`: Coverage-Constraint `Σ x_j >= Bedarf_i` statt
    `>= 1`.
  - `CoverResult` um das Feld `k` erweitert; `summary()` zeigt `k=…` nur bei
    `k > 1`. `n_coverable`/`n_covered` beziehen sich jetzt auf den gekappten
    Bedarf.

- **`run.py`**, **`inspector.py`** — neues CLI-Flag `--k` (Default `1`), das an
  beide Solver durchgereicht wird.

- **`README.md`**, **`usage.md`** — `--k` in den Optionstabellen ergänzt und
  k-Coverage gegenüber der Overlap-Reserve eingeordnet.

### Verifikation

- `--scene box --no-overlap --no-tracking`: `k = 1` → **49 Scans** (unverändert),
  `k = 2` → **103 Scans**, jeweils 100 % der erreichbaren Facetten abgedeckt
  (bei `k = 2` doppelt). Die Kappung hält auch nur einfach sichtbare Facetten als
  „abgedeckt".
- Greedy und ILP liefern auf einer kleinen Testmatrix für `k = 1` das bisherige
  Ergebnis; für `k = 2` die erwartete redundante Auswahl.

## 2026-07-01 — Zweistufige Sequenzierung mit Tracking-Integration

### Motivation

Das bodengebundene Tracking-System (mobile Roboter mit kabelgebundenen Sensoren)
hat einen begrenzten Arbeitsradius. Sobald die Drohne ihn verlässt, muss der
Tracker **repositioniert** werden — teuer in Zeit und Koordinationsaufwand.

Die bisherige Sequenzierung (`sequencing.py`) löste einen **einzigen flachen TSP
über alle Drohnenposen**, ohne zu berücksichtigen, an welchem Tracking-Standort
sich die Drohne dabei befand. Tracking und Sequenzierung waren damit entkoppelt.
Die Folge: Die Route konnte ständig zwischen verschiedenen Tracking-Bereichen
hin- und herspringen und erzwang so viele unnötige Repositionierungen.

### Was geändert wurde

Die Sequenzierung ist nun **zweistufig entlang der hierarchischen Kopplung an
das Tracking-System** aufgebaut (übernommen aus dem Ansatz der großen
`pipeline`):

1. **Äußere Tour** — TSP über die Tracking-Standorte. Minimiert Anzahl und
   Weglänge der Repositionierungen. Startpunkt ist der Standort mit den meisten
   zugeordneten Posen (pragmatischer Messbeginn).
2. **Innere Tour** — je Tracking-Segment ein *offener* TSP über die diesem
   Standort zugeordneten Drohnenposen. Der Startpunkt jedes Segments liegt an
   der Pose, die dem **Austritt des vorherigen Segments** am nächsten liegt, für
   einen nahtlosen Übergang ohne Rücksprünge.

Damit wird jeder Tracking-Bereich zusammenhängend abgeflogen, bevor
repositioniert wird — Sprünge zwischen den Bereichen entfallen.

### Betroffene Dateien (nur `vpp3d`)

- **`sequencing.py`** — komplett überarbeitet:
  - Zweistufige `sequence_route(pose_positions, tracking=None)`: Verrechnet den
    `TrackingPlan` (Standorte + Zuordnung der Posen) in die Reihenfolge.
  - Nicht trackbare Posen (`assignment == -1`) werden dem XY-nächsten Standort
    zur Sequenzierung zugeschlagen, in `station_ids` aber weiterhin als `-1`
    markiert (die Constraint-Verletzung bleibt sichtbar).
  - TSP-Heuristik auf die matrixbasierte Variante (Nearest-Neighbour-Aufbau +
    offener 2-opt, rein numpy) umgestellt — schneller bei gleicher Qualität.
  - **Rückwärtskompatible Schnittstelle:** `Route.order` bleibt eine Permutation
    über die gewählten Posen und `Route.length` der euklidische Flugweg — die
    Visualisierung (`viz.py`) funktioniert unverändert. Neu ergänzt:
    `station_ids` (Tracking-Segment je Routenpose) und `n_repositions`.
  - **Fallback:** Ohne Tracking-Plan (`tracking=None` oder keine Standorte, z. B.
    `--no-tracking`) läuft weiterhin der einzelne flache TSP wie zuvor.

- **`run.py`** — reicht den geplanten `TrackingPlan` (`trk`) an
  `sequence_route(...)` durch, statt die Posen ohne Tracking-Kontext zu
  sequenzieren.

- **`__init__.py`**, **`README.md`** — Beschreibung von Schritt [7] an die neue,
  zweistufige Kopplung angepasst.

### Abgrenzung

Das Tracking bleibt in `vpp3d` **nachgelagert** zur Posenwahl (kein bi-level
Set Cover wie in der großen `pipeline`). Neu gekoppelt ist ausschließlich die
*Sequenzierung*: Standortwahl und Posenwahl werden nach wie vor getrennt
optimiert, aber die Abfahrreihenfolge respektiert nun die Tracking-Segmente.

### Verifikation

- `--scene notched_box`: 80 Posen → **2 Tracking-Segmente, 1 Repositionierung**;
  die Route bleibt je Segment zusammenhängend statt zwischen Bereichen zu springen.
- `--scene box --no-tracking`: Fallback auf den flachen TSP bestätigt
  (79 Posen, 90,3 m Flugweg).

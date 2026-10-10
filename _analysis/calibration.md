# Kalibracja w repozytoriach referencyjnych

Przekrojowy przegląd tego, jak kalibracja jest zaimplementowana w submodułach wroclaw-twin.
Dokument odnosi się do konkretnych plików i linii kodu; ścieżki są względem korzenia danego submodułu.

---

## TL;DR

| Repo | Co kalibruje | Algorytm | Entry point |
|------|--------------|----------|-------------|
| **matsim-python-tools** | Udziały modalne, utilities od dystansu, korekty grupowe | Iteracyjna aktualizacja ASC + Optuna (TPE) | `matsim/calibration/__init__.py` → `create_calibration()` |
| **matsim-berlin** | Udziały modalne (SrV 2018), natężenia ruchu (VMZ), wszystkie linki (Cadyts) | j.w. + OptaPlanner (plan-selection ILP) + Cadyts | `src/main/python/calibrate*.py`, `RunCountOptimization.java`, `RunOpenBerlinCalibration.java` |
| **matsim-scenario-template** | Udziały modalne (MID / custom) | Identyczny z matsim-python-tools | `src/main/python/calibrate.py` (szablon z FIXME-ami) |
| **eqasim-france** | — | brak aktywnego kodu; tylko komentarze w syntezie populacji | — |
| **dlr-tsc** | — | brak | — |

---

## 1. matsim-python-tools — biblioteka kalibracji ASC

> Repo: `matsim-vsp/matsim-python-tools` (PyPI: `matsim-tools`)
> Lokalnie: `matsim-python-tools/`

### Struktura pakietu kalibracji

```
matsim/calibration/
  __init__.py          ← główny orkiestrator, create_calibration()
  base.py              ← CalibratorBase + wzór aktualizacji ASC
  calib_asc.py         ← ASCCalibrator — podstawowy (udziały modalne)
  calib_asc_dist.py    ← ASCDistCalibrator — ASC + utilities od dystansu
  calib_group_asc.py   ← ASCGroupCalibrator — korekty per grupa wiekowa/zatrudnienie
  analysis.py          ← obliczanie udziałów modalnych z trips.csv / persons.csv
  constraints.py       ← ograniczenia parametrów (positive, negative, bounded, zero)
  group.py             ← parsing grup (np. "age_group_21-35")
  utils.py             ← schedulery learning-rate (linear, exponential, distance-aware)
tests/calibration/
  test_calibration.py  ← 50 iteracji softmax; waliduje MAE < 0.03
```

### Jak działa pętla

**Wzór aktualizacji ASC** (`base.py`, linie ~219–230):

```
Δ_asc[i] = ln(z_i / m_i) - ln(z_0 / m_0)
```

gdzie `z_i` = docelowy udział trybu `i`, `m_i` = zasymulowany udział, indeks `0` = tryb referencyjny (zwykle `walk` z ASC=0).

**Jedna iteracja kalibracji:**

1. Optuna proponuje zestaw parametrów (trial).
2. Orkiestrator zapisuje je do `params/run{N}.yaml`.
3. MATSim uruchamiany jako subprocess: `java -cp scenario.jar … --yaml params/run{N}.yaml --output runs/{N:03d}`.
4. Z wynikowego `trips.csv` obliczane są udziały modalne → błąd MAE/MSE → wartość objective Optuny.
5. Opcjonalnie: output plans z iteracji N stają się inputem iteracji N+1 (`chain_runs`).

**Storage:** SQLite via `optuna.storages.RDBStorage` (`{name}.db`). Wznowienie = ponowne `study.optimize()`.

**Schedulery LR** (`utils.py`):
- `linear_scheduler(start, interval)` — domyślny w Berlin/template
- `linear_dist_scheduler(start_rate, end_rate, end_km)` — dedykowany dla kalibracji utilities od dystansu
- dowolna `Callable[[int], float]`

**Kalibracja odległościowa** (`calib_asc_dist.py`) — bins dystansu (0–1, 1–2, 2–5, 5–10, 10–20, 20+ km); osobny LR per bin, `fixed_mode_dist` (zazwyczaj `"car"`) jest pomijany.

**Kalibracja grupowa** (`calib_group_asc.py`) — format SBB `BehaviorGroups`: każda grupa ma oddzielne korekty `deltaConstant`; używane np. do różnych grup wiekowych.

**Parametry przekazywane do MATSim (YAML):**
```yaml
scoring:
  scoringParameters[]:
    modeParams[]:
      constant: <ASC>                      # dla ASCCalibrator
      deltaPerDistGroup: [...]             # dla ASCDistCalibrator
SBBBehaviorGroups:
  behaviorGroup[]:
    personGroupAttributeValues[]:
      absoluteModeCorrections:
        deltaConstant: <korekta>           # dla ASCGroupCalibrator
```

### Istotne pliki z liniami

| Plik | Co zawiera |
|------|-----------|
| `matsim/calibration/__init__.py:98–279` | `create_calibration()` — main entry point; logika tworzenia trial, wywoływania MATSim, parsowania wyników |
| `matsim/calibration/base.py:219–230` | Wzór aktualizacji ASC |
| `matsim/calibration/utils.py` | `linear_scheduler`, `linear_dist_scheduler`, `default_chain_scheduler` |
| `matsim/calibration/analysis.py` | Parsowanie `trips.csv` i `persons.csv` → udziały modalne |
| `tests/calibration/test_calibration.py` | Test zbieżności 50 iteracji, cel [0.1, 0.5, 0.3, 0.1] |

---

## 2. matsim-berlin — kalibracja scenariusza Berlin

> Repo: `matsim-scenarios/matsim-berlin` (branch `release/v6`)
> Lokalnie: `matsim-berlin/`

Berlin implementuje **trzy oddzielne poziomy kalibracji**:

### 2a. Kalibracja ASC (udziały modalne) — Python

Skrypty w `src/main/python/`:

| Skrypt | Cel | Docelowe wartości |
|--------|-----|-------------------|
| `calibrate.py` | Podstawowa kalibracja ASC | SrV 2018: walk 29.7%, bike 17.8%, PT 26.5%, car 20.1%, ride 5.9% |
| `calibrate_dist.py` | ASC + utilities od dystansu | j.w. + `fixed_mode_dist="car"`, `dist_lr` ze startem 0.3 → 3.5 do 25 km |
| `calibrate_groups.py` | Korekty per grupa wiekowa | Filtr: tylko osoby z prefiksem `"berlin"` w populacji |

Inicjalne ASC (`calibrate.py`):
```python
initial = {"bike": -1.4, "pt": 0.6, "car": -1, "ride": -1.4, "walk": 0}
```

Każdy skrypt woła `study.optimize(obj, N)` — Berlin uruchamia 4 triala (podstawowy) lub 10 (dystans/grupy).

**Makefile** (`Makefile:262–300`) — powiązane targety:
- `prepare-calibration` → tworzy wejściowe plany + sieć + liczniki do kalibracji
- `eval-opt` → uruchamia `RunCountOptimization` (patrz niżej)
- Wynikowe plany z iteracji: `berlin-cadyts-input-v6.4-25pct.plans.xml.gz`

### 2b. Kalibracja liczników ruchu — OptaPlanner (Java)

**Plik:** `src/main/java/org/matsim/prepare/opt/RunCountOptimization.java`

**Idea:** mając `experienced_plans` (kilka planów na agenta z biegów symulacyjnych), wybierz po jednym planie na agenta tak, żeby sumaryczne natężenia na linkach były jak najbliżej pomiarów z detektorów VMZ.

**Algorytm:**
- Solver: OptaPlanner (SAT/heuristic) z konfiguracją iteracyjną
- `problem.iterate(5000, 0.5, beta, 0.01)` — 5000 kroków, beta skaluje metrykę błędu
- Metryki błędu (`ErrorMetric` enum): `abs_error`, `log_error`, `symmetric_percentage_error`
- Liczniki skalowane przez rozmiar próbki (25pct → ×4) i opcjonalnie `CAR_FACTOR=5`

**Entry point z Makefile:**
```bash
java -cp matsim-berlin-*.jar org.matsim.prepare.RunCountOptimization \
  --counts berlin-v6.4-counts-vmz.xml.gz \
  --plans experienced_plans.xml.gz \
  --metric log_error
```

### 2c. Kalibracja Cadyts — pętla MATSim

**Plik:** `src/main/java/org/matsim/prepare/RunOpenBerlinCalibration.java`

Kalibracja Cadyts działa **wewnątrz** biegu MATSim jako moduł scoringu — nie jest osobną pętlą zewnętrzną:

- Enum `CalibrationMode.cadyts` (linie 282–315): włącza `CadytsCarModule`
- Scoring (linie 439–445): `CadytsScoring<Link>` z wagą `30 × beta`
- Strategia wymuszania innowacji (linia 455): `ForceInnovationStrategyChooser` z częstotliwością `ceil(1/weight)` — żeby Cadyts miał z czego wybierać
- Minimalistyczny scoring: tylko Cadyts + travel time (linie 439–445)
- Cel: dopasowanie natężeń na wszystkich linkach sieci, nie tylko stacjach pomiarowych

**Cadyts vs OptaPlanner:**

| Aspekt | Cadyts | OptaPlanner |
|--------|--------|-------------|
| Kiedy | W trakcie biegu MATSim (co iterację) | Po biegach; post-processing |
| Wejście | Plany MATSim live | Gotowe `experienced_plans` |
| Zasada | Modyfikuje scoring agentów | Selekcja planów z gotowego zbioru |
| Granulacja | Każdy link | Wybrane linki (stacje pomiarowe) |

### 2d. Kalibracja sieci (przepustowości i prędkości)

**Pliki:** `src/main/python/capacity/`, `src/main/java/org/matsim/prepare/BerlinNetworkParams*.java`

Nie jest to kalibracja modalna, ale parametrów sieci:
- Mikrosymulacje SUMO dla skrzyżowań (różne typy: `priority`, `right_before_left`, `traffic_light`)
- Na wynikach trenowane modele XGBoost (`BerlinNetworkParams*.java`) → predykcja capacity i free speed
- Uruchamiane: `capacity/runSUMO.sh`, `capacity/runOPT.sh` (tablice SGE/SLURM)

---

## 3. matsim-scenario-template — szablon kalibracji

> Repo: `matsim-scenarios/matsim-scenario-template`
> Lokalnie: `matsim-scenario-template/`

**Plik:** `src/main/python/calibrate.py` (84 linie)

Identyczny wzorzec jak matsim-python-tools / Berlin, ale jest to szablon z miejscami do uzupełnienia:

```python
# FIXME: Adjust initial ASC values
initial = {"bike": -0.141210, "pt": 0.0781478, "car": 0.871977, "ride": -2.22873}

# FIXME: Adjust target mode shares (or load from mid.csv)
target = {...}

# FIXME: Adjust spatial filter for persons
transform_persons = filter_persons   # geopandas + shapefile

study, obj = create_calibration(
    "calib",
    ASCCalibrator(modes, initial, target, lr=linear_scheduler(start=0.3, interval=15)),
    "matsim-template-1.0.jar",
    "../input/v1.0/[name]-v1.0.config.xml",
    args="--10pct",
    chain_runs=utils.default_chain_scheduler
)
study.optimize(obj, 1)
```

Nasz punkt startowy do adopcji tego wzorca. Pliki FIXME wskazują dokładnie, co trzeba dostarczyć.

---

## 4. eqasim-france i dlr-tsc — brak kalibracji

**eqasim-france** skupia się na syntezie populacji z danych ankietowych; kalibracja pojawia się wyłącznie w komentarzach:
- `synthesis/population/spatial/secondary/locations.py` — „Resampling for calibration" (algorytm rozmieszczenia, nie pętla kalibracyjna)
- `data/hts/output.py` — wzmianka, że dane HTS mogą być użyte do kalibracji zewnętrznej

**dlr-tsc** — repozytorium Traffic Signal Control; brak kodu kalibracyjnego.

---

## 5. Przepływ danych — diagram koncepcyjny

```
Dane ankietowe / pomiary ruchu
  (SrV, VMZ, MID, liczniki SUMO)
          │
          ▼ wartości docelowe
  ┌──────────────────────┐
  │    Calibrator init   │  (ASC/Dist/Group, LR scheduler, constraints)
  └──────────┬───────────┘
             │
         Optuna Trial
             │
             ▼
  params/run{N}.yaml  ◄──── wzór ASC: Δ = ln(z_i/m_i) - ln(z_0/m_0)
             │
             ▼
  MATSim Run (subprocess)
  java -cp scenario.jar … --yaml … --output runs/{N:03d}
             │
             ▼
  trips.csv / persons.csv
             │
             ▼
  Udziały modalne / błąd
             │
       ┌─────┴─────┐
       │           │
   MAE < próg?   OptaPlanner
    STOP         (selekcja planów → liczniki)
                    │
                 Cadyts
              (scoring in-loop → linki)
```

---

## 6. Co z tego wynika dla Wrocławia

1. **Biblioteka:** `matsim-python-tools` dostarcza gotowy `create_calibration()` + Optuna. Nie trzeba pisać pętli od zera — wystarczy dostarczyć `target` (udziały modalne z danych Wrocławia) i `initial` ASC.

2. **Wzorzec pliku kalibracyjnego:** `matsim-scenario-template/src/main/python/calibrate.py` to szablon do skopiowania i uzupełnienia FIXME-ami (dane źródłowe Wrocławia, shapefile obszaru).

3. **Trzy oddzielne poziomy** (jak w Berlin): ASC (udziały), OptaPlanner (liczniki), Cadyts (linki) — nie muszą być wdrażane jednocześnie; ASC to minimum.

4. **LR = 0.3, interval = 15** — wartości startowe sprawdzone w Berlin; można adoptować wprost.

5. **Kalibracja sieci (capacity/freespeed):** wymaga mikrosymulacji SUMO dla lokalnej sieci Wrocławia — osobny, kosztowny etap (`capacity/runSUMO.sh` jako wzorzec).

6. **Storage Optuna (SQLite):** wznowienia są bezkosztowe — `study.optimize()` kontynuuje od ostatniego triala. Dobra właściwość na środowisko klastrowe.

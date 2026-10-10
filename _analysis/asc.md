# ASC — Alternative Specific Constant

Wyjaśnienie pojęcia w kontekście modelu wyboru środka transportu i kalibracji MATSim.

---

## Co to jest ASC?

**Alternative Specific Constant** (stała specyficzna dla alternatywy) to wyraz wolny w funkcji użyteczności modelu dyskretnego wyboru. Reprezentuje **średnią preferencję** dla danego środka transportu, której nie da się wyjaśnić żadną mierzalną cechą podróży (czasem, kosztem, dystansem).

Innymi słowy: ASC „zbiera" wszystko, co sprawia, że ludzie wybierają rower/auto/PT nawet wtedy, gdy czas i koszt wszystkich opcji byłby identyczny — przyzwyczajenie, wygodę, prestiż, niezmierzone atrybuty.

---

## Skąd się bierze w modelu?

### Funkcja użyteczności (model logitowy)

Dla osoby `n` i środka transportu `i`:

```
V(n, i) = ASC_i + β_tt · czas_i(n) + β_cost · koszt_i(n) + ...
```

Prawdopodobieństwo wyboru trybu `i` z zestawu `C`:

```
P(i | n) = exp(V(n, i)) / Σ_j∈C exp(V(n, j))
```

Ponieważ liczy się tylko **różnica** użyteczności między trybami, jedna ASC musi być zablokowana na 0 — to tryb referencyjny. W praktyce zwykle `ASC_walk = 0`.

Przykład z matsim-berlin (inicjalne wartości):

| Tryb | ASC (inicjalny) |
|------|----------------|
| walk | 0.0 (referencyjny) |
| bike | −1.4 |
| pt   | +0.6 |
| car  | −1.0 |
| ride | −1.4 |

Ujemna ASC roweru oznacza, że — przy identycznym czasie i koszcie — ludzie *rzadziej* wybierają rower niż chodzenie. Dodatnia ASC PT oznacza odwrotnie.

---

## Dlaczego ASC trzeba kalibrować?

Model estymowany na ankiecie (np. SrV Berlin, MiD Niemcy) daje wartości `β` (wrażliwości na czas/koszt). ASC wyznaczane podczas estymacji odzwierciedlają jednak **preferencje ankietowanych**, nie populacji symulowanej.

Gdy uruchomimy MATSim z tymi wartościami, zasymulowane udziały modalne prawie na pewno **rozminą się z obserwowanymi** — bo:
- Populacja syntetyczna ≠ populacja ankietowa
- Sieć MATSim upraszcza rzeczywistość (brakuje utrudnień, komfortu, itd.)
- Dane ankietowe mają własne błędy

Kalibracja ASC to iteracyjna korekta tych wyrazów wolnych tak, żeby symulacja odtworzyła rzeczywiste udziały modalne.

---

## Wzór aktualizacji ASC

Stosowany w `matsim-python-tools` (`base.py`, linie ~219–230):

```
ASC_i(t+1) = ASC_i(t) + lr(t) · [ ln(z_i / m_i(t)) − ln(z_0 / m_0(t)) ]
```

gdzie:
- `z_i` — docelowy udział trybu `i` (dane obserwowane)
- `m_i(t)` — zasymulowany udział trybu `i` w iteracji `t`
- `z_0`, `m_0` — analogicznie dla trybu referencyjnego (walk)
- `lr(t)` — learning rate (maleje z czasem; typowo `linear_scheduler(start=0.3)`)

**Intuicja:** jeśli symulacja produkuje zbyt mało rowerzystów (`m_bike < z_bike`), to `ln(z/m) > 0`, więc `ASC_bike` rośnie → rower staje się atrakcyjniejszy w kolejnej iteracji.

Człon z trybem referencyjnym (`− ln(z_0/m_0)`) zapewnia, że suma prawdopodobieństw pozostaje znormalizowana — korekta jednego trybu nie "kradnie" masy tylko temu jednemu trybowi kosztem wszystkich pozostałych.

---

## ASC a inne parametry scoringu

ASC jest jednym z wielu parametrów modelu. Hierarchia w MATSim:

```
scoring.scoringParameters[subpopulation]
  └── modeParams[mode]
        ├── constant          ← to jest ASC
        ├── marginalUtilityOfTraveling_util_hr
        ├── monetaryDistanceRate
        └── deltaPerDistGroup[bin]  ← kalibracja odległościowa (ASCDistCalibrator)
```

Kalibracja **tylko ASC** (najprostszy przypadek) nie dotyka `β` — zmienia tylko wyraz wolny. To odpowiada założeniu, że wrażliwości na czas/koszt są poprawne, a problem leży w przeciętnych preferencjach.

---

## Rozszerzenia ASC używane w projektach referencyjnych

### Kalibracja odległościowa (`ASCDistCalibrator`)

Zamiast jednej stałej per tryb, kalibruje użyteczność dla **przedziałów dystansu**:

```
V(n, i) = ASC_i + Σ_b δ_{i,b} · 𝟙[dist(n) ∈ bin_b] + ...
```

Biny: 0–1 km, 1–2 km, 2–5 km, 5–10 km, 10–20 km, 20+ km.
Parametr `YAML`: `deltaPerDistGroup`.

Używane gdy model dobrze odtwarza udziały ogólne, ale błędnie rozkłada je po dystansach (np. PT za rzadko na krótkich dystansach).

### Kalibracja grupowa (`ASCGroupCalibrator`)

Korekty `Δ_ASC` per **grupa demograficzna** (wiek, zatrudnienie):

```
V(n, i) = ASC_i + Δ_{i, grupa(n)} + ...
```

Format w YAML: `SBBBehaviorGroups → absoluteModeCorrections → deltaConstant`.

Używane gdy model dobrze odtwarza udziały populacyjne, ale błędnie rozkłada je po grupach wiekowych.

---

## Zbieżność i kryteria zatrzymania

Pętla kalibracyjna jest zbieżna gdy:

```
MAE = (1/|C|) · Σ_i |z_i − m_i| < ε
```

Typowe `ε` w testach matsim-python-tools: `0.03` (3 p.p.).

W praktyce (Berlin): 4–10 iteracji zewnętrznej pętli, każda = pełny bieg MATSim (100–150 iteracji wewnętrznych).

---

## Związek z kodem

| Plik | Rola |
|------|------|
| `matsim-python-tools/matsim/calibration/base.py:219–230` | Implementacja wzoru aktualizacji |
| `matsim-python-tools/matsim/calibration/calib_asc.py` | `ASCCalibrator` — prosty wariant |
| `matsim-python-tools/matsim/calibration/calib_asc_dist.py` | `ASCDistCalibrator` — biny dystansu |
| `matsim-python-tools/matsim/calibration/calib_group_asc.py` | `ASCGroupCalibrator` — per grupa |
| `matsim-python-tools/matsim/calibration/utils.py` | `linear_scheduler`, `linear_dist_scheduler` |
| `matsim-berlin/src/main/python/calibrate.py` | Przykład wywołania z wartościami Berlin |
| `matsim-scenario-template/src/main/python/calibrate.py` | Szablon do adopcji |

Szczegółowy opis pętli kalibracyjnej i przepływu danych: [`calibration.md`](calibration.md).

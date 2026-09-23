# wroclaw-twin — repozytoria referencyjne (Faza 1–2)

Zbiór repozytoriów open-source (jako git submodules) wykorzystanych jako materiał dowodowy do analizy architektury dla projektu symulacji Wrocławia (MATSim → SUMO, wieloagentowe RL do sterowania sygnalizacją świetlną).

## Co tu jest

- **`_analysis/notes/<repo>.md`** — 12 notatek (Faza 1), po jednej na repozytorium referencyjne, z ustaleniami popartymi cytatami `plik:linia`.
- **`_analysis/comparison.md`** — porównanie między repozytoriami (Faza 2): macierz 13 wymiarów architektonicznych (orkiestracja, konfiguracja, dane, proweniencja, kontrakt MATSim→SUMO, kalibracja, testy, licencje, runtime Python↔Java, pakietowanie, dokumentacja).
- **`MANIFEST.tsv`** — lista repozytoriów referencyjnych: tier, URL, tryb pobrania (`full` / `light` / `sparse`), status, przypięty commit i jego data.
- **`_trees/*.txt`** — pełne listingi plików dla repozytoriów pobranych tylko częściowo (`sumo`, `sumo-scenarios`, `sumo-berlin`), żeby notatki mogły odwoływać się do plików, których zawartości nie ma lokalnie.
- **12 submoduły git** — repozytoria referencyjne, każdy przypięty do konkretnego commita wymienionego w `MANIFEST.tsv`.

**To repozytorium NIE zawiera** danych lokalnych projektu (ankiety KBR, macierze OD, dane GPS/ZDiUM, geobazy) — są zastrzeżone i trzymane poza gitem, zgodnie z ustaleniami monorepo `wust-terrain`.

## Dla zespołu: co analizować

Zacznij od `_analysis/comparison.md` (streszczenie porównawcze), potem sięgnij do `_analysis/notes/<repo>.md` dla szczegółów per repozytorium. Wszystkie twierdzenia są tam podparte ścieżką pliku i numerem linii w odpowiednim submodule — warto je zweryfikować w kodzie źródłowym.

## Jak sklonować

Repozytoria referencyjne są dołączone jako **git submodules**, część jest spłycona (`--depth 1`, tylko 1 commit — bez pełnej historii), a `sumo` dodatkowo ma **sparse checkout** (tylko katalogi związane z importerem MATSim).

### Krok 1: sklonuj główne repo razem z submodułami

```bash
git clone --recurse-submodules <URL-tego-repo>
cd wroclaw-twin
```

Jeśli repo jest już sklonowane bez `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

### Krok 2: spłyć submoduły (opcjonalnie, ale zalecane — repo jak `sumo` są bardzo duże)

Domyślnie powyższe polecenie pobierze **pełną historię** każdego submodułu (kilka GB w przypadku `sumo` czy `matsim-berlin`). Żeby pobrać tylko przypięty commit (tak jak było to zrobione lokalnie), użyj:

```bash
git submodule update --init --recursive --depth 1
```

### Krok 3: `sumo` wymaga ręcznego sparse checkout

Submodule `sumo` w oryginale ma pobrane tylko katalogi związane z importem z MATSim (`tools/import/matsim`, `tests/tools/import/MATSim`, `docs/web/docs/Tools/Import`) — cały monorepo SUMO ma dziesiątki tysięcy plików. Po `submodule update` katalog `sumo/` będzie zawierał pełny checkout, chyba że skonfigurujesz sparse checkout ręcznie:

```bash
cd sumo
git sparse-checkout init --cone
git sparse-checkout set docs/web/docs/Tools/Import tests/tools/import/MATSim tools/import/matsim
cd ..
```

### Weryfikacja

Po sklonowaniu porównaj commity submodułów z `MANIFEST.tsv`:

```bash
git submodule status
```

Każdy wiersz powinien zaczynać się od commita zgodnego z kolumną `commit` w `MANIFEST.tsv` (bez `-` z przodu — `-` oznacza niepobrany submoduł, `+` oznacza commit inny niż zapisany w indeksie).

## Uwaga o `pam-sdist`

W `MANIFEST.tsv` pozycja `pam-sdist` (pakiet PyPI `cml-pam==0.3.2`) ma status `FAILED` — pobranie się nie powiodło i pakiet **nie jest częścią analizy**. Wnioski dotyczące PAM w `_analysis/comparison.md` są oznaczone jako pośrednie / wnioskowane.

## Licencje repozytoriów referencyjnych

Repozytoria `eqasim-*` i `matsim-*` są na licencji GPL/AGPL. Notatki w `_analysis/` opisują wzorce własnymi słowami i nie zawierają kopiowanego kodu (poza pojedynczymi identyfikatorami i krótkimi fragmentami konfiguracji jako dowodem). Przy dalszej pracy z kodem tych repozytoriów zachowaj warunki ich licencji.

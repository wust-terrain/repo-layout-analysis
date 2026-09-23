# Brief dla analizy repozytoriów referencyjnych (Faza 1)

## Nasz kontekst

Budujemy symulację Wrocławia na potrzeby badań nad wieloagentowym RL do sterowania sygnalizacją świetlną.

- **Agentowy model popytu (MATSim)** wytwarza popyt w stanie równowagi z danych lokalnych.
- Popyt jest eksportowany jako **wersjonowany artefakt** i konsumowany przez **SUMO** (mikrosymulacja + środowisko RL).
- MATSim jest tu generatorem popytu, NIE środowiskiem ewaluacji sterowania.

Zespół: 5 studentów MSc (każdy własna praca, wspólna publikacja) + ~10-osobowe koło naukowe. Praca asynchroniczna, jeden właściciel + jeden reviewer na issue.

Dane lokalne: badanie ankietowe KBR 2018 i 2024 (różne podziały modalne), macierze OD z sieci komórkowej, dane prędkości GPS, pomiary ruchu ZDiUM, geobazy GIS ZDiUM (detektory, pętle, sterowniki), OSM, GTFS. **Część z nich jest zastrzeżona i nie może być commitowana ani redystrybuowana.**

Ustalone (nie otwieramy ponownie): monorepo w organizacji GitHub `wust-terrain`, **workspace'y `uv`**; pre-commit: `ruff`, `mypy`; `docs/dane/` i `docs/decyzje/` wspólne dla wszystkich faz.

Obecny stan monorepo: `packages/kbr-eda/` (pakiet `src/kbr`, hatchling, pandas/geopandas, notebooki), `docs/dane/_SZABLON.md`, `docs/decyzje/_SZABLON.md`, dane na wspólnym dysku wskazanym w `.env` (`raw/` tylko do odczytu, `interim/`, `processed/`), `.gitignore` blokuje csv/parquet/shp itd. Brak jeszcze root `pyproject.toml` z `[tool.uv.workspace]`.

## Zadanie

Napisz `_analysis/notes/<name>.md` (katalog `/home/dominik/refs/wroclaw-twin/_analysis/notes/`) dla przydzielonego repozytorium. **Po polsku**, terminy techniczne i identyfikatory po angielsku.

Omów TYLKO te wymiary, o których repozytorium faktycznie coś mówi; dla pozostałych napisz jedną linię „nie dotyczy” (ewentualnie z informacją, gdzie szukałeś).

Wymiary:
1. **Orkiestracja** — deklarowanie, kolejność, cache i unieważnianie etapów; jak działa częściowe ponowne uruchomienie. (Style do porównania: stage'e `synpp`, DAG celów plikowych w Make, numerowane skrypty z `--skip-to`, instalowalny pakiet z CLI entry points.)
2. **Konfiguracja i wiele scenariuszy** — jedno źródło prawdy dla przebiegu; jak wyrażone są warianty/miasta.
3. **Układ danych i duże pliki** — raw vs pośrednie vs wyjście; co jest w git; jak pobierane są wejścia; wzorce `.gitignore`; gdzie publikowane są wyniki.
4. **Proweniencja i wersjonowanie** — wersja scenariusza vs wersja kodu; hash commita w wynikach; changelog; plik cytowania; automatyzacja release'ów.
5. **Źródło prawdy dla sieci** — która sieć jest kanoniczna i jak utrzymywana jest spójność ID między narzędziami.
6. **Przekazanie MATSim → SUMO** — co konsumuje i emituje `matsim_importPlans.py`, co ginie w tłumaczeniu (trasy, semantyka odjazdu, tryby, osoby vs pojazdy), co to oznacza dla kontraktu naszego artefaktu. Fixtury testowe są źródłem prawdy, nie dokumentacja.
7. **Kalibracja między narzędziami** — gdzie wynik jednego symulatora parametryzuje drugi.
8. **Próbka / skalowanie** — jak reprezentowane są próbki populacji i skalowanie przepustowości.
9. **Testy i determinizm** — małe scenariusze testowe, syntetyczne dane testowe, testy determinizmu, walidacja schematów.
10. **Dane zastrzeżone i licencje** — licencje per katalog, separacja danych nieredystrybuowalnych.
11. **Runtime Python ↔ Java** — jak JAR jest budowany, przypinany i wywoływany z Pythona; czy Java żyje w tym samym repo.
12. **Pakietowanie** — układ biblioteka vs aplikacja; jak to się mapuje na członków workspace'u `uv`.
13. **Dokumentacja** — dokumenty per miasto/przypadek, notatki obok scenariuszy.

## Format każdego ustalenia

```
### <N>. <Wymiar>

**<krótki tytuł ustalenia>**
- **Zaobserwowane:** co repo robi, z dowodem: ścieżka pliku (i numery linii, gdy przydatne), np. `synpp/src/synpp/pipeline.py:412-440`.
- **Dlaczego:** jakie ograniczenie ta praktyka rozwiązuje. Oznacz *(wnioskowane)*, jeśli zgadujesz; bez oznaczenia tylko wtedy, gdy repo mówi to wprost (cytat/ścieżka).
- **Czy dotyczy nas?** tak / częściowo / nie — jedno zdanie odnoszące się do naszego kontekstu.
```

Na początku pliku: nagłówek z nazwą repo, commitem (z `MANIFEST.tsv`), jednym akapitem „czym jest to repo”, i informacją o ewentualnej częściowości klonu.
Na końcu: sekcja **„Najważniejsze dla nas”** (3–6 punktów) oraz **„Nie znaleziono”** (czego szukałeś i gdzie).

## Zasady

- Każde twierdzenie opieraj na przeczytanym kodzie/pliku. Numery linii weryfikuj — nie zgaduj.
- Oddzielaj *zaobserwowane* od *wnioskowanego*. Jeśli czegoś nie znalazłeś, napisz „nie znaleziono” i gdzie szukałeś.
- Popularność, gwiazdki, reputacja instytucji nie są dowodem.
- **Licencje:** repozytoria `eqasim-*` i `matsim-*` są GPL/AGPL. **Nie kopiuj kodu** do notatek (poza pojedynczymi identyfikatorami, nazwami funkcji/plików, krótkimi fragmentami konfiguracji potrzebnymi jako dowód — max kilka linii). Opisuj wzorce własnymi słowami.
- Nie modyfikuj niczego poza swoim plikiem notatek. Nie uruchamiaj sieciowych instalacji; możesz czytać pliki, grep, git log w repo (read-only).
- Część repo to częściowe klony: pełne listingi plików są w `/home/dominik/refs/wroclaw-twin/_trees/<name>.txt` (dla `sumo-scenarios`, `sumo-berlin`, `sumo`). Wykorzystaj je, żeby opisać, co istnieje w repo, choć nie jest pobrane.
- Długość: tyle, ile trzeba; typowo 150–400 linii. Gęsto, bez waty.
- W odpowiedzi końcowej do mnie: 5–10 linii podsumowania najważniejszych ustaleń + ścieżka do pliku. Nie wklejaj całej notatki.

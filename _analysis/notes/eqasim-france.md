# eqasim-france

- **Repo:** https://github.com/eqasim-org/eqasim-france
- **Commit:** `a9db00cf776244874fd2c80c7ce2281b69180c69` (z `_analysis/MANIFEST.tsv`, pobrano 2026-09-23)
- **Klon:** pełne drzewo plików (`full`), ale historia płytka — `git log` pokazuje tylko 1 commit (`a9db00c feat: Jenkinsfile (#556)`), więc historii zmian nie dało się analizować; posiłkowałem się `CHANGELOG.md`.
- **Licencja:** GPL-2.0 (`LICENSE`, `docs/misc/legal.md`). Poniżej żadnego kodu, tylko opisy wzorców i identyfikatory.

**Czym jest to repo.** Pipeline w Pythonie (oparty na `synpp`) generujący syntetyczną populację (gospodarstwa, osoby, aktywności, podróże, pojazdy) dla Île-de-France, a przez konfigurację także dla innych regionów Francji (Lyon, Nantes, Tuluza, Korsyka, Bordeaux). Opcjonalnie buduje z tego kompletny scenariusz MATSim: sieć z OSM + GTFS przez `pt2matsim`, populację/facilities/households/vehicles w XML, `config.xml` i gotowy JAR `eqasim-java`. Java nie leży w tym repo, tylko jest klonowana i budowana Mavenem w trakcie przebiegu. Projekt jest *aplikacją*, nie biblioteką: `uv` + `pyproject.toml` bez `build-system`, uruchamiany przez `uv run -m synpp config.yml`.

---

### 1. Orkiestracja

**Stage'e `synpp` jako moduły Pythona z `configure/execute/validate`, DAG wynika z deklaracji**
- **Zaobserwowane:** Stage to po prostu moduł Pythona (np. `data/hts/entd/raw.py`, `synthesis/population/sampled.py`), a jego identyfikator to ścieżka z kropkami. W `configure` stage deklaruje zależności (`context.stage(...)`) i parametry, które czyta (`context.config(...)`). `execute` zwraca wynik, który trafia do cache. Cele przebiegu wymienia się w liście `run:` w YAML-u (`config.yml:8-10`: `synthesis.output`, opcjonalnie `matsim.output`). Kolejność wykonania nigdzie nie jest zapisana ręcznie; `synpp` wyznacza ją z grafu zależności.
- **Dlaczego:** dzięki temu stage'e są modułowe, a każdy wariant regionu/ankiety to tylko inny podgraf *(wnioskowane)*.
- **Czy dotyczy nas?** tak. To jeden z kandydatów na nasz styl orkiestracji: ma jawny DAG w kodzie i działa bez Make.

**Warunkowe gałęzie DAG przez aliasy (wybór źródła danych w `configure`)**
- **Zaobserwowane:** `data/hts/selected.py:1-20` czyta `hts` z configu i podpina jeden z kilku łańcuchów (`data.hts.entd.reweighted`, `data.hts.egt.filtered`, `data.hts.edgt_lyon.reweighted`, `data.hts.edgt_44.reweighted`, `data.hts.emp.reweighted`, `data.hts.emc2_33.reweighted`, `data.hts.mobisurvstd.filtered`) pod wspólnym aliasem `hts`. Na nieznaną wartość rzuca wyjątek. Podobnie `synthesis/population/sampled.py:11-15` wybiera źródło `data.census.filtered` albo `synthesis.population.projection.reweighted` (zależnie od `projection_year`), a `synthesis/output.py:33-34` dołącza `matsim.simulation.prepare` tylko przy `mode_choice`.
- **Dlaczego:** kod downstream czyta zawsze `context.stage("hts")` i nie musi wiedzieć, z której ankiety pochodzą dane *(wnioskowane)*.
- **Czy dotyczy nas?** tak. KBR 2018 i KBR 2024 pasują wprost do wzorca „jeden alias, kilka implementacji `raw → cleaned → filtered → reweighted`”.

**Cache i unieważnianie: hash configu i zależności plus „odcisk” z `validate`**
- **Zaobserwowane:** Stage'e wczytujące surowe dane zwracają z `validate` odcisk pliku wejściowego, najczęściej `os.path.getsize` (`data/census/raw.py:56-60`, `data/hts/entd/raw.py:75-86`, `data/od/raw.py:75-76`; łącznie 30 funkcji `validate` w `data/`). `matsim/runtime/eqasim.py:86` zwraca mtime lokalnej ścieżki `eqasim_path`. `documentation/meta_output.py:45-47` zwraca wersję połączoną z hashem commita, więc zmiana commita unieważnia ten stage. Parametry oznaczone `volatile = True` (`processes`, `java_memory`: `matsim/runtime/java.py:6`, `synthesis/population/matched.py:28`) nie wpływają na klucz cache *(znaczenie `volatile` wnioskowane z nazwy i użycia; samego `synpp` nie ma w tym repo)*. `docs/simulation/simulation_execution.md:29-31` mówi wprost, że po dodaniu `matsim.output` ponowne wywołanie uruchomi tylko brakujące stage'e.
- **Dlaczego:** rozmiar pliku jest tanim odciskiem wielogigabajtowych archiwów. Wada *(wnioskowane)*: podmiana pliku na inny o identycznym rozmiarze nie unieważni cache. Stage `data/hts/mobisurvstd/raw.py:27-29` w ogóle nie zwraca odcisku.
- **Czy dotyczy nas?** tak. Wzorzec „validate zwraca fingerprint wejścia” warto przejąć, ale z hashem (lub przynajmniej rozmiarem i mtime), a nie z samym rozmiarem.

**Częściowe ponowne uruchomienie i nadpisywanie z CLI**
- **Zaobserwowane:** dowolny klucz configu można nadpisać z linii poleceń (`docs/population/population_execution.md:102-105`, np. `--output_prefix`, `--output_path`). Jenkins tak właśnie przechodzi po kilku `sampling_rate` na jednym cache (`Jenkinsfile:117-124`). Testy wywołują `synpp.run(stages, config, working_directory=...)` bezpośrednio z Pythona (`tests/test_pipeline.py:22`).
- **Dlaczego:** jeden cache obsługuje wiele wariantów; stage'e przed próbkowaniem się nie powtarzają *(wnioskowane)*.
- **Czy dotyczy nas?** tak. Wywoływanie pipeline'u jako API z testów jest ważne dla naszego CI.

### 2. Konfiguracja i wiele scenariuszy

**Jeden plik YAML na miasto; różnice to kilka kluczy**
- **Zaobserwowane:** w repo jest `config.yml` (IDF, minimalny), `config_full.yml` (316 linii, wszystkie opcje z domyślnymi wartościami w komentarzach) i `config_{lyon,nantes,toulouse,corsica}.yml`. Po porównaniu wszystkie miejskie pliki są kopią `config.yml` z tym samym szkieletem (`working_directory`, `run`, `processes`, `sampling_rate: 0.001`, `random_seed: 1234`, `java_memory: 14G`). Różnią się tylko:
  - zakresem: `regions` / `departments` (Lyon `["01","38","42","69"]` `config_lyon.yml:36-37`; Nantes `[44]`; Tuluza 6 departamentów `config_toulouse.yml:37`; Korsyka `regions: [94]` `config_corsica.yml:36`),
  - `output_prefix` (`lyon_`, `nantes_`…),
  - ścieżkami danych względem `data_path` (`gtfs_path`, `osm_path`, `ban_path`, `bdtopo_path` z sufiksem miasta),
  - wyborem ankiety `hts` (Nantes `edgt_44` `config_nantes.yml:18`, reszta `entd`),
  - pojedynczymi parametrami algorytmów (Korsyka `matching_minimum_observations: 5` `config_corsica.yml:41`).
  Bordeaux ma dokumentację (`docs/cases/bordeaux.md`), ale nie ma swojego `config_bordeaux.yml` i nie jest w toctree `docs/cases/cases_summary.md:61-69`.
- **Dlaczego:** kod jest wspólny, a region to tylko filtr przestrzenny i zestaw ścieżek. Wartości domyślne żyją w `configure` stage'y, a nie w YAML-u (np. `output_prefix` domyślnie `ile_de_france_`, `synthesis/output.py:25`).
- **Czy dotyczy nas?** częściowo. Mamy jedno miasto, ale warianty (KBR 2018 vs 2024, próbka, zakres przestrzenny) można wyrazić tym samym wzorcem: mały plik wariantu na wspólnych domyślnych. Unikałbym kopiowania całego szkieletu, bo tu każdy plik powtarza te same 30 linii.

**Warianty źródeł danych per region (ankiety HTS)**
- **Zaobserwowane:** klucz `hts` przyjmuje `entd`, `egt`, `edgt_lyon`, `edgt_44`, `emp`, `emc2`, `mobisurvstd` (`data/hts/selected.py:5-18`). Każda ankieta ma własny podkatalog z łańcuchem `raw → cleaned → filtered → (reweighted)` (`data/hts/*/`). Lyon ma dodatkowo pod-wariant źródła tej samej ankiety: `edgt_lyon_source: adisp|cerema`. Wartość domyślna `unchosen` celowo rzuca wyjątek (`data/hts/edgt_lyon/filtered.py:9-18`, `docs/cases/lyon.md:149`), bo ten sam survey przychodzi w dwóch różnych formatach plików (`raw_adisp.py`, `raw_cerema.py`). Wspólny kontrakt wyjściowy wszystkich ankiet sprawdza `data/hts/hts.py:245-268` (`check()`: wymagane kolumny households/persons/trips, brak ujemnych czasów, spójność first/last trip). Uogólnieniem jest `mobisurvstd`: zewnętrzny pakiet standaryzuje dowolną francuską ankietę do jednego formatu (`data/hts/mobisurvstd/raw.py:18-24`, `docs/cases/cases_summary.md:24-26`).
- **Dryf dokumentacji:** w `config_full.yml:43` komentarz wymienia tylko `entd, egt, edgt_lyon, edgt_44, mobisurvstd`, a `emp` i `emc2` pomija. Wartość klucza to `emc2`, ale katalog nazywa się `emc2_33`.
- **Dlaczego:** każda ankieta ma inną strukturę, a z tego samego badania mogą istnieć dwie dystrybucje (ADISP vs CEREMA). Jawny wybór z twardym błędem zapobiega cichemu użyciu złego parsera *(wnioskowane)*.
- **Czy dotyczy nas?** tak, bardzo. KBR 2018 i KBR 2024 mają różne podziały modalne. Wzorzec „osobny łańcuch per ankieta + wspólna funkcja `check()` na wspólnym schemacie + brak ukrytego defaultu” przenosi się 1:1.

**Wycinek przestrzenny (cutter)**
- **Zaobserwowane:** sekcja `cutter` (`config_full.yml:176-188`) i stage `matsim.simulation.cutter.cut` (test: `tests/test_simulation.py:178-199`) tną gotowy scenariusz do wielokąta GeoJSON i dodają prefiks `cutter_`.
- **Czy dotyczy nas?** częściowo. Przyda się, gdy SUMO ma obejmować tylko fragment Wrocławia, a MATSim całą aglomerację.

### 3. Układ danych i duże pliki

**Dane, cache i wyjście poza repo; trzy katalogi podawane w configu**
- **Zaobserwowane:** `data_path`, `working_directory` (cache) i `output_path` to ścieżki absolutne w YAML-u. Rekomendowany układ `/project/{code,data,cache,output}` opisuje `docs/population/population_execution.md:9-15`. Katalog `data/` w repo zawiera **kod** stage'y wczytujących dane (`data/census`, `data/hts`, `data/osm`…), a nie dane. `.gitignore` ignoruje `cache`, `/output`, `env`, `config_local_*.yml`, `.script_data/` oraz dwa historyczne katalogi `data/nantes_2015`, `data/lyon_2015` (pozostałość z czasów, gdy dane leżały w `data/`, *wnioskowane*). `synthesis/output.py:37-41` wymaga, żeby katalog wyjściowy istniał.
- **Dlaczego:** nie da się commitować dużych i licencjonowanych danych. Wzorzec `config_local_*.yml` pozwala trzymać lokalne ścieżki poza gitem *(wnioskowane)*.
- **Czy dotyczy nas?** tak. Zgadza się z naszym `.env` i `raw/interim/processed`. Wzorzec `config_local_*.yml` w `.gitignore` warto przejąć. Nazwa `data/` dla kodu jest myląca i jej nie powielać.

**Pobieranie wejść: skrypt wyliczający potrzebne pliki z configu, plus cache-proxy**
- **Zaobserwowane:** `scripts/download.py` (typer) czyta ten sam `config.yml`. Na podstawie `regions`/`departments` i kodów INSEE wylicza departamenty, a potem rejestruje URL i ścieżkę docelową każdego pliku. Dotyczy to ENTD (`:230-245`), BD TOPO per departament z zaszytą datą wydania (`:276-281`), BAN, urban type (tylko gdy `use_urban_type`) oraz OSM/GTFS (tylko gdy w `run` jest `matsim.output` albo włączone jest `mode_choice`, `:300`, `:358`). Na koniec raportuje brakujące pliki. `scripts/verify_data.py` z cotygodniowego workflowu `.github/workflows/data.yml` sprawdza, czy źródłowe URL-e nadal działają. `scripts/cache_server/` to FastAPI-proxy cache'ujące pobrania (klucz: sha256 URL-a, TTL, limit rozmiaru; `scripts/cache_server/main.py:115-117`, `config.yml`), używany przez `--cache-server` w Jenkinsie.
- **Dryf dokumentacji:** `download.py` pobiera BD TOPO 3-5 z 2025-12-15, a `docs/cases/lyon.md` każe ściągać edycję 2022.
- **Dlaczego:** otwarte dane francuskie znikają lub zmieniają URL-e; cotygodniowy test to wykrywa, a proxy chroni CI przed niestabilnymi serwerami *(częściowo wprost w komentarzach Jenkinsfile:98-102)*.
- **Czy dotyczy nas?** częściowo. Dla OSM i GTFS przyda się skrypt „manifest pobrań wyliczany z configu”. Dla danych zastrzeżonych (KBR, ZDiUM) pobierania nie ma, więc zostaje tylko raport brakujących plików.

**Publikacja wyników**
- **Zaobserwowane:** repo samo nie publikuje populacji. Jenkins archiwizuje `pipeline_output/*`, a opcjonalnie cache, dane i repo jako artefakty builda (`Jenkinsfile:131-199`). Obraz Dockera trafia do `ghcr.io` (`.github/workflows/docker-publish.yml`). Odrębna publikacja danych (Data in Brief) jest tylko wspomniana w README.
- **Czy dotyczy nas?** częściowo. Nasz wersjonowany artefakt popytu wymaga osobnego miejsca publikacji; tutaj tę rolę pełnią artefakty CI.

### 4. Proweniencja i wersjonowanie

**`meta.json` obok wyników: wersja, commit, seed, próbka, ankieta**
- **Zaobserwowane:** stage `documentation.meta_output` zapisuje `<prefix>meta.json` z polami `sampling_rate`, `hts`, `random_seed`, `created` (UTC), `version` (z `version.txt`) i `commit` (`git rev-parse HEAD`, a przy błędzie `"unknown"`) (`documentation/meta_output.py:15-43`). Zależą od niego zarówno `synthesis.output`, jak i `matsim.output` (`synthesis/output.py:22`, `matsim/output.py:19`). README (sekcja Versioning) mówi wprost, że `meta.json` podaje dokładny commit.
- **Luki *(wnioskowane)*:** brak flagi „dirty working tree”, brak wersji/commita `eqasim-java` i `pt2matsim`, brak wersji ani odcisków danych wejściowych. Obraz Dockera kopiuje kod bez `.git` (`Dockerfile:18-24`), więc w kontenerze `commit` będzie najpewniej `"unknown"`.
- **Czy dotyczy nas?** tak. To minimum dla naszego artefaktu. Trzeba je rozszerzyć o dirty flag, wersję JAR-a MATSim, hash configu i odciski wejść (KBR 2018/2024, OD, OSM snapshot).

**JAR symulacji pakowany razem ze scenariuszem**
- **Zaobserwowane:** przy `write_jar: true` (domyślnie) `matsim/output.py:50-54` kopiuje zbudowany JAR do wyjścia jako `<prefix>run.jar`. Scenariusz uruchamia się potem samodzielnie (`java -cp ile_de_france_run.jar ... RunSimulation`, `docs/simulation/simulation_execution.md:51`).
- **Dlaczego:** tak można odtworzyć wynik bez odtwarzania środowiska budowania.
- **Czy dotyczy nas?** tak. Artefakt popytu powinien mieć przypiętą wersję (lub sam JAR) MATSim, który go wytworzył.

**Automatyzacja release'u: release-please + Conventional Commits w tytułach PR**
- **Zaobserwowane:** `.github/workflows/release-please.yml` (`release-type: simple`) prowadzi `CHANGELOG.md` i `version.txt` (1.5.0) oraz `.release-please-manifest.json`. Wersję w README aktualizują znaczniki `x-release-please-start-version` (`README.md:76-82`). `.github/workflows/pr_lint.yml` wymusza konwencjonalny tytuł PR; commity w gałęzi nie muszą być konwencjonalne (`CONTRIBUTING.md`, sekcja o tytule PR), bo PR-y są squashowane (`CONTRIBUTING.md:87`).
- **Dryf:** `pyproject.toml` ma `version = "1.4.0"`, a `version.txt`, `CITATION.cff` i manifest mają 1.5.0. Release-please nie aktualizuje `pyproject.toml`, bo nie ma go w `extra-files`; ten klucz stoi zresztą w pliku manifestu, a nie w pliku konfiguracyjnym release-please, którego w repo brak. Źródłem prawdy dla `meta.json` jest `version.txt`.
- **Czy dotyczy nas?** tak. Squash z konwencjonalnym tytułem PR i release-please dobrze pasują do pracy „jeden właściciel + reviewer na issue”. Trzeba jednak mieć jedno źródło wersji albo dopisać `pyproject.toml` do release-please, żeby nie powtórzyć tego dryfu.

**`CITATION.cff`**
- **Zaobserwowane:** plik `CITATION.cff` cytuje oprogramowanie jako `eqasim` z `version: 1.5.0` i ma `preferred-citation` wskazujący artykuł TR-C 2021 (DOI 10.1016/j.trc.2021.103291). Nie ma w nim DOI samego oprogramowania ani Zenodo.
- **Czy dotyczy nas?** tak. Przy wspólnej publikacji 5 prac warto mieć `CITATION.cff` z `preferred-citation`.

### 5. Źródło prawdy dla sieci

**Sieć MATSim budowana z OSM + GTFS przez pt2matsim; brak sieci kanonicznej w repo**
- **Zaobserwowane:** stage'e `matsim/scenario/supply/{osm,gtfs,processed}.py` budują sieć przez `pt2matsim`. GTFS jest filtrowany na jeden dzień (`gtfs_date: dayWithMostServices`, `matsim/scenario/supply/gtfs.py:12-19`). Opcja `export_detailed_network` zapisuje geometrię sprzed uproszczenia do `<prefix>detailed_network.csv` (`matsim/output.py:44-48`). Nie ma żadnego mechanizmu spójności ID z innym symulatorem.
- **Czy dotyczy nas?** częściowo. Ta sieć jest pochodną OSM, a ID linków pochodzą z pt2matsim, więc jeśli SUMO ma własną sieć, mapowanie ID musimy zbudować sami. Eksport detailed network to ewentualny punkt zaczepienia.

### 6. Przekazanie MATSim → SUMO

Nie dotyczy. Szukałem w `matsim/`, `scripts/`, `docs/simulation/` (`grep sumo`): brak jakiegokolwiek eksportu do SUMO.

### 7. Kalibracja między narzędziami

Nie dotyczy (brak drugiego symulatora). Jedyne sprzężenie: przy `mode_choice: true` wyjście populacji przechodzi przez `matsim.simulation.prepare`, czyli wybór trybu po stronie Javy (`synthesis/output.py:33-34`).

### 8. Próbka / skalowanie

**Próbka populacji: stochastyczne zaokrąglenie wag spisu, potem losowanie gospodarstw z `sampling_rate`**
- **Zaobserwowane:** `synthesis/population/sampled.py:26-58` najpierw zaokrągla wagi gospodarstw stochastycznie do całkowitych krotności i replikuje gospodarstwa, dostając 100% populacji. Potem każde gospodarstwo wybiera z prawdopodobieństwem `sampling_rate`, więc próbka jest bernoulliowska na poziomie gospodarstw, a nie osób. Całość używa jednego generatora `np.random.default_rng(random_seed)`. `sampling_rate` trafia do Javy jako `--sample-size` w `RunGenerateConfig` (`matsim/simulation/prepare.py:90-91`). Skalowanie przepustowości (flow/storage capacity) liczy więc `eqasim-java`, a nie to repo. Domyślna próbka we wszystkich configach to `0.001`; testy używają `1.0`. `sampling_rate` jest zapisywany w `meta.json`.
- **Dlaczego:** losowanie całych gospodarstw zachowuje ich strukturę (wspólne auta, eskorta) *(wnioskowane)*.
- **Czy dotyczy nas?** tak. Do kontraktu artefaktu: próbka jest parametrem wyjścia zapisanym w metadanych, a SUMO musi znać `sampling_rate`, żeby przeskalować popyt albo przepustowość (po stronie SUMO odpowiednik `--scale`, *wnioskowane*).

### 9. Testy i determinizm

**Syntetyczne dane testowe generowane od zera w strukturze oryginałów**
- **Zaobserwowane:** `tests/testdata.py` (1192 linie) w fixture sesyjnym (`tests/conftest.py:6-22`) generuje „sztuczną Francję”: dwa regiony 50×50 km, departamenty 25×25 km, gminy 5×5 km, IRIS 500×500 m (docstring `tests/testdata.py:12-60`). Powstają pliki o tej samej strukturze i nazwach co prawdziwe: IRIS, spis, BPE, Filosofi, ENTD, **EGT** (niepubliczna, `:493-602`), OD, BD TOPO, BAN, SIRENE, OSM (siatka dróg 500 m, `:879-975`), GTFS, flota pojazdów. Każdy zbiór ma własny ustalony seed (`default_rng(1000)`, `(2533)`, `(5632)`…). Braki i przypadki brzegowe (NaN w koordynatach BPE, brakujące decyle dochodu) są wprowadzone celowo. Docstring mówi wprost, że dane są generowane od zera, a nie wycinane z prawdziwych, żeby można je było stopniowo rozszerzać pod konkretne funkcje.
- **Dlaczego:** CI może testować ścieżki kodu dla danych niepublicznych (EGT) bez dostępu do nich, a same fixture'y nie łamią licencji.
- **Czy dotyczy nas?** tak. To najważniejszy wzorzec dla nas: syntetyczne „mini-KBR 2018/2024”, OD i pomiary ZDiUM w identycznym schemacie pozwalają testować w publicznym CI bez danych zastrzeżonych.

**Test determinizmu: dwa pełne przebiegi i porównanie ze złotymi hashami MD5**
- **Zaobserwowane:** `tests/test_determinism.py:81-160` uruchamia `synthesis.output` dwukrotnie w osobnych cache (`processes = 1`, stały seed) i porównuje MD5 każdego pliku CSV i GPKG z zaszytymi w teście wartościami. Test sprawdza więc nie tylko powtarzalność, ale też brak zmian względem wartości referencyjnych. Normalizacja: `.gz` jest hashowany po dekompresji (gzip zapisuje timestamp, `:48-52`). GPKG hashowany jest jako posortowany zrzut SQL bez tabel `rtree` i z zamianą `MEDIUMINT→INTEGER` dla zgodności Linux/Windows (`:62-79`). Po stronie produkcji `synthesis/output.py:43-64` (`clean_gpkg`) nadpisuje `last_change` stałą datą i zaokrągla bbox. Wariant MATSim (`:162-205`) sprawdza tylko `config.xml`, `households.xml.gz` i `vehicles.xml.gz`, bez populacji i sieci; powód nie jest podany. Test uruchamia się w macierzy ubuntu/windows/macos (`.github/workflows/tests.yml:13-15`).
- **Dlaczego:** determinizm między OS-ami jest warunkiem porównywalności wyników. Każda zmiana wyniku wymusza jawną aktualizację hashy w PR *(wnioskowane)*.
- **Czy dotyczy nas?** tak. Chcemy tego samego dla artefaktu popytu. Warto od początku pisać wyjścia w formie deterministycznej (bez timestampów, z sortowaniem) i nie ograniczać się do hashy części plików.

**Testy „smoke” dla wariantów configu i kontrakt schematu**
- **Zaobserwowane:** `tests/test_pipeline.py:54-147` uruchamia pipeline dla kolejnych wariantów (`entd`, `egt`, `mode_choice`, `fleet_sample`, `bhepop2`, `urban_type`, motocykle, `force_model`, `census_attributes`, `housing_type`) i sprawdza dokładne liczności (np. 441 osób, 147 gospodarstw, 2961 aktywności) oraz obecność kolumn. `tests/test_simulation.py` buduje pełny scenariusz MATSim łącznie z klonowaniem i budową Javy (w CI instalowana jest Java 25, `tests.yml:19-22`) i sprawdza pliki wyjściowe oraz cutter. Test noise jest wyłączony domyślnie i sterowany zmienną `TEST_NOISE`. Walidacja schematu to asercje w kodzie (`data/hts/hts.py:245-268`); ani pandera, ani JSON Schema tu nie ma.
- **Czy dotyczy nas?** tak. Macierz testów „jeden test na wariant configu” to tani sposób, żeby każdy student nie psuł wariantów pozostałych.

### 10. Dane zastrzeżone i licencje

**Dane niepubliczne tylko jako opcja z dokumentacją, bez pobierania, z fixture'em w testach**
- **Zaobserwowane:** repo nie ma licencji per katalog, tylko GPL-2.0 dla całości. Dane niepubliczne są w dokumentacji jawnie oznaczone jako „Optional”: EGT („not available publicly”, `docs/population/population_data.md:81-87`), EDGT Lyon („available on request”, `docs/cases/lyon.md:58-62`), EMC2 Gironde („proprietary data”, `docs/cases/bordeaux.md:51`). Dokumentacja podaje dokładną listę wymaganych plików, ale `scripts/download.py` ich nie rejestruje (brak `egt` w skrypcie), a domyślna ankieta jest otwarta (`entd`). Z kolei EDGT 44 jest open data (`docs/cases/nantes.md:51`). Separację zapewnia wyłącznie to, że dane leżą poza repo (`data_path`); nie ma mechanizmu technicznego typu flag „restricted”.
- **Czy dotyczy nas?** tak. Wzorzec „otwarty wariant jako domyślny, zastrzeżony jako opcja z listą plików w docs i syntetycznym odpowiednikiem w testach” pasuje do KBR i ZDiUM. My potrzebujemy dodatkowo jawnego oznaczenia w `docs/dane/`.

### 11. Runtime Python ↔ Java

**Java w osobnym repo, klonowana i budowana w stage'u; wersja przypięta commitem**
- **Zaobserwowane:** stage `matsim.runtime.eqasim` (`matsim/runtime/eqasim.py`) w cache stage'u klonuje `eqasim-java` (`--single-branch -b <branch>`), robi checkout `eqasim_commit` i buduje moduł `ile_de_france` Mavenem (`-Pstandalone --projects ile_de_france --also-make package -DskipTests=true`, `:33-50`). Następnie sprawdza, czy powstał `ile_de_france-<version>.jar` (`:52-53`). Domyślne wartości w kodzie to `eqasim_version 2.2.0`, `branch main`, `commit fc85693` (`:7-9`). Alternatywą jest `eqasim_path`, czyli gotowy lokalny build: JAR jest wtedy kopiowany, a odciskiem jest mtime (`:57-62`, `:86`), co służy do generowania fixture'ów testowych dla eqasim-java (`config_corsica.yml:43-46`). Ścieżka JAR-a jest wynikiem stage'u, a inne stage'e wołają `eqasim.run(context, <main class>, args)` (`:23-27`), które deleguje do `java.run`. `java.run` składa `java -cp <jar> -Xmx<java_memory> -Djava.io.tmpdir=<cache>/__java_temp -Dmatsim.preferLocalDtds=true <class> <args>` (`matsim/runtime/java.py:20-46`) i uruchamia przez `subprocess.check_call`. `pt2matsim` jest obsługiwane identycznie: płytki klon tagu `v26.1`, build Mavenem, smoke-test `CreateDefaultOsmConfig` (`matsim/runtime/pt2matsim.py:12-43`). `validate` sprawdza obecność binarek i wersje (Java ≥17 `java.py:71`, Maven 3.x, git 2.x). Opcjonalny `maven_local_cache` izoluje repozytorium Mavena w cache (`maven.py:30-36`). Kroki po stronie Javy to osobne main-klasy wywoływane sekwencyjnie z plikami jako interfejsem (`RunPreparation`, `RunGenerateConfig`, `RunAdaptConfig`, `EditConfig`, `RunImputeSpatialAttribute`, `matsim/simulation/prepare.py:50-140`).
- **Uwagi *(zaobserwowane)*:**
  - wszystkie regiony (Lyon, Korsyka…) używają modułu Javy `ile_de_france` i klas `org.eqasim.ile_de_france.*`;
  - `config_full.yml:241-243` w komentarzach podaje nieaktualne wartości (`1.5.0`, `ece4932`), niezgodne z kodem (`2.2.0`, `fc85693`);
  - `eqasim_tag` jest walidowany (`:79-84`), ale `execute` go nie używa.
- **Dlaczego:** budowa ze źródła z przypiętym commitem daje odtwarzalność bez publikowania JAR-ów w repozytorium Mavena. Kosztem jest wymóg Mavena i Javy na każdej maszynie oraz długi pierwszy przebieg *(wnioskowane)*.
- **Czy dotyczy nas?** tak. Jeśli mamy własny kod MATSim, to „stage zwracający ścieżkę JAR-a + cienki wrapper `java.run` + commit/wersja jako parametr configu, liczony do cache” jest gotowym wzorcem. Lepiej jednak pobierać gotowy, przypięty JAR (np. z GitHub Releases/Packages) niż budować Mavenem u każdego studenta, a wersję JAR-a zapisywać w `meta.json`, czego to repo nie robi.

### 12. Pakietowanie

**Aplikacja jako wirtualny projekt `uv`, bez `build-system` i bez entry pointów**
- **Zaobserwowane:** `pyproject.toml` ma tylko sekcję `[project]` z listą zależności. Nie ma `[build-system]`, `[project.scripts]` ani `[tool.*]`. `uv.lock` pokazuje `source = { virtual = "." }` dla `eqasim-france` (`uv.lock:349-351`). Pakiety najwyższego poziomu (`data`, `synthesis`, `matsim`, `analysis`, `documentation`, `noise`) importuje się z katalogu roboczego; nie ma układu `src/`. Uruchomienie: `uv run -m synpp config.yml` (`docs/population/population_execution.md:61`), a skrypty pomocnicze to `uv run scripts/*.py`. `pytest` i `mock` są w zależnościach runtime, bo grup dev brak. `typer`, `rich`, `requests` i `pyyaml`, używane przez `scripts/`, nie są zadeklarowane bezpośrednio i przychodzą tranzytywnie (np. `typer` przez `mobisurvstd`, `uv.lock:802`). `requires-python >=3.13`, `.python-version` = 3.13. Docker kopiuje `pyproject.toml`, `uv.lock` i `.python-version`, robi `uv sync`, a dopiero potem kopiuje kod (`Dockerfile:14-24`).
- **Dlaczego:** to aplikacja-pipeline, a nie biblioteka do reużycia; stage'e muszą być importowalne przez `synpp` po ścieżce modułu *(wnioskowane)*.
- **Czy dotyczy nas?** częściowo. W workspace `uv` pipeline popytu może być podobnym wirtualnym członkiem, ale lepiej z `src/`, grupą `dev` dla pytest oraz jawnie zadeklarowanymi zależnościami skryptów (tu jest ukryta zależność tranzytywna). Reużywalne części (np. parser KBR) powinny być osobnym członkiem-biblioteką z `build-system`, jak nasz `kbr-eda`.

### 13. Dokumentacja

**Dokument per miasto/przypadek w Sphinx + MyST, publikowany automatycznie**
- **Zaobserwowane:** `docs/cases/{lyon,nantes,toulouse,corsica,bordeaux}.md` (120–190 linii) opisują dla każdego regionu według jednego szablonu:
  1. „Additional data”: skąd pobrać BD TOPO, OSM, GTFS i BAN, dokładne nazwy plików i katalogi docelowe;
  2. opcjonalną regionalną ankietę ze statusem dostępności;
  3. „Overview” z pełną listą oczekiwanych plików;
  4. fragmenty YAML do „Generating the population” i „Running the simulation”.
  `docs/cases/cases_summary.md` podaje ogólny proces i szablon configu dla nowego regionu. Pozostałe katalogi to `docs/population/`, `docs/simulation/`, `docs/misc/` (changelog, roadmap, legal) i `docs/containers/`. Budowa: `.github/workflows/sphinx.yml` publikuje na GitHub Pages, jest też `.readthedocs.yaml`.
- **Dryf *(zaobserwowane)*:** `lyon.md` wciąż odsyła do „Anaconda environment” i BD TOPO 2022; `population_execution.md:57` wspomina `mamba` obok `uv run`; `bordeaux.md` linkuje nieistniejące `../population.md` i brak go w toctree.
- **Czy dotyczy nas?** tak. Szablon „dodatkowe dane → lista oczekiwanych plików → fragment configu” dobrze pasuje do `docs/dane/`. Pokazuje też, że dokumentacja per przypadek szybko się rozjeżdża z kodem, jeśli nic jej nie sprawdza (np. check linków w CI albo generowanie listy plików z `download.py`).

---

## Najważniejsze dla nas

1. **Syntetyczne fixture'y danych zastrzeżonych** (`tests/testdata.py`) mają schemat oryginału, są generowane od zera z ustalonymi seedami i obejmują niepubliczną EGT. Pozwalają testować ścieżki KBR/ZDiUM w publicznym CI bez łamania licencji.
2. **Wybór źródła danych przez alias w DAG** (`data/hts/selected.py`) plus wspólny kontrakt `check()` (`data/hts/hts.py:245`) i twardy błąd dla niejednoznacznego wariantu (`edgt_lyon_source`). To bezpośredni wzorzec dla KBR 2018 vs 2024.
3. **Test determinizmu ze złotymi hashami** na trzech OS-ach, z normalizacją formatów (gzip, GPKG) po stronie zarówno testu, jak i zapisu. Przenieść na nasz artefakt popytu, i to od razu dla *wszystkich* plików.
4. **`meta.json` z commitem, wersją, seedem i próbką oraz JAR dołączony do scenariusza** to dobry początek proweniencji. Brakuje dirty flag, wersji Javy/JAR-a i odcisków wejść, więc my musimy to uzupełnić.
5. **Runtime Java jako stage `synpp`**: klon + checkout commita + Maven, ścieżka JAR-a jako wynik stage'u, cienki `java.run`. Wzorzec dobry, ale budowanie Mavenem u każdego jest drogie; lepiej przypięty gotowy JAR.
6. **release-please + konwencjonalne tytuły PR + `version.txt`**. Uwaga na dryf: `pyproject.toml` (1.4.0) nie jest bumpowany razem z resztą (1.5.0).

## Nie znaleziono

- Eksportu do SUMO ani odniesień do SUMO (`grep -ri sumo` w całym repo).
- Mechanizmu spójności ID sieci z innym narzędziem (`matsim/scenario/supply/`).
- Licencji per katalog, flag „restricted” ani technicznej separacji danych niepublicznych (`LICENSE`, `docs/misc/legal.md`, `scripts/download.py`).
- Walidacji schematu biblioteką (pandera, JSON Schema); są tylko asercje (`data/hts/hts.py`).
- Zapisu wersji `eqasim-java`/`pt2matsim` ani odcisków danych wejściowych w `meta.json` (`documentation/meta_output.py`).
- DOI oprogramowania (Zenodo) w `CITATION.cff`.
- Historii git: klon ma 1 commit, więc ewolucji konfiguracji i układu nie dało się prześledzić.
- `config_bordeaux.yml` mimo istnienia `docs/cases/bordeaux.md`.
- Implementacji skalowania przepustowości: jest po stronie `eqasim-java` (`RunGenerateConfig --sample-size`), poza tym repo.

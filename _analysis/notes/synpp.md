# synpp

- **Repo:** `synpp` — https://github.com/eqasim-org/synpp
- **Commit:** `7b4f70b199fcc58b9a1343f250e6369e7d7eda88` (z `MANIFEST.tsv`: `full`, `ok`, 2026-08-11), gałąź `develop`, wersja pakietu `1.6.1` (`setup.py:11`).
- **Licencja:** MIT (`LICENSE`, `setup.py:27`) — w odróżnieniu od `eqasim-*`/`matsim-*` nie jest GPL/AGPL.
- **Częściowość klonu:** pliki kompletne (manifest: `full`), ale **historia git jest płytka** (`git rev-parse --is-shallow-repository` → `true`, jeden commit w `git log`). Częstotliwości commitów ani liczby autorów nie da się ustalić z tego klonu (patrz „Nie znaleziono”).

**Czym jest to repo.** Mała (≈1,5 tys. linii Pythona: `src/synpp/pipeline.py` 1149, `parallel.py` 165, `progress.py` 144) biblioteka do łączenia „stage'ów” (etapów) w DAG z cache'owaniem wyników na dysku. Stage to moduł/klasa/funkcja z `configure` (deklaruje wymagane opcje configu i stage'e nadrzędne), `execute` (liczy wynik, zwracany i pickle'owany) i opcjonalnym `validate` (zwraca token; zmiana tokenu unieważnia cache). Uruchamiana z YAML przez `python -m synpp config.yml` albo z Pythona (`synpp.run`, klasa `Synpp`). To silnik, na którym stoją pipeline'y `eqasim-*` (populacja syntetyczna → scenariusz MATSim); samo repo nie zawiera nic z MATSim/SUMO.

Twierdzenia o zachowaniu zweryfikowałem, czytając kod oraz **uruchamiając eksperymenty na kopii repo w scratchpadzie** (niczego nie zmieniałem w `/home/dominik/refs`). Testy z repo: 56/57 przechodzi (uruchomione bez `pytest`, którego nie ma w środowisku, własnym prostym runnerem; jedyny błąd — `test_downstream_externals.py::test_without_externals` — wynika z tego, że mój runner uruchamia testy alfabetycznie, a `synpp` trzyma moduły w globalnym `sys.modules`, więc kolejność testów ma znaczenie).

---

### 1. Orkiestracja

**Stage = obiekt z `configure`/`execute`/`validate`, rozwiązywany z nazwy modułu**
- **Zaobserwowane:** `resolve_stage` (`src/synpp/pipeline.py:118-157`) przyjmuje string (import modułu lub `moduł.atrybut`), moduł, klasę, funkcję z dekoratorem `@synpp.stage` albo dowolny obiekt z `execute`. `StageInstance` (`:66-88`) wymaga `execute`, `configure`/`validate` są opcjonalne. Kompaktowa forma `@synpp.stage` (`:1068-1087`, `DecoratedStage` `:1100-1149`) mapuje argumenty funkcji na opcje configu lub stage'e; README zaznacza, że forma kompaktowa nie wspiera własnego `validate` (sekcja „Compact stage definition”) — w kodzie `DecoratedStage` rzeczywiście nie ma `validate`.
- **Dlaczego:** zależności deklarowane w kodzie stage'a (`context.stage(...)` w `configure`), a nie w centralnym pliku — graf buduje się sam przez eksplorację od żądanych stage'ów (README, „Running a pipeline”).
- **Czy dotyczy nas?** częściowo — to jeden z porównywanych stylów; wygodny, ale graf jest „ukryty” w kodzie i poznawalny dopiero po `configure` wszystkich stage'ów (jest `dryrun`/`flowchart_path`, `:661-672`).

**Klucz cache = nazwa stage'a + MD5 z *wymaganego* configu (bez opcji `volatile`); kod NIE wchodzi do klucza**
- **Zaobserwowane:** `hash_name` (`:216-222`) liczy MD5 z `json.dumps(config bez kluczy volatile, sort_keys=True)`; gdy stage nie wymaga żadnej opcji, klucz to sama nazwa (np. `pkg.c` — bez sufiksu). Do configu stage'a trafiają (a) opcje, o które sam prosi w `configure` (`ConfigurationContext.config`, `:265-280`), (b) opcje przekazane jawnie w `context.stage(name, {...})`, (c) **niejawnie** opcje odziedziczone po stage'ach nadrzędnych (propagacja „w dół”, `:530-567`), z pominięciem volatile (`:550`). Pliki w katalogu roboczym: `<klucz>.p` (pickle wyniku) i `<klucz>.cache/` (katalog na pliki stage'a) — `:685-686`, `:842-861`. Kod stage'a jest hashowany osobno jako `module_hash` = MD5 z `inspect.getsource(...)` (`:91-95`) i zapisywany tylko w metadanych `pipeline.json` (`:872`), nie w nazwie pliku.
- **Dlaczego:** różne parametryzacje tego samego stage'a współistnieją w jednym katalogu roboczym; zmiana kodu nadpisuje tę samą ścieżkę zamiast mnożyć wpisy *(wnioskowane)*.
- **Czy dotyczy nas?** tak — kluczowe dla decyzji, czy `synpp` może być naszym orkiestratorem popytu; patrz ryzyka niżej.

**Kolejność unieważniania (`run`, `:716-800`)**
- **Zaobserwowane:** stage jest „stale”, gdy: 4.1 jest żądany w `run` i `rerun_required=True` (domyślne, także z CLI — `:1001-1010`); 4.2 brak go w `pipeline.json`; 4.4 zmienił się `module_hash`; 4.5 brak `.p` lub `.cache/`; 4.6 rodzic ma nowszy znacznik `updated` (czas zegarowy UTC, `:866`); 4.7 zmienił się zbiór kluczy rodziców; 4.8 `validate()` zwrócił inny token niż zapisany; 4.9 wszyscy potomkowie stale'a (`nx.descendants`); 4.10 efemeryczni rodzice stale'a są odtwarzani. Punkt 4.3 („zmiana configu”) jest pusty z komentarzem, że jest zbędny, bo config jest w kluczu (`:729-730`). Wykonanie: `nx.topological_sort`, **sekwencyjnie**, jeden stage na raz (`:821-899`).
- **Dlaczego:** README: „in almost any case of pipeline modification we end up in a consistent situation (though we cannot prove it)” i zaleca „by convention” zawsze uruchamiać stage po zmianie kodu.
- **Czy dotyczy nas?** tak — to wprost mówi, że gwarancja spójności jest „best effort”.

**Wykrywanie zmiany kodu obejmuje tylko źródło samego stage'a — sprawdzone eksperymentalnie**
- **Zaobserwowane:** `get_stage_hash` hashuje `inspect.getsource` modułu (dla stage'a-modułu: cały plik), klasy (tylko ciało klasy) albo funkcji (tylko ciało funkcji) — `:135-154`. Eksperyment: stage `pkg.a` importuje `pkg.helper`; po zmianie stałej w `helper.py` ponowne uruchomienie zwróciło **stary wynik** (`B(X1*1)`), a `pkg.a` nie był stale. Dodanie samego komentarza do `a.py` unieważniło `a` i wszystkich potomków (hash liczy tekst, nie semantykę). Zmiana wersji bibliotek (pandas, geopandas), samego `synpp` czy JAR-a wywoływanego z `execute` nie jest wykrywana.
- **Dlaczego:** tanie przybliżenie „czy kod się zmienił” bez analizy importów *(wnioskowane)*.
- **Czy dotyczy nas?** tak — przy naszym układzie `src/<pakiet>/` z helperami w osobnych modułach zmiana w `utils.py` nie unieważni cache; 15 osób pracujących na wspólnym cache dostanie cicho stare wyniki.

**Zmiana pliku wejściowego nie jest wykrywana, chyba że stage napisze `validate()`**
- **Zaobserwowane:** eksperyment: zmiana treści `in.txt` czytanego w `execute` → brak unieważnienia, stary wynik. Mechanizmem jest wyłącznie token z `validate` (`:773-783`, zapis `:851`, `:871`); README pokazuje przykład z rozmiarem pliku, fixture `tests/fixtures/devalidation/token_a.py` zwraca całą treść pliku jako token. `ValidateContext` daje tylko `config` i `path()` (`:301-313`).
- **Dlaczego:** `synpp` nie wie, które pliki czyta stage; odpowiedzialność przerzucona na autora stage'a.
- **Czy dotyczy nas?** tak — nasze dane `raw/` (KBR, OD, ZDiUM) leżą na wspólnym dysku i mogą być podmieniane; bez obowiązkowej konwencji `validate` (np. hash/mtime pliku) cache jej nie zauważy.

**Błąd: niepełna propagacja configu w diamentach → kolizja kluczy i CICHO BŁĘDNY wynik**
- **Zaobserwowane:** propagacja odziedziczonego configu idzie od źródeł w dół, ale z każdego stage'a kolejkuje **tylko pierwszego** potomka: `pending.append(stage["downstream"][0]["hash"])` (`:566-567`). Eksperyment na grafie `a(seed) → b`, `a → c`, `b,c → top` (b i c same nie czytają configu): klucz `c` nie dostaje sufiksu (`pkg.c`), a przy uruchomieniu w jednym `run` dwóch wariantów `top` z `seed=1` i `seed=2` wynik dla `seed=1` to `B(a1)C(a2)` — `c` z innego scenariusza. Rejestr zwija stage'e po kluczu (`:589-596`), więc oba warianty `c` stają się jednym. Przy uruchamianiu scenariuszy po kolei w jednym katalogu wynik jest poprawny (ratuje go reguła 4.7), ale `pkg.c.p` jest nadpisywany przy każdej zmianie scenariusza (thrashing). Łatka w kopii scratchpadowej (kolejkowanie wszystkich potomków) daje poprawne `C(a1)` i osobne klucze `pkg.c__…`. Istniejące testy tego nie łapią (`tests/test_downstream.py` testuje łańcuch, nie diament z dwoma potomkami).
- **Dlaczego:** nie dotyczy (błąd implementacji).
- **Czy dotyczy nas?** tak — jeśli użyjemy `synpp` i będziemy liczyć kilka wariantów (KBR 2018 vs 2024, różne próbki) w jednym przebiegu/katalogu, możemy dostać niepoprawne wyniki bez żadnego błędu.

**`ephemeral` — pośrednie wyniki kasowane po użyciu; zmiana ich kodu może zostać niezauważona**
- **Zaobserwowane:** `context.stage(name, ephemeral=True)` (`:282-292`); stage jest efemeryczny tylko, jeśli wszyscy żądający go tak oznaczą (`:584-586`). Po wykonaniu ostatniego konsumenta jego `.p` i `.cache/` są usuwane (`:878-892`), ale wpis w `pipeline.json` zostaje. Efemeryczne bez cache są wyłączone ze sprawdzeń 4.2–4.8 (`sorted_cached_hashes`, `:717`). Eksperyment: `e`(ephemeral) → `f` → `g`; po zmianie kodu `e` uruchomienie `g` zwróciło **stary** `G(F(E1))`, bo `f` był w cache, a `e` nie był sprawdzany. Dopiero bezpośrednie żądanie `f` przeliczyło `F(E2)`.
- **Dlaczego:** oszczędność dysku na dużych pośrednich tabelach (CHANGELOG 1.2.0: „use as much caching as possible”).
- **Czy dotyczy nas?** częściowo — przydatne przy dużych plikach pośrednich, ale łączy się z ryzykiem nieaktualnych wyników.

**Częściowe uruchomienia**
- **Zaobserwowane:** brak `--skip-to`/`--from`; „częściowość” wynika z cache. CLI `--run stage` zastępuje listę `run` z YAML (`:939-941`, `:1025`). Żądane stage'e zawsze są przeliczane (`rerun_required=True`); `Synpp.run_single(..., rerun_if_cached=False)` pozwala wczytać je z cache (`:1012-1016`, `:901-907`). Nie ma polecenia „wymuś przeliczenie X” ani „wyczyść cache” — trzeba ręcznie usuwać pliki lub zmienić kod.
- **Dlaczego:** model „Make przez hashe” — przelicz tylko to, co się zmieniło.
- **Czy dotyczy nas?** częściowo.

**Równoległość tylko wewnątrz stage'a**
- **Zaobserwowane:** stage'e wykonują się sekwencyjnie; `context.parallel(data, processes)` daje pulę `multiprocessing` (`forkserver`, fallback `spawn`) — `parallel.py:50-65`, `pipeline.py:373-385`; liczba procesów z top-level `processes` w configu. Proces potomny nie ma dostępu do stage'ów (`parallel.py:26-30`). Drobny błąd: `parallel.py:16,112` rzucają `PipelineError`, którego plik nie importuje (NameError zamiast czytelnego błędu).
- **Dlaczego:** prostsze przekazywanie danych do workerów niż surowe `multiprocessing` (README, „Parallel execution”).
- **Czy dotyczy nas?** częściowo.

**Obsługa błędów i współbieżność**
- **Zaobserwowane:** brak `try/except` wokół `execute`; wyjątek przerywa cały przebieg. `pipeline.json` jest zapisywany po każdym udanym stage'u (`:875-876`, atomowo przez `.new` + `move`, z `.bk` — `:604-611`), więc udane etapy zostają w cache. Eksperyment: stage rzucający wyjątek zostawia katalog `.cache/` z częściowymi plikami, ale bez `.p` i bez wpisu w meta → przy kolejnym uruchomieniu katalog jest kasowany i stage liczony od nowa (`:845-847`). Brak blokady pliku (`grep lock` w `src/` — brak): dwa równoległe przebiegi na tym samym katalogu roboczym ścigają się o `pipeline.json`. Reguła 4.6 porównuje czasy zegarowe maszyny.
- **Dlaczego:** narzędzie projektowane dla jednego użytkownika i jednego katalogu roboczego *(wnioskowane)*.
- **Czy dotyczy nas?** tak — wspólny cache na wspólnym dysku dla wielu studentów jest wykluczony bez zewnętrznej blokady.

**Format przechowywania: pickle**
- **Zaobserwowane:** `pickle.dump(result, protocol=4)` (`:859-861`), odczyt `pickle.load` (`:346-348`, `:905-907`). README: wynik jest „pickled … and cached on the hard drive”. Pliki zewnętrzne (CSV, XML, JAR-owe wyjścia) stage pisze do `context.path()`; inne stage'e czytają je przez `context.path("inny.stage")` (`:352-360`), bez egzekwowania zakazu zapisu do cudzego katalogu (README: „by convention”).
- **Dlaczego:** dowolne obiekty Pythona (DataFrame) bez projektowania formatu.
- **Czy dotyczy nas?** tak — pickle zależy od wersji pandas/geopandas (cache może się nie wczytać po aktualizacji lub wczytać się inaczej), nie nadaje się jako artefakt do wymiany, a `pickle.load` z zapisywalnego wspólnego dysku wykonuje dowolny kod.

### 2. Konfiguracja i wiele scenariuszy

**Jeden YAML: `working_directory`, `run`, `config`, `aliases`, `externals`**
- **Zaobserwowane:** `Synpp.build_from_yml` (`:1018-1065`); przykład `example/config.yml`. Opcje hierarchiczne adresowane kropką (`flatten`/`unflatten`, `:38-60`; `get_config_value` zwraca poddrzewo, `:177-195`). Stage'e w `run` mogą mieć własne parametry (`- stage: {param: 1}`, `:1026-1036`).
- **Dlaczego:** jeden plik opisuje cały przebieg (README, „Command-line tool”).
- **Czy dotyczy nas?** tak — wzorzec „config.yml per wariant” pasuje do KBR 2018/2024.

**Nadpisywanie z CLI z rzutowaniem typu**
- **Zaobserwowane:** `--config key value`, `-c:int|float|json` (`:942-971`); jeśli klucz już istnieje, wartość jest rzutowana na typ istniejącej: `type(current[option])(value)` (`:1051-1052`). Dla opcji typu `bool` daje to `bool("false") == True` (sprawdzone w Pythonie: rzutowanie niepustego stringa).
- **Dlaczego:** CHANGELOG 1.6.0 „override config values to the command line interface”.
- **Czy dotyczy nas?** częściowo — pułapka dla flag boolowskich.

**Warianty = parametryzowane stage'e w jednym katalogu roboczym; `volatile` dla opcji nieistotnych**
- **Zaobserwowane:** ten sam stage z innym configiem to inny klucz; oba współistnieją. `config(..., volatile=True)` wyklucza opcję z klucza i propagacji (`:277-278`, `:219`, `:550`); README: dla liczby rdzeni/pamięci. `aliases` podmieniają stage na inny (`:119-120`), `externals` mapują nazwę modułu na plik spoza pakietu (`:105-106`) — w teście podmieniają implementację stage'a (`tests/test_downstream_externals.py`). Nie ma pojęcia „miasto”/„scenariusz” w samym `synpp`.
- **Dlaczego:** CHANGELOG 1.5.1 („drop-in replacement for existing stages”), 1.6.0 (`volatile`).
- **Czy dotyczy nas?** częściowo — `aliases`/`externals` to sposób na wariant „Wrocław” bez forka pipeline'u, ale patrz błąd diamentowy w wymiarze 1.

### 3. Układ danych i duże pliki

**Katalog roboczy = płaski katalog `<klucz>.p` + `<klucz>.cache/` + `pipeline.json`(`.bk`)**
- **Zaobserwowane:** potwierdzone eksperymentem (`ls wd`). `.gitignore` repo ignoruje `.synpp_cache` (domyślna nazwa, `:622-623`). Brak rozróżnienia raw/interim/processed; wyjście końcowe to to, co stage zapisze gdzie chce (np. przez ścieżkę z configu). Przykład pobiera dane z URL w stage'u (`example/nyc_taxi/download.py`).
- **Dlaczego:** nie dotyczy (brak wyjaśnienia w repo).
- **Czy dotyczy nas?** częściowo — katalog roboczy `synpp` odpowiadałby naszemu `interim/`, ale z kluczami-hashami zamiast czytelnych nazw.

### 4. Proweniencja i wersjonowanie

**Metadane per stage bez commita**
- **Zaobserwowane:** `pipeline.json` trzyma per klucz: `config`, `updated`, `dependencies`, `info` (`set_info`), `validation_token`, `module_hash` (`:864-873`). Brak hasha commita, wersji pakietów i wersji `synpp`. Repo ma `CHANGELOG.md` (ręczny, sekcje per wersja). Brak `CITATION.cff`, brak automatyzacji release'ów w `.github/workflows/` (jest tylko `tests.yml`).
- **Dlaczego:** nie dotyczy.
- **Czy dotyczy nas?** częściowo — `set_info`/`get_info` (`:362-371`) to gotowy kanał na statystyki do raportu, ale proweniencję artefaktu musimy dodać sami.

### 5. Źródło prawdy dla sieci
Nie dotyczy (biblioteka ogólna; przeszukano `src/`, `example/`).

### 6. Przekazanie MATSim → SUMO
Nie dotyczy (brak kodu MATSim/SUMO w repo).

### 7. Kalibracja między narzędziami
Nie dotyczy.

### 8. Próbka / skalowanie
Nie dotyczy (brak pojęć próbki; `synpp` nie obsługuje seedów losowych — to robi kod stage'ów).

### 9. Testy i determinizm

**Testy jednostkowe na mikro-fixture'ach stage'ów**
- **Zaobserwowane:** `tests/` — 14 plików, ~57 testów; fixture'y to 2–6-liniowe moduły-stage'e (`tests/fixtures/**`). Pokrycie: unieważnianie przez config, config przekazany, rodzica, potomków, token, zmianę pliku z kodem (`tests/test_devalidate.py:4-266`), `volatile`, efemeryczność (`tests/test_ephemeral.py`), aliasy, externals, równoległość, progress, cykle, `dryrun`/flowchart, test szybkości na losowym grafie (`tests/test_speed.py:72`). CI: `.github/workflows/tests.yml` — ubuntu + windows, Python 3.10, tylko push/PR na `develop`. README ma nieaktualną plakietkę Travis.
- **Brakuje (zaobserwowane):** testu diamentu z dwoma potomkami bez własnego configu (błąd z wymiaru 1), testu zmiany modułu pomocniczego, testu zmiany kodu efemerycznego rodzica przy zcache'owanym dziecku, testu determinizmu wyników.
- **Dlaczego:** testują mechanikę grafu, nie domenę.
- **Czy dotyczy nas?** tak — wzorzec „mikro-stage'e jako fixture'y do testu orkiestracji” jest tani; jeśli przyjmiemy `synpp`, musimy dopisać własne testy regresyjne na powyższe przypadki.

### 10. Dane zastrzeżone i licencje
Nie dotyczy dla danych. Licencja kodu: MIT (`LICENSE`) — można go używać/vendorować bez ograniczeń GPL.

### 11. Runtime Python ↔ Java
Nie dotyczy — `synpp` nie zna Javy; wywołanie JAR-ów to sprawa stage'ów (eqasim). Zmiana JAR-a nie wpływa na `module_hash`.

### 12. Pakietowanie

**Mała biblioteka `src/`-layout z `setup.py`, trzy zależności**
- **Zaobserwowane:** `setup.py` (setuptools, `package_dir={"": "src"}`), brak `pyproject.toml`. `install_requires`: `networkx>=2.4`, `PyYAML>=5.1.2`, `pyzmq>=18.1.0` (`setup.py:22`); `requirements.txt` podnosi do `networkx>=3.4` (`node_link_data(..., edges=...)` w `:669` wymaga nowego networkx). `pyzmq` służy tylko do raportowania postępu z procesów potomnych (`progress.py`). `python_requires='>=3.0'`. Entry point tylko jako `python -m synpp` (`__main__.py`), bez `console_scripts`.
- **Dlaczego:** nie dotyczy.
- **Czy dotyczy nas?** tak — jako zależność zewnętrzna w workspace `uv` wchodzi bez problemu; nasze stage'e to zwykłe moduły w pakietach workspace'u, a `synpp` jest tylko „runnerem”. Brak typów (`mypy` na naszym kodzie wywołującym `context.*` nic nie da, bo `Context` nie ma adnotacji).

### 13. Dokumentacja
**README jako jedyna dokumentacja**
- **Zaobserwowane:** `README.md` opisuje koncepcje, reguły unieważniania, `validate`, cache paths, aliasy, równoległość, CLI i przykład NYC taxi (`example/`). Brak osobnych docs.
- **Czy dotyczy nas?** częściowo — lista reguł unieważniania z README jest dobrym wzorem do opisania naszej orkiestracji w `docs/decyzje/`.

---

## Najważniejsze dla nas

1. **Cache NIE jest wrażliwy na zmiany w modułach pomocniczych, bibliotekach, JAR-ach ani plikach wejściowych** (sprawdzone eksperymentem) — wykrywa tylko zmianę źródła samego stage'a (`pipeline.py:91-95`, `:732-743`) i token z `validate()`. Przy 15 osobach to przepis na ciche, nieaktualne wyniki; gdybyśmy użyli `synpp`, obowiązkowa konwencja: każdy stage czytający `raw/` implementuje `validate` (hash pliku), a „stale cache” czyścimy przy zmianie wersji zależności.
2. **Potwierdzony błąd w propagacji configu (`pipeline.py:566-567`)**: w grafie typu diament stage bez własnego configu, będący nie-pierwszym potomkiem, dostaje klucz bez hasha configu → dwa scenariusze w jednym przebiegu współdzielą jego wynik (`B(a1)C(a2)` zamiast `C(a1)`). Dotyczy dokładnie naszego przypadku „kilka wariantów w jednym katalogu roboczym”.
3. **Zmiana kodu stage'a `ephemeral` bywa ignorowana**, gdy jego konsument jest w cache (eksperyment `G(F(E1))`).
4. **Pickle + brak blokady + zegar ścienny** → katalog roboczy `synpp` musi być lokalny per osoba, nie na wspólnym dysku; artefakt popytu eksportujemy osobnym stage'em w jawnym formacie (nie `.p`), z własnymi metadanymi proweniencji (commit, wersje).
5. Warte przejęcia niezależnie od narzędzia: deklarowanie zależności i wymaganych opcji w `configure`, hash z *wymaganych* opcji (a nie całego configu), `volatile` dla opcji technicznych, `set_info` na statystyki do raportu, `validate` jako token danych zewnętrznych.
6. Utrzymanie: jeden plik rdzenia, licencja MIT, ostatni commit 2026-08-11 (Sebastian Hörl); historii nie da się ocenić z płytkiego klonu. Przy tak małym kodzie ewentualny fork/vendoring z poprawkami jest realny.

## Nie znaleziono

- **Częstotliwość commitów i liczba autorów** — klon płytki (1 commit); `git log`, `git shortlog -sn --all` zwracają tylko `7b4f70b` (Sebastian Hörl, 2026-08-11, „feat: cleaner command line configuration (#95)”). `setup.py` wskazuje jednego autora; numer PR `#95` sugeruje ≥95 PR/issue w historii *(wnioskowane)*.
- **Hashowania zależności importowanych przez stage / wersji pakietów / pliku wejściowego z automatu** — szukano w `pipeline.py` (`get_stage_hash`, `hash_name`, `calculate_identification_hash`), brak.
- **Blokady plików / obsługi współbieżnych przebiegów** — `grep -n lock src/` pusty.
- **Polecenia wymuszenia przeliczenia lub czyszczenia cache** — brak w `run_from_cmd` (`:928-985`).
- **Testów determinizmu, walidacji schematów wyników** — brak w `tests/`.
- **Formatu innego niż pickle** — brak (`pickle` jedyny serializator w `src/`).

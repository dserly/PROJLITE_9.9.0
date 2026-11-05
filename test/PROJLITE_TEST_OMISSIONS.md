# PROJLITE: részletes tesztkimaradási jelentés

Állapot: 2026-11-05, Windows, MSVC, Release, megosztott PROJLITE-könyvtár.

## 1. Mit jelentenek a számok?

| Fogalom                     | Jelentés                                                                                                                                             |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| CTest-bejegyzés             | A CMake `add_test()` által regisztrált futtatás. Egy GIE-fájl vagy egy teljes GTest-program is csak egy ilyen bejegyzés.                             |
| Mért GIE-eset               | A GIE fájlonkénti `total: ... tests succeeded, ... tests skipped` számlálójának egysége. Nem egyenlő a koordináták vagy a pipeline-lépések számával. |
| Mért GTest-eset             | A ténylegesen lefuttatott GTest által jelentett teszt.                                                                                               |
| Geodéziai ellenőrzés        | A `geodtest` és a `geodsigntest` belső összehasonlítása. Nem önálló GTest-eset.                                                                      |
| Forrásbeli GTest-deklaráció | A `TEST`, `TEST_F`, illetve `TEST_P` makrók statikus száma. A feltételes fordítás miatt ez nem bizonyított futtatható esetszám.                      |
| Forrásbeli GIE-parancs      | Az aktív, nem kommentelt `expect` és `roundtrip` sorok statikus száma. Nem mért futási eredmény.                                                     |
| Nem regisztrált teszt       | A CMake feltétele miatt a CTest listájába sem kerül be. A CTest nem jelenti automatikusan kihagyottként.                                             |

**A mért skipet, a forrásbeli deklarációkat és a geodéziai ellenőrzéseket nem szabad egyetlen testcase-összegként összeadni.**

### Ellenőrzött konfiguráció

| Beállítás                     | Érték                        | Következmény                                                                                           |
| ----------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------ |
| `PROJLITE`                    | `ON`                         | Nincs SQLite/`proj.db`, teljes ISO 19111 API és CRS-műveletválasztás.                                  |
| `BUILD_TESTING`               | `ON`                         | Tesztek engedélyezve.                                                                                  |
| `BUILD_SHARED_LIBS`           | `ON`                         | MSVC DLL-build; a `pj_phi2_test` nem regisztrálódik.                                                   |
| `BUILD_APPS`                  | `OFF`                        | A kapcsolódó CLI-programok tesztjei nem regisztrálódnak.                                               |
| `BUILD_GIE`                   | `OFF`                        | A GIE runner a tesztbuildhez ettől még megépült, de a külön `test_gie.yaml` CLI-teszt feltétele hamis. |
| `ENABLE_TIFF`                 | `OFF`                        | A TIFF-függő GIE-bejegyzések nem regisztrálódnak.                                                      |
| `ENABLE_CURL`                 | `OFF`                        | A CURL-függő GIE-bejegyzések nem regisztrálódnak.                                                      |
| `TESTING_USE_NETWORK`         | `ON`                         | A GTest letölthető; a függőség már rendelkezésre áll.                                                  |
| `RUN_NETWORK_DEPENDENT_TESTS` | `ON`                         | Önmagában nem elég: a TIFF/CURL-feltételek továbbra is hiányoznak.                                     |
| Futási kizárás                | `-E "^pytest_run_cli_test$"` | Egy regisztrált CTest-bejegyzés nem indul el.                                                          |

### Mért összesítés

| Kategória                         | Lefutott/sikeres |            Kihagyott |        Hibás | Bizonyíték                                                     |
| --------------------------------- | ---------------: | -------------------: | -----------: | -------------------------------------------------------------- |
| Regisztrált CTest-bejegyzések     |               46 |                    1 |            0 | Összesen 47 regisztrált bejegyzés; a pytest kizárva.           |
| GIE belső esetek, 37 fájl         |             6570 |                  803 |            0 | A fájlonkénti összesítések összege.                            |
| GTest belső esetek, 7 program     |               33 |                    0 |            0 | A GTest-programok futási összesítése.                          |
| Geodéziai ellenőrzések, 2 program |              991 |                    0 |            0 | 813 + 178 ellenőrzés.                                          |
| Kizárt pytest belső esetei        |                0 | 10 nem indított eset | Nem futtatva | `python -m pytest --collect-only -q test/cli/run_cli_test.py`. |

A 37 GIE-fájlból **22-ben minden számlált eset skipet kapott**. Ezek együtt 795 kihagyást jelentenek; három további, részben lefutott fájlban összesen 8 eset maradt ki. A 22 fájl CTest-szinten mégis sikeres, mert a runner nem talált hibás esetet.

## 2. Teljesen kihagyott GIGS-fájlok

Mind a 20 alábbi fájlban három `operation` blokk van. Mind a 60 blokk tartalmaz `+init=epsg:...` hivatkozást. A [GIE Lite-feltétele](../src/apps/gie.cpp#L668) ezért az egész érintett műveletet kihagyja, és az utána következő `expect` parancsokat skipként számolja.

**Nem a táblázatban szereplő vetületmatematika bizonyult használhatatlannak.** Az EPSG-definíció feloldása hiányzik. Ellenőrzött, explicit PROJ-definíciókkal a matematikai tesztek egy része vagy egésze visszanyerhető lehet, de az már nem ugyanannak az authority-feloldási útvonalnak a tesztje lenne.

| CTest-név és forrás                                  | Tesztelt terület                      | EPSG-hivatkozások | Mért skip | Miért nem fut?                                                                         |
| ---------------------------------------------------- | ------------------------------------- | ----------------- | --------: | -------------------------------------------------------------------------------------- |
| [GIGS-5101.1-jhs](gigs/5101.1-jhs.gie)               | Transverse Mercator, JHS, 1. rész     | 4326              |       118 | EPSG-init a pipeline-ban. Az `etmerc` lépés explicit, de az egész pipeline skipet kap. |
| [GIGS-5101.2-jhs](gigs/5101.2-jhs.gie)               | Transverse Mercator, JHS, 2. rész     | 32631, 4326       |        46 | A földrajzi és/vagy vetületi definíció adatbázisos feloldást igényel.                  |
| [GIGS-5101.3-jhs](gigs/5101.3-jhs.gie)               | Transverse Mercator, JHS, 3. rész     | 28354, 4283       |        46 | EPSG-init lépések.                                                                     |
| [GIGS-5101.4-jhs-etmerc](gigs/5101.4-jhs-etmerc.gie) | Transverse Mercator, JHS, 4. rész     | 4190              |        46 | EPSG-init a pipeline-ban.                                                              |
| [GIGS-5102.1](gigs/5102.1.gie)                       | Lambert Conic Conformal, 1SP          | 2192, 4230        |        38 | EPSG-init lépések.                                                                     |
| [GIGS-5102.2](gigs/5102.2.gie)                       | Lambert Conic Conformal, 1SP, 2. rész | 27572, 4807       |        38 | EPSG-init lépések.                                                                     |
| [GIGS-5103.1](gigs/5103.1.gie)                       | Lambert Conic Conformal, 2SP, 1. rész | 31370, 4313       |        40 | EPSG-init lépések.                                                                     |
| [GIGS-5103.2](gigs/5103.2.gie)                       | Lambert Conic Conformal, 2SP, 2. rész | 2921, 4152        |        20 | EPSG-init lépések.                                                                     |
| [GIGS-5103.3](gigs/5103.3.gie)                       | Lambert Conic Conformal, 2SP, 3. rész | 3568, 4152        |        20 | EPSG-init lépések.                                                                     |
| [GIGS-5104](gigs/5104.gie)                           | Oblique Stereographic                 | 28992, 4289       |        40 | EPSG-init lépések.                                                                     |
| [GIGS-5105.2](gigs/5105.2.gie)                       | Oblique Mercator, B változat          | 23700, 4237       |        24 | EPSG-init lépések.                                                                     |
| [GIGS-5106](gigs/5106.gie)                           | Hotine Oblique Mercator, A változat   | 3376, 4742        |        46 | EPSG-init lépések.                                                                     |
| [GIGS-5107](gigs/5107.gie)                           | American Polyconic                    | 4674              |        26 | EPSG-init a pipeline-ban.                                                              |
| [GIGS-5108](gigs/5108.gie)                           | Cassini-Soldner                       | 3377, 4742        |        34 | EPSG-init lépések.                                                                     |
| [GIGS-5109](gigs/5109.gie)                           | Albers Equal Area                     | 3577, 4283        |        26 | EPSG-init lépések.                                                                     |
| [GIGS-5111.1](gigs/5111.1.gie)                       | Mercator, A változat                  | 3001, 4211        |        70 | EPSG-init lépések.                                                                     |
| [GIGS-5112](gigs/5112.gie)                           | Mercator, B változat                  | 3388, 4284        |        10 | EPSG-init lépések.                                                                     |
| [GIGS-5113](gigs/5113.gie)                           | Transverse Mercator, South Oriented   | 2049, 4148        |        10 | EPSG-init lépések.                                                                     |
| [GIGS-5201](gigs/5201.gie)                           | Földrajzi/geocentrikus konverzió      | 4326, 4978        |        54 | A CRS-definíciókat EPSG-init adja meg.                                                 |
| [GIGS-5208](gigs/5208.gie)                           | Kezdőmeridián/longitude rotation      | 4275, 4807        |        28 | Mindkét oldal EPSG-init hivatkozást használ.                                           |
| **Összesen**                                         | **20 fájl, 60 műveletblokk**          |                   |   **780** | **Mindegyik fájlban 0 sikeres belső eset.**                                            |

## 3. Másik két teljesen kihagyott GIE-fájl

### EPSG CRS-műveletválasztás

Forrás: [gie/epsg_no_grid.gie](gie/epsg_no_grid.gie). A `no_grid` elnevezés csak azt jelenti, hogy nem kell rácsfájl; **nem** jelenti azt, hogy nincs szükség `proj.db`-re.

| Eset         | Bemenet/cél                         | Mi marad ellenőrizetlenül?                                        | Skip oka                                       |
| ------------ | ----------------------------------- | ----------------------------------------------------------------- | ---------------------------------------------- |
| 1            | EPSG:4258 -> EPSG:25832             | ETRS89 -> UTM32, CRS-tengelyrend és műveletkiválasztás.           | `crs_src` / `crs_dst` nem támogatott Lite-ban. |
| 2            | EPSG:4979 -> EPSG:4978              | WGS84 földrajzi 3D -> geocentrikus konverzió a CRS-API útvonalán. | Ugyanaz.                                       |
| 3            | Ugyanazon CRS-pár másik ellenőrzése | A földrajzi/geocentrikus értékpár ellenőrzése.                    | Ugyanaz.                                       |
| 4            | EPSG:7843 -> EPSG:7912, 2020.0      | GDA2020/ITRF2014 időfüggő transzformáció a referenciaepochán.     | Ugyanaz.                                       |
| 5            | EPSG:7843 -> EPSG:7912, 2026.0      | Ugyanazon transzformáció időfüggése más epochán.                  | Ugyanaz.                                       |
| 6            | EPSG:4123 -> EPSG:2393              | Finn KKJ/YKJ konverzió és northing/easting tengelyrend.           | Ugyanaz.                                       |
| **Összesen** |                                     | **0 lefutott, 6 mért skip.**                                      |                                                |

Explicit pipeline-nal a matematika egy része tesztelhető lenne, de az eredeti CRS-factory viselkedését Lite nem biztosítja.

### SQLite GeoPackage TINShift

Forrás: [gie/tinshift_gpkg.gie](gie/tinshift_gpkg.gie). A Lite megőrzi a JSON TINShift támogatását, de a SQLite-os GeoPackage-olvasót nem.

| Művelet bemeneti fájlja                         | Mért skipnek megfelelő `expect` sor | Mi marad ellenőrizetlenül?                                  | Kizárás oka / szűkíthetőség                                                                                             |
| ----------------------------------------------- | ----------------------------------: | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `i_do_not_exist.gpkg`                           |                                   1 | Nem létező GPKG megnyitásának hibakódja.                    | A `.gpkg` bemenetet a runner előre kihagyja. Negatív teszt: nem bizonyított, hogy feltétlenül egészében ki kell hagyni. |
| `tests/tinshift_empty_file.gpkg`                |                                   1 | Üres/érvénytelen GPKG hibakódja.                            | Ugyanaz a kiterjesztésalapú skip; a hibaút külön vizsgálattal megtartható lehet.                                        |
| `tests/tinshift_crs_implicit.gpkg`              |                                   3 | Implicit CRS, sikeres konverzió és két hibás bemeneti eset. | Tényleges GPKG-olvasás és kapcsolódó adatok szükségesek.                                                                |
| `tests/tinshift_simplified_kkj_etrs.gpkg`       |                                   1 | Vízszintes KKJ/ETRS TINShift.                               | SQLite GeoPackage-olvasó hiányzik.                                                                                      |
| `tests/tinshift_simplified_n60_n2000.gpkg`      |                                   1 | N60/N2000 magassági TINShift.                               | Ugyanaz.                                                                                                                |
| `tests/tinshift_fallback_nearest_side.gpkg`     |                                   1 | Legközelebbi oldal alapú fallback.                          | Ugyanaz.                                                                                                                |
| `tests/tinshift_fallback_nearest_centroid.gpkg` |                                   1 | Legközelebbi centroid alapú fallback.                       | Ugyanaz.                                                                                                                |
| **Összesen**                                    |                               **9** | **0 lefutott, 9 mért skip.**                                |                                                                                                                         |

## 4. Részben lefutott GIE-fájlok

| Fájl                                                              | Sikeres / kihagyott        | Pontosan mi maradt ki?                                                                      | Tényleges ok                                                 | Értékelés                                                                                                                 |
| ----------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| [gie/more_builtins.gie](gie/more_builtins.gie#L194)               | 181 / 2                    | EPSG:25832 és EPSG:25833 init lépésekből álló pipeline két ellenőrzése.                     | EPSG authority lookup.                                       | A többi művelet fut, beleértve a helyi init-definíciókat; nincs egész fájlos kizárás.                                     |
| [gie/ellipsoid.gie](gie/ellipsoid.gie#L75)                        | 36 / 1                     | `operation cobra`, utána `expect failure`.                                                  | Nem üres bemenet, de nincs benne `proj=` vagy `init=`.       | Negatív parser-teszt, nem adatbázis-függő matematikai eset. A jelenlegi skip-feltétel szélesebb a DB-függőség szűrésénél. |
| [gie/4D-API_cs2cs-style.gie](gie/4D-API_cs2cs-style.gie#L206)     | 73 / 5 összesen            | EPSG:26915 -> EPSG:3857 pipeline: 2 eset; EPSG:26915 -> explicit `webmerc`: további 2 eset. | A pipeline-okban maradt EPSG-init.                           | Explicit forrásdefinícióval átdolgozható lehet.                                                                           |
| [Ugyanazon 4D-fájl parser-esete](gie/4D-API_cs2cs-style.gie#L539) | Az előző sor 5 skipjéből 1 | Szándékosan hibás szabad szöveges CRS, elvárt `invalid_op_wrong_syntax`.                    | A `proj=`/`init=` nélküli bemenetet a runner előre kihagyja. | Negatív parser-teszt; Lite-specifikus hibaelvárással vizsgálandó, nem automatikusan DB-függő.                             |
| **Összesen**                                                      | **290 / 8**                |                                                                                             |                                                              | **A 4D-fájlt csak egyszer szabad számolni.**                                                                              |

GIE-kihagyások ellenőrző összege: **780 + 6 + 9 + 2 + 1 + 5 = 803**.

## 5. Kézzel kizárt pytest: 10 gyűjtött eset

A [CLI harness](cli/run_cli_test.py) `pytest_run_cli_test` nevű CTest-bejegyzése a futási `-E` szűrő miatt nem indul el. Az alábbi lista `--collect-only` eredmény, nem új pytest-futtatás.

| Gyűjtött pytest-eset                | Aktuális státusz | Miért nem fut?                          |
| ----------------------------------- | ---------------- | --------------------------------------- |
| `test_sub_env_var`                  | Nem indult el    | A teljes szülő CTest-bejegyzés kizárva. |
| `test_eval_skipif`                  | Nem indult el    | Ugyanaz.                                |
| `test_tmpdir`                       | Nem indult el    | Ugyanaz.                                |
| `test_file`                         | Nem indult el    | Ugyanaz.                                |
| `test_tester_from_yaml`             | Nem indult el    | Ugyanaz.                                |
| `test_tester_from_yaml_with_update` | Nem indult el    | Ugyanaz.                                |
| `test_test`                         | Nem indult el    | Ugyanaz.                                |
| `test_test_tmpdir`                  | Nem indult el    | Ugyanaz.                                |
| `test_test_file`                    | Nem indult el    | Ugyanaz.                                |
| `test_out_cmds`                     | Nem indult el    | Ugyanaz.                                |

A kizárás háttere a korábbi Windows-futtatás: POSIX shellparancsokra és eltérő útvonalkezelésre épülő esetek hibáztak. **Ez nem jelenti azt, hogy mind a 10 pytest-eset hibás vagy Windows alatt eleve futtathatatlan.** A kizárás jelenleg teljes harness-szintű, ezért működő eseteket is eltakar.

## 6. PROJLITE miatt nem létrehozott unit targetek

Feltételek és forráslista: [unit/CMakeLists.txt](unit/CMakeLists.txt#L152).

| Target              | Forrásbeli GTest-deklaráció | Közvetlen CMake-ok                                                                                       | Mi marad ki?                                                                                         | Fontos korlát                                                                                                                             |
| ------------------- | --------------------------: | -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `proj_test_cpp_api` |                        1419 | Teljes target `if(NOT PROJLITE)` alatt; közvetlen `SQLite3::SQLite3` link.                               | ISO 19111 C++ API, CRS, datum, WKT/JSON I/O, factory, műveletválasztás, C API és kapcsolódó tesztek. | Nem bizonyított, hogy mind az 1419 deklaráció külön-külön adatbázis-függő; monolit target-szintű kizárás.                                 |
| `gie_self_tests`    |                          20 | Teljes target `if(NOT PROJLITE)` alatt.                                                                  | Core önellenőrzések és CRS/ISO API-t használó esetek egyaránt.                                       | A fájl vegyes: pl. `cart_selftest`, unitconvert és Horner esetek is kimaradnak. További szétválasztás lehetséges, de még nincs validálva. |
| `test_network`      |                          26 | Teljes target `if(NOT PROJLITE)` alatt; közvetlen SQLite-link. CTest-regisztrációja emellett TIFF-függő. | Hálózati/grid/cache viselkedés és a SQLite-os cache tesztjei.                                        | Egyszer kell számolni, akkor is, ha a Lite- és a TIFF-feltétel is kizárja.                                                                |
| `test_tmerczoned`   |                           8 | Teljes target `if(NOT PROJLITE)` alatt.                                                                  | EPSG:32600/32700 zónahatárok, földrajzi szélesség/hosszúság határesetei.                             | Közös segédje `proj_create_crs_to_crs()`-t hív EPSG CRS-ekkel; ez nem Lite API.                                                           |
| **Összesen**        |         **1473 deklaráció** | **4 teljes target**                                                                                      |                                                                                                      | **Nem mért runtime-skip szám.**                                                                                                           |

### A monolit C++ API target részletezése

| Forrás                                                           | Statikus deklaráció | Terület                                                  |
| ---------------------------------------------------------------- | ------------------: | -------------------------------------------------------- |
| [unit/test_util.cpp](unit/test_util.cpp)                         |                   2 | Segédobjektumok/util API.                                |
| [unit/test_common.cpp](unit/test_common.cpp)                     |                  13 | Common API, mennyiségek/mértékegységek.                  |
| [unit/test_coordinates.cpp](unit/test_coordinates.cpp)           |                   3 | Koordináta-rendszer objektumok.                          |
| [unit/test_crs.cpp](unit/test_crs.cpp)                           |                 197 | CRS-objektumok és kapcsolódó működés.                    |
| [unit/test_metadata.cpp](unit/test_metadata.cpp)                 |                  12 | Metaadat API.                                            |
| [unit/test_io.cpp](unit/test_io.cpp)                             |                 454 | WKT/PROJ/JSON beolvasás és kiírás.                       |
| [unit/test_operation.cpp](unit/test_operation.cpp)               |                 162 | Koordinátaművelet-objektumok.                            |
| [unit/test_operationfactory.cpp](unit/test_operationfactory.cpp) |                 274 | Koordinátaművelet-keresés és -választás.                 |
| [unit/test_datum.cpp](unit/test_datum.cpp)                       |                  22 | Datum/ellipszoid objektumok.                             |
| [unit/test_factory.cpp](unit/test_factory.cpp)                   |                 113 | Authority/database factory.                              |
| [unit/test_c_api.cpp](unit/test_c_api.cpp)                       |                 152 | Teljes C API tesztjei.                                   |
| [unit/test_grids.cpp](unit/test_grids.cpp)                       |                  10 | Gridhez kapcsolódó API.                                  |
| [unit/test_projinfo_lib.cpp](unit/test_projinfo_lib.cpp)         |                   5 | Projinfo library működés.                                |
| **Összesen**                                                     |            **1419** | **Az előző táblázat részletezése; nem újabb 1419 eset.** |

A statikus számlálás a forrásban talált makródeklarációkat méri. Például a CRS-tesztekben feltételesen letiltott blokk is van; a teljes build által ténylegesen gyűjtött GTest-esetszámot ebből nem szabad pontosnak állítani.

## 7. TIFF/CURL miatt nem regisztrált GIE-tesztek

Regisztrációs feltételek: [CMakeLists.txt](CMakeLists.txt#L82). A mostani cache-ben a hálózati futási opció `ON`; a hiányzó képesség itt a TIFF és/vagy CURL.

| CTest-név / forrás                                     | Hiányzó feltétel                      |                    Forrásbeli `expect` + `roundtrip` | Mi marad ki?                                                                                    |
| ------------------------------------------------------ | ------------------------------------- | ---------------------------------------------------: | ----------------------------------------------------------------------------------------------- |
| [Deformation](gie/deformation.gie)                     | `TIFF_ENABLED`                        |                                         26 + 17 = 43 | Rácsalapú deformációs műveletek ellenőrzése.                                                    |
| [geotiff_grids](gie/geotiff_grids.gie)                 | `TIFF_ENABLED`                        |                                          50 + 1 = 51 | GeoTIFF gridolvasás és interpoláció.                                                            |
| [defmodel](gie/defmodel.gie)                           | `TIFF_ENABLED`                        |                                         23 + 11 = 34 | A deformációs modell teljes GIE-integrációja. A külön JSON/model unit teszt ettől még lefutott. |
| [gridshift](gie/gridshift.gie)                         | `TIFF_ENABLED`                        |                                         35 + 24 = 59 | Általános gridshift integráció.                                                                 |
| [gridshift_network_enabled](gie/gridshift.gie)         | `TIFF_ENABLED`                        |                 Ugyanaz a 59 parancs, külön futtatás | Ugyanazon fájl `PROJ_NETWORK=ON` környezetben.                                                  |
| [tinshift_gpkg_network](gie/tinshift_gpkg_network.gie) | `CURL_ENABLED` és hálózati futás      |                                            3 + 2 = 5 | Hálózati GeoPackage TINShift. A Lite SQLite-hiánya további akadály.                             |
| [nkg](gie/nkg.gie)                                     | TIFF, CURL és hálózati futás          |                                          33 + 0 = 33 | NKG-transzformációk/rácsok integrációja.                                                        |
| [epsg_grid](gie/epsg_grid.gie)                         | TIFF, CURL és hálózati futás          |                                            7 + 0 = 7 | EPSG-alapú, rácsot igénylő CRS-műveletek. A Lite CRS-factory hiánya további akadály.            |
| **Összesen**                                           | **8 nem regisztrált CTest-bejegyzés** | **291 parancselőfordulás a tervezett futtatásokban** | **Statikus leltár, nem 291 bizonyított runtime-testcase.**                                      |

A képesség bekapcsolása önmagában nem garantálja az összes eset futását: adatfájl, hálózat, SQLite vagy teljes CRS-API is kellhet. Ezért ezek a számok nem részei a mért 803 skipnek.

## 8. Nem regisztrált CLI-tesztek

Feltételek: [cli/CMakeLists.txt](cli/CMakeLists.txt). Python/YAML rendelkezésre áll; itt a CLI-programok buildfeltételei hiányoznak. A belső YAML-esetszámokat ez a jelentés nem mérte fel.

| Meglévő tesztforrás                                            | Hiányzó buildfeltétel                     | Mi marad ki?                                                                                |
| -------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------- |
| [cli/test_cct.yaml](cli/test_cct.yaml)                         | `BUILD_CCT`                               | A `cct` parancssori viselkedése.                                                            |
| [cli/test_cs2cs_datumfile.yaml](cli/test_cs2cs_datumfile.yaml) | `BUILD_CS2CS`                             | Datumfájlok kezelése a `cs2cs` CLI-ben.                                                     |
| [cli/test_cs2cs_flaky.yaml](cli/test_cs2cs_flaky.yaml)         | `BUILD_CS2CS`                             | A külön flaky cs2cs tesztkészlet.                                                           |
| [cli/test_cs2cs_ignf.yaml](cli/test_cs2cs_ignf.yaml)           | `BUILD_CS2CS`                             | IGNF-hez kapcsolódó cs2cs esetek.                                                           |
| [cli/test_cs2cs_ntv2.yaml](cli/test_cs2cs_ntv2.yaml)           | `BUILD_CS2CS`                             | NTv2-rácsos cs2cs esetek.                                                                   |
| [cli/test_cs2cs_various.yaml](cli/test_cs2cs_various.yaml)     | `BUILD_CS2CS`                             | Vegyes cs2cs CLI-esetek.                                                                    |
| [cli/test_proj.yaml](cli/test_proj.yaml)                       | `BUILD_PROJ`                              | A `proj` CLI alapképességei.                                                                |
| [cli/test_proj_nad27.yaml](cli/test_proj_nad27.yaml)           | `BUILD_PROJ`                              | NAD27-alapú CLI-esetek.                                                                     |
| [cli/test_proj_nad83.yaml](cli/test_proj_nad83.yaml)           | `BUILD_PROJ`                              | NAD83-alapú CLI-esetek.                                                                     |
| [cli/test_invproj.yaml](cli/test_invproj.yaml)                 | `BUILD_PROJ`                              | Az `invproj` CLI.                                                                           |
| [cli/test_projinfo.yaml](cli/test_projinfo.yaml)               | `BUILD_PROJINFO`                          | A `projinfo` CLI.                                                                           |
| [cli/test_gie.yaml](cli/test_gie.yaml)                         | `BUILD_GIE`                               | Magának a GIE parancssori interfészének tesztje; nem azonos a 37 futtatott GIE-adatfájllal. |
| **Összesen**                                                   | **12 nem regisztrált YAML/CLI-bejegyzés** | **Belső esetszám nincs mérve.**                                                             |

## 9. Platformfeltételek és nem CTest-célok

| Teszt/cél                                                                | Miért marad ki?                                                                                 | Darabszám jellege                              | Megjegyzés                                                                                                      |
| ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| [unit/pj_phi2_test.cpp](unit/pj_phi2_test.cpp)                           | `MSVC AND BUILD_SHARED_LIBS` esetén a CMake nem építi; a forrás szerinti ok DLL-kompatibilitás. | 1 target, 1 statikus GTest-deklaráció.         | Nem PROJLITE-specifikus kizárás.                                                                                |
| [unit/test_fork.c](unit/test_fork.c)                                     | POSIX pthread/fork platformfeltétele nem teljesül Windows alatt.                                | 1 target; nem GTest-program.                   | Adatbázis-elérést tesztel fork után, ezért Lite-on más platformon sem tekinthető automatikusan kompatibilisnek. |
| [cli/test_cs2cs_locale.sh](cli/test_cs2cs_locale.sh)                     | `UNIX` és `BUILD_CS2CS` kell.                                                                   | 1 nem regisztrált shellteszt.                  | Windows és a CLI-buildfeltétel egyaránt kizárja.                                                                |
| [cli/test_projinfo.sh](cli/test_projinfo.sh)                             | `UNIX` és `BUILD_PROJINFO` kell.                                                                | 1 nem regisztrált shellteszt.                  | Ugyanaz a két kizárási kategória.                                                                               |
| [cli/test_projsync.sh](cli/test_projsync.sh)                             | `UNIX` és `BUILD_PROJSYNC` kell.                                                                | 1 nem regisztrált shellteszt.                  | Ugyanaz a két kizárási kategória.                                                                               |
| [benchmark/bench_proj_trans.cpp](benchmark/bench_proj_trans.cpp)         | A benchmark almappát Lite kihagyja.                                                             | Benchmarkprogram, nem regisztrált CTest-teszt. | A benchmark CMake nem tartalmaz `add_test()`-et; nem hiányzó testcase-ként számolandó.                          |
| [fuzzers/proj_crs_to_crs_fuzzer.cpp](fuzzers/proj_crs_to_crs_fuzzer.cpp) | A fuzzer almappát Lite kihagyja; a saját CMake-je Windows alatt is tiltja.                      | Fuzzerprogram, nem regisztrált CTest-teszt.    | Nincs véges, előre felsorolható GTest-esetszáma.                                                                |

## 10. Eleve kikommentelt upstream GIGS-regisztrációk

A [teszt-CMake](CMakeLists.txt#L99) tíz további GIGS-regisztrációt kommentben tartalmaz. Ezek nem az aktuális Lite-skip következményei. A megjegyzés általánosan ismert vetületi tesztproblémákra utal; az egyedi történeti hibák itt nem lettek újravizsgálva.

| Kikommentelt CTest-név | Hivatkozott adatfájl állapota ebben a checkoutban                        | Számolható belső esetszám? |
| ---------------------- | ------------------------------------------------------------------------ | -------------------------- |
| `GIGS-5101.4-jhs`      | Hiányzik. A külön `GIGS-5101.4-jhs-etmerc` változat viszont regisztrált. | Nem.                       |
| `GIGS-5105.1`          | Hiányzik.                                                                | Nem.                       |
| `GIGS-5110`            | Hiányzik.                                                                | Nem.                       |
| `GIGS-5111.2`          | Hiányzik.                                                                | Nem.                       |
| `GIGS-5203`            | Hiányzik.                                                                | Nem.                       |
| `GIGS-5204.1`          | Hiányzik.                                                                | Nem.                       |
| `GIGS-5205.1`          | Hiányzik.                                                                | Nem.                       |
| `GIGS-5206`            | Hiányzik.                                                                | Nem.                       |
| `GIGS-5207.1`          | Hiányzik.                                                                | Nem.                       |
| `GIGS-5207.2`          | Hiányzik.                                                                | Nem.                       |

Ezek kommentben szereplő nevek, nem további tíz létező, aktuálisan futtatható teszt.

## 11. Mely kizárások szűkíthetők tovább?

| Terület                                | Jelenlegi veszteség                                   | Lehetséges következő lépés                                                                           | Mit nem szabad feltételezni?                                                                               |
| -------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| GIGS explicit definíciók               | 780 mért skip                                         | Az EPSG-definíciók hiteles, ellenőrzött PROJ-paraméterekre bontása; összehasonlítás teljes PROJ-jal. | Nem garantált, hogy egyszerű szövegcserével minden tengelyrend, kezdőmeridián és mértékegység helyes lesz. |
| Szabad szöveges negatív parser-tesztek | 2 mért skip                                           | Lite hibaút és hibakód validálása; célzott elvárás.                                                  | Ezek nem szükségszerűen DB-függők.                                                                         |
| GPKG negatív tesztek                   | A 9 skipből 2 közvetlenül hibás/hiányzó fájlt tesztel | Vizsgálni, hogy a megfelelő hibaút SQLite nélkül ellenőrizhető-e.                                    | Nem szabad ezzel működő GPKG-olvasást állítani.                                                            |
| Vegyes `gie_self_tests` target         | 20 forrásbeli deklaráció                              | Core és CRS/ISO tesztek szétválasztása, majd build és futás.                                         | Nem mind a 20 eset szükségszerűen adatbázis-függő.                                                         |
| Monolit C++ API suite                  | 1419 forrásbeli deklaráció                            | A linkelhető, valóban Lite API-t vizsgáló részek célzott leválasztása.                               | A teljes suite sem elérhető, sem automatikusan értelmes nem lesz Lite-ban.                                 |
| Pytest harness                         | 10 gyűjtött, nem indított eset                        | Windows-kompatibilis harness vagy csak az érintett pytest-esetek kizárása.                           | A korábbi hibákból nem következik mind a 10 eset hibája.                                                   |

## 12. Reprodukálás és számlálási korlát

Részletes futás, a jelenlegi kizárással:

```powershell
ctest --test-dir build-projlite-tests -C Release -V -E "^pytest_run_cli_test$" --output-log build-projlite-tests/testcase-details.log
```

Regisztrált tesztek listája és a kizárt pytest gyűjtése:

```powershell
ctest --test-dir build-projlite-tests -C Release -N
python -m pytest --collect-only -q test/cli/run_cli_test.py
```

A GIE végső `Grand total` skip-száma jelenleg nem megbízható: a [process_file összesítője](../src/apps/gie.cpp#L461) a `grand_skip` változót önmagával növeli a fájl `total_skip` száma helyett. Ez a jelentés a fájlonkénti, helyes `total:` sorokat használja. A hibát a jelentés elkészítése nem módosította.

**Biztos, futásból ismert kihagyás:** 803 GIE-eset, továbbá a szűrő miatt el nem indított 10 gyűjtött pytest-eset. A nem regisztrált targetek belső testcase-einek teljes, runtime szerint hiteles száma külön teljes PROJ-referenciabuild és azonosított platform-/képességfeltételek nélkül nincs megmérve. A forrásbeli leltár ezt egészíti ki, nem helyettesíti.

# Dobro mjesto: objašnjenje projekta i priprema za prezentaciju

Skripta za predstavljanje na konferenciji Ministarstva pravosuđa, uprave i digitalne transformacije (Datathon Portala otvorenih podataka 2026., izazov B: Okoliš i održivi prostor).

Stanje koda: grana `main`, commit `cca9b14` (20. 11. 2026.). Sve putanje su relativne u odnosu na korijen repozitorija. Brojke o podacima pročitane su iz lokalne mape `data/processed/` (podaci preuzeti 2.–9. 11. 2026.). Evaluacija modela za zrak ponovno je pokrenuta pri pisanju ove skripte (`uv run python -m analysis.ml_air`) i dala je iste rezultate kao `docs/ML.md`.

Oznaka **[PROVJERITI]** znači da tvrdnju nije moguće potvrditi iz koda ili datoteka. Uz svaku piše što nedostaje.

---

## 1. Projekt u 5 rečenica

1. **Dobro mjesto** je web aplikacija koja za bilo koju adresu u Hrvatskoj složi izvještaj o okolišu i svakodnevnom životu: poplave, kvaliteta zraka, priroda i hlad, udaljenost od velikih prometnica, požari raslinja, sunčev potencijal krova te što je pješice dostupno u 5, 10 i 15 minuta.
2. Namijenjena je ljudima koji biraju gdje živjeti (kupci, najmoprimci, obitelji, umirovljenici, studenti), a preko otvorenog API-ja i gradovima, bankama i oglasnicima.
3. Rješava problem da su podaci o okolišu i uslugama rasuti po desetak institucija i formata (GeoTIFF, INSPIRE GML, Excel, GTFS) i neupotrebljivi za običnog čovjeka, pa se okolišni rizici i ovisnost o autu obično otkriju tek nakon useljenja.
4. Aplikacija povezuje 34 skupa otvorenih podataka (od toga 22 s data.gov.hr) i dva javna API-ja, a svaku brojku računa backend iz podataka i uz nju navodi izvor, datum i vrstu (izmjereno, procjena modela ili gruba procjena).
5. Uz izvještaj za adresu postoje karta svih 6.759 naselja, usporedba dviju adresa, vođeni tok „Pronađi mjesto za mene” koji rangira naselja prema profilu korisnika i asistent koji odgovara na pitanja samo iz izračunatih podataka.

---

## 2. Problem i ciljni korisnici

**Problem.** Pri kupnji ili najmu ljudi gledaju cijenu, kvadraturu i lokaciju. Okoliš (poplave, zrak, hlad, buka prometa) i dostupnost usluga bez auta (vrtić, škola, liječnik, ljekarna, autobus) obično otkriju tek kad usele. Podaci o tome postoje, ali su:

- rasuti po institucijama: DGU, DZS, Ministarstvo znanosti, HZZO, gradovi i općine, prijevoznici, europske agencije (EEA, JRC, ESA, Copernicus);
- u formatima koji traže GIS znanje: rasteri, INSPIRE GML od 2,6 GB, GTFS vozni redovi, Excel tablice bez koordinata;
- izraženi stručnim pojmovima („dubina vode za povratno razdoblje od 100 godina”, „µg/m³ PM2,5”).

**Ciljni korisnici** (README.md, odjeljak „Za koga”; `config/profiles.json`):

| Korisnik | Zašto bi ga koristio |
|---|---|
| Kupci, najmoprimci, oni koji razmišljaju o selidbi | jedna adresa, jedan izvještaj, prije potpisa ugovora |
| Obitelji s djecom (profil `obitelj`) | vrtić, škola, pedijatar i igralište pješice, uz zrak i poplave |
| Umirovljenici (profil `umirovljenik`) | trgovina, ljekarna i liječnik pješice, bolnica u blizini, hlad i javni prijevoz |
| Studenti (profil `student`) | studij u mjestu ili blizu, javni prijevoz, život bez auta |
| Rad od kuće (profil `rad-od-kuce`) | mirno mjesto, priroda, svakodnevne potrebe |
| Gradovi, agencije, banke, oglasnici | REST API i GeoJSON za ugradnju u vlastite usluge |
| Izdavači podataka | nalazi o kvaliteti njihovih skupova (`docs/KVALITETA_PODATAKA.md`) |

Profil `vlastiti` također postoji u `config/profiles.json`: korisnik sam namješta težine.

---

## 3. Struktura projekta

```
datathon_projekt/
├── README.md              opis rješenja za ocjenjivače (problem, funkcionalnosti, izvori, pokretanje)
├── CLAUDE.md              izvorni plan i pravila projekta za AI alat Claude Code (zadaci P0–P2, tvrda pravila)
├── PROGRESS.md            popis zadataka i stanje (napravljeno, odustano, za ljude)
├── PRODUCT.md, DESIGN.md  opis proizvoda i dizajnerskog sustava sučelja
├── LICENSE                EUPL-1.2 (licenca koda)
├── Makefile               naredbe: setup, data, valhalla, dev, test, lint, build, e2e, screenshots, verify
├── .env.example           predložak postavki (CONTACT_EMAIL, LLM_*, VALHALLA_URL, DOMAIN)
├── .gitignore, .gitattributes, .dockerignore   pravila za git i Docker (data/ i .env se ne commitaju)
├── .githooks/pre-push     blokira izravan push na main (promjene idu preko pull requesta)
├── .claude/, .impeccable/ postavke razvojnih AI alata (Claude Code, dodatak Impeccable za pregled dizajna); nisu dio aplikacije
│
├── backend/               Python 3.12 (uv)
│   ├── pyproject.toml     ovisnosti i postavke alata (ruff, mypy strict, pytest)
│   ├── uv.lock            zaključane točne verzije paketa
│   ├── ingest/            KONEKTORI IZVORA: preuzimanje i obrada (make data)
│   │   ├── run_all.py         redoslijed svih 16 koraka obrade
│   │   ├── common.py          pristojno preuzimanje, User-Agent, manifest.json
│   │   ├── dgu.py             adrese i upravne jedinice (INSPIRE GML)
│   │   ├── osm.py             OpenStreetMap: trgovine, igrališta, parkovi, ceste, stajališta
│   │   ├── centers.py         stvarna središta naselja (OSM čvorovi place=*)
│   │   ├── dzs.py             Popis 2021., stanovništvo po naseljima
│   │   ├── rasters.py         isijecanje rastera poplava, zraka i pokrova tla za Hrvatsku
│   │   ├── effis.py           opožarene površine (WFS)
│   │   ├── stations.py        službene mjerne postaje zraka (EEA), za provjeru i ML
│   │   ├── transit.py         vozni redovi (GTFS, JSON) u polaske na sat
│   │   ├── transit_lines.py   linije javnog prijevoza po stajalištima iz OSM relacija
│   │   ├── registries.py      vrtići, škole, ordinacije, ljekarne: geokodiranje nad DGU adresama
│   │   ├── recycling.py       općinski skupovi s Portala (CKAN) + OSM
│   │   ├── pois.py            spajanje svih točaka u jedan sloj, uklanjanje duplikata
│   │   ├── settlement_profiles.py   isti izračun kao izvještaj za središta svih 6.759 naselja
│   │   ├── facilities.py      visoka učilišta, studijski programi, bolnice, domovi, menze
│   │   ├── settlement_traffic.py    udaljenost svih adresa naselja od prometnica
│   │   ├── settlement_landcover.py  medijan prirode, drveća i zraka po adresama naselja
│   │   └── listing_locations.py     održavanje: UUID-ovi mjesta za Index Oglasi (nije dio make data)
│   ├── analysis/          IZRAČUNI kategorija
│   │   ├── environment.py     poplave, zrak, priroda i hlad, prometnice, požari (najveći modul)
│   │   ├── walking.py         kandidati u blizini + jedan poziv Valhalle za vremena hoda
│   │   ├── daily.py           svakodnevni život, javni prijevoz, život bez auta
│   │   ├── solar.py           sunce na krovu (PVGIS)
│   │   ├── score.py           bodovi 0–100 po kategoriji i ponderirani prosjek
│   │   ├── status.py          nazivi statusa (Dobro, Upozorenje, Loše…)
│   │   ├── ml_air.py          evaluacija strojnog učenja za zrak (ne koristi se u aplikaciji)
│   │   └── validation.py      usporedba karte zraka s mjernim postajama (ne koristi se u aplikaciji)
│   ├── app/               API (FastAPI)
│   │   ├── main.py            sve rute /api/*, zagrijavanje pri pokretanju
│   │   ├── report.py          slaganje izvještaja i sažetka iz predložaka, GeoJSON
│   │   ├── geocode.py         vlastita tražilica adresa
│   │   ├── layers.py          slojevi učitani u memoriju (točke, parkovi, ceste, požari)
│   │   ├── admin.py           naselje, općina, županija za točku
│   │   ├── address_grid.py    broj kućnih brojeva oko točke (za karticu prirode)
│   │   ├── valhalla.py        klijent usmjerivača (izokrone, matrica, rute)
│   │   ├── pvgis.py           klijent PVGIS-a s predmemorijom
│   │   ├── walk_route.py      pješački put na klik
│   │   ├── stop_lines.py      linije na stajalištima bez voznog reda
│   │   ├── assistant.py       asistent (pravila ili jezični model samo za namjeru)
│   │   ├── finder.py, finder_rules.py   „Predloži mjesto”: filtri, uvjeti profila, rang
│   │   ├── map_layer.py       podaci za kartu Hrvatske (granice, pokazatelji)
│   │   ├── live_sources.py    datumi zadnje izmjene skupova uživo s Portala (CKAN)
│   │   ├── sources.py         izvori iz manifest.json
│   │   ├── models.py          Pydantic modeli odgovora (svaka vrijednost: source, retrieved_at, kind)
│   │   ├── profiles.py, config.py, geo.py, fmt.py, brand.py   pomoćni moduli
│   └── tests/             pytest: 123 testne funkcije, fixtures/golden (zlatni testovi)
│
├── frontend/              React 19 + TypeScript (Vite)
│   ├── package.json, package-lock.json   ovisnosti
│   ├── vite.config.ts     razvojni poslužitelj na 5173, proxy /api na 8000
│   ├── src/pages/         stranice: HomePage, ReportPage, ComparePage, FindPage, MapPage, AboutPage, NotFoundPage
│   ├── src/components/    19 komponenti (pretraga, kartice kategorija, karte, asistent, ocjena mjesta…)
│   ├── src/lib/           API klijent, profili, izračun ocjene, rute, slojevi karte, poveznice na oglase
│   └── tests/             unit (Vitest, 67 testova), e2e (Playwright + axe), snimke zaslona
│
├── config/                ZAJEDNIČKO backendu i frontendu
│   ├── profiles.json      profili i težine kategorija (0–3)
│   ├── finder_rules.json  obavezni uvjeti i kriteriji profila za „Predloži mjesto”
│   ├── listings.json      kako se slažu poveznice na oglasnike
│   └── listing_locations.json   UUID-ovi mjesta za Index Oglasi
│
├── deploy/                Docker Compose (produkcija i razvoj), Caddyfile, Dockerfile.api, Dockerfile.web
├── docs/                  14 dokumenata (arhitektura, izvori, metodologija, ML, kvaliteta, odluke…) i images/
├── scripts/               check_readme.py, demo_addresses.json, geocode_check.py, upload_data.sh
└── data/                  NIJE u gitu: raw/ (2,6 GB preuzetih datoteka), processed/ (554 MB obrađenih)
```

Napomene:
- Nema Jupyter notebooka ni datoteke `requirements.txt`. Ovisnosti backenda su u `backend/pyproject.toml` i `backend/uv.lock`.
- Nema baze podataka. Obrađeni podaci su datoteke (Parquet i GeoTIFF) koje API pri pokretanju učita u memoriju.
- Cache datoteke (`__pycache__`, `node_modules`, `dist`, `test-results`) nisu u gitu i nisu dio rješenja.

---

## 4. Skupovi podataka

Oznaka **[Portal]** znači da je skup objavljen na data.gov.hr. Gdje kod preuzima datoteku s drugog URL-a od stranice na Portalu, navedena su oba.

| # | Naziv skupa | Izdavatelj | Izvor / URL | Format | Gdje se učitava | Polja koja se koriste | Zašto nam treba |
|---|---|---|---|---|---|---|---|
| 1 | Adrese INSPIRE **[Portal]** | Državna geodetska uprava | data.gov.hr/ckan/dataset/adrese-inspire; preuzimanje: geoportal.dgu.hr/services/atom/INSPIRE_Addresses_(AD).zip | INSPIRE GML u ZIP-u (2,6 GB) | `backend/ingest/dgu.py`, `addresses()` | položaj (`gml:pos`), kućni broj (`LocatorDesignator`), ulica (`ThoroughfareName`), naselje i šifra naselja (`AdminUnitName`), poštanski broj | tražilica adresa; geokodiranje registara; medijani po naseljima |
| 2 | Upravne jedinice INSPIRE **[Portal]** | Državna geodetska uprava | data.gov.hr/ckan/dataset/upravne-jedinice-inspire; geoportal.dgu.hr/…/INSPIRE_Administrative_Units_(AU).zip | INSPIRE GML | `dgu.py`, `admin_units()` | `nationalCode`, `LocalisedCharacterString` (razina), `text` (naziv), geometrija | naselje, općina i županija za točku; granice naselja na karti |
| 3 | Popis 2021., stanovništvo po naseljima **[Portal]** | Državni zavod za statistiku | data.gov.hr/ckan/dataset/statistika-u-nizu-popis-stanovnistva-kucanstava-i-stanova-2021; podaci.dzs.hr/…/popis_2021-stanovnistvo_po_naseljima.xlsx | XLSX | `backend/ingest/dzs.py`, `run()` | županija, grad/općina, naselje, spol (`sv.`), ukupno | broj stanovnika; filtar „najmanje 200 stanovnika” u prijedlozima |
| 4 | Registar dječjih vrtića **[Portal]** | Ministarstvo znanosti, obrazovanja i mladih | data.gov.hr/ckan/dataset/djecji-vrtici | CSV | `backend/ingest/registries.py`, `kindergartens()` | `Naziv`, `Adresa`, `Mjesto`, `PostanskiBroj` | najbliži vrtić (geokodiran, jer registar nema koordinate) |
| 5 | Osnovne škole **[Portal]** | Ministarstvo znanosti, obrazovanja i mladih | data.gov.hr/ckan/dataset/osnovne-skole | CSV | `registries.py`, `schools()` | `Naziv`, `Adresa`, `Mjesto`, `PoštanskiBroj` | najbliža osnovna škola |
| 6 | Ustanove iz sustava visokog obrazovanja **[Portal]** | Ministarstvo znanosti, obrazovanja i mladih | data.gov.hr/ckan/dataset/ustanove-iz-sustava-visokog-obrazovanja | JSON | `backend/ingest/facilities.py` | naziv i mjesto ustanove | profil Student u „Predloži mjesto” |
| 7 | Studijski programi visokih učilišta **[Portal]** | Ministarstvo znanosti, obrazovanja i mladih | data.gov.hr/ckan/dataset/studijski-programi-visokih-ucilista | JSON | `facilities.py` | vrsta programa, mjesto izvođenja (`IzvodjacMjesto`) | broj preddiplomskih i diplomskih programa po mjestu |
| 8 | Geoportal: dječji vrtići Grada Zagreba **[Portal]** | Grad Zagreb | data.gov.hr/ckan/dataset/geoportal-djecji-vrtici; preuzimanje: opendata.arcgis.com (CSV) | CSV s koordinatama | `registries.py`, `kindergartens()` | `naziv`, `adresa`, `X`, `Y` | vrtići u Zagrebu s točnim koordinatama (umjesto registra MZOM-a) |
| 9 | Geoportal: osnovne škole Grada Zagreba **[Portal]** | Grad Zagreb | data.gov.hr/ckan/dataset/geoportal-osnovne-skole; preuzimanje: opendata.arcgis.com (GeoJSON) | GeoJSON | `registries.py`, `schools()` | `naziv`, `adresa`, geometrija | matične i područne škole u Zagrebu |
| 10–20 | 11 općinskih skupova o reciklažnim dvorištima i zelenim otocima **[Portal]** (Zagreb, Cernik, Dugopolje, Kloštar Podravski, Hvar, Bistra, Mikleuš, Velika Ludina, Kalinovac, Posedarje, Sopje) | gradovi i općine | pronalaze se pretragom CKAN API-ja (`data.gov.hr/ckan/api/3/action/package_search`) | CSV, JSON, XLSX (različito) | `backend/ingest/recycling.py`, `run()` | stupci koordinata (traže se po nazivima: longitude, lon, x, geo_duzina…) i naziv | mjesta za reciklažu |
| 21 | HŽPP vozni red (GTFS) **[Portal]** | HŽ Putnički prijevoz d.o.o. | data.gov.hr/ckan/dataset/vozni-red-h-putni-kog-prijevoza-u-gtfs-obliku; preuzimanje: www.hzpp.hr/GTFS_files.zip | GTFS (ZIP s CSV) | `backend/ingest/transit.py`, `gtfs_departures()` | `trips`, `routes`, `stops`, `stop_times`, `calendar` | polasci vlakova na sat |
| 22 | Vozni redovi Autotroleja (radni dan) **[Portal]** | Grad Rijeka | data.gov.hr/ckan/dataset/at-vozni-redovi; preuzimanje: e-usluge2.rijeka.hr/OpenData/ATvoznired-tjedan.json | JSON | `transit.py`, `autotrolej_departures()` | `PolazakId`, `StanicaId`, `Polazak`, `RedniBrojStanice`, `GpsX`, `GpsY`, `Naziv` | polasci autobusa u Rijeci |
| 23 | ZET vozni red (GTFS) | Zagrebački električni tramvaj d.o.o. | zet.hr/gtfs-scheduled/latest (stranica: zet.hr/odredbe/datoteke-u-gtfs-formatu/669) | GTFS | `transit.py` | isto kao HŽPP | polasci tramvaja i autobusa u Zagrebu |
| 24 | Gradski parking Šibenik, vozni red (GTFS) | Gradski parking d.o.o. (Grad Šibenik) | gradski-parking.hr/…/gtfs.zip | GTFS | `transit.py` | isto kao HŽPP | polasci autobusa u Šibeniku |
| 25–27 | HZZO: ugovoreni timovi opće medicine, pedijatrije i ugovorene ljekarne | Hrvatski zavod za zdravstveno osiguranje | hzzo.hr/sites/default/files/web_opca_092026_0.xlsx, web_ped_092026_0.xlsx, „Popis ugovorenih ljekarni u RH_1.xls” | XLSX, XLS | `registries.py`, `_hzzo_teams()`, `doctors()`, `pharmacies()` | mjesto, ulica, kućni broj (stupci 10–12); za ljekarne `Ulica`, `Kućni broj`, `HPT broj`, `HPT naziv` | najbliži liječnik, pedijatar, ljekarna; **imena liječnika se ne koriste** |
| 28 | JRC karte opasnosti od riječnih poplava v3.1.1 | Europska komisija, JRC | jeodpp.jrc.ec.europa.eu/…/CEMS-EFAS/flood_hazard/Europe_RP{10,100,500}_filled_depth.tif | GeoTIFF | `backend/ingest/rasters.py`, `flood()`; čita `analysis/environment.py`, `read_flood()` | dubina vode u metrima | poplave |
| 29–30 | EEA interpolirane karte kvalitete zraka 2024. (PM2,5, NO2), 1 km | Europska agencija za okoliš | sdi.eea.europa.eu/…/pm25_avg_24.tif, no2_avg_24.tif | GeoTIFF | `rasters.py`, `air()`; `environment.py`, `read_air()` | godišnji prosjek µg/m³ | zrak |
| 31 | Službene mjerne postaje zraka, 2024. (E1a) | Ministarstvo zaštite okoliša i zelene tranzicije, preko EEA-e | EEA Air Quality Download Service (API) | Parquet + CSV metapodaci | `backend/ingest/stations.py` | vrijednosti, valjanost, šifra i tip postaje, koordinate | provjera karte zraka i ML evaluacija (ne prikazuje se u izvještaju) |
| 32 | ESA WorldCover 10 m 2021 v200 | Europska svemirska agencija | esa-worldcover.s3…/ESA_WorldCover_10m_2021_v200_{6 pločica}_Map.tif | GeoTIFF (COG) | `rasters.py`, `worldcover()`; `environment.py`, `landcover_pixels()` | razred pokrova tla (10 drveće, 20 grmlje, 30 travnjaci, 40 oranice, 50 izgrađeno…) | priroda, drveće, izgrađenost |
| 33 | EFFIS opožarene površine (MODIS) | Europska komisija, Copernicus EFFIS | maps.effis.emergency.copernicus.eu/effis (WFS, sloj `ms:modis.ba.poly`, filtar COUNTRY=HR) | GeoJSON | `backend/ingest/effis.py` | `FIREDATE`, `AREA_HA`, `COMMUNE`, geometrija | požari raslinja u krugu od 2 km |
| 34 | OpenStreetMap, izvadak za Hrvatsku | suradnici OpenStreetMapa (Geofabrik) | download.geofabrik.de/europe/croatia-latest.osm.pbf | OSM PBF | `osm.py`, `centers.py`, `transit_lines.py`, `facilities.py`; i Valhalla | `amenity`, `shop`, `leisure`, `highway`, `railway`, `ref`, `place`, relacije `route` | mreža hoda, trgovine, igrališta, parkovi, ceste, stajališta, linije, središta naselja |

**Uživo, bez preuzimanja:**

| Servis | Izdavatelj | Gdje u kodu | Za što |
|---|---|---|---|
| JRC PVGIS 5.3 API (`re.jrc.ec.europa.eu/api/v5_3/PVcalc`) | Europska komisija, JRC | `backend/app/pvgis.py` | sunce na krovu, kWh po kWp godišnje |
| CKAN API Portala (`data.gov.hr/ckan/api/3/action/package_search`) | Portal otvorenih podataka | `backend/app/live_sources.py`, `backend/ingest/recycling.py` | datumi zadnje izmjene 22 skupa s Portala; pretraga općinskih skupova |

**Samo poveznice (ne izvode se podaci):** karte opasnosti od poplava Hrvatskih voda (licenca nije navedena) i strateške karte buke EEA-e (licenca nije otvorena), `analysis/environment.py`, konstante `HV_VIEWER` i `EEA_NOISE_VIEWER`.

**Licence** su zapisane u `docs/IZVORI_PODATAKA.md` i u `data/processed/manifest.json` (funkcija `record_source()` u `backend/ingest/common.py`). Otvorena dozvola RH: DGU, DZS, MZOM, ZET, Šibenik, općinski skupovi. CC BY 4.0: EEA, ESA, EFFIS, Autotrolej, vrtići Zagreba. ODbL: OpenStreetMap.

- **[PROVJERITI]** Licenca HZZO-ovih popisa: u kodu piše „nije navedena; javno objavljen popis tijela javne vlasti”. Izdavač licencu ne navodi, pa za službenu upotrebu treba pisana potvrda HZZO-a.
- **[PROVJERITI]** Licenca HŽPP GTFS-a: „nije navedena; skup je objavljen na data.gov.hr radi izrade aplikacija”. Vrijedi isto kao za HZZO.
- **[PROVJERITI]** Licenca JRC karata poplava: katalog JRC-a navodi ODbL 1.0, a datoteka `copyright.txt` CC BY 4.0. Izvor sam sebi proturječi.
- **[PROVJERITI]** GTFS Šibenika: kod i `docs/ODLUKE.md` (9. 11. 2026.) navode Otvorenu dozvolu RH, a `PROGRESS.md` pod „Odustano” još piše „bez navedene licence i bez objave na Portalu”. Treba provjeriti stranicu Gradskog parkinga i uskladiti `PROGRESS.md`.
- **[PROVJERITI]** Vrtići i škole Zagreba preuzimaju se s `opendata.arcgis.com`, a u manifestu je poveznica na stranicu skupa na data.gov.hr. Treba potvrditi da je to isti skup, tj. da poveznica s Portala vodi na taj ArcGIS resurs.

---

## 5. Obrada podataka korak po korak (pipeline)

Sve pokreće `make data`, odnosno `uv run python -m ingest.run_all` u mapi `backend/`. Datoteka `backend/ingest/run_all.py` (lista `STEPS`, retci 28–45) izvodi 16 koraka redom. Ako jedan korak padne, ispiše grešku i nastavlja s ostalima (`main()`, retci 48–64).

### 5.1. Učitavanje (preuzimanje)

**Što:** svaki konektor preuzima svoj izvor u `data/raw/<izvor>/`.

**Kako:** `download()` u `backend/ingest/common.py`:
- ne preuzima ponovno ako datoteka već postoji;
- između dva zahtjeva istom poslužitelju čeka barem 1 s (`_throttle()`);
- nastavlja prekinuto preuzimanje (HTTP zaglavlje `Range`);
- pokušava do 5 puta s rastućim čekanjem (`5 · 2^pokušaj` s).

Svaki zahtjev nosi User-Agent `PrijePotpisa/0.1 (+CONTACT_EMAIL)` (`backend/app/config.py`, svojstvo `user_agent`). Nakon obrade `record_source()` upisuje naslov, izdavatelja, URL, licencu i datum preuzimanja u `data/processed/manifest.json`. Iz te datoteke aplikacija prikazuje izvor uz svaku vrijednost.

**Zašto:** izvještaj ne smije ovisiti o tome radi li u trenutku upita svaki od tridesetak izvora. Osim toga, pristojnim preuzimanjem ne opterećujemo poslužitelje izdavača.

**Posebnosti učitavanja:**
- **DGU adrese** (`dgu.py`, `iter_members()`, retci 56–66): `Address.gml` je jedan redak od 2,6 GB, pa se čita strujno iz ZIP-a u blokovima od 16 MB i dijeli po oznaci `</wfs:member>`. Svaki zapis čitaju regularni izrazi (struktura je fiksna INSPIRE shemom), što traje oko minutu.
- **OSM** (`osm.py`, `extract()`, retci 113–159): jedan prolaz kroz PBF s bibliotekom pyosmium i filtrom ključeva (`FILTER_KEYS`).
- **EEA postaje** (`stations.py`, `parquet_urls()`): POST zahtjev servisu vraća popis Parquet datoteka po mjernom mjestu.
- **Općinski skupovi** (`recycling.py`, `search_packages()`): četiri upita CKAN API-ju („reciklažno dvorište”, „reciklažna dvorišta”, „zeleni otok”, „zeleni otoci”), pa preuzimanje svakog CSV/JSON/XLSX resursa.

### 5.2. Čišćenje

**Nedostajuće vrijednosti:**
- DGU: adresa bez položaja ili kućnog broja se preskače (`dgu.py`, `addresses()`: `if pos is None or not number: continue`). Selo bez ulica dobiva naziv naselja umjesto ulice (`street or place`).
- DZS: crtica u stupcu „Ukupno” znači 0 (`str(r.total).replace("-", "0")`). Grad Zagreb je i županija i grad, pa mu se prazan stupac grada popunjava županijom (`rows["jls"].fillna(rows["county"])`).
- Registri: `_clean()` u `registries.py` pretvara prazne i NaN vrijednosti u prazan tekst i uklanja „.0” koje Excel dodaje brojevima.
- GTFS: polazak bez vremena se izbacuje, kao i stajalište s koordinatama (0, 0) (`transit.py`, `gtfs_departures()`: `stops["lat"].abs() > 1`).
- EFFIS: poligon bez geometrije se preskače.
- EEA postaje: računaju se samo valjana mjerenja (`Validity > 0` i vrijednost postoji), uz pokrivenost godine (`coverage`).
- Općinski skupovi: redovi bez koordinata broje se kao `missing`, a neispravni brojevi kao `malformed` (`recycling.py`, `extract_points()`) i zapisuju se kao nalaz o kvaliteti.

**Duplikati:**
- HZZO: više timova u istoj ordinaciji daje jednu točku (skup `seen` u `_hzzo_teams()`). Ljekarne se uklanjaju po ključu (poštanski broj, ulica, broj).
- Registri nakon geokodiranja: `drop_duplicates(["category", "lon", "lat"])` u `pois.py`, `build()`.
- OSM točka iste vrste unutar 100 m od službene se izbacuje (`OSM_DEDUP_M = 100`, `pois.py`, funkcija `_far_from()` s mrežom ćelija za brzinu).
- OSM stajalište unutar 60 m od stajališta s voznim redom smatra se istim (`STOP_DEDUP_M = 60`).
- Općinska reciklaža: isti skup je često objavljen u 3–4 formata, pa se uzima samo prvi upotrebljiv resurs (`break` u `recycling.py`). Uklanjaju se i duplikati koordinata, a OSM točke unutar 30 m od općinskih (`DEDUP_M = 30`).
- EEA postaje: od više mjernih mjesta iste tvari na postaji ostaje ono s najboljom pokrivenošću.

**Pretvorba tipova:**
- Sve tablice čitaju se kao tekst (`dtype=str`), a brojevi se pretvaraju izričito. Tako se ne gube vodeće nule u šiframa naselja („000132”) i poštanskim brojevima.
- Vrijeme „HH:MM:SS” pretvara se u sekunde (`transit.py`, `_seconds()`). GTFS dopušta i vremena iza ponoći, npr. „25:10:00”.
- Decimalni zarez postaje točka (`recycling.py`, `_to_float()`).
- Kućni broj se slaže iz INSPIRE dijelova: broj, slovo i podbroj („45”, „45A”, „25/2”) (`dgu.py`, `house_number()`, retci 73–87). Do 6. 11. 2026. slova su se gubila (60.030 od prvih 401.839 adresa ima slovo).
- **Koordinatni sustavi:** DGU je u EPSG:3035 (ETRS89-LAEA) s redoslijedom osi sjever, istok, pa se okreće i pretvara u WGS84 (`dgu.py`, retci 141–144). Za udaljenosti se sve pretvara u HTRS96/TM (EPSG:3765, metri) u `backend/app/geo.py`.

**Filtriranje:**
- Točke izvan okvira Hrvatske se odbacuju (`CROATIA_BBOX` u `app/geo.py`). Općinskim točkama s **zamijenjenom** širinom i dužinom koordinate se okreću (`recycling.py`, retci 100–102).
- Iz registra MZOM-a izbacuje se Zagreb, jer za Zagreb postoji točniji gradski geoportal s koordinatama.
- OSM škole ulaze samo ako naziv govori da je osnovna ili područna škola (`pois.py`, `_is_primary_school()`). Liječnici i pedijatri dolaze samo iz HZZO-a (`OFFICIAL_ONLY`), jer OSM `amenity=doctors` miješa specijaliste.
- Pruge: izbacuju se sporedni kolosijeci i ranžirni kolodvori (`RAIL_EXCLUDED_SERVICE` u `osm.py`). Županijske ceste (četveroznamenkasta oznaka Ž) ne broje se kao prometnice (`analysis/environment.py`, `is_traffic_road()`).
- GTFS: broje se samo polasci od 6 do 20 h prve srijede s prometom, bez dolaska na zadnje stajalište (`transit.py`, `_service_date()` i `gtfs_departures()`).
- OSM linije: izbacuju se relacije s rasponom većim od 30 km ili s manje od 5 stajališta, jer su to međugradske linije (`transit_lines.py`, `local_routes()`).

**Primjer (DGU adresa) prije i poslije:**

Prije (sirovi INSPIRE GML, skraćeno):
```
<ad:alternativeIdentifier>Kamenjačka ulica 19 Andraševec 49243 Oroslavje</ad:alternativeIdentifier>
<gml:pos>2560213.96618063 4781662.24108698</gml:pos>      (EPSG:3035, sjever pa istok)
<ad:designator>19</ad:designator> … addressNumber
<ad:component xlink:href="#ThoroughfareName.2156774454"/> …
```
Poslije (redak u `data/processed/addresses.parquet`):
```
street=Kamenjačka ulica  housenumber=19  place=Andraševec  postcode=49243
settlement_code=000132  lon=15.945399  lat=45.989097  source=dgu_ad
```

### 5.3. Spajanje skupova

| Što se spaja | Ključ | Kako su riješena nepoklapanja | Gdje |
|---|---|---|---|
| adresa ↔ naselje | šifra naselja: 6-znamenkasti `alternativeIdentifier` komponente `AdminUnitName` | država nema šifru od 6 znakova, pa se razlikuje | `dgu.py`, `addresses()` |
| naselje → općina → županija | **prostorno**: reprezentativna točka naselja unutar poligona općine | GDAL ne čita roditeljske veze iz GML-a, pa se hijerarhija izvodi geometrijom; duplikati po šifri se uklanjaju | `dgu.py`, `admin_units()`, `parents()` |
| DZS ↔ DGU naselja | **po nazivu**: (normalizirani grad/općina, normalizirano naselje) | DZS nema šifre naselja. Normalizacija uklanja dijakritike, interpunkciju i „dj”→„d” (`app/geocode.py`, `normalize()`). Dvojezični nazivi („Ime – Name”) režu se kod crtice. Spojeno je **6.633 od 6.759** naselja; ostala nemaju broj stanovnika (prikazuje se „nema podataka”) | `dzs.py`, `_key()` |
| središte naselja (OSM) ↔ DGU naselje | isti normalizirani naziv **i** OSM čvor unutar poligona naselja | **6.536** naselja dobilo je stvarno središte (trg, crkva), a **223** zadržava reprezentativnu točku poligona | `centers.py`, `run()` |
| registri (vrtići, škole, HZZO) ↔ DGU adrese | (poštanski broj, kućni broj) pa (naselje, kućni broj), uz bodovanje riječi ulice | najdulja riječ ulice (obično prezime) mora se poklopiti; preferira se točan broj. Ako broja nema, uzima se sredina ulice (preciznost „street”). Nepronađeni se izbacuju i broje u `geocoding_report.json` | `registries.py`, `AddressIndex.match()`, `_best()`, `street_score()` |
| GTFS: polasci ↔ putovanja ↔ linije | `trip_id`, `route_id`, `service_id` | putovanja samo za aktivne servise odabranog dana | `transit.py`, `gtfs_departures()` |
| stajališta s voznim redom ↔ OSM stajališta | prostorno, 60 m | OSM duplikat se uklanja | `pois.py` |
| OSM linije ↔ stajališta u izvještaju | prostorno, 30 m | perón i točka na cesti često su 10–20 m razmaknuti | `app/stop_lines.py`, `MATCH_M` |
| mjerenja postaja ↔ metapodaci postaja | `sampling_point` = `Sampling Point Id` | od više godina metapodataka uzima se zadnja | `stations.py`, `run()` |
| visoka učilišta ↔ naselja | po nazivu mjesta | bez geokodiranja adresa, jer pravila trebaju samo naselje | `facilities.py` |

**Uspješnost geokodiranja registara** (`data/processed/geocoding_report.json`):

| Izvor | Na adresu | Na ulicu | Nije pronađeno | Uspješno |
|---|---|---|---|---|
| Vrtići (MZOM) | 599 | 53 | 50 | 92,9 % |
| Osnovne škole (MZOM) | 726 | 26 | 57 | 93,0 % |
| Obiteljski liječnici (HZZO) | 1.048 | 63 | 90 | 92,5 % |
| Pedijatri (HZZO) | 180 | 4 | 10 | 94,8 % |
| Ljekarne (HZZO) | 955 | 83 | 163 | 86,4 % |

**Primjer (registar vrtića) prije i poslije:**

Prije (CSV registra MZOM-a): `Naziv=Dječji vrtić Mali princ, Adresa=Dunavska 86, Mjesto=Osijek, PostanskiBroj=31000` (bez koordinata).
Poslije (`registry_pois.parquet`): `category=kindergarten, name=Dječji vrtić Mali princ, lon=18.685818, lat=45.55115, source=mzom_vrtici, detail=Dunavska 86, Osijek, precision=address`.

**Primjer (DZS) prije i poslije:**

Prije (XLSX, tri retka po naselju): `Vukovarsko-srijemska | Općina | Nijemci | Nijemci | sv. | 1330`, zatim isti redak za `m` (671) i `ž` (659).
Poslije: zadržava se samo `sv.`, ključ je (`nijemci`, `nijemci`), a rezultat je `settlement_code=043052, population=1330`.

### 5.4. Transformacije i izračuni

1. **Isijecanje rastera** (`rasters.py`, `clip_to_croatia()`): europski rasteri izrezuju se na okvir Hrvatske (`CROATIA_BOUNDS = (13.3, 42.3, 19.5, 46.6)`) i spremaju kao komprimirani GeoTIFF u pločicama 256 × 256. Pločice WorldCovera kopiraju se nepromijenjene.
2. **Polasci na sat** (`transit.py`): broj polazaka sa stajališta od 6 do 20 h podijeljen s 14. Primjer iz `transit_stops.parquet`: `Trg bana J. Jelačića, tram, 50.43 polaska/h, gtfs_zet, radni dan 2026-11-11`.
3. **Spajanje točaka** (`pois.py`, `build()`): jedan sloj `pois.parquet` s 26.566 točaka:

   | Kategorija | Broj točaka |
   |---|---|
   | stajališta | 10.170 |
   | trgovine | 4.959 |
   | igrališta | 3.289 |
   | škole | 2.131 |
   | reciklaža | 2.116 |
   | vrtići | 1.367 |
   | ljekarne | 1.330 |
   | liječnici | 1.031 |
   | pedijatri | 173 |

4. **Linije na stajalištima** (`transit_lines.py`): 11.510 redaka (stajalište, vrsta, broj linije, prijevoznik).
5. **Profili naselja** (`settlement_profiles.py`, `profile()`): za središte svakog od 6.759 naselja računaju se iste kategorije kao u izvještaju i bodovi 0–100, u 4 paralelne niti. Ovaj korak treba pokrenutu Valhallu.
6. **Medijan po adresama** (`settlement_landcover.py`, `settlement_traffic.py`): jedna točka ne opisuje naselje, pa se za prirodu, drveće, zrak i prometnice računa medijan i raspon od 25. do 75. percentila preko adresa naselja. Naselja s više od 100 adresa ulaze sa slučajnim uzorkom od 100 adresa (`SAMPLE = 100`, `SEED = 2026`). Za prometnice se računaju sve adrese.
7. **Visoka učilišta i bolnice** (`facilities.py`): studiji unutar 40 km, najbliža bolnica, studentski dom i menza.

**Primjer izlaza (profil naselja Nijemci, `settlement_profiles.parquet`):**
```
population=1330  flood_status=good  air_band=umjerena  pm25=18.6  no2=16.5
tree_share=18.2  traffic_band=Blizu prometnice  burnt_areas=0  needs_15min=5
min_school=3.0  min_doctor=4.4  min_kindergarten=12.0  has_park=False
score_flood=100  score_air=50  score_green=100  score_noise=25  score_car_free=71  score_transit=NaN
```
`score_transit=NaN` znači da Nijemci imaju stajališta bez otvorenog voznog reda, pa se učestalost ne zna i kategorija se ne boduje.

### 5.5. Izlaz

`data/processed/` (554 MB):
- Parquet tablice:

  | Datoteka | Sadržaj |
  |---|---|
  | `addresses.parquet` | 1.683.411 adresa |
  | `admin_units.parquet` | 7.336 jedinica |
  | `settlements.parquet` | 6.759 naselja |
  | `pois.parquet` | 26.566 točaka |
  | `transit_stops.parquet` | 4.274 stajališta: ZET 2.525, HŽPP 460, Šibenik 227, Autotrolej 1.062 |
  | `effis_burnt.parquet` | 1.030 opožarenih površina |
  | `osm_noise_lines.parquet` | 50.060 dionica cesta i pruga |
  | `osm_parks.parquet` | 2.237 parkova |
  | `recycling.parquet` | 2.116 točaka, od toga 45 iz općinskih skupova |
  | `settlement_*.parquet` | profili i medijani naselja |

- GeoTIFF-ovi: `flood_rp10/100/500.tif`, `air_pm25_2024.tif`, `air_no2_2024.tif`, `worldcover/` (6 pločica).
- `manifest.json`: 38 zapisa s izvorom, licencom i datumom.
- Izvještaji o kvaliteti: `geocoding_report.json` i `recycling_ckan.json` (33 općinska skupa pronađena, 11 upotrebljivih).

**Što se proizvede, a ne koristi:** `osm_addresses.parquet` (268.457 adresa iz OSM-a) piše `ingest/osm.py`, ali ga ne čita nijedan dio aplikacije. To je ostatak rezervnog plana iz `CLAUDE.md` za slučaj da DGU adrese ne rade. `stations_2024.parquet` koristi se samo u `analysis/ml_air.py` i `analysis/validation.py`, ne u izvještaju.

### 5.6. Izračun izvještaja za jednu adresu (u stvarnom vremenu)

`backend/app/report.py`, `build_report()`:
1. `locate()`: naselje, općina i županija za točku (`app/admin.py`). Točka izvan kopna Hrvatske daje grešku 422.
2. U tri paralelne niti: PVGIS (sunce), Valhalla izokrone (5, 10, 15 min) i `compute_reach()` (`analysis/walking.py`). Ta funkcija za svaku vrstu mjesta uzima kandidate unutar 1.300 m zračne udaljenosti (barem 3 najbliža, najviše 40), pa **jednim** pozivom Valhalle (`sources_to_targets`) dobiva vremena hoda do svih (`app/valhalla.py`, `walk_minutes()`).
3. Kategorije: `flood_category`, `air_category`, `green_category`, `noise_category`, `fire_category` (`analysis/environment.py`), `solar_category` (`analysis/solar.py`), `car_free_category`, `transit_category`, `daily_category` (`analysis/daily.py`).
4. `score.attach()`: bodovi 0–100 po kategoriji (`analysis/score.py`).
5. `summary()`: dvije do četiri rečenice iz predložaka nad statusima.
6. Popis korištenih izvora iz `manifest.json`.

---

## 6. Analitika, modeli i umjetna inteligencija

### 6.1. U aplikaciji nema modela strojnog učenja

Sve kategorije računaju se determinističkim pravilima s pragovima iz izvora (`docs/METODOLOGIJA.md`):

| Kategorija | Metoda | Pragovi i izvor | Kod |
|---|---|---|---|
| Poplave | vrijednost piksela na točki i najveća dubina unutar 100 m za povratna razdoblja 10, 100 i 500 godina | loše: 10 ili 100 godina; upozorenje: 500 godina ili poplava unutar 100 m (Direktiva 2007/60/EZ, karte Hrvatskih voda). Vjerojatnost u 30 godina: P = 1 − (1 − 1/T)^30, oko 26 % za T = 100 | `environment.py`, `flood_category()`, `flood_chance()` |
| Zrak | vrijednost ćelije od 1 km | 4 razine: WHO 2021 (PM2,5 5, NO2 10 µg/m³), EU 2030. (10 i 20), današnja granica EU-a (25 i 40, Direktiva (EU) 2024/2881) | `environment.py`, `AIR_BANDS`, `air_category()` |
| Priroda i hlad | udio razreda WorldCovera u metričkom krugu od 300 m | gusto izgrađeno je više od 50 % izgrađenog (naša pretpostavka). Tada vrijedi pravilo 3-30-300 (Konijnendijk, 2023.): 30 % drveća i park do 300 m | `environment.py`, `landcover_pixels()`, `green_category()` |
| Udaljenost od prometnica | zračna udaljenost do autoceste, brze, državne ili glavne gradske ceste | do 300 m, 300–500 m, više od 500 m (Health Effects Institute, 2010.) | `environment.py`, `TRAFFIC_BANDS`, `noise_category()` |
| Požari | opožarene površine koje dodiruju krug od 2 km | samo informacija, bez ocjene | `environment.py`, `fire_category()` |
| Sunce | PVGIS, 1 kWp, gubici 14 %, optimalni nagib | samo informacija | `app/pvgis.py`, `analysis/solar.py` |
| Hod | Valhalla, pješački profil, 4,5 km/h | obrazložena pretpostavka (`docs/ODLUKE.md`) | `app/valhalla.py`, `WALKING_SPEED_KMH` |
| Javni prijevoz | najviše polazaka na sat sa stajališta do 10 min hoda | 6 polazaka na sat = 100 bodova (naša normalizacija) | `analysis/daily.py`, `analysis/score.py` |

**Ocjena mjesta** (`analysis/score.py`): svaka kategorija dobiva bodove 0–100.

| Kategorija | Pravilo bodovanja |
|---|---|
| poplave | 100 / 50 / 0 |
| zrak | 100 / 80 / 50 / 10 po razini |
| prometnice | 100 / 60 / 25 |
| požari | 100 ako nema požara, 60 za jedan, 30 za više |
| sunce | linearno od 1.000 do 1.500 kWh/kWp |
| život bez auta | broj ispunjenih potreba od 7 × 100 / 7 |

Ocjena je ponderirani prosjek s težinama 0–3 iz `config/profiles.json`. Kategorije bez bodova preskaču se (`weighted()`). Isti izračun postoji u pregledniku (`frontend/src/lib/score.ts`, `lifeScore()`), pa korisnik mijenja težine bez novog zahtjeva prema poslužitelju. „Najbolje” i „najslabije” su kategorije s najvećim pozitivnim, odnosno negativnim doprinosom: težina × (bodovi − ocjena).

### 6.2. Evaluacija strojnog učenja za kvalitetu zraka (nije ušla u aplikaciju)

`backend/analysis/ml_air.py`, opis u `docs/ML.md`.

- **Pitanje:** može li model procijeniti godišnji prosjek PM2,5 i NO2 na razini ulice bolje od EEA karte od 1 km?
- **Zašto baš to:** jedino ovdje postoji službena „istina” za provjeru (mjerne postaje) i jasna osnova za usporedbu (EEA karta).
- **Ulazi** (`FEATURES`):
  - vrijednost EEA karte;
  - udaljenost od najbliže velike ceste (OSM);
  - udio izgrađenog i drveća u 300 m (WorldCover);
  - logaritam broja stanovnika naselja (DZS).
- **Izlaz:** odstupanje postaje od karte. Procjena je karta plus predviđeno odstupanje.
- **Modeli:**
  - prosječno odstupanje;
  - linearna regresija;
  - Random Forest (300 stabala, `min_samples_leaf=2`, `random_state=0`, scikit-learn).
- **Provjera:** leave-one-station-out. Svaka postaja predviđa se modelom koji je nikad nije vidio. Mjere su RMSE, MAE i pristranost.
- **Pravilo odluke, postavljeno prije izračuna:** model ulazi u aplikaciju samo ako je RMSE niži od osnove za obje tvari.

| Tvar | Postaja | RMSE EEA karte | RMSE Random Foresta | Promjena |
|---|---|---|---|---|
| PM2,5 | 18 | 1,73 µg/m³ | 1,96 µg/m³ | +13 % (lošije) |
| NO2 | 16 | 4,91 µg/m³ | 4,17 µg/m³ | −15 % (bolje) |

**Odluka:** model nije ušao u aplikaciju. Za NO2 je bolji, ali počiva na samo 16 postaja, od kojih su 4 prometne. **Ograničenja:** postaja je malo, a EEA karte izrađene su i uz pomoć istih postaja, pa osnova ima prednost koja nije neovisna.

Povezani alat: `analysis/validation.py` uspoređuje vrijednosti karte zraka s najbližom postajom u Zagrebu, Osijeku i Požegi. Provjera je pokazala da su prometne postaje i do 17 µg/m³ NO2 iznad karte (`docs/METODOLOGIJA.md`).

### 6.3. Jezični model u asistentu (opcionalan)

`backend/app/assistant.py`:
- Bez ključa radi `understand_rules()`: ključne riječi nad tekstom bez dijakritika (`CATEGORY_KEYWORDS`, `PURCHASE_PATTERNS`, `OUT_OF_SCOPE`…) određuju namjeru i kategorije.
- S ključem (`LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY` u `.env`) radi `understand_llm()`: model preko OpenAI-kompatibilnog API-ja vraća **samo JSON s namjerom i kategorijama**. Ako ne odgovori ili vrati nešto neispravno, koriste se pravila.
- Odgovor (`short_answer()`, `answer()`) se uvijek slaže iz predložaka nad rezultatom `build_report()`. Model ne piše nijednu brojku.
- Namjere su `report`, `overview`, `explain`, `purchase_advice` (odbija savjet o kupnji), `out_of_scope`, `no_data` i `unknown`.
- Isti obrazac koristi `/api/find/parse` (`app/finder.py`, `parse_question()`): slobodno napisano pitanje pretvara se u filtre, a rang ostaje deterministički.
- **[PROVJERITI]** Je li na javnom demou (`https://dobromjesto.domagojlukas.online/`) postavljen ključ za jezični model, iz koda se ne vidi. To je postavka u `.env` na poslužitelju.

---

## 7. Tehnička arhitektura

**Backend** (točne verzije iz `backend/uv.lock`; Python 3.12):

| Biblioteka | Verzija | Uloga |
|---|---|---|
| FastAPI | 0.162.0 | REST API, automatski OpenAPI |
| Uvicorn | 0.54.0 | poslužitelj |
| Pydantic | 2.14.2 | modeli i provjera podataka |
| httpx | 0.28.1 | HTTP klijent (preuzimanje, Valhalla, PVGIS, CKAN) |
| NumPy / pandas | 2.5.4 / 3.1.0 | izračuni, tablice |
| rasterio | 1.5.2 | čitanje GeoTIFF-ova |
| Shapely | 2.2.0 | geometrija, prostorni indeks STRtree |
| pyproj | 3.8.0 | koordinatni sustavi |
| PyArrow | 26.0.0 | Parquet |
| GeoPandas | 1.2.0 | samo obrada (upravne jedinice) |
| pyosmium | 4.3.2 | samo obrada (OSM) |
| scikit-learn | 1.9.1 | samo evaluacija ML-a |
| ruff, mypy (strict), pytest | razvoj | kvaliteta koda |

**Frontend** (točne verzije iz `frontend/package-lock.json`):

| Biblioteka | Verzija | Uloga |
|---|---|---|
| React | 19.3.1 | sučelje |
| React Router | 8.5.1 | stranice |
| TanStack Query | 5.105.2 | dohvat i predmemorija podataka |
| MapLibre GL JS | 6.16.0 | karte (podloga OpenFreeMap) |
| Vite | 8.3.5 | izgradnja i razvojni poslužitelj |
| TypeScript | 5.9.3 | jezik |
| Vitest, Playwright, axe-core, ESLint | 5.1.0, 1.64.1, 4.x, 9.39.5 | testovi i pristupačnost |

**Usluge:**
- Valhalla 3.9.1 (Docker image `ghcr.io/valhalla/valhalla-scripted:3.9.1`): pješačka mreža iz OSM-a.
- Caddy 2: HTTPS i posluživanje izgrađenog frontenda.
- Nema baze podataka: podaci su datoteke koje API drži u memoriji (`app/layers.py`, `get_layers()` s `@lru_cache`).

**Kako komponente komuniciraju:**
- Preglednik zove samo `/api/*` na istom poslužitelju. U razvoju Vite prosljeđuje `/api` na port 8000 (`frontend/vite.config.ts`), a u produkciji to radi Caddy (`deploy/Caddyfile`, `reverse_proxy api:8000`).
- Pločice karte preglednik dohvaća izravno s OpenFreeMapa.
- API zove Valhallu (HTTP, port 8002), PVGIS i CKAN Portala. Jezični model zove samo ako je postavljen.
- Pri pokretanju API (`app/main.py`, `warm_up()`):
  - učita geokoder, slojeve i mrežu adresa;
  - u pozadini zatraži PVGIS za demo adrese;
  - izgradi podatke za kartu.

```
                         OBRADA (make data, jednom mjesečno)
 data.gov.hr (DGU, DZS, MZOM, Zagreb, općine, HŽPP, Autotrolej)  ┐
 HZZO (XLSX)   ZET, Šibenik (GTFS)                               │   backend/ingest/*.py
 JRC, EEA, ESA (GeoTIFF)   EFFIS (WFS)   EEA postaje             ├──► data/raw ──► data/processed
 OpenStreetMap (PBF) ────────────────────────────────────────────┘    (Parquet, GeoTIFF, manifest.json)
        │                                                                     │
        └──► Valhalla (pločice pješačke mreže)                                │ učitava se u memoriju
                    ▲                                                         ▼
                    │ izokrone, matrica, rute                ┌───────────────────────────────┐
                    └────────────────────────────────────────│  FastAPI (backend/app)        │
 PVGIS API ◄──── sunce (predmemorija 0,01°) ─────────────────│  /api/report, /api/geocode,   │
 CKAN API ◄──── datumi skupova (predmemorija 1 h) ───────────│  /api/find, /api/map/*, …     │
 Jezični model ◄─ samo namjera pitanja (opcionalno) ─────────└───────────────▲───────────────┘
                                                                             │ /api/*
                                                               ┌─────────────┴─────────────┐
                                                               │ Caddy (HTTPS) ili Vite    │
                                                               └─────────────▲─────────────┘
                                                                             │
                                          Preglednik: React + MapLibre ──────┘ ──► OpenFreeMap (podloga)
```

**API rute** (`backend/app/main.py`):

| Ruta | Što radi |
|---|---|
| `GET /api/health` | stanje |
| `GET /api/geocode?q=` | prijedlozi adresa |
| `GET /api/reverse` | naselje za točku |
| `GET /api/report?lat=&lon=[&format=geojson]` | cijeli izvještaj |
| `GET /api/route` | pješački put |
| `GET /api/sources`, `/api/sources/live` | izvori i datumi s Portala |
| `POST /api/assistant` | asistent |
| `GET /api/find/options`, `POST /api/find`, `POST /api/find/parse` | „Predloži mjesto” |
| `GET /api/map/geometry/{overview,detail}`, `/api/map/settlements`, `/api/map/point` | karta |

OpenAPI opis je na `/api/docs`.

**Standardi:**
- REST, OpenAPI 3.1, JSON, GeoJSON (RFC 7946);
- INSPIRE GML (ulaz), GTFS, CKAN API, WFS;
- EPSG:4326, 3765, 3035;
- GeoParquet i GeoTIFF;
- WCAG 2.2 AA.

**Produkcija** (`deploy/docker-compose.yml`): tri usluge, `web` (Caddy s izgrađenim frontendom), `api` (2 Uvicorn radnika, bez pristupnih zapisa) i `valhalla`. Prema `docs/ARHITEKTURA.md` dovoljno je 2 vCPU, 4 GB RAM-a i 10 GB diska.

---

## 8. Kako pokrenuti projekt

Preduvjeti: Git, Docker Desktop, `make` (na Windowsu iz Git for Windows SDK-a ili MSYS2), [uv](https://docs.astral.sh/uv/) i Node.js 22+. Detaljne upute za Windows bez `make`-a su u `docs/RAZVOJ.md`.

```bash
# 1. Preuzimanje koda
git clone https://github.com/dlukas99/datathon_projekt.git
cd datathon_projekt

# 2. Postavke (upišite CONTACT_EMAIL; LLM_* samo ako želite asistenta s modelom)
cp .env.example .env

# 3. Ovisnosti: uv sync (backend), npm ci i Playwright Chromium (frontend)
make setup

# 4. Preuzimanje i obrada svih izvora (prvi put oko 20 min i 2,6 GB)
make data
```

Korak „Profili naselja” u `make data` treba pokrenutu Valhallu, a Valhalla treba OSM izvadak koji `make data` tek preuzme. Zato pri prvom pokretanju taj korak javi grešku, a ostali koraci se dovrše (`run_all.py` nastavlja nakon greške). Profile zatim treba dovršiti ručno:

```bash
make valhalla                       # Docker: Valhalla gradi pločice oko 3 minute
cd backend && uv run python -m ingest.settlement_profiles && cd ..
```

```bash
# 5. Razvojno pokretanje: Valhalla, backend :8000 i frontend :5173
make dev
```

Na Windowsu `uvicorn --reload` (koji koristi `make dev`) zna zapeti pri ponovnom učitavanju. Pouzdanije je pokrenuti dijelove u zasebnim terminalima:
```bash
docker compose -f deploy/docker-compose.dev.yml up -d valhalla
cd backend && uv run python -m uvicorn app.main:app --port 8000 --no-access-log
cd frontend && npm run dev
```

6. Otvorite http://localhost:5173 (API opis: http://localhost:8000/api/docs).

```bash
# Provjere
make test      # pytest + Vitest
make lint      # ruff, mypy, ESLint, tsc
make verify    # sve provjere, izgradnja, e2e i provjera README-a

# Produkcija (na poslužitelju s Dockerom, nakon make data ili scripts/upload_data.sh)
DOMAIN=vasa-domena.hr docker compose -f deploy/docker-compose.yml up -d --build
```

---

## 9. Glavne funkcionalnosti iz perspektive korisnika

Stranice su definirane u `frontend/src/App.tsx`.

1. **Naslovnica (`/`, `HomePage.tsx`):**
   - pitanje „Kako je živjeti ovdje?” i dva puta: **Provjeri adresu** i **Pronađi mjesto za mene**;
   - mozaik svih naselja obojenih prema ocjeni mjesta (`CroatiaMosaic.tsx`);
   - šest okolišnih mjerenja i kratko objašnjenje.
2. **Pretraga adrese (`AddressSearch.tsx` → `/api/geocode`):**
   - prijedlozi dok korisnik tipka;
   - razumije „12a”, „12 A”, „25/2”, „bb”, poštanski broj i „Grad/Općina”;
   - prepoznaje stara i nova imena ulica („Brune Bušića” ↔ „Bušićeva”);
   - kad kućnog broja nema, nudi ulicu, a kad nema ulice, naselje, uz napomenu.
3. **Izvještaj (`/izvjestaj`, `ReportPage.tsx`):**
   - **iskaznica mjesta**: sažetak i šest ključnih očitanja;
   - **okolišni pregled**: kartice poplava, zraka, prirode, prometnica, požara i sunca, svaka s jednom rečenicom i dijelom „Detalji i izvor”;
   - **život bez auta**: izokrone 5/10/15 min na karti, „X od 7” potreba i javni prijevoz s polascima ili linijama iz OSM-a;
   - **svakodnevni život**: najbliže ustanove s minutama hoda. Klik iscrta pješački put (`/api/route`).
4. **Ocjena mjesta (`LifeScore.tsx`, `ProfilePicker.tsx`):**
   - npr. „77 od 100”;
   - riječima piše što je najbolje, a što najslabije;
   - korisnik bira profil ili mijenja težine u 4 stupnja, a izbor se sprema u URL.
5. **Usporedba (`/usporedi`, `ComparePage.tsx`):** dvije adrese jedna uz drugu.
6. **Asistent (`Assistant.tsx` → `/api/assistant`):** pitanja na hrvatskom. Odgovor ima jednu do dvije rečenice, redak „Izvor” i „Prikaži detalje”.
7. **Predloži mjesto (`/predlozi`, `FindPage.tsx`):**
   - tri koraka: tko traži, gdje, što je važno;
   - prijedlozi naselja s ispunjenim uvjetima, razlozima i jednom napomenom;
   - gumb „Pogledaj oglase” (`ListingLinks.tsx`) slaže poveznicu na Njuškalo, Index Oglasi ili Nekretnine.hr, a oglase ne dohvaća.
8. **Karta Hrvatske (`/karta`, `MapPage.tsx`, `CroatiaMap.tsx`):**
   - 6.759 naselja obojenih prema sloju (ocjena za profil, zrak, poplave, usluge pješice, javni prijevoz);
   - klik otvara karticu naselja;
   - ispod karte je pet najboljih naselja i pretraga, za korištenje bez karte.
9. **O alatu (`/o-alatu`, `AboutPage.tsx`):** dijelovi Ukratko, Kako računamo, Izvori podataka (s datumima s Portala uživo), Kvaliteta podataka, Privatnost i Tim.
10. **Ostalo:**
    - dijeljenje poveznicom (stanje je u URL-u);
    - ispis i preuzimanje GeoJSON-a;
    - tamna tema prati uređaj;
    - prikaz radi na mobitelu od 320 px.

---

## 10. Slabosti i ograničenja

**Kvaliteta i pokrivenost podataka:**
- **Geokodiranje registara** uspijeva za 86–95 % zapisa. Nepronađeni vrtići, škole, liječnici i ljekarne (ukupno 370) ne prikazuju se, a vrtići, škole i ljekarne djelomično se dopunjuju iz OSM-a.
- **Vozni redovi** postoje u otvorenom obliku samo za ZET, HŽPP, Autotrolej i Šibenik. Za Split, Osijek, Zadar, Pulu i druge prikazuju se samo stajališta i linije iz OSM-a, bez učestalosti.
- **Vozni red HŽPP-a vrijedi do 13. 12. 2026.** Nakon toga treba ponovno pokrenuti `make data`, inače će polasci vlakova biti zastarjeli.
- **HZZO popis ljekarni** zadnji je put izmijenjen u rujnu 2023.
- **URL-ovi HZZO-a sadrže mjesec** (`web_opca_092026_0.xlsx`). Kad HZZO objavi novu datoteku, adresu u `registries.py` treba ručno promijeniti.
- **OpenStreetMap** je izvan gradova nepotpun: „nije pronađeno” može značiti i da objekt nije ucrtan.
- **Zrak** je modelirana karta na mreži od 1 km, a ne mjerenje na adresi. Uz prometne ulice NO2 može biti znatno viši.
- **Poplave** obuhvaćaju samo modelirane riječne poplave većih vodotoka, bez bujica i mora.
- **Buka** se ne mjeri, nego se procjenjuje udaljenošću od ceste, jer karte buke EEA-e nisu pod otvorenom licencom. Županijske ceste se ne broje, pa prometna Ž-cesta kroz grad (npr. Ž4290 u Vinkovcima) nije vidljiva.
- **WorldCover** je iz 2021. i ne vidi pojedinačna stabla uz ulice. Prag „gusto izgrađeno > 50 %” je naša pretpostavka.
- **Požari:** EFFIS bilježi uglavnom požare od oko 30 ha naviše.
- **Licence:** HZZO i HŽPP nemaju navedenu licencu (vidi odjeljak 4).

**Tehnička ograničenja:**
- **Ovisnost o vanjskim servisima:** PVGIS (prvi upit za novu točku čeka do 6 s, nakon čega se prikazuje „Nema podataka”) i CKAN (zadnji dobar odgovor čuva se na disku).
- **Sve je u memoriji:** API treba nekoliko GB RAM-a. Skaliranje znači više kopija API-ja, a za to nema baze ni dijeljene predmemorije.
- **Osvježavanje podataka je ručno** (`make data`). U repozitoriju nema zakazanog zadatka (cron).
- Valhalla nakon prvog pokretanja oko 3 minute gradi pločice. Za to vrijeme kategorije hoda pokazuju „Nema podataka”.
- Mjerenja brzine (0,1–0,5 s za API s toplom predmemorijom, 0,5–1,6 s u pregledniku) su iz `docs/ARHITEKTURA.md`, izmjerena 8. 11. 2026. na razvojnom računalu, ne na produkciji.

**Nedovršeno ili nekorišteno:**
- P2 zadaci **procjena auto-kilometara i CO2** te **izvoz PDF-a izvještaja** nisu napravljeni (`PROGRESS.md`). Postoji samo ispis iz preglednika.
- ML model za zrak napravljen je i evaluiran, ali nije u aplikaciji (odjeljak 6.2).
- `osm_addresses.parquet` se proizvodi, ali se ne koristi.
- **Poveznice na Index Oglasi** rade preciznije samo za dio mjesta, jer UUID-ovi mjesta nisu upisani za sva (`config/listing_locations.json`). Kategorije najma na Njuškalu nisu provjerene, jer Njuškalo blokira automatske provjere.
- **Pristupačnost** je provjerena samo automatski (axe-core, tipkovnica, 320 px). Provjera s čitačem zaslona i stvarnim korisnicima još nije provedena.
- Neki testovi preskaču se ako nema obrađenih podataka (npr. `tests/test_traffic.py`, `skipif`).

---

## 11. Daljnji razvoj

1. **Automatsko osvježavanje podataka:** zakazano pokretanje `make data` (npr. mjesečno) s provjerom promjena preko CKAN API-ja (`metadata_modified`) i obavijest kad se izvor promijeni ili poveznica prestane raditi.
2. **Više voznih redova:** GTFS za Split, Osijek, Zadar, Pulu i Dubrovnik čim ih prijevoznici objave. Kod je spreman, dovoljan je novi zapis u `GTFS_FEEDS` (`backend/ingest/transit.py`).
3. **Model NO2 na razini ulice** uz gradske mjerne mreže i podatke o prometnom opterećenju. Evaluacija je već pokazala −15 % RMSE, ali uz premalo postaja.
4. **Vraćanje podataka Portalu:** spojeni nacionalni sloj reciklažnih dvorišta i geokodirani registri kao prijedlog koordinata izdavačima (MZOM, HZZO).
5. **Ugradnja preko API-ja** u oglasnike nekretnina, aplikacije banaka za stambene kredite i gradske portale.
6. **Novi slojevi:** toplinski stres, kvaliteta vode za piće, karte buke i HAKOM-ova dostupnost interneta (ako postanu otvoreni) i obavijest kad se promijeni službena karta poplava.

---

## 12. Pitanja koja vam mogu postaviti

**Podaci i izvori**

1. **Koliko podataka s Portala otvorenih podataka koristite?**
   Od 34 skupa njih 22 je s data.gov.hr: adrese i upravne jedinice DGU-a, Popis 2021., četiri skupa MZOM-a, vrtići i škole Grada Zagreba, 11 skupova gradova i općina o reciklaži te vozni redovi HŽPP-a i Autotroleja. Za njih aplikacija uživo preko CKAN API-ja prikazuje datum zadnje izmjene (`backend/app/live_sources.py`). Europske izvore koristimo samo gdje domaćeg otvorenog skupa nema (zrak, satelitski pokrov tla, poplave za cijelu zemlju).

2. **Zašto koristite europske karte poplava, a ne karte Hrvatskih voda?**
   Karte Hrvatskih voda nemaju navedenu licencu, pa iz njih ne izvodimo podatke, nego samo dajemo poveznicu na službeni preglednik uz svaki izvještaj. JRC karte su otvorene i pokrivaju cijelu Hrvatsku za povratna razdoblja od 10, 100 i 500 godina. Podjelu na veliku, srednju i malu vjerojatnost preuzeli smo iz opisa karata Hrvatskih voda i Direktive 2007/60/EZ.

3. **Kako dobivate koordinate vrtića i ordinacija kad ih registri nemaju?**
   Imamo vlastito geokodiranje nad 1,68 milijuna adresa DGU-a (`backend/ingest/registries.py`, `AddressIndex.match()`). Traži se isti poštanski broj ili naselje i isti kućni broj, a zatim se boduje poklapanje riječi ulice. Najdulja riječ, obično prezime, mora se poklopiti. Uspješnost je 86–95 %, a nepronađeni zapisi se ne prikazuju.

4. **Što je s gradovima koji nemaju otvoreni vozni red?**
   Ne pišemo da prijevoza nema. Potreba za stajalištem s polascima označava se kao „nije provjereno”, a iz OpenStreetMapa navodimo koje linije ondje staju, npr. „tramvaj 1 i 2, autobus 3, 4, 5, 7 i 8 (GPP Osijek)” (`backend/ingest/transit_lines.py`, `backend/app/stop_lines.py`). Takva kategorija ne ulazi u ocjenu mjesta, jer učestalost ne znamo.

5. **Koliko su podaci svježi?**
   Svaka vrijednost nosi datum preuzimanja iz `manifest.json`. Trenutni podaci preuzeti su 2.–9. 11. 2026. Neki izvori su stariji sami po sebi: zrak je za 2024., WorldCover za 2021., a HZZO ljekarne su iz 2023. Osvježavanje je jedna naredba (`make data`), ali se zasad pokreće ručno.

6. **Jeste li našli probleme u podacima na Portalu?**
   Jesmo i zapisali smo ih u `docs/KVALITETA_PODATAKA.md`:
   - od 33 općinska skupa o reciklaži samo 11 ima upotrebljive koordinate;
   - neki imaju zamijenjenu širinu i dužinu, koje kod sam ispravlja (`recycling.py`, `extract_points()`);
   - poveznica na vozni red HŽPP-a na Portalu vraća 404;
   - registri vrtića i škola nemaju koordinate.

**Obrada i točnost**

7. **Kako znate da su vremena hoda točna?**
   Računa ih Valhalla, otvoreni usmjerivač, po stvarnoj mreži ulica i staza iz OSM-a, brzinom 4,5 km/h (`backend/app/valhalla.py`). To je nešto sporije od zadane vrijednosti, da bude realno i za starije osobe. Na 435 mjesta oko demo adresa vrijeme rute odstupa od vremena u izvještaju najviše 8 sekundi (`docs/METODOLOGIJA.md`). Ograničenje je točnost OSM mreže, npr. neucrtana pješačka staza.

8. **Kako ste spojili DZS i DGU kad DZS nema šifre naselja?**
   Po nazivu općine i naselja nakon normalizacije: bez dijakritika i interpunkcije, uz „dj”→„d” i rezanje dvojezičnih naziva kod crtice (`backend/ingest/dzs.py`, `_key()`). Spojeno je 6.633 od 6.759 naselja. Ostala nemaju broj stanovnika i to tako i piše.

9. **Zašto za naselje koristite medijan, a ne prosjek?**
   Jedna točka ne opisuje naselje: središte Splita je u pješačkoj jezgri, a središte sela na glavnoj cesti. Zato za kartu i prijedloge računamo vrijednost za adrese u naselju (`settlement_traffic.py`, `settlement_landcover.py`). Medijan je vrijednost tipične adrese, a nekoliko kuća uz šumu ili uz cestu ne može ga povući kao prosjek.

10. **Može li ocjena mjesta zavarati?**
    Ocjena je samo pomoć za usporedbu. Kategorije se uvijek prikazuju i zasebno, težine su vidljive i korisnik ih mijenja (`config/profiles.json`, `frontend/src/lib/score.ts`). Neke normalizacije (sunce, prijevoz, požari) su naše pretpostavke i tako su označene u metodologiji. Profil mijenja samo težine, nikad vrijednosti.

11. **Je li ovo službena procjena rizika?**
    Nije. Na svakom izvještaju je napomena da je alat informativan i da ne zamjenjuje karte i upozorenja Hrvatskih voda, DHMZ-a i civilne zaštite (`backend/app/report.py`, `DISCLAIMER`). Svaka vrijednost označena je kao izmjereno, procjena modela ili gruba procjena.

**Arhitektura i skalabilnost**

12. **Zašto nemate bazu podataka?**
    Podaci se mijenjaju jednom mjesečno i samo se čitaju, pa ih držimo u Parquet i GeoTIFF datotekama koje API učita u memoriju s prostornim indeksima (`backend/app/layers.py`). To je jednostavnije i brže od baze za ovaj obrazac korištenja. Da se dodaju korisnički računi ili spremanje, trebala bi baza.

13. **Koliko je brz izvještaj i kako se skalira?**
    Prema mjerenju iz `docs/ARHITEKTURA.md` API odgovara za 0,1–0,5 s s toplom predmemorijom. Jedan izvještaj šalje Valhalli samo jedan upit matrice i jedan upit izokrona. Poslužitelju su dovoljni 2 vCPU i 4 GB RAM-a. API nema stanje, pa se skalira dodavanjem kopija iza Caddyja. Mjerenja pod opterećenjem nismo radili.

14. **Kako se rješenje povezuje s drugim sustavima?**
    Preko REST API-ja s OpenAPI opisom (`/api/docs`). Izvještaj je dostupan i kao GeoJSON (`/api/report?…&format=geojson`) za QGIS i ArcGIS. Na ulazu koristimo standarde INSPIRE, GTFS, WFS i CKAN, pa se novi izvor istog standarda dodaje lako. Oglasnik ili banka mogu jednim pozivom dobiti pregled za adresu.

15. **Što ako PVGIS ili Portal ne rade?**
    Izvještaj se i dalje složi. PVGIS se čeka najviše 6 s, nakon čega sunce pokazuje „Nema podataka”, a rezultat se ipak sprema u predmemoriju za sljedeći put (`backend/app/pvgis.py`). Datumi s Portala imaju predmemoriju od sat vremena i zadnji dobar odgovor na disku (`live_sources.py`).

**AI i tehnologija**

16. **Gdje je tu umjetna inteligencija?**
    Na tri mjesta, ali uvijek ograničeno. Asistent može koristiti jezični model samo za prepoznavanje namjere pitanja, a brojke dolaze iz predložaka (`backend/app/assistant.py`). Isto vrijedi za pretvaranje slobodnog pitanja u filtre. Napravili smo i evaluaciju strojnog učenja za zrak (Random Forest, leave-one-station-out). Model nije ušao u aplikaciju, jer nije bio bolji za obje tvari.

17. **Zašto model za zrak nije u aplikaciji kad je za NO2 bio 15 % bolji?**
    Pravilo smo postavili prije izračuna: model ulazi samo ako je bolji za obje tvari. Za PM2,5 bio je 13 % lošiji. Za NO2 rezultat počiva na samo 16 postaja, od kojih su 4 prometne, što nije dovoljno stabilno za prikaz građanima. Radije prikazujemo službenu EEA kartu s jasnim ograničenjem.

18. **Može li asistent izmisliti odgovor?**
    Ne može izmisliti brojku, naziv ni datum, jer se svaka rečenica s podatkom slaže iz predloška nad izračunatim izvještajem. Model vraća samo JSON s namjerom i kategorijama, a ako vrati nešto neispravno, rade pravila. Na pitanja o kupnji ne odgovara „kupite”, a za teme bez podataka (kriminal, potres) kaže da podataka nema. Ponašanje provjerava 30 testnih pitanja (`backend/tests/test_assistant.py`).

19. **Jeste li sami pisali kod?**
    Pri pisanju koda, testova i dokumentacije koristili smo alat Claude Code (Anthropic), a odluke o izvorima, pragovima i arhitekturi pregledavao je i donosio tim. Pravila natjecanja (čl. VIII.) dopuštaju AI alate uz navođenje, pa smo ih naveli u README-u i `docs/ALATI_I_LICENCE.md`. Plan i pravila za alat su u `CLAUDE.md`, npr. „nikad izmišljeni podaci” i „svaka vrijednost ima izvor”.

**Korisnici, licence, privatnost i održavanje**

20. **Spremate li adrese koje ljudi upisuju? Što je s GDPR-om?**
    Ne spremamo:
    - API i Caddy ne vode pristupne zapise (`deploy/Caddyfile`: `output discard`; Uvicorn `--no-access-log`);
    - nema kolačića za praćenje ni analitike;
    - PVGIS dobiva koordinate zaokružene na 0,01°;
    - ako je uključen jezični model, njegovu pružatelju ide samo tekst pitanja, bez adrese;
    - iz HZZO popisa ne prikazujemo imena liječnika.

21. **Pod kojom je licencom rješenje i smijete li koristiti sve izvore?**
    Kod je pod EUPL-1.2, a izvedeni podaci pod CC BY 4.0. Izvore koristimo prema njihovim licencama: Otvorena dozvola RH, CC BY 4.0, ODbL za OSM. Za HZZO i HŽPP licenca nije navedena, pa prije službenog uvođenja preporučujemo pisanu potvrdu izdavača. Karte Hrvatskih voda i karte buke EEA-e samo povezujemo.

22. **Tko bi ovo održavao?**
    Treba vlasnik usluge, npr. jedinica lokalne samouprave, razvojna agencija ili tijelo nadležno za Portal (`docs/ARHITEKTURA.md`, organizacijski preduvjeti). Tehnički je to mjesečni `make data` i jedan Docker Compose. Vozni red HŽPP-a treba osvježiti nakon 13. 12. 2026., a URL-ove HZZO-a ručno ažurirati kad se promijene.

23. **Koliko je rješenje testirano?**
    - backend: 123 testne funkcije u pytestu, uključujući zlatne testove izvještaja za demo adrese i 30 pitanja asistenta;
    - frontend: 67 jediničnih testova (Vitest) i e2e testovi u Playwrightu s automatskom provjerom pristupačnosti (axe);
    - sve pokreće `make verify`;
    - ručno testiranje i popravci opisani su u `docs/NALAZI_TESTIRANJA.md`.

24. **Je li dostupno ljudima s invaliditetom?**
    Rađeno je prema WCAG 2.2 AA, a axe ne javlja nalaze ni na jednoj stranici, u svijetloj i tamnoj temi. Sve što je na karti dostupno je i u tekstu, a radi i na mobitelu od 320 px. Provjeru s čitačem zaslona i stvarnim korisnicima još nismo napravili.

25. **Može li ovo raditi i za druge države?**
    Okolišni izvori (EEA, JRC, ESA, EFFIS, PVGIS, OSM) pokrivaju cijelu Europu. Domaći dio (adrese, registri, popis, vozni redovi) treba nove konektore u `backend/ingest/`. Arhitektura i metodologija ostaju iste.

---

## 13. Rječnik pojmova

| Pojam | Objašnjenje |
|---|---|
| API | Sučelje preko kojeg jedan program traži podatke od drugoga (npr. preglednik od našeg poslužitelja). |
| REST | Način izrade API-ja u kojem se resursi dohvaćaju HTTP zahtjevima na adrese poput `/api/report`. |
| OpenAPI | Standardni strojno čitljivi opis API-ja; kod nas automatski na `/api/docs`. |
| JSON | Tekstualni format za razmjenu strukturiranih podataka. |
| GeoJSON | JSON za geografske podatke (točke, linije, poligoni), standard RFC 7946. |
| CKAN | Softver na kojem radi data.gov.hr; ima API za pretragu skupova i njihove metapodatke. |
| INSPIRE | Europska direktiva i skup standarda za prostorne podatke; DGU po njemu objavljuje adrese. |
| GML | XML format za geografske podatke koji koristi INSPIRE. |
| GTFS | Standardni format voznog reda (CSV datoteke: linije, putovanja, stajališta, vremena). |
| WFS | Standardni web servis za dohvat vektorskih geopodataka (kod nas EFFIS). |
| ATOM | Format za objavu datoteka za preuzimanje; DGU tako objavljuje INSPIRE skupove. |
| Raster / GeoTIFF | Slika u kojoj svaki piksel ima vrijednost (npr. dubinu vode) i poznati položaj na Zemlji. |
| COG | GeoTIFF organiziran u pločice, pa se može čitati samo dio datoteke. |
| Parquet / GeoParquet | Stupčasti format za tablice; GeoParquet uz to sprema geometriju. |
| PBF | Sažeti binarni format izvatka OpenStreetMapa. |
| OpenStreetMap (OSM) | Otvorena karta svijeta koju uređuju dobrovoljci (ceste, trgovine, parkovi…). |
| EPSG:4326 (WGS84) | Koordinate u stupnjevima (geografska širina i dužina), kao GPS. |
| EPSG:3765 (HTRS96/TM) | Službeni hrvatski projekcijski sustav u metrima; koristimo ga za udaljenosti. |
| EPSG:3035 (ETRS89-LAEA) | Europski projekcijski sustav u metrima; u njemu su EEA i INSPIRE podaci. |
| Geokodiranje | Pretvaranje tekstualne adrese u koordinate. |
| Normalizacija teksta | Uklanjanje dijakritika, velikih slova i interpunkcije da se nazivi mogu uspoređivati. |
| Prostorni spoj (spatial join) | Spajanje dvaju skupova po položaju (npr. točka unutar poligona) umjesto po šifri. |
| STRtree | Prostorni indeks koji brzo pronalazi geometrije u blizini točke. |
| Reprezentativna točka | Točka za koju je zajamčeno da leži unutar poligona. |
| Izokrona | Područje dostupno u zadanom vremenu (npr. 10 minuta hoda) po stvarnoj mreži. |
| Usmjerivač (router) | Program koji računa put i vrijeme putovanja po mreži ulica; kod nas Valhalla. |
| Valhalla | Otvoreni usmjerivač nad OSM-om; računa izokrone, matrice vremena i rute. |
| Matrica vremena (sources_to_targets) | Jedan upit koji vraća vrijeme hoda od jedne točke do mnogo odredišta. |
| Polyline | Sažeto kodirani zapis niza koordinata (oblik rute). |
| Povratno razdoblje | Prosječan razmak između poplava te veličine; „100 godina” znači 1 % vjerojatnosti svake godine. |
| PM2,5 | Čestice prašine manje od 2,5 mikrometra, štetne za pluća. |
| NO2 | Dušikov dioksid, plin koji uglavnom dolazi iz prometa. |
| µg/m³ | Mikrogram po kubnom metru, mjera koncentracije onečišćenja u zraku. |
| Pravilo 3-30-300 | Smjernica za gradove: 3 stabla vidljiva iz doma, 30 % krošnji u četvrti, park do 300 m. |
| kWp / kWh | Vršna snaga solarne elektrane / proizvedena energija. |
| PVGIS | Alat Europske komisije za procjenu proizvodnje solarne elektrane. |
| Medijan | Srednja vrijednost poredanog niza; na nju ne utječu ekstremi. |
| Interkvartilni raspon | Raspon između 25. i 75. percentila; tu je polovica vrijednosti. |
| Ponderirani prosjek | Prosjek u kojem svaka vrijednost ima svoju težinu (važnost). |
| Normalizacija bodova | Pretvaranje različitih mjernih jedinica u istu ljestvicu 0–100. |
| Random Forest | Model strojnog učenja koji uprosječuje mnogo stabala odlučivanja. |
| Linearna regresija | Model koji predviđa vrijednost kao ponderirani zbroj ulaznih značajki. |
| Leave-one-station-out | Provjera u kojoj se svaka postaja predviđa modelom treniranim bez nje. |
| RMSE / MAE | Prosječna veličina pogreške predviđanja; RMSE jače kažnjava velike pogreške. |
| Pristranost (bias) | Prosječno sustavno odstupanje predviđanja (previše ili premalo). |
| Jezični model (LLM) | Model umjetne inteligencije koji razumije i piše tekst. |
| OpenAI-kompatibilan API | Standardni oblik poziva jezičnog modela koji podržavaju mnogi pružatelji i lokalni modeli. |
| Namjera (intent) | Što korisnik pitanjem želi (npr. činjenica iz izvještaja, objašnjenje pojma). |
| Predložak | Unaprijed napisana rečenica u koju se umeću izračunate vrijednosti. |
| Deterministički | Isti ulaz uvijek daje isti izlaz. |
| Predmemorija (cache) | Spremljeni rezultat koji se ponovno koristi umjesto novog izračuna ili upita. |
| FastAPI | Python okvir za izradu API-ja. |
| Pydantic | Python biblioteka za opis i provjeru struktura podataka. |
| React | JavaScript biblioteka za izradu korisničkog sučelja. |
| TypeScript | JavaScript s tipovima, koji otkriva greške prije pokretanja. |
| Vite | Alat koji gradi frontend i pokreće razvojni poslužitelj. |
| MapLibre GL JS | Otvorena biblioteka za interaktivne karte u pregledniku. |
| OpenFreeMap | Besplatan servis kartografskih podloga bez ključa. |
| Docker / Docker Compose | Pakiranje programa u kontejnere / pokretanje više kontejnera jednom naredbom. |
| Caddy | Web poslužitelj koji automatski dobiva HTTPS certifikat. |
| Reverse proxy | Poslužitelj koji prima zahtjeve i prosljeđuje ih drugom servisu (Caddy → API). |
| uv | Brzi upravitelj Python paketa i okruženja. |
| pytest / Vitest / Playwright | Alati za testove backenda, frontenda i testove u pravom pregledniku. |
| axe-core | Alat za automatsku provjeru pristupačnosti web stranica. |
| WCAG 2.2 AA | Međunarodni standard pristupačnosti weba (razina AA). |
| ruff / mypy | Provjera stila i grešaka u Pythonu / provjera tipova. |
| Zlatni test | Test koji uspoređuje rezultat s unaprijed spremljenim točnim rezultatom. |
| EUPL-1.2 | Licenca otvorenog koda Europske unije. |
| ODbL | Licenca otvorene baze podataka (OSM); izvedene baze moraju ostati pod njom. |
| CC BY 4.0 | Licenca koja dopušta upotrebu uz navođenje autora. |
| Otvorena dozvola RH | Hrvatska licenca za ponovnu uporabu informacija javnog sektora. |
| Manifest | Datoteka `manifest.json` s izvorom, licencom i datumom svakog skupa. |
| User-Agent | Zaglavlje kojim se naš program predstavlja poslužitelju izvora. |
| Ocjena mjesta | Naša skupna ocjena 0–100 s vidljivim i podesivim težinama. |
| Profil | Skup težina i uvjeta za tip korisnika (obitelj, umirovljenik, student…). |
| GDPR | Opća uredba EU-a o zaštiti osobnih podataka. |

---

## 14. Plan prezentacije od 7 minuta

Ukupno 8 slajdova i oko 590 izgovorenih riječi. Uz oko 125 riječi u minuti to je oko 5 minuta govora, a ostatak vremena ide na klikanje u demou i prijelaze između slajdova.

### Slajd 1: Kako je živjeti ovdje? (0:00–0:45)
**Na slajdu:** naziv „Dobro mjesto”, slika naslovnice (`docs/images/naslovnica.png`), tim Požega, FERIT, izazov B.

**Govor:** „Dobar dan. Mi smo tim Požega s FERIT-a u Osijeku. Gdje živjeti je najveća odluka većine ljudi, a često i najveći kredit. Kad kupujemo ili unajmljujemo, gledamo cijenu, kvadraturu i kat. A ono što zapravo određuje svakodnevni život, je li kuća u poplavnom području, kakav je zrak, ima li hlada, može li se do vrtića ili ljekarne bez auta, otkrijemo tek kad uselimo. Zato smo napravili Dobro mjesto.”

### Slajd 2: Problem: podaci postoje, ali nisu za ljude (0:45–1:35)
**Na slajdu:** logotipi ili nazivi izvora (DGU, DZS, MZOM, HZZO, gradovi, EEA, JRC, ESA, OSM) i formati (GML 2,6 GB, GeoTIFF, GTFS, Excel bez koordinata).

**Govor:** „Svi ti podaci postoje i većinom su otvoreni. Ali razbacani su po desetak institucija i formata: adrese u INSPIRE datoteci od 2,6 gigabajta, karte poplava i zraka kao rasteri, vozni redovi u GTFS-u, a registri vrtića i HZZO-ovi popisi ordinacija uopće nemaju koordinate. Prosječan čovjek ne može provjeriti ni jednu adresu. Mi to radimo umjesto njega i odgovor kažemo jednostavnim jezikom, s izvorom za svaku brojku.”

### Slajd 3: Podaci: 34 skupa, 22 s Portala (1:35–2:30)
**Na slajdu:** tablica po temama (adrese, stanovništvo, ustanove, zdravstvo, prijevoz, okoliš, reciklaža), istaknut broj 22 s data.gov.hr i dva API-ja uživo (PVGIS, CKAN).

**Govor:** „Koristimo 34 skupa otvorenih podataka, od toga 22 s Portala otvorenih podataka: adrese i upravne jedinice Državne geodetske uprave, Popis 2021., registre vrtića i škola, vozne redove HŽPP-a i Autotroleja i jedanaest skupova gradova i općina o reciklaži. Europske izvore koristimo samo za ono čega domaćeg nema: kvalitetu zraka, satelitski pokrov tla i karte poplava. Uživo pitamo dva API-ja: PVGIS za sunce i CKAN API Portala, pa uz svaki skup prikazujemo kad je zadnji put izmijenjen.”

### Slajd 4: Obrada: od rasutih datoteka do jedne točke (2:30–3:30)
**Na slajdu:** dijagram toka: izvori → `make data` (16 koraka) → Parquet/GeoTIFF + manifest → API. Natuknice: vlastito geokodiranje (86–95 %), spajanje DZS–DGU po nazivu (6.633 od 6.759), medijan adresa po naselju.

**Govor:** „Cijela obrada pokreće se jednom naredbom i ima šesnaest koraka. Adrese čitamo strujno, bez raspakiravanja. Registre bez koordinata sami geokodiramo nad milijun i sedamsto tisuća adresa i uspijevamo u 86 do 95 posto slučajeva. Stanovništvo DZS-a spajamo s naseljima po nazivu, jer DZS nema šifre. Za cijela naselja ne gledamo jednu točku, nego medijan svih adresa, jer je središte Splita u pješačkoj zoni, a središte sela na glavnoj cesti. Svaki korak zapisuje izvor, licencu i datum.”

### Slajd 5: Demo (3:30–5:00)
**Na slajdu:** uživo `dobromjesto.domagojlukas.online` (rezerva: snimke `izvjestaj.png`, `zivot-bez-auta.png`, `predlozi-rezultati.png`).

**Govor (uz klikanje):** „Upišem adresu, recimo gradsku upravu u Vinkovcima. Za sekundu dobijem iskaznicu mjesta: izvan je poplavnih područja, zrak je umjeren, gusto je izgrađeno i ima malo drveća, a velika cesta je 790 metara daleko. Ispod je život bez auta: zone od 5, 10 i 15 minuta hoda po stvarnim ulicama i šest od sedam svakodnevnih potreba pješice. Usporedim s Nijemcima: tamo je puno prirode, ali je glavna cesta samo 40 metara daleko. Pitam asistenta: ‚Je li ova kuća u poplavnom području?’, a on kaže ‚Ne’, uz izvor. I na kraju ‚Pronađi mjesto za mene’: umirovljenik, Osječko-baranjska županija, i dobijem naselja s ljekarnom, liječnikom i trgovinom pješice, uz razloge.”

### Slajd 6: Arhitektura (5:00–5:45)
**Na slajdu:** jednostavan dijagram: preglednik (React, MapLibre) → Caddy → FastAPI → podaci u memoriji + Valhalla; strelice prema PVGIS-u i CKAN-u. Natuknice: REST + OpenAPI, GeoJSON, INSPIRE, GTFS; Docker Compose; 2 vCPU, 4 GB RAM.

**Govor:** „Arhitektura je jednostavna. Backend je FastAPI, vremena hoda računa otvoreni usmjerivač Valhalla nad OpenStreetMapom, a sučelje je React s MapLibre kartama. Sve se pokreće jednim Docker Compose naredbom na skromnom poslužitelju. API je otvoren i opisan OpenAPI-jem, a izvještaj je dostupan i kao GeoJSON, pa ga oglasnik, banka ili grad mogu ugraditi u svoju uslugu. Ne spremamo nijednu upisanu adresu ni pitanje.”

### Slajd 7: Vrijednost i povjerenje (5:45–6:25)
**Na slajdu:** tri natuknice: „svaka brojka ima izvor, datum i vrstu”, „AI ne izmišlja brojke”, „ML smo testirali i nismo ga uključili jer nije bio dovoljno dobar”; plus „nalazi o kvaliteti podataka vraćeni izdavačima”.

**Govor:** „Najviše smo pazili na povjerenje. Svaka brojka nosi izvor, datum i oznaku je li izmjerena ili procijenjena. Jezični model u asistentu samo razumije pitanje, a brojke dolaze iz izračuna. Isprobali smo i strojno učenje za zrak na razini ulice. Za NO2 je bilo 15 posto bolje, ali za PM2,5 lošije, pa ga po pravilu koje smo postavili unaprijed nismo uključili. Usput smo zapisali probleme u podacima na Portalu, poput općinskih skupova bez koordinata, i to može koristiti izdavačima.”

### Slajd 8: Daljnji razvoj (6:25–7:00)
**Na slajdu:** automatsko osvježavanje, vozni redovi ostalih gradova (GTFS), model NO2 uz gradske mreže, nacionalni sloj reciklaže natrag na Portal, ugradnja u oglasnike i banke.

**Govor:** „Dalje želimo automatsko mjesečno osvježavanje podataka, vozne redove Splita, Osijeka i drugih gradova čim budu objavljeni, i precizniji model dušikovog dioksida uz gradske mjerne mreže. Spojeni sloj reciklažnih dvorišta možemo vratiti na Portal kao jedan nacionalni skup. Cilj je da svatko, prije nego što potpiše ugovor, zna kako je živjeti na toj adresi. Hvala.”

---

## Popis mjesta označenih s [PROVJERITI]

1. Licenca HZZO-ovih popisa liječnika, pedijatara i ljekarni nije navedena (odjeljak 4).
2. Licenca HŽPP GTFS-a nije navedena (odjeljak 4).
3. Licenca JRC karata poplava navedena je dvaput različito: ODbL u katalogu, CC BY 4.0 u `copyright.txt` (odjeljak 4).
4. GTFS Šibenika: kod i `docs/ODLUKE.md` navode Otvorenu dozvolu RH, a `PROGRESS.md` ga vodi pod „Odustano” zbog licence (odjeljak 4).
5. Vrtići i škole Grada Zagreba preuzimaju se s `opendata.arcgis.com`, a u manifestu je stranica na data.gov.hr: treba potvrditi da je riječ o istom resursu (odjeljak 4).
6. Je li na javnom demou uključen jezični model za asistenta (odjeljak 6.3).

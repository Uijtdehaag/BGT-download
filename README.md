# BGT-download - CODE VIBE WITH CLAUDE
Download een deel van de Basiskaart Grootschalige Topografie met deze BGT viewer.
Bekijk de demo: https://uijtdehaag.github.io/BGT-download/

# Herbruikbare AI prompt — BGT download webapp

> Plak alles onder de streep aan een capabel codemodel. Het produceert één bestand: `index.html`.

---

Bouw een **single-page webapp** in **één bestand `index.html`** (inline CSS + JS, geen build-stap) genaamd **"BGT download"** met subtitel **"Basiskaart Grootschalige Topografie"**. De app laat de gebruiker BGT-vlakken op een kaart selecteren, hun attributen bekijken/exporteren, en het officiële BGT-extract downloaden via de PDOK Download API. Taal van de interface: Nederlands.

## Stack
- Eén `index.html`, Leaflet (via CDN) voor de kaart. Geen frameworks/bundler. Geen `localStorage`/`sessionStorage`.
- Nette, rustige UI: zijbalk links (zoeken, gereedschappen, filters, selectielijst, download- en exportknoppen), kaart rechts. Inklapbare zijbalk op desktop, hamburger op mobiel.

## Kaartlagen
- **Basiskaart:** CartoDB Positron tegels.
- **BGT-basiskaart (default aan):** PDOK WMTS
  `https://service.pdok.nl/lv/bgt/wmts/v1_0/achtergrondvisualisatie/EPSG:3857/{z}/{x}/{y}.png`
  Deze laag is dekkend; zet de opacity op **0,5 zodra de luchtfoto aan staat**, anders 1.
- **Luchtfoto (toggle):** PDOK Actueel Ortho HR (WMTS, EPSG:3857).
- Twee Leaflet `layerGroup`s bovenop alles: `selLayer` (oranje gemarkeerde geselecteerde vlakken) en `markers`. Houd selectie altijd vooraan.

## Zoeken
Zoekbalk met autocomplete via PDOK Locatieserver (`api.pdok.nl/bzk/locatieserver/search/v3_1`). Een suggestie kiezen **navigeert alleen** de kaart naar die locatie (voegt niets toe aan de selectie).

## Vlakselectie — PDOK BGT OGC API Features
- Endpoint: `https://api.pdok.nl/lv/bgt/ogc/v1/collections/{type}/items?bbox=minLon,minLat,maxLon,maxLat&limit=N&f=json` (Accept: `application/geo+json`).
- Vlak-objecttypen: `wegdeel, waterdeel, begroeidterreindeel, onbegroeidterreindeel, ondersteunendwegdeel, ondersteunendwaterdeel, overbruggingsdeel, pand, overigbouwwerk, scheiding_vlak, kunstwerkdeel_vlak, tunneldeel`.
- Negeer historie: objecten met een gevulde `eind_registratie`.
- **Klikmodus:** vraag een kleine bbox rond het klikpunt op, doe ray-casting punt-in-polygoon, en selecteer het **kleinste omsluitende** vlak. Nogmaals klikken = deselecteren.
- **Polygoonmodus:** punten klikken, dubbelklik sluit de polygoon; voeg alle vlakken toe waarvan het **zwaartepunt binnen de polygoon** valt. Minimum 3 punten.
- Min. zoomniveau 14 voor selecteren; toon een hint in de kaarthoek.
- Bewaar per vlak: id, objecttype, lokaal_id, de **volledige `properties`**, berekende oppervlakte (shoelace met meter-correctie via `cos(lat)`), representatief punt (gemiddelde van de eerste ring), en de geometrie. Teken elk geselecteerd vlak oranje met popup.

## Filters
- **Objecttype-filter** (dropdown): "alle vlakken" of één specifiek type; bepaalt welke collecties doorzocht worden.
- **Bronhouder-filter** (dropdown): automatisch gevuld met de bronhouders die in de huidige selectie voorkomen; filtert de zichtbare lijst én de CSV-export. Teller toont `zichtbaar/totaal` als er gefilterd wordt. Uitgeschakeld als de selectie leeg is.

## Selectielijst (zijbalk)
Per geselecteerd vlak: objecttype-label, een samenvatting met functie/fysiek voorkomen + `Plus: <plus_fysiekvoorkomen>` + oppervlakte, een regel met bronhouder, en een inklapbare **"Toon alle attributen"**-tabel. **Toon het Lokaal ID NIET in de samenvatting** (wel in de attributentabel/popup/CSV). Klik op een item zoomt ernaartoe; verwijderknop per item; "Wis alles"-knop.

## Attributen opschonen
Maak een helper die bepaalt welke velden verborgen zijn: **alle velden waarvan de naam op `codespace` eindigt of `codespace` bevat**, plus geometrie-/interne velden (`geometry, geom, geometrie, id, gml_id`). Pas dit toe in lijst, popup **en** CSV. Voor weergave: strip een leidende codespace (alles vóór `:`, of een korte code vóór de eerste `-`), splits camelCase in losse woorden, en zet ISO-datums om naar NL-notatie. **In de CSV de ruwe waarden behouden.**

## CSV-export
Exporteer de zichtbare (bronhouder-gefilterde) selectie. Kolommen = `objecttype` + de unie van alle niet-verborgen attribuutsleutels + `oppervlakte_m2, lat, lon`. UTF-8 met BOM, dubbele-quote-escaping.

## BGT Download service (PDOK BGT Download API)
- Knop **"BGT downloaden (polygoon)"**, uitgeschakeld tot er een polygoon getekend is; weer uitgeschakeld bij wissen.
- **Formaatkeuze**-dropdown ernaast: `citygml` (CityGML/GML), `gmllight` (GML-light), `stufgeo` (StUF-Geo).
- Flow:
  1. `POST https://api.pdok.nl/lv/bgt/download/v1_0/full/custom` met JSON `{ featuretypes, format, geofilter }`, headers `Content-Type` en `Accept: application/json`. Verwacht **HTTP 202**.
  2. Lees `_links.status.href`, **poll** die status-URL elke ~2s tot `status === "COMPLETED"` (toon voortgang `progress`).
  3. Download via `_links.download.href` (relatief t.o.v. de status-URL); trigger de ZIP-download.
- **geofilter** = WKT `POLYGON((...))` in **EPSG:28992 (RD)**. Zet de getekende WGS84-ring (lat/lon) om naar RD met de Schreutelkamp/Strang-van-Hees-benadering (controlepunt Amersfoort → X 155000, Y 463000). Sluit de ring (eerste punt == laatste punt). Geen inner rings.
- **featuretypes** = het gekozen objecttype, of de volledige set bij "alle vlakken". Map de collectie-namen `scheiding_vlak`→`scheiding` en `kunstwerkdeel_vlak`→`kunstwerkdeel`.
- Statusvenster met voortgangsbalk, succes (incl. "download opnieuw"-link) en foutmeldingen.

## Info-popup
Een "i"-knop opent een modal met uitleg over de app, de databronnen (CartoDB, PDOK Ortho, PDOK BGT WMTS, PDOK BGT Features OGC API), de objecttypen, de CSV-export en de download-service. 

## Kwaliteitseisen
- Robuuste error handling (try/catch) rond alle fetches; faal stil per objecttype en ga door.
- Gebruik de actuele datum, geen verouderde jaartallen in queries.
- Lever uitsluitend het complete `index.html` op, klaar om te openen.

## Bekende aandachtspunten (vermeld deze in code-commentaar)
- CORS: de PDOK OGC- en Download-API's moeten cross-origin requests vanuit de browser toestaan; zo niet, dan is een kleine proxy nodig.
- Attribuut- en featuretype-namen kunnen per objecttype licht verschillen — toon daarom alle teruggegeven properties en verifieer download-featuretypes via de Swagger UI (`api.pdok.nl/lv/bgt/download/v1_0/ui/`).

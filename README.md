# Eisdiele Kasse San Remo

Browser-basierte Touch-Kasse für die Eisdiele San Remo, geschrieben in
Delphi 12.1 mit TMS Web Core (Trial v2.4.6.1, kompiliert mit pas2js 2.3.1).
Läuft im Browser auf Windows-Rechnern und Android-Tablets, ist
touch-optimiert und wird über GitHub Pages ausgeliefert.

![Lokales Bild](EisdieleV01Kl.png)

## Live-URL

`https://rwhexa.github.io/eisdielerw/EisdieleKasse.html`

Repository: `https://github.com/rwhexa/eisdielerw`

## Funktionsumfang (aktueller Stand)

- **Tisch-Verwaltung** für Tisch 1–8, Theke und Mitnahme
- **Eigener Warenkorb pro Tisch**, der beim Tischwechsel automatisch
  gespeichert wird – Nachbestellungen addieren sich zur bestehenden
  Bestellung
- **Tisch-Leiste** oben zeigt jederzeit alle offenen Tische mit ihrer
  Summe; aktive Tische in Koralle, belegte Tische in Mint
- **MwSt-Logik:** Tische 1–8 standardmäßig Vor Ort (19 %), Theke und
  Mitnahme standardmäßig zum Mitnehmen (7 %); pro Tisch umschaltbar;
  Getränke immer 19 %
- **Artikel-Kacheln** in vier Kategorien: Eis, Becher, Waffel, Getränk
- **Netto/MwSt/Brutto-Splittung** im Warenkorb
- **Bar/Karte/Storno-Buttons** als Platzhalter für spätere
  TSE-Anbindung und Kartenterminal
- **Eisdielen-Pastell-Farbschema** mit Koralle/Mint/Cremé
- **RW-Logo** und **Versions-Anzeige** in der Topbar
- **localStorage-Persistenz** aller offenen Tische als Absturz-Sicherung

## Projektstruktur

```
EisdieleKasse/
├── EisdieleKasse.dpr      Projektdatei
├── EisdieleKasse.html     Project-HTML mit CSS, Boot-Wrapper, PWA-Hooks
├── UMain.pas              Main-Unit mit Datenmodell und Form-Aufbau
├── UMain.dfm              Form-Definition (epIgnore, efCSS)
├── UMain.html             Platzhalter (leer, aber von TMS referenziert)
├── logorw.png             RW-Logo mit transparentem Hintergrund (256x256)
├── index.html             Redirect-Stub für Aufruf der Repo-Root-URL
├── manifest.json          PWA-Manifest (für späteren PWA-Ausbau bereit)
├── service-worker.js      Service Worker (für späteren PWA-Ausbau bereit)
└── README.md              Dieses Dokument
```

Im Output-Verzeichnis `TMSWeb\Release\` landet nach dem Compile:

- `EisdieleKasse.html` – gerenderte HTML (vom Build-Prozess)
- `EisdieleKasse.js` – kompilierte JS, ca. 2,4 MB

Wichtig: **logorw.png, manifest.json und service-worker.js müssen manuell**
nach `TMSWeb\Release` kopiert werden, da TMS Web Core sie beim Build
nicht automatisch übernimmt. Gleiches gilt für `UMain.html` – die wird
vom kompilierten JS zur Laufzeit referenziert, also auch ins Release.

## Datenmodell

```
TProduct
├── Id, Cat, Nm (Name), Pr (Preis brutto), Ico (Emoji)
└── Vat19: Boolean (true = immer 19 %, z. B. Getränke)

TOrderItem
├── Product: TProduct
└── Qty: Integer

TTable
├── Id, Name, Mode (cmTakeAway / cmEatHere)
└── Items: array of TOrderItem
```

`TForm1` hält dynamische Arrays `FProducts`, `FTables` und einen Zeiger
`FActiveTable` auf den gerade aktiven Tisch.

## localStorage-Persistenz

Der komplette Tisch-Zustand wird nach jeder Bestelländerung in den
localStorage geschrieben (`StoreTablesState`) und beim App-Start wieder
geladen (`RestoreTablesState`). Speicher-Key: `sanremo_state_v1`.

Was gespeichert wird:

- Pro Tisch: ID, Verzehr-Modus
- Pro Artikel: nur **Produkt-ID + Menge** (nicht Name und Preis)

Beim Laden werden die Produkt-IDs wieder mit den echten Produkten aus
`SeedProducts` verknüpft. Dadurch wirken Preisänderungen auch auf bereits
offene Tische.

Robustheit:

- localStorage nicht verfügbar (Speicher voll etc.) → App läuft normal
  weiter, nur ohne Sicherung
- Korrupter Speicherinhalt → sauberer Neustart mit leeren Tischen
- Produkt-ID nicht mehr auffindbar → Artikel wird übersprungen, mit
  kurzem Hinweis (greift erst wenn später die JSON-Artikelliste kommt)
- Speicher-Key versionsiert – bei Formatänderung einfach `_v2` setzen,
  alter Stand wird automatisch ignoriert

## Bekannte Stolperfallen mit TMS Web Core Trial 2.4.6.1

Diese fünf Punkte sind in der Trial-Version 2.4.6.1 zu beheben. Bei einer
Vollversion oder neueren Releases können die Workarounds schrittweise
zurückgebaut werden.

### 1. pas2js Trial vergisst den rtl.run-Boot-Aufruf

Der pas2js-Output endet mit `});` und ohne den eigentlich erwarteten
`rtl.run('program');`-Aufruf am Dateiende. Folge: JS lädt, wird aber
nicht gestartet, Browser bleibt leer.

Workaround in `EisdieleKasse.html`:

```html
<script src="$(ProjectName).js"></script>
<script>
  if (typeof rtl !== 'undefined' && typeof rtl.run === 'function') {
    rtl.run('program');
  } else {
    alert('FEHLER: rtl ist nicht definiert.');
  }
</script>
```

### 2. Generics überfordern den Trial-Compiler

`TObjectList<T>` mit drei verschiedenen Typparametern (TProduct, TTable,
TOrderItem) bricht den JS-Output mittendrin ab.

Workaround: Statt Generics-Container dynamische Arrays:

```pascal
FProducts: array of TProduct;
FTables  : array of TTable;
```

Plus eigene Helper-Methoden `AddProduct`, `AddTable`, `FreeProducts`,
`FreeTables`.

### 3. ElementPosition und ElementFont müssen explizit gesetzt werden

Default: Komponenten bekommen Inline-Styles mit
`position: absolute; width: 100px; height: 25px` mitgegeben, die CSS-Klassen
überschreiben.

Workaround auf jedem programmatisch erzeugten Control:

```pascal
Btn := TWebButton.Create(Self);
Btn.ElementPosition := epIgnore;
Btn.ElementFont := efCSS;
Btn.Parent := pnlTop;
Btn.ElementClassName := 'mode-btn';
```

In der DFM analog:

```
ElementClassName = 'host'
ElementFont = efCSS
ElementPosition = epIgnore
```

### 4. TMS injiziert anonyme div-Wrapper

Zwischen einem `<span class="kasse-root">` und seinen Kindern liegt ein
unstyled `<div>`. Grid- und Flex-Layouts der Kinder brechen zusammen.

Workaround in CSS:

```css
.kasse-root > div, .main > div, .cart > div, .totals > div /* etc. */ {
  display: contents !important;
}
```

### 5. Inline-Styles brauchen !important-Override

TMS gibt trotz `epIgnore` weiter inline `width: 100px; height: 25px` mit.

Workaround in CSS: Container-Klassen mit `!important` auf `width: auto`,
`height: auto` und gewünschten `display`-Modus zwingen.

### 6. Beim Deployment alles aus TMSWeb\Release committen

Die kompilierte JS-Datei referenziert intern `UMain.html`. Wird diese
nicht mitgepusht, kommt eine 404-Antwort vom GitHub-Pages-Server, die
sich in der Anzeige mit der Kasse mischt. Reihenfolge der Sichtbarkeit:
oben die 404-Meldung, weiter unten die korrekt gerenderte Kasse.

### 7. Geerbte Methoden nicht überschreiben

`TWebForm` erbt von `TControl`, das schon `SaveState`/`LoadState`
besitzt. Deshalb wurden die Persistenz-Methoden bewusst
`StoreTablesState`/`RestoreTablesState` benannt.

## Build und Start

1. Projekt in Delphi 12.1 öffnen
2. Build-Konfiguration auf **Release** stellen (Projekt-Manager →
   Build Configurations → Rechtsklick auf Release → Aktivieren)
3. **Projekt → Neu erstellen** (Strg+F9 reicht nicht für saubere
   HTML-Übernahme)
4. F9 zum Starten – TMS Web Core öffnet den Browser automatisch über den
   eingebauten Dev-Server auf `http://localhost:8000/`
5. Bei Änderungen: Strg+F5 im Browser für Hard-Reload (Cache umgehen)

## Deployment auf GitHub Pages

1. Inhalt aus `TMSWeb\Release\` ins Repo `eisdielerw` committen:
   - `EisdieleKasse.html`
   - `EisdieleKasse.js`
   - `UMain.html`
2. Zusätzlich (einmalig, beim Update bei Bedarf):
   - `logorw.png`
   - `index.html`
   - `manifest.json`
   - `service-worker.js`
3. Push nach `main`
4. GitHub Pages liefert die Kasse unter der oben genannten URL aus

## Betrieb auf Android-Tablet

Empfohlene Browser-App: **Fully Kiosk Browser** (Vollbild ohne
Adresszeile, gegen versehentliches Verlassen sperrbar, Auto-Start).

Wichtige Einstellungen:

- **Start URL** auf die Live-URL setzen
- **Keep Screen On** an
- **Kiosk Mode** mit PIN aktivieren
- **Auto-Confirm JS Dialogs** an (umgeht Trial-Nag-Dialog)
- **Start on Boot** an (Kasse kommt nach Tablet-Einschalten automatisch)

## Geplante nächste Schritte

| Modul | Aufwand | Beschreibung |
|---|---|---|
| **JSON-Artikelliste** | mittel | `SeedProducts` ersetzen durch Fetch einer `produkte.json` zur Laufzeit. Artikel-Pflege ohne Compile möglich. **Nächste Aufgabe.** |
| **PWA-Aktivierung** | klein | Manifest und Service Worker sind vorhanden, müssen nur committed werden. Home-Screen-Installation + Offline-Betrieb |
| **Artikelverwaltung in der App** | groß | UI zum Anlegen/Ändern von Artikeln. Voraussetzung: Backend oder zumindest Import/Export der JSON über die App |
| **Tagesabschluss / Z-Bericht** | mittel | Verkaufsliste mit CSV-Export für den Steuerberater |
| **Bondruck** | mittel | ESC/POS via Netzwerkdrucker (z. B. Epson TM-T20), HTTP-Brücke als Helper |
| **TSE-Anbindung** | groß | **Pflicht im Echtbetrieb** (Kassensicherungsverordnung). Cloud-TSE (fiskaly, Deutsche Fiskal) per REST oder Hardware-TSE (Swissbit USB) |
| **Mehrere Kassen synchron** | groß | REST-Backend (Supabase, eigene API) oder MQTT-Sync |

## Versionsinfo

- Delphi 12.1 Athens
- TMS Web Core Trial v2.4.6.1
- pas2js 2.3.1 (Build 2023-11-19)
- App-Version: v0.1 (sichtbar in der Topbar)
- Getestet in Chrome (Windows) und Fully Kiosk Browser (Android-Tablet 7")

## TSE-Hinweis für Echtbetrieb

Sobald die Software echte Verkäufe abwickelt, schreibt die deutsche
Kassensicherungsverordnung eine zertifizierte TSE vor. Ohne TSE darf die
Kasse nicht eingesetzt werden. Optionen:

- **Cloud-TSE** (fiskaly, Deutsche Fiskal): einfache HTTPS-Anbindung,
  ca. 10–15 € pro Monat
- **Hardware-TSE** (Swissbit USB, Epson TSE-Drucker): einmalig teurer,
  funktioniert offline

Der Bon muss zusätzlich QR-Code, Signatur, Transaktionszähler und
Seriennummer der TSE enthalten – das übernimmt die TSE-API.

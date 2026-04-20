# GANDO – AV-Import DM.01-AV-CH (ITF) nach QGIS

Dieses Dokument ist der persistente Projekt-Kontext für Claude. Grundlage ist
`Umsetzungsplan_AV_Import_DM01AVCH_QGIS.docx` (Version 1.0, 20.04.2026, Status: Entwurf zur Freigabe).

## Projekt-Metadaten

| Feld | Inhalt |
|---|---|
| Projekt | GANDO – WebGIS / GIS-Automatisierung |
| Auftraggeber | Ganeos AG |
| Autor | Stefan (Projektlead) |
| Dokumenttyp | Umsetzungsplan / Konzept |
| Version | 1.0 |
| Datum | 20.04.2026 |
| Status | Entwurf zur Freigabe |

## 1. Ausgangslage und Zielsetzung

AV93-Daten im INTERLIS-1-Transferformat (ITF) nach Bundesmodell **DM.01-AV-CH**
automatisiert in QGIS einlesen und mit den offiziellen Darstellungsempfehlungen
(swisstopo / Eidg. Vermessungsdirektion gemäss AV-WMS / MOpublic) stilisieren.

Erst nach Freigabe dieses Plans wird mit der Umsetzung begonnen.

### 1.1 Festgelegte Rahmenbedingungen
- Ausschliesslich Bundesmodell **DM.01-AV-CH** (Version 24, 04.06.2004). Keine kantonalen Mehranforderungen.
- Darstellungsvorgabe: offizielle Styles des Bundes (AV-WMS / MOpublic). Falls keine QMLs verfügbar, werden QMLs nach Spezifikation aufgebaut.
- Umsetzung als **Standalone-Python-Skript** auf Basis von **PyQGIS** (kein Plugin).
- Eingangsformat: **INTERLIS 1 / ITF**.
- Bezugsrahmen: **LV95 (EPSG:2056)**.

### 1.2 Nicht-Ziele
- Kein INTERLIS 2 / XTF.
- Keine kantonalen Modelle (BE, ZH, SG …).
- Keine Schreib-/Export-Funktion.
- Kein GUI / kein Plugin.

## 2. Grundlagen

### 2.1 Datenmodell DM.01-AV-CH
Bundesmodell 2001 der amtlichen Vermessung in INTERLIS 1. Gültig bis 31.12.2027
(danach Ablösung durch DMAV).

**Topics pro ITF-Transferfile (eine Gemeinde):**

| Topic | Inhalt |
|---|---|
| FixpunkteKategorie1/2/3 | LFP, HFP, Hilfsfixpunkte |
| Bodenbedeckung | Flächen- und Linienelemente BB |
| Einzelobjekte | Flächen-, Linien- und Punkt-EO |
| Hoehen | Höhenpunkte, Höhenkurven |
| Nomenklatur | Flur- und Geländenamen |
| Liegenschaften | Grundstücke, Projektgrundstücke |
| Rohrleitungen | Leitungsnetze der AV |
| Nummerierungsbereiche | NUM / Vermessungsbezirke |
| Hoheitsgrenzen | Gemeinde-, Bezirks-, Kantons-, Landesgrenzen |
| Planeinteilungen | Planeinteilung Grundbuchplan |
| TSEinteilung | Toleranzstufen |
| Rutschgebiete | Dauernde Bodenverschiebungen |
| Gebaeudeadressen | Gebäudeadressen, Eingänge |
| PLZOrtschaft | Postleitzahlbereiche |
| Planrahmen | Planrahmen PfG |

### 2.2 Platzierung von Beschriftungen
Nummern, Texte, Symbole über Attribute **Pos / Ori / HAli / VAli**:
- `HAli` = Left, `VAli` = Bottom
- `Ori` = 100 gon für Nummern/Texte, 0 gon für Symbole
- Umsetzung über datendefinierte Eigenschaften in QML.

## 3. Architektur und Projektstruktur

Standalone-Python-Paket, CLI-startbar oder aus QGIS-Standalone. Nutzt PyQGIS
und ili2gpkg (Java) für ITF-Konvertierung.

### 3.1 Verzeichnisstruktur

```
gando_av_import/
├── config.yaml                 Konfiguration (Pfade, Layer-Style-Mapping)
├── import_av.py                Einstiegspunkt / CLI
├── requirements.txt
├── lib/
│   ├── validator.py            ilivalidator-Aufruf (ITF-Prüfung)
│   ├── converter.py            ili2gpkg-Aufruf (ITF -> GPKG)
│   ├── loader.py               Layer in QGIS laden, CRS setzen
│   ├── styler.py               QML zuweisen, SVG-Pfad registrieren
│   └── project.py              Layerbaum, Gruppen, Massstab, Speichern
├── styles/
│   ├── qml/                    QML-Styles nach swisstopo / MOpublic
│   └── svg/                    SVG-Symbolbibliothek
├── models/
│   └── DM01AVCH24D.ili         Referenzmodell
├── templates/
│   └── av_projekt.qgz          QGIS-Projektvorlage
└── logs/
```

### 3.2 Komponentenübersicht

| Komponente | Aufgabe | Technologie |
|---|---|---|
| validator.py | Fachliche/formale Prüfung ITF gegen DM.01-AV-CH | ilivalidator (Java) |
| converter.py | ITF → GeoPackage, CRS 2056 | ili2gpkg (Java) |
| loader.py | Vektorlayer aus GPKG in QGIS laden | PyQGIS (QgsVectorLayer) |
| styler.py | QML pro Layer, SVG-Pfad registrieren | PyQGIS (loadNamedStyle) |
| project.py | Layerbaum, Gruppen, Massstabsbereiche, .qgz speichern | PyQGIS (QgsProject) |
| import_av.py | Orchestrierung, CLI, Logging | Python 3.11+ |

## 4. Verarbeitungs-Pipeline

Strikt sequenziell. Abbruch bei Fehler + Log-Protokoll.

1. Parameter/Konfiguration einlesen (`config.yaml`, CLI-Argumente).
2. ITF mit `ilivalidator` prüfen; Validierungsreport ablegen.
3. ITF mit `ili2gpkg` in GeoPackage konvertieren (ein Layer pro Klasse, EPSG:2056).
4. QGIS-Standalone initialisieren (`QgsApplication`), SVG-Suchpfad setzen.
5. Pro konfigurierte Klasse: Layer aus GPKG laden, CRS prüfen, dem Projekt hinzufügen.
6. Pro Layer zugehörige QML zuweisen (`loadNamedStyle`).
7. Layerbaum aufbauen (Gruppen, Reihenfolge, Massstabsbereiche, Sichtbarkeit).
8. Projekt als `.qgz` speichern; Zusammenfassungs-Log schreiben.

### 4.1 Prozessdiagramm

```
[ITF] -> ilivalidator -> [Validierungsreport]
           |
           v
     ili2gpkg -> [GeoPackage EPSG:2056]
           |
           v
PyQGIS: Layer laden -> QML zuweisen -> Layerbaum aufbauen
           |
           v
  [QGIS-Projekt .qgz + Logfiles]
```

## 5. Darstellung / Styling-Konzept

### 5.1 Strategie
- Eine QML-Datei pro Zielklasse (z. B. `bodenbedeckung_flaeche`, `liegenschaft`, `lfp1`).
- Regelbasierte Symbolisierung über `BB_Art`, `EO_Art`, `Fixpunktkategorie` usw.
- SVG-Symbole in `styles/svg/`, Pfad beim Start in QGIS registrieren.
- Massstabsabhängige Sichtbarkeit (1:500, 1:1000, 1:2000, 1:5000, Übersicht).
- Beschriftung datendefiniert über Pos/Ori/HAli/VAli.

### 5.2 Quellen
| Quelle | Nutzung |
|---|---|
| AV-WMS / MOpublic-Spezifikation (Bund) | Primärreferenz (Farben, Linien, Symbole, Layerstruktur) |
| cadastre.ch / cadastre-manual.admin.ch | Modelldokumentation + Objektkatalog |
| SVG-Bibliothek des Bundes (AV) | Symbole für EO, Fixpunkte, Gebäudeadressen |
| Projekteigene Ergänzungen | Nur wo offizielle QMLs fehlen; klar als intern markiert |

### 5.3 QML-Pakete
Swisstopo publiziert aktuell keine vollständig vorgefertigten QGIS-QML-Pakete für
DM.01-AV-CH. Fallback: eigene Umsetzung nach MOpublic. Falls während der Umsetzung
offizielle QMLs erscheinen, werden diese bevorzugt eingebunden.

## 6. Konfiguration (`config.yaml`)

```yaml
crs: EPSG:2056
ili_model: DM01AVCH24D

tools:
  ili2gpkg:     /opt/interlis/ili2gpkg.jar
  ilivalidator: /opt/interlis/ilivalidator.jar
  java:         /usr/bin/java

paths:
  input_itf:   ./data/input/
  output_gpkg: ./data/output/
  styles_qml:  ./styles/qml/
  styles_svg:  ./styles/svg/
  project_out: ./data/projects/
  logs:        ./logs/

layer_style_map:
  bodenbedeckung_flaeche: bb_flaeche.qml
  bodenbedeckung_linie:   bb_linie.qml
  einzelobjekt_flaeche:   eo_flaeche.qml
  einzelobjekt_linie:     eo_linie.qml
  einzelobjekt_punkt:     eo_punkt.qml
  liegenschaft:           ls_grundstueck.qml
  lfp1:                   fp_lfp1.qml
  lfp2:                   fp_lfp2.qml
  lfp3:                   fp_lfp3.qml
  gebaeudeadresse:        ga_gebaeudeadresse.qml
```

## 7. Umgebung

### 7.1 Software
| Komponente | Version / Hinweis |
|---|---|
| QGIS LTR | ≥ 3.34 (PyQGIS, Standalone) |
| Python | 3.11+ (QGIS-eigenes Python) |
| Java Runtime | ≥ 17 (für ili2gpkg / ilivalidator) |
| ili2gpkg | aktuellste Version |
| ilivalidator | aktuellste Version |
| OS | Linux (ARM64 / Raspberry Pi 5) und Windows |

### 7.2 Daten und Referenzen
- ITF-Testdatei einer Referenzgemeinde (stellt Stefan bereit).
- Modelldatei `DM01AVCH24D.ili` aus Model Repository.
- AV-WMS / MOpublic-Spezifikation.
- Offizielle SVG-Symbolbibliothek AV (soweit verfügbar).

### 7.3 Python-Abhängigkeiten
- `pyyaml` – Konfiguration
- `click` – CLI
- `logging` – Standardbibliothek, strukturierte Logs
- `PyQGIS` – aus QGIS-Installation (kein Pip-Paket)

## 8. Fehlerbehandlung und Logging
- Jeder Pipeline-Schritt kapselt Fehler, strukturierte Log-Zeile (Zeitstempel, Schritt, Status, Detail).
- ilivalidator-Fehler 1:1 als Report; Pipeline bricht ab.
- ili2gpkg-Fehler: Abbruch, unvollständige GPKG verwerfen.
- Fehlende QML/SVG: Warnung (kein Abbruch); Layer mit Standardstil.
- Abschlussbericht als JSON + Textlog pro Import.

## 9. Qualitätssicherung
- Testdatensatz einer Referenzgemeinde als Regressionsbasis.
- Automatisierter Lauf: ITF → ilivalidator → ili2gpkg → QGIS → Soll-Vergleich.
- Visueller Abgleich gegen AV-WMS / Plan für das Grundbuch bei ≥ 3 Massstäben.
- Stichprobenreview der QMLs gegen MOpublic (dokumentiert).

## 10. Meilensteine

| Nr. | Meilenstein | Lieferobjekt |
|---|---|---|
| M0 | Plan-Freigabe | Dieses Dokument, freigegeben |
| M1 | Tooling bereit | ili2gpkg + ilivalidator lauffähig, Testdatei validiert |
| M2 | Prototyp Konvertierung | ITF → GPKG, Layer in QGIS ladbar |
| M3 | Styling v1 | Erste QML-Sets nach MOpublic, drei Kern-Topics |
| M4 | Styling komplett | QML für alle relevanten Klassen, Layerbaum |
| M5 | Abnahme | Standalone-Skript + Doku + Testprotokoll |

## 11. Offene Punkte / Risiken
- Verfügbarkeit offizieller swisstopo-QMLs unklar; Fallback = eigene MOpublic-Umsetzung.
- Performance auf Raspberry Pi 5 (ARM64) für grosse Gemeinden früh testen.
- Unterschiedliche ITF-Versionen in der Praxis; Validator-Verhalten dokumentieren.
- Umgang mit leeren / optionalen Topics (Rutschgebiete, Rohrleitungen) im Layerbaum festlegen.

## 12. Freigabe

Verbindlicher Umsetzungsplan. Entwicklung startet erst nach schriftlicher Freigabe.

---

## Arbeitshinweise für Claude

- **Sprache:** Deutsch (Schweizer Hochdeutsch, "ss" statt "ß"), wie im Quelldokument.
- **Status:** Plan ist **Entwurf zur Freigabe** (M0). Vor Freigabe KEINE produktive Code-Entwicklung starten — nur Gerüste/Skelette.
- **Scope-Disziplin:** Nur DM.01-AV-CH, nur ITF, nur Import, nur PyQGIS-Standalone. Nicht-Ziele aus 1.2 strikt einhalten.
- **Projektwurzel:** `gando_av_import/` (unterhalb von `av01_itf/`).
- **Quelldokument:** `Umsetzungsplan_AV_Import_DM01AVCH_QGIS.docx` — bei Konflikten hat das Dokument Vorrang.

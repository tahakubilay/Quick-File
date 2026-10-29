<div align="center">

# QuickFile
### Kontextbezogene Desktop-Arbeitsoberfläche für intelligente Dateioperationen

<img src="./assets/hero.svg" alt="QuickFile – visuelle Projektidentität" width="100%" />

**Kontextbefehle** · **KI-gestützte Dateiabläufe** · **Desktop Interaction Engineering**

[English](./README.md) · [**Deutsch**](./README.de.md) · [Türkçe](./README.tr.md)

</div>

---

## Konzept

**QuickFile** bringt Dateioperationen direkt an den Ort der Aufmerksamkeit. Dateisystemkontext, aktionsorientierte Oberfläche und optionale KI-Unterstützung bilden einen gemeinsamen Desktop-Arbeitsablauf.

Klassische Dateimanager sind auf Navigation ausgelegt, Launcher auf Befehle und KI-Tools häufig auf Chats. QuickFile untersucht die Verbindung: **Element auswählen → Kontext erfassen → Aktion festlegen → Vorschau prüfen → kontrolliert ausführen**.

<img src="./assets/interaction-system.svg" alt="Interaktionsoberflächen von QuickFile mit gemeinsamem Befehlsmodell" width="100%" />

## Native Anwendung und Laufzeitarchitektur

QuickFile basiert auf einer **Tauri-2-Desktopanwendung** mit **React 18 / TypeScript / Vite 6** für die Oberfläche und einem modularen **Rust-Backend**. Es handelt sich nicht nur um einen browserbasierten Dateimanager: Das Interaktionsmodell verwendet native Dateisystembefehle, Desktop-Events, lokale Prozesskoordination und persistierten Betriebszustand.

`Tauri 2` · `Rust` · `React 18` · `TypeScript` · `Vite 6` · `SQLite / WAL` · `Unix Domain Sockets (Linux)`

<img src="./assets/native-runtime.svg" alt="QuickFile-Laufzeit mit Tauri-WebView, Rust-Aktions- und Jobmodulen, lokaler Persistenz und Linux-IPC" width="100%" />

| Laufzeitgrenze | Technische Verantwortung |
|:---|:---|
| **Desktop-WebView** | Kontextoberflächen, Tastatur/Maus, Vorschau, Fortschritt, Einstellungen |
| **Tauri-Befehlsbrücke** | Typisierte Frontend-zu-Rust-Befehle, Fenstersteuerung und Events |
| **Rust Action Registry** | Dateikontextabhängige Befehlsfilterung und Vorschlagsranking |
| **Rust Job Engine** | Dateioperationen, Jobzustand und operationsabhängiges Undo |
| **SQLite mit WAL** | Lokale Persistenz mit Write-Ahead-Logging |
| **CLI / Resident Host** | Unter Linux Aktivierung über einen lokalen Unix-Domain-Socket |

### Desktop-Aktivierung und IPC

Ein wichtiger Architekturpunkt ist die Trennung zwischen **kurzlebigem Aufruf** und **dauerhaft laufender Desktop-Oberfläche**. Unter Linux kann ein CLI-Prozess über einen Unix-Domain-Socket mit dem Resident Host kommunizieren. Dieser veranlasst dann ein Event in Richtung Tauri-Frontend. Dadurch muss ein externer Befehl nicht jedes Mal einen vollständigen neuen Anwendungsprozess starten.

```mermaid
sequenceDiagram
    participant CLI as CLI / Externer Aufruf
    participant HOST as Resident Host (Linux)
    participant RUST as Tauri / Rust
    participant UI as React-Oberfläche
    participant JOBS as Aktionen + Jobs

    CLI->>HOST: Unix-Domain-Socket-Anfrage
    HOST->>RUST: Desktop-Oberfläche aktivieren
    RUST-->>UI: Natives Event / Auswahlkontext
    UI->>RUST: Tauri-Befehl
    RUST->>JOBS: Aktion auflösen oder ausführen
    JOBS-->>RUST: Ergebnis / Jobstatus
    RUST-->>UI: Neuer Aktionszustand
```

Das Diagramm beschreibt den Architekturpfad. Fokusverhalten, Event-Verarbeitung und plattformübergreifende Aktivierung sind von der jeweiligen Desktopumgebung abhängig.

### Konkrete Desktop-Oberflächen

Der Anwendungscode trennt mehrere Bedienmodi:

- **Mini Popup:** kompakter Zugriff auf Aktionen zur aktuellen Auswahl.
- **Command Palette:** durchsuchbare Befehle mit Tastaturbedienung und Clipboard-Interaktion.
- **Radial / Marking Menu:** SVG-Menü mit acht Sektoren, Pointer-Richtungsauswertung sowie den Tasten `1–8`, Pfeiltasten, Enter und Escape.
- **QuickCompress HUD:** kompaktes Fenster für kompressionsbezogene Aktionen.
- **AI Explain:** eigenständige Oberfläche für kontextbezogene Erklärungen.
- **Settings:** umfangreiche, in mehrere Bereiche unterteilte Einstellungen für Darstellung und Interaktion.

Die native Rust-Schicht definiert unterschiedliche Fenstergrößen für Kompressions-HUD, Radialmenü und größere Arbeitsflächen. Diese Geometrie ist Teil des Desktop-Designs und nicht nur ein CSS-Breakpoint.

### Aktionsfilterung, Ausführung und Wiederherstellung

Im Backend existieren **dateispezifische Aktionsfilter** (`ActionRegistry::actions_for_files`) und ein **Vorschlagsranking** (`rank_suggestions`). Dadurch kann die Anwendbarkeit einer Aktion unabhängig von der Darstellung im Menü modelliert werden. Ein separates Rust-Jobmodul behandelt Ausführung und Undo-bezogene Logik.

Ein Undo-Modell bedeutet nicht, dass jede destruktive Dateisystemaktion garantiert rückgängig gemacht werden kann. Die Wiederherstellbarkeit hängt von der tatsächlichen Operation und ihrem Ergebnis ab.

### KI-Oberfläche und Modellinferenz

Die Anwendung enthält eine **AI-Explain-Oberfläche und beispielhafte UI-Antworten**. Dies ist nicht gleichbedeutend mit einem vollständig nachgewiesenen produktiven KI-Backend für alle Funktionen. Semantische Suche, Zusammenfassung, Kategorisierung und natürlichsprachliche Stapelverarbeitung werden unten als **Funktionsbereiche** beschrieben. Die praktische Umsetzung hängt von Modellanbindung, Dateiadaptern, Regelprüfung und Zustimmung ab.

### Plattformintegration

Die native Integration fokussiert zunächst **Linux-Desktops**, mit eigenen Anforderungen für **X11 und Wayland**. Fensterfokus, globale Pointer-Gesten, Transparenz und Compositorverhalten sind zwischen beiden Systemen nicht identisch. Strukturen für Windows und macOS sind ebenfalls vorhanden, begründen aber noch keine vollständige Funktionsparität.

---

## Interaktionsflächen

| Oberfläche | Interaktionsmodell | Technische Anforderung |
|---|---|---|
| **Kontextmenü** | Dateityp- und auswahlabhängige Befehle | Filterung, Verfügbarkeit, Berechtigungen |
| **Schnellaktionspalette** | Befehle suchen und ohne großes Fenster ausführen | Tastaturfokus, Ranking, Shortcuts |
| **Radiale Aktionen** | Häufige Operationen räumlich gruppieren | Zielgrößen, Abbruch, Bedienbarkeit |
| **Verschachtelte Menüs** | Komplexe Befehle schrittweise erschließen | Navigation, Zustandsübergänge |
| **Dateiinspektor** | Pfad, Größe, Zeitstempel, Eigenschaften und Aktionen | Asynchrone Metadaten, veraltete Auswahl |
| **Stapelverarbeitung** | Einen Befehl auf mehrere Dateien anwenden | Konflikte, Teilerfolge, Einzelergebnisse |
| **Vorschau / Bestätigung** | Geplante Änderungen vor Ausführung prüfen | Sicherheit, Änderungsvergleich |
| **Aktivität / Wiederherstellung** | Laufende und abgeschlossene Vorgänge erklären | Fortschritt, Abbruch, Wiederholung |

Die verschiedenen Oberflächen sollen dieselbe Befehlssemantik verwenden. Ein Tastaturkürzel und ein Kontextmenü dürfen unterschiedliche Einstiegspunkte, aber nicht unterschiedliche Dateisystemregeln besitzen.

### Kontextbezogene Befehlsauflösung

Die Aktion **Umbenennen** benötigt je nach Auswahl unterschiedliche Vorbedingungen:

- Eine einzelne beschreibbare Datei kann direkt umbenannt werden.
- Mehrere Dateien benötigen einen Batch-Plan inklusive Kollisionsprüfung.
- Schreibgeschützte Ordner dürfen nicht als regulär editierbar erscheinen.
- Zwischen Vorschau und Ausführung verschwundene Einträge müssen erneut geprüft werden.
- KI-generierte Dateinamen dürfen Zeichen-, Pfad- und Überschreibungsregeln nicht umgehen.

Die Oberfläche zeigt den Zustand eines **Befehlsvertrags**, erteilt aber selbst keine Dateisystemberechtigung.

## KI-gestützte Dateiverarbeitung

Die KI-Schicht wird als **Vorschlagsmaschine** betrachtet, nicht als Instanz, die selbstständig das Dateisystem verändert. Folgende Fähigkeiten gehören zum Interaktionsmodell; ihre konkrete Verfügbarkeit hängt von Dateiadaptern, Modellkonfiguration und Validierung ab.

| Fähigkeit | Anwendung | Sicherheitsgrenze |
|---|---|---|
| **Semantische Dateisuche** | Inhalte und Bedeutung statt nur Dateinamen suchen | Definierter Indexbereich, nachvollziehbare Treffer |
| **Zusammenfassungen** | Ausgewählte Dokumente oder Dateigruppen zusammenfassen | Dateiformat- und Größenlimits |
| **Intelligentes Umbenennen** | Namen aus Inhalt und Metadaten vorschlagen | Alte und neue Pfade einzeln prüfen |
| **Kategorisierung** | Rechnungen, Bilder, Dokumente und Projektdateien gruppieren | Konfidenz und manuelle Korrektur |
| **Stapelorganisation** | Natürliche Sprache in Kopier-/Verschiebe-/Umbenennungspläne übersetzen | Simulation, Kollisionsprüfung, Freigabe |
| **Metadatenextraktion** | Strukturierte Tags und Suchattribute ableiten | Herkunft und Extraktionsfehler |
| **Workflow-Vorschläge** | Passende Aktionen im aktuellen Kontext anbieten | Nachvollziehbare, optionale Ausführung |

### Beispiel: natürlicher Sprachbefehl

> „Sortiere diese PDFs nach Jahr, gib ihnen einheitliche Namen und behalte die Originale.“

Daraus entsteht ein **prüfbarer Aktionsplan** mit erkanntem Jahr, Zielordner, ursprünglichem und vorgeschlagenem Dateipfad, Unklarheiten sowie Kopier- oder Verschiebesemantik. Erst nach der Prüfung wird geschrieben. Fehlende Daten und Namenskonflikte dürfen nicht stillschweigend geraten werden.

<img src="./assets/ai-workflow.svg" alt="Vom KI-Vorschlag über Validierung und Vorschau zur bestätigten Ausführung" width="100%" />

**Datenschutzgrenze:** Lokale Metadatenverarbeitung und externe Modellinferenz sind nicht gleichwertig. Eine sichere Architektur unterscheidet lokale Verarbeitung, optionale externe Anbieter und ausdrückliche Zustimmung. Das Auswählen eines KI-Befehls ist keine pauschale Erlaubnis, Dokumentinhalte an fremde Dienste zu übermitteln.

## Technische Architektur

```mermaid
flowchart TB
    UI["Kontextmenü · Palette · Radialmenü · Inspektor"]
    CTX["Auswahlkontext / Dateimetadaten"]
    REG["Befehlsregister / Verfügbarkeitsprüfung"]
    PLAN["Aktionsplanung"]
    AI["Optionaler KI-Adapter"]
    POLICY["Regeln / Berechtigungen / Kollisionen"]
    PREVIEW["Vorschau + Nutzerfreigabe"]
    EXEC["Dateisystemadapter"]
    EVENTS["Ergebnis / Fortschritt / Fehler"]

    UI --> CTX --> REG --> PLAN
    AI -. "strukturierte Vorschläge" .-> PLAN
    PLAN --> POLICY --> PREVIEW --> EXEC --> EVENTS
    EVENTS --> UI
```

Das Diagramm beschreibt **Architekturgrenzen**, nicht die Behauptung, dass alle Adapter bereits als ausführbare Module veröffentlicht wurden.

### Beispiel eines Befehlsvertrags

```ts
type FileActionPlan = {
  operation: "copy" | "move" | "rename" | "delete";
  targets: Array<{
    source: string;
    destination?: string;
    expectedFingerprint?: string;
  }>;
  preconditions: string[];
  collisions: Array<{ path: string; reason: string }>;
  confirmationRequired: boolean;
};
```

Die Schnittstelle ist **illustrativ**, keine Zusage einer veröffentlichten API. Sie trennt UI- bzw. KI-Vorschläge von validierten Dateioperationen.

## Robustheit und Fehlerfälle

| Risiko | Technische Maßnahme |
|---|---|
| **Time-of-check / time-of-use** | Auswahl und Ziel unmittelbar vor Ausführung erneut prüfen |
| **Namenskollisionen** | Definierte Strategien: überspringen, umbenennen, bewusst ersetzen |
| **Teilerfolg bei Batch-Aktionen** | Einzelne Resultate protokollieren statt künstliche Atomizität zu behaupten |
| **Abbruch** | Unterbrechbare Vorgänge und Wiederherstellungsgrenzen unterscheiden |
| **Berechtigungen / Dateisperren** | Echte Fehler anzeigen, keine scheinbaren Erfolge |
| **Pfad- und Link-Sicherheit** | Symbolische Links, Traversal und geschützte Ziele berücksichtigen |
| **Unzuverlässige KI-Ausgabe** | Strukturierte Befehle parsen, zulässige Aktionen prüfen, Freigabe verlangen |
| **Lange Operationen** | Fortschritt ohne blockierende Oberfläche darstellen |

Undo-Garantien müssen zur Operation passen: Eine reversible Umbenennung und eine endgültige Löschung besitzen nicht denselben Wiederherstellungsweg.

## Desktop-Designsystem

QuickFile setzt auf hohe Informationsdichte, präzise Hierarchie, reduzierte Bewegung und eindeutige Zustände für Auswahl, Hover, Tastaturfokus, Deaktivierung, Wartezeiten, destruktive Aktionen und Erfolg.

Zu den Interaktionsbausteinen gehören Aktionszeilen, Suchpaletten, verschachtelte Menüs, Inspektoren, Bestätigungsdialoge, Vorschauen, Fortschrittsanzeigen und Aktivitäten.

**Projektkontext:** QuickFile ist als Open-Source-Desktop-Produkt ausgerichtet. Dieses Repository beschreibt das Produkt- und Interaktionsmodell; Diagramme und Codebeispiele sind keine Leistungsnachweise oder Behauptungen über den Implementierungsstand jeder Funktion.

---

<div align="center">

Entwickelt von [Taha Kubilay](https://github.com/tahakubilay) · [English](./README.md) · [Türkçe](./README.tr.md)

</div>

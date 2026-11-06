# Plectyra – Projektplan

> Status: Entwurf · Stand: 2026-11-06
>
> Dieses Dokument ist die Grundlage für Architektur- und Umsetzungsentscheidungen.
> Getroffene Entscheidungen werden im [Entscheidungslog](#18-entscheidungslog)
> festgehalten, offene Punkte stehen unter [Offene Entscheidungen](#17-offene-entscheidungen).

---

## Inhalt

1. [Vision](#1-vision)
2. [Ziele und Nicht-Ziele](#2-ziele-und-nicht-ziele)
3. [Recherche: bestehende Projekte](#3-recherche-bestehende-projekte)
4. [Signal-Konzept: Audio, MIDI und MIDI-Gitarre](#4-signal-konzept-audio-midi-und-midi-gitarre)
5. [Anforderungen](#5-anforderungen)
6. [Architektur](#6-architektur)
7. [Audio-Engine](#7-audio-engine)
8. [DSP-Module](#8-dsp-module)
9. [Plugin-System](#9-plugin-system)
10. [Benutzeroberfläche (Tauri)](#10-benutzeroberfläche-tauri)
11. [Presets und Persistenz](#11-presets-und-persistenz)
12. [MIDI-Steuerung](#12-midi-steuerung)
13. [Tests und Qualitätssicherung](#13-tests-und-qualitätssicherung)
14. [Build, CI und Distribution](#14-build-ci-und-distribution)
15. [Roadmap](#15-roadmap)
16. [Risiken](#16-risiken)
17. [Offene Entscheidungen](#17-offene-entscheidungen)
18. [Entscheidungslog](#18-entscheidungslog)
19. [Referenzen](#19-referenzen)

---

## 1. Vision

Plectyra ist ein quelloffenes, plattformübergreifendes Gitarren-Rig als
Standalone-Anwendung. Man schließt die Gitarre an ein Audio-Interface an und
baut sich aus Pedalen, Amps, Boxen und Effekten sein eigenes Rig zusammen –
vergleichbar mit kommerziellen Lösungen wie Guitar Rig, aber offen und
erweiterbar.

Kernideen:

- **Standalone zuerst.** Eine eigenständige Desktop-App für macOS, Windows und
  Linux, kein DAW-Plugin als Primärprodukt.
- **Offene Plugin-Schnittstelle.** Jeder kann eigene Pedale und Effekte bauen,
  ohne den Plectyra-Code anzufassen. Dafür setzen wir auf einen etablierten
  offenen Standard (CLAP) statt auf ein proprietäres Format.
- **Rust im Kern.** Engine, DSP und Plugin-Host in Rust, Oberfläche mit Tauri 2.
- **Live-tauglich.** Niedrige Latenz, keine Aussetzer, Steuerung per
  MIDI-Fußleiste.

## 2. Ziele und Nicht-Ziele

### Ziele (v1.0)

- Latenz vom Eingang bis zum Ausgang unter 10 ms bei 48 kHz auf üblicher
  Hardware mit passendem Treiber (CoreAudio, ASIO, JACK/PipeWire).
- Pedalboard mit frei anordenbaren Modulen, seriell und parallel.
- Solide Grundausstattung an eingebauten Effekten (Gate, Drive, Amp, Cab, EQ,
  Modulation, Delay, Reverb, Tuner, Looper).
- Laden von Neural-Amp-Modeler-Profilen (`.nam`) und Cab-Impulsantworten (`.wav`).
- Hosting von CLAP-Plugins inklusive Zustandsspeicherung in Presets.
- Steuerung per MIDI: Fußschalter, Expression-Pedal, Program Change, MIDI Learn.
- Plugin-SDK mit Vorlage, Dokumentation und Beispiel-Pedalen.
- Installer bzw. Pakete für alle drei Plattformen aus der CI.

### Nicht-Ziele (vorerst)

- Kein DAW-Plugin (VST3/CLAP/AU) von Plectyra selbst. Kann später kommen, weil
  die Engine davon unabhängig ist.
- Keine Mehrspur-Aufnahme oder DAW-Funktionen. Höchstens ein einfacher Looper
  und optional ein Mitschnitt des Ausgangs.
- Kein Cloud-Dienst, kein Account-System, kein Plugin-Store in v1.
- Keine Mobile-App in v1. Eine Fernsteuerung per Handy ist eine spätere Option.
- Kein eigenes Hardware-Gerät.

## 3. Recherche: bestehende Projekte

Stand der Recherche: November 2026.

| Projekt | Beschreibung | Stärken | Warum es unser Ziel nicht abdeckt |
|---|---|---|---|
| [MOD Desktop](https://mod.audio/desktop/) | Pedalboard-App von MOD Audio, AGPLv3, Linux/macOS/Windows | Konzeptionell am nächsten dran, großes LV2-Ökosystem, Patchbay-Ansatz | C/C++/Python, Web-UI über eigenen Server, LV2 ist außerhalb von Linux kaum verbreitet, noch Beta |
| [OpenRig](https://github.com/jpfaria/OpenRig) | Echtzeit-Audioplattform für Gitarre in Rust, GPL-3, Desktop für alle drei Plattformen | Gleiche Sprache, ähnliche Vision, YAML-Presets | Sehr früh (`0.1.0-dev`), UI-Stack und Plugin-Schnittstelle unklar |
| [AmpForge](https://github.com/Loursy/AmpForge) | Amp-Sim mit internem Pedalboard, C++, GPL-3, Plugin und Standalone | Fertig nutzbar, viele Module | Geschlossenes, internes Pedalboard ohne Fremd-Plugins |
| Guitarix | Etablierte Amp-Sim für Linux/JACK | Ausgereifte DSP | Primär Linux, Windows/macOS nur über einen inoffiziellen, veralteten VST3-Port |
| Rakarrack-plus | Multieffekt für Linux | Viele Effekte | Linux, kein offenes Plugin-Konzept |
| [Neural Amp Modeler](https://github.com/sdatkinson/NeuralAmpModelerCore) | Amp-Modelling per neuronalem Netz, Core in C++ | De-facto-Standard, tausende freie Profile | Nur ein Baustein, kein Rig |
| [NeuralAmpModeler-rs](https://docs.rs/crate/NeuralAmpModeler-rs/latest/source/README.md) | NAM in Rust | Reines Rust, keine Allokation im Audio-Thread | Alpha, setzt x86-64 mit AVX2/FMA voraus, läuft also nicht auf Apple Silicon |
| [Rustone](https://github.com/acaala/rustone), RustRiff | Amp-Sims mit Rust und Tauri | Zeigen, dass der Stack grundsätzlich funktioniert | Hobby- bzw. Lernprojekte ohne Plugin-Schnittstelle |
| [dsp-stuff](https://github.com/simmsb/dsp-stuff) | Node-basierte Audioeffekte in Rust (cpal) | Node-Graph-Ansatz | Seit 2024 inaktiv, kein Gitarren-Fokus |

### Lehren daraus

- **MOD Desktop** zeigt, dass das Pedalboard-Konzept mit offenem
  Plugin-Ökosystem funktioniert. Die Bindung an LV2 ist aber der Grund, warum
  das Ökosystem außerhalb von Linux klein bleibt. Wir wählen deshalb CLAP.
- **NAM** ist gesetzt. Ohne NAM-Unterstützung ist eine moderne Amp-Sim kaum
  konkurrenzfähig, weil die Community dort ihre Profile teilt.
- **OpenRig** verfolgt eine sehr ähnliche Idee. Vor dem eigentlichen Start
  wird geprüft, ob eine Zusammenarbeit oder ein Fork sinnvoller ist
  (siehe [Offene Entscheidungen](#17-offene-entscheidungen)).

### Fazit

Es gibt derzeit kein ausgereiftes, plattformübergreifendes Gitarren-Rig in
Rust mit Tauri-Oberfläche und offener, standardbasierter Plugin-Schnittstelle.
Die Lücke ist real.

## 4. Signal-Konzept: Audio, MIDI und MIDI-Gitarre

Eine normale E-Gitarre liefert ein **analoges Audiosignal**, kein MIDI. Das
Audio-Interface digitalisiert es, und Plectyra verarbeitet diesen Audiostrom.
Daraus ergibt sich:

| Signal | Rolle in Plectyra | Priorität |
|---|---|---|
| Audio (Gitarre über Interface) | Hauptsignalweg durch das Rig | v1 |
| MIDI von Controllern (Fußleiste, Expression-Pedal) | Steuerung: Bypass, Parameter, Presets, Snapshots, Looper | v1 |
| MIDI von MIDI-Gitarren (z. B. Fishman TriplePlay, Roland GK mit Konverter) | Noten an Instrument-Plugins (Synthesizer) weiterleiten, parallel zum Audiosignal | später |
| MIDI Clock | Tempo-Synchronisation für Delay und Modulation | später |

Der Graph unterstützt von Anfang an **Event-Ports** neben Audio-Ports, damit
MIDI-Noten später ohne Umbau durch das Rig geleitet werden können.

## 5. Anforderungen

### Funktional

- Auswahl von Audio-Interface, Eingangskanal, Ausgangskanälen, Samplerate und
  Puffergröße.
- Module hinzufügen, entfernen, umsortieren, bypassen, duplizieren.
- Parallele Signalwege (Split und Mix), z. B. zwei Amps gleichzeitig.
- Mono-Eingang, Stereo-Verarbeitung ab beliebigem Punkt im Graph.
- Presets speichern, laden, umbenennen, exportieren und importieren.
- Snapshots innerhalb eines Presets (gleiche Module, andere Parameterwerte)
  für schnelle Wechsel ohne Neuaufbau des Graphen.
- Tuner, der den Ausgang optional stummschaltet.
- Eingangs- und Ausgangspegelanzeigen, Clipping-Warnung.
- Looper mit Aufnahme, Overdub, Undo.
- Plugin-Browser für eingebaute Module, CLAP-Plugins, NAM-Profile und IRs.

### Nicht-funktional

- **Latenz:** Unter 10 ms vom Eingang bis zum Ausgang bei 48 kHz und 64–128
  Samples Puffer, mit nativen Treibern.
- **Stabilität:** Keine Aussetzer bei Preset-Wechsel, Modulumbau oder
  UI-Interaktion. Ein abstürzendes Plugin darf idealerweise nicht die App
  beenden (siehe [Risiken](#16-risiken)).
- **CPU:** Ein typisches Rig (Gate, Drive, NAM-Amp, Cab, Delay, Reverb) soll auf
  einem aktuellen Mittelklasse-Laptop unter 25 % eines Kerns bleiben.
- **Plattformen:** macOS 12+ (Apple Silicon und Intel), Windows 10/11 (x64),
  Linux (x86_64, später aarch64).
- **Startzeit:** App betriebsbereit in unter 3 Sekunden, das
  Plugin-Scanning läuft im Hintergrund.

## 6. Architektur

### 6.1 Überblick

```
┌───────────────────────── Tauri 2 App ─────────────────────────┐
│ Web-Frontend                                                  │
│ Pedalboard · Graph-Ansicht · Browser · Tuner · Einstellungen  │
└───────────────▲─────────────────────────────┬─────────────────┘
                │ Events (gedrosselt, ~30 Hz) │ Commands (Tauri IPC)
┌───────────────┴─────────────────────────────▼─────────────────┐
│ Steuer-Schicht (Rust, normaler Thread)                        │
│ Rig-Modell · Graph-Compiler · Preset-Verwaltung · MIDI-Mapping│
│ Plugin-Verwaltung · Asset-Laden (Worker-Threads)              │
└───────────────▲─────────────────────────────┬─────────────────┘
                │ Rückkanal (alte Pläne,      │ Ausführungspläne,
                │ Messwerte, Tuner-Daten)     │ Parameteränderungen
┌───────────────┴─────────────────────────────▼─────────────────┐
│ Audio-Thread (Echtzeit)                                       │
│ Eingang → Ausführungsplan (Nodes in fester Reihenfolge) → Out │
│ keine Allokation · keine Locks · keine Syscalls · kein IPC    │
└───────────────────────────────────────────────────────────────┘
```

Grundsatz: **Tauri ist ausschließlich die Oberfläche.** Kein Audio fließt
durch die Webview oder über IPC. Die Engine läuft auch ohne UI (wichtig für
Tests und eine spätere Headless- oder Plugin-Variante).

### 6.2 Thread-Modell

| Thread | Aufgabe | Regeln |
|---|---|---|
| Audio-Thread | Callback des Audio-Treibers, führt den aktuellen Plan aus | Echtzeit-Regeln (siehe 6.3), erhöhte Priorität |
| Steuer-Thread | Hält das Rig-Modell, kompiliert Pläne, verarbeitet Commands | Darf allozieren und blockieren, aber nie den Audio-Thread blockieren |
| Main-Thread (Tauri-Eventloop) | Fenster, Webview, native Plugin-GUIs | CLAP verlangt, dass bestimmte Aufrufe (u. a. GUI, Aktivierung) auf dem „Main Thread" des Hosts passieren. Diese Aufrufe werden über Tauris Main-Thread-Dispatch ausgeführt |
| Worker-Threads | NAM-Modelle laden, IRs laden und resamplen, Preset-Import | Ergebnisse gehen fertig vorbereitet an den Steuer-Thread |
| Analyse-Thread | Tuner (Tonhöhenerkennung), Spektrum, Pegel aufbereiten | Bekommt Audio-Kopien über einen Ringpuffer |
| Scanner-Prozess | Durchsucht und prüft Plugins | Separater Prozess, damit ein fehlerhaftes Plugin nicht die App abstürzen lässt |

### 6.3 Echtzeit-Regeln für den Audio-Thread

Verbindlich für allen Code, der im Audio-Callback läuft:

- Keine Heap-Allokation und -Freigabe (`Box`, `Vec::push` mit Wachstum,
  `String`, `Arc::drop` des letzten Verweises …).
- Keine Mutexe, kein `RwLock`, keine blockierenden Channels.
- Keine Datei-, Netzwerk- oder Konsolen-I/O, kein Logging mit Allokation.
- Kein Aufruf von Tauri oder anderen UI-APIs.
- Kommunikation ausschließlich über lock-freie Single-Producer/Single-Consumer-
  Ringpuffer (`rtrb`) und atomare Werte.
- Nicht mehr benötigte Objekte (alte Pläne, entfernte Nodes) werden über einen
  Rückkanal an den Steuer-Thread geschickt und dort freigegeben.
- Denormale Zahlen werden abgeschaltet (Flush-to-Zero/Denormals-are-Zero auf
  x86, FZ-Bit auf ARM), um CPU-Spitzen bei abklingenden Signalen zu vermeiden.
- In Debug-Builds und Tests prüft `assert_no_alloc`, dass im Callback nichts
  alloziert wird.

### 6.4 Repository- und Crate-Struktur

Cargo-Workspace:

```
plectyra/
├─ Cargo.toml                  Workspace-Definition
├─ crates/
│  ├─ plectyra-core/           Gemeinsame Typen: IDs, Parameter, Fehler, Sample-Formate
│  ├─ plectyra-engine/         Graph, Compiler, Ausführungsplan, Scheduler, Parameter-Glättung
│  ├─ plectyra-io/             Audio-I/O über cpal, Geräteverwaltung, Duplex-Brücke
│  ├─ plectyra-midi/           MIDI-Ein/Ausgabe über midir, Mapping, MIDI Learn
│  ├─ plectyra-dsp/            Eingebaute Effekte und DSP-Bausteine
│  ├─ plectyra-nam/            NAM-Anbindung (FFI an NeuralAmpModelerCore)
│  ├─ plectyra-host/           CLAP-Host auf Basis von clack-host
│  ├─ plectyra-scan/           Binary für den Plugin-Scanner-Prozess
│  ├─ plectyra-preset/         Preset-Format, Serialisierung, Migration
│  └─ plectyra-plugin-sdk/     Hilfen und Vorlage für Plugin-Entwickler, eigene CLAP-Erweiterung
├─ app/
│  ├─ src-tauri/               Tauri-Backend: Commands, Events, App-Lebenszyklus
│  └─ ui/                      Web-Frontend
├─ plugins/
│  └─ examples/                Beispiel-Pedale als CLAP-Plugins
├─ assets/                     Mitgelieferte IRs, Standard-Presets, Grafiken
├─ docs/                       Architektur, Plugin-Entwicklung, Benutzerhandbuch
└─ .github/workflows/          CI und Releases
```

Abhängigkeitsrichtung (keine Zyklen):

```
core ← dsp ← engine ← io
core ← host ← engine
core ← nam ← dsp
core ← preset
core ← midi
app → engine, io, midi, host, preset
```

### 6.5 Datenmodell

```
Rig
├─ id, name
├─ graph: Graph
│  ├─ nodes: [Node]
│  │  ├─ id: NodeId (stabil, UUID)
│  │  ├─ kind: Builtin(type_id) | Clap(plugin_id) | Nam(asset) | Ir(asset) | Io(Input|Output)
│  │  ├─ bypass: bool
│  │  ├─ params: { ParamId → Wert }
│  │  ├─ state: Option<Bytes>   (Plugin-Zustand)
│  │  └─ ui: { Position, Farbe, Label }
│  └─ connections: [ (NodeId, PortId) → (NodeId, PortId) ]
├─ snapshots: [Snapshot { name, param_overrides, bypass_overrides }]
├─ midi_map: [Mapping]
└─ meta: { tempo, Tags, Notizen }
```

- **Parameter** haben eine stabile ID, einen Bereich, eine Einheit, eine
  Glättungsstrategie und einen Standardwert. Intern werden sie normalisiert
  (0–1) übertragen und im Node in Klartextwerte umgerechnet.
- **Ports** sind typisiert: `Audio { channels }` oder `Events`.

### 6.6 Graph-Kompilierung und Austausch

1. Die UI schickt eine Änderung (z. B. „Node einfügen") an den Steuer-Thread.
2. Der Steuer-Thread ändert das Rig-Modell und kompiliert daraus einen
   **Ausführungsplan**:
   - Topologische Sortierung, Zyklen sind verboten.
   - Alle Puffer werden vorab alloziert und den Ports zugewiesen.
   - Neue Nodes werden auf dem Steuer-Thread erzeugt und aktiviert
     (bei CLAP: `activate` auf dem Main-Thread).
   - Latenzausgleich: Bei parallelen Pfaden mit unterschiedlicher Latenz
     (z. B. ein Plugin mit Lookahead) werden kürzere Pfade verzögert.
3. Der fertige Plan geht über einen Ringpuffer an den Audio-Thread.
4. Der Audio-Thread wechselt zu Beginn des nächsten Blocks auf den neuen Plan
   und blendet bei strukturellen Änderungen kurz über (5–20 ms), um Knackser
   zu vermeiden.
5. Der alte Plan geht über den Rückkanal zurück und wird dort freigegeben.

Parameteränderungen laufen **nicht** über eine Neukompilierung, sondern über
atomare Werte bzw. eine Event-Queue pro Block. So können CLAP-Plugins
sampelgenaue Parameter-Events erhalten.

### 6.7 Fehlerbehandlung

- Fehler auf dem Steuer-Thread werden als typisierte Fehler (`thiserror`) an
  die UI gemeldet und dort verständlich angezeigt.
- Fehlt beim Laden eines Presets ein Plugin oder eine Datei, wird ein
  **Platzhalter-Node** eingesetzt, der den Zustand unverändert aufbewahrt. Beim
  erneuten Speichern geht so nichts verloren.
- Verliert die App das Audiogerät (z. B. USB-Interface abgezogen), stoppt die
  Engine sauber, die UI zeigt den Zustand an und versucht auf Wunsch eine
  Wiederverbindung.

## 7. Audio-Engine

### 7.1 Audio-I/O

Basis ist [cpal](https://github.com/RustAudio/cpal).

| Plattform | Backend | Hinweise |
|---|---|---|
| macOS | CoreAudio | Geringe Latenz ohne Zusatztreiber |
| Windows | ASIO (bevorzugt), WASAPI | ASIO über die cpal-Option `asio`, braucht das ASIO-SDK zum Bauen (inzwischen auch unter GPLv3 verfügbar). WASAPI als Fallback, Exklusivmodus prüfen |
| Linux | JACK, PipeWire (über JACK-Kompatibilität), ALSA | Echtzeit-Priorität über rtkit bzw. passende Gruppenrechte |

**Duplex-Problem:** cpal stellt Eingang und Ausgang als getrennte Streams mit
eigenen Callbacks bereit. Für ein Gitarren-Rig braucht man aber Eingang und
Ausgang im selben Takt. Lösungsansatz:

- Der Eingangs-Callback schreibt in einen kleinen Ringpuffer, der
  Ausgangs-Callback liest daraus und führt den Plan aus.
- Der Füllstand wird überwacht. Laufen beide Takte auseinander (verschiedene
  Geräte), wird nachgeregelt oder eine Warnung angezeigt.
- Bei JACK und ASIO laufen Ein- und Ausgang ohnehin synchron.

Ob dieser Ansatz die Latenzziele erreicht, ist die zentrale Frage von
**Phase 0**. Falls nicht, ist der Plan B ein dünnes, eigenes Backend pro
Plattform für echtes Duplex (z. B. CoreAudio-AUHAL auf macOS), während cpal für
die Geräteliste erhalten bleibt.

Weitere Punkte:

- Thread-Priorität des Audio-Threads über `audio_thread_priority` anheben,
  falls das Backend das nicht selbst erledigt.
- Unterstützte Samplerates: 44,1 / 48 / 88,2 / 96 kHz. Puffergrößen 32–1024.
- Intern wird mit `f32` gerechnet, nicht-interleaved (ein Puffer pro Kanal).
- Latenzanzeige in der UI: Puffer, Treiberlatenz (soweit gemeldet),
  Plugin-Latenz.

### 7.2 Verarbeitungsblock

Pro Callback:

1. Eingangssamples aus dem Duplex-Puffer holen, Eingangsgain anwenden,
   Eingangspegel messen.
2. Ausstehende Pläne und Parameter-Events aus den Queues übernehmen.
3. Nodes in Plan-Reihenfolge verarbeiten, bei Bedarf in Sub-Blöcke
   unterteilen (z. B. für sampelgenaue Events).
4. Ausgangsgain und Limiter (Schutz vor Feedback und Lautstärkespitzen),
   Ausgangspegel messen.
5. Kopien für Tuner, Looper und Meter in die Analyse-Ringpuffer schreiben.

### 7.3 Parameter-Glättung

- Lineare oder exponentielle Rampen (konfigurierbar pro Parameter, ca. 5–50 ms)
  gegen Zipper-Geräusche.
- Bypass mit kurzer Überblendung, kein harter Schnitt.
- Snapshot-Wechsel glätten alle betroffenen Parameter gleichzeitig.

## 8. DSP-Module

Eingebaute Module in `plectyra-dsp`. Jedes Modul implementiert dasselbe interne
`Processor`-Trait wie gehostete Plugins über einen Adapter, damit der Graph
nicht zwischen beiden unterscheiden muss.

| Modul | Umsetzung | Phase |
|---|---|---|
| Input/Output | Gain, Phasenumkehr, Kanalwahl, Pegelmessung | 1 |
| Noise Gate | Hysterese, Attack/Hold/Release, Sidechain vom Eingang | 1 |
| Tuner | YIN oder MPM auf Analyse-Thread, chromatisch, Kalibrierung (A4), optional Mute | 1 |
| Kompressor | Feed-Forward, RMS/Peak, Soft Knee, Parallel-Mix | 1 |
| Overdrive/Distortion/Fuzz | Waveshaper (verschiedene Kennlinien), Tone-Stack, 4- bis 8-faches Oversampling mit Polyphasen-Halbbandfiltern | 1 |
| EQ | Biquads (Low/High Shelf, Peak, Hoch-/Tiefpass), parametrisch und grafisch | 1 |
| Cab-IR | Gleichmäßig partitionierte FFT-Faltung (`realfft`), IR wird beim Laden auf die Session-Samplerate resampelt (`rubato`) und normalisiert, Mix und Hoch-/Tiefpass | 1 |
| Delay | Digital, Tape (Modulation, Sättigung), Ping-Pong, Tap-Tempo, Feedback-Filter | 1 |
| Reverb | Algorithmisch (FDN), Room/Hall/Plate/Spring-Charakter, Pre-Delay, Mix | 1 |
| NAM-Amp | Siehe 8.1 | 2 |
| Klassischer Amp | Vorstufe, Tone-Stack, Endstufen-Sättigung als Alternative zu NAM | 2 |
| Chorus / Flanger / Phaser / Tremolo / Vibrato | Modulierte Delay-Lines bzw. Allpass-Ketten, LFO mit Tempo-Sync | 2 |
| Wah | Resonanter Bandpass, Expression-Pedal oder Auto-Wah (Hüllkurve) | 2 |
| Pitch / Octaver | Später, hoher Aufwand | später |
| Looper | Aufnahme, Overdub, Undo, Halbe Geschwindigkeit, Reverse | 4 |
| Split / Mix | Parallele Pfade, A/B, Y-Split, Pegel und Pan pro Zweig | 5 |

### 8.1 NAM

- Grundlage ist [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore)
  (C++, MIT-Lizenz).
- Anbindung über `cxx` und einen kleinen C++-Wrapper, gebaut mit dem
  `cc`- oder `cmake`-Crate.
- Die vorhandene Rust-Implementierung `NeuralAmpModeler-rs` ist derzeit
  Alpha und braucht AVX2, läuft also nicht auf Apple Silicon. Sie wird
  beobachtet. Ein reiner Rust-Port ist eine spätere Option.
- Modelle werden auf einem Worker-Thread geladen und erst danach in den
  Graph eingesetzt.
- NAM-Modelle sind in der Regel auf eine feste Samplerate trainiert. Läuft die
  Session mit einer anderen Rate, wird um das Modell herum resampelt.
- CPU-Last pro Modell wird gemessen und in der UI angezeigt.

### 8.2 Allgemeine DSP-Regeln

- Kein Code im Audio-Pfad, der von der Puffergröße abhängige Allokationen
  braucht. Alle Puffer werden bei `prepare(sample_rate, max_block_size)`
  angelegt.
- Nichtlinearitäten grundsätzlich mit Oversampling.
- Jedes Modul hat Referenz-Tests (siehe Abschnitt 13).

## 9. Plugin-System

### 9.1 Warum CLAP

| Kriterium | CLAP | VST3 | LV2 | Eigenes Format | WASM |
|---|---|---|---|---|---|
| Lizenz | MIT | MIT (seit SDK 3.8, Okt. 2025) | ISC | frei wählbar | – |
| Stabile C-ABI | ja | ja (COM-artig) | ja | müsste gebaut werden | ja |
| Aufwand als Host | mittel | hoch | mittel | hoch | mittel |
| Rust-Unterstützung | clack-host / clack-plugin, nih-plug | Bindings vorhanden, aufwändig | gering | – | wasmtime |
| Verbreitung auf Win/macOS | wachsend | sehr hoch | gering | null | null |
| Eigene Erweiterungen | vorgesehen | eingeschränkt | ja | ja | ja |

Entscheidung (Vorschlag): **CLAP ist das primäre Plugin-Format.** VST3-Hosting
folgt später, WASM bleibt eine Option für abgeschottete Community-Plugins.

### 9.2 CLAP-Host

Umsetzung mit [clack-host](https://github.com/prokopyl/clack). Benötigte
Erweiterungen:

| Erweiterung | Zweck |
|---|---|
| `audio-ports` | Kanalanzahl und Port-Layout |
| `note-ports` | MIDI-Noten (für MIDI-Gitarre und Instrumente) |
| `params` | Parameterliste, Werte, Automation, generische UI |
| `state` | Zustand in Presets speichern und laden |
| `latency` | Latenzausgleich in parallelen Pfaden |
| `tail` | Ausklingzeit (Reverb, Delay) bei Bypass beachten |
| `gui` | Native Plugin-Fenster (frei schwebend) |
| `timer-support`, `posix-fd-support` | Für Plugin-GUIs, insbesondere unter Linux |
| `thread-check`, `log` | Diagnose |

Standard-Suchpfade gemäß CLAP-Konvention:

| Plattform | Pfade |
|---|---|
| macOS | `~/Library/Audio/Plug-Ins/CLAP`, `/Library/Audio/Plug-Ins/CLAP` |
| Windows | `%LOCALAPPDATA%\Programs\Common\CLAP`, `%COMMONPROGRAMFILES%\CLAP` |
| Linux | `~/.clap`, `/usr/lib/clap` |
| alle | zusätzlich `CLAP_PATH` und eigene Pfade aus den Einstellungen |

### 9.3 Plugin-Scanning

- Separates Binary `plectyra-scan`, wird von der App als Kindprozess gestartet.
- Lädt jedes Plugin einzeln, liest die Deskriptoren aus und schreibt das
  Ergebnis als JSON-Cache (Pfad, Änderungszeit, Hash, Plugin-IDs, Features).
- Stürzt der Scanner bei einem Plugin ab, landet es auf einer Sperrliste. Der
  Nutzer kann es manuell wieder freigeben.
- Erneutes Scannen nur bei geänderten Dateien.

### 9.4 Plectyra-Erweiterung für Pedale

CLAP erlaubt herstellerspezifische Erweiterungen. Plectyra definiert
`com.plectyra.pedal/1`. Plugins ohne diese Erweiterung funktionieren trotzdem,
bekommen dann aber eine generische Darstellung.

Entwurf der Inhalte:

```c
// com.plectyra.pedal/1 (Entwurf, wird in Phase 6 finalisiert)
typedef struct plectyra_pedal_info {
    const char *category;        // "drive", "amp", "cab", "modulation", "delay", "reverb", "utility", ...
    clap_id     bypass_param_id; // Parameter, der als Fußschalter dient (CLAP_INVALID_ID = keiner)
    uint32_t    accent_rgba;     // Grundfarbe des Pedals in der UI
    const char *layout_hint;     // "stomp-2", "stomp-3", "stomp-4", "rack", ...
    const char *primary_params;  // kommagetrennte Param-IDs für die Regler auf der Pedalfläche
} plectyra_pedal_info_t;

typedef struct plectyra_pedal {
    bool (*get_info)(const clap_plugin_t *plugin, plectyra_pedal_info_t *out);
} plectyra_pedal_t;
```

Die genaue Form (z. B. ob Grafiken oder ein deklaratives Layout mitgeliefert
werden können) wird in Phase 6 entschieden.

### 9.5 Plugin-Oberflächen

- **Standard:** Plectyra erzeugt aus der `params`-Erweiterung automatisch eine
  Pedal-Oberfläche im Web-Frontend. Damit sehen alle Pedale einheitlich aus.
- **Optional:** Über „Original-GUI öffnen" wird das native Plugin-Fenster als
  eigenes, schwebendes Fenster geöffnet. Eine Einbettung in die Webview ist
  technisch schwierig und kein Ziel für v1.

### 9.6 Plugin-SDK

`plectyra-plugin-sdk` und die Doku in `docs/plugin-development.md` sollen es
leicht machen, ein Pedal zu bauen:

- Vorlage (`cargo generate`) für ein Pedal in Rust. Basis ist entweder
  [nih-plug](https://github.com/robbert-vdh/nih-plug) oder `clack-plugin`. Zu
  klären ist, ob nih-plug eigene CLAP-Erweiterungen sauber unterstützt;
  andernfalls wird `clack-plugin` direkt genutzt.
- C/C++-Header für die Plectyra-Erweiterung, damit auch JUCE- oder
  C++-Entwickler sie nutzen können.
- Beispiel-Pedale: Boost, Tremolo, Bitcrusher.
- Validierung der Beispiele mit `clap-validator` in der CI.

### 9.7 Spätere Ausbaustufen

- **VST3-Hosting:** Seit der MIT-Lizenzierung des SDK problemlos einbindbar,
  aber aufwändiger als CLAP.
- **Out-of-Process-Hosting:** Plugins in einem eigenen Prozess mit
  Shared-Memory-Puffern. Schützt die App vor Abstürzen, kostet aber Latenz
  und Komplexität. Sinnvoll als optionaler Modus pro Plugin.
- **WASM-Plugins:** Abgeschottet, eine Datei für alle Plattformen, aber ohne
  native GUI und mit Leistungseinbußen.

## 10. Benutzeroberfläche (Tauri)

### 10.1 Technik

- Tauri 2 mit einem Web-Frontend. Framework-Wahl ist offen (Vorschlag: Svelte
  wegen geringer Laufzeitkosten und einfacher Reaktivität).
- Typen für Commands und Events werden aus Rust generiert (`specta` /
  `tauri-specta`), damit Frontend und Backend nicht auseinanderlaufen.
- Pedale, Regler und Kabel als SVG bzw. Canvas. Keine schweren
  Grafik-Bibliotheken ohne Not.

### 10.2 Ansichten

| Ansicht | Inhalt |
|---|---|
| Pedalboard | Pedale nebeneinander wie auf dem Boden, Drag & Drop, Fußschalter-Optik |
| Graph | Signalfluss mit Kabeln, für parallele Pfade und komplexe Rigs |
| Browser | Eingebaute Module, CLAP-Plugins, NAM-Profile, IRs, Presets, Suche und Filter |
| Tuner | Vollbild-Tuner, gut auf Distanz lesbar |
| Live-Modus | Große Kacheln für Presets und Snapshots, wenig Ablenkung |
| Einstellungen | Audiogerät, Kanäle, Samplerate, Puffer, MIDI-Geräte, Plugin-Pfade, Darstellung |

### 10.3 IPC-Schnittstelle (Entwurf)

Commands (Frontend → Backend):

```
audio.list_devices · audio.configure · audio.start · audio.stop
rig.get · rig.add_node · rig.remove_node · rig.move_node · rig.connect · rig.disconnect
rig.set_param · rig.set_bypass · rig.snapshot_save · rig.snapshot_recall
preset.list · preset.load · preset.save · preset.delete · preset.import · preset.export
plugin.list · plugin.rescan · plugin.open_gui · plugin.close_gui
midi.list_devices · midi.learn_start · midi.learn_cancel · midi.map_remove
assets.import_nam · assets.import_ir
```

Events (Backend → Frontend):

```
meters (≈30 Hz) · tuner (≈20 Hz) · rig.changed · param.changed (z. B. durch MIDI)
audio.state (läuft/gestoppt/Fehler/Gerät verloren) · plugin.scan_progress
cpu_load · xrun (Aussetzer) · midi.learned
```

Messwerte werden gedrosselt und nur bei relevanter Änderung gesendet.

### 10.4 Bedienung

- Tastaturkürzel für die wichtigsten Aktionen.
- Rückgängig/Wiederholen für Änderungen am Rig.
- Gut lesbar im Dunkeln (Bühne), Kontraste nach WCAG AA.

## 11. Presets und Persistenz

### 11.1 Format

JSON mit Versionsnummer. Beispiel:

```json
{
  "format": "plectyra-preset",
  "version": 1,
  "name": "Crunch Lead",
  "tags": ["crunch", "lead"],
  "tempo": 120.0,
  "graph": {
    "nodes": [
      { "id": "a1f…", "kind": { "builtin": "gate" }, "bypass": false,
        "params": { "threshold_db": -55.0, "release_ms": 80.0 } },
      { "id": "b2e…", "kind": { "nam": { "path": "models/plexi.nam", "sha256": "…" } },
        "params": { "input_db": 0.0, "output_db": -6.0 } },
      { "id": "c3d…", "kind": { "clap": { "id": "com.example.tapedelay", "version": "1.2.0" } },
        "state": "base64…" }
    ],
    "connections": [["input:0", "a1f:in"], ["a1f:out", "b2e:in"], ["b2e:out", "c3d:in"], ["c3d:out", "output:0"]]
  },
  "snapshots": [
    { "name": "Rhythm", "params": { "b2e.output_db": -9.0 } },
    { "name": "Solo", "params": { "b2e.output_db": -4.0 }, "bypass": { "c3d": false } }
  ],
  "midi_map": [
    { "source": { "device": "*", "channel": 1, "cc": 11 }, "target": "b2e.input_db", "range": [-12.0, 12.0] }
  ]
}
```

### 11.2 Regeln

- Jede Formatänderung erhöht `version`. Migrationen laufen beim Laden
  automatisch und sind getestet.
- Externe Dateien (NAM, IR) werden relativ zur Bibliothek referenziert und
  zusätzlich per Hash identifiziert, damit verschobene Dateien wiedergefunden
  werden.
- Export als Bundle (ZIP mit Preset, Modellen und IRs), sofern die Lizenzen
  der Dateien das erlauben.
- Speicherorte über das `directories`-Crate (plattformübliche Pfade).
  Einstellungen und Presets liegen getrennt.

## 12. MIDI-Steuerung

- MIDI-Ein- und Ausgabe über `midir`.
- **MIDI Learn:** Regler anklicken, Controller bewegen, fertig.
- Zuordnungen: CC → Parameter (mit Bereich und Kurve), CC/Note → Bypass
  (Toggle oder Momentary), Program Change → Preset, CC → Snapshot, Looper-
  Funktionen, Tap-Tempo.
- Globale Zuordnungen (gelten für alle Presets) und Preset-spezifische
  Zuordnungen.
- MIDI-Ausgabe für Rückmeldungen an Controller mit LEDs (später).
- Geräte-Hotplug: Neu angeschlossene Controller werden erkannt.
- MIDI-Gitarre (später): Noten-Events werden über Event-Ports an
  Instrument-Plugins geleitet.

## 13. Tests und Qualitätssicherung

| Ebene | Werkzeug / Vorgehen |
|---|---|
| Unit-Tests | `cargo test` für DSP-Bausteine, Graph-Compiler, Preset-Migration |
| Referenz-Renderings | Feste Testsignale (Sinus, Sweep, Impuls, aufgenommene Gitarren-DI) offline durch Module schicken und mit gespeicherten Referenzen vergleichen (Toleranz-basiert) |
| Echtzeit-Sicherheit | `assert_no_alloc` in Tests und Debug-Builds |
| Benchmarks | `criterion` für alle Module und typische Rigs, Regressionen in der CI sichtbar machen |
| Property-Tests | `proptest` für Graph-Operationen (keine Zyklen, keine verlorenen Verbindungen, Undo/Redo konsistent) |
| Plugin-Validierung | `clap-validator` für die Beispiel-Plugins, Host-Tests mit bekannten freien CLAP-Plugins |
| Latenzmessung | Loopback-Kabel vom Ausgang zum Eingang, Messung per Impuls, dokumentiert pro Plattform |
| Lint und Format | `cargo fmt --check`, `cargo clippy -- -D warnings`, Frontend-Linting |
| Manuelle Tests | Checkliste pro Release: Gerätewechsel, Abziehen des Interfaces, Preset-Wechsel unter Last, lange Laufzeit (Soak-Test über Stunden) |

## 14. Build, CI und Distribution

### 14.1 Entwicklungsablauf

- Hauptzweig `main`, Feature-Branches, Merge per Pull Request.
- Commit-Nachrichten nach Conventional Commits (`feat:`, `fix:`, `docs:` …).
- Versionierung nach SemVer, Changelog pro Release.

### 14.2 CI (GitHub Actions)

- Matrix: macOS (arm64), Windows (x64), Linux (x64).
- Schritte: Format, Clippy, Tests, Benchmarks (nur bei Bedarf), Frontend-Build,
  Tauri-Build.
- Release-Workflow bei Tags: Installer bauen, signieren, als GitHub-Release
  veröffentlichen.

### 14.3 Plattformspezifisches

**macOS**

- Universal-Binary oder getrennte Builds für arm64 und x64.
- Hardened Runtime, Signierung und Notarisierung.
- Entitlement für Audioeingang und `NSMicrophoneUsageDescription` in der
  `Info.plist`, sonst bekommt die App kein Eingangssignal.
- Zum Laden fremder Plugins unter Hardened Runtime ist
  `com.apple.security.cs.disable-library-validation` nötig.
- Auslieferung als `.dmg`.

**Windows**

- ASIO-Unterstützung erfordert das ASIO-SDK beim Build. Lizenzfrage klären
  (siehe Offene Entscheidungen).
- Installer als MSI oder NSIS über Tauri. WebView2 wird vorausgesetzt bzw.
  mitinstalliert.
- Code-Signing-Zertifikat für weniger SmartScreen-Warnungen (später).

**Linux**

- AppImage und `.deb`, später Flatpak.
- Flatpak-Sandbox erschwert den Zugriff auf Plugin-Pfade und JACK; das muss
  gesondert gelöst werden.
- Hinweise zu Echtzeit-Rechten (Gruppe `audio`, rtkit) in der Doku.
- Tauri nutzt WebKitGTK; Darstellungsunterschiede zu anderen Plattformen
  einplanen.

### 14.4 Lizenz

Vorschlag (noch nicht entschieden):

- App und Engine: **GPL-3.0-or-later**, wie MOD Desktop, OpenRig und Guitarix.
- `plectyra-plugin-sdk` und die Header der Plectyra-Erweiterung:
  **MIT OR Apache-2.0**, damit auch proprietäre Plugins möglich sind.
- Abhängigkeiten müssen dazu kompatibel sein (wird per `cargo-deny` in der CI
  geprüft).

## 15. Roadmap

Zeitangaben sind grobe Schätzungen für ein bis zwei Personen in der Freizeit.

### Phase 0 – Machbarkeit (1–2 Wochen)

Ziel: Klären, ob der gewählte Stack die Latenzanforderung erreicht.

- [ ] Cargo-Workspace und Tauri-2-Grundgerüst anlegen
- [ ] Audio-Durchschleifen mit cpal (Eingang → Gain → Ausgang) über die Duplex-Brücke
- [ ] Latenz per Loopback auf macOS, Windows (ASIO und WASAPI) und Linux (JACK/PipeWire) messen
- [ ] Ein Regler in der UI steuert den Gain über eine lock-freie Queue
- [ ] Pegelanzeige per gedrosseltem Event
- [ ] Minimale CI (Format, Clippy, Tests auf drei Plattformen)
- [ ] OpenRig im Detail prüfen (Architektur, UI, Plugin-Konzept, Aktivität)

Abnahme: Unter 10 ms vom Eingang bis zum Ausgang auf allen drei Plattformen
oder ein dokumentierter Plan B (eigenes Duplex-Backend).

### Phase 1 – MVP-Engine (4–6 Wochen)

- [ ] `plectyra-engine`: lineare Kette, Ausführungsplan, Plan-Austausch mit Überblendung
- [ ] Parameter-System mit Glättung
- [ ] Eingebaute Module: Gate, Kompressor, Drive (mit Oversampling), EQ, Cab-IR, Delay, Reverb, Tuner
- [ ] Preset-Format v1 mit Speichern und Laden
- [ ] UI: Pedalboard-Ansicht, Browser für eingebaute Module, Einstellungen
- [ ] Referenz-Renderings und `assert_no_alloc` in der CI

Abnahme: Ein Rig aus mindestens fünf Modulen ist spielbar, Preset-Wechsel
und Umbauen erzeugen keine Aussetzer.

### Phase 2 – Amp-Modelling (3–4 Wochen)

- [ ] NAM-Anbindung per FFI, Laden auf Worker-Thread
- [ ] Resampling um das Modell bei abweichender Samplerate
- [ ] Klassischer Amp als Alternative
- [ ] Modulationseffekte und Wah
- [ ] Bibliothek für NAM-Profile und IRs mit Import

Abnahme: Gängige `.nam`-Profile laufen auf Apple Silicon, Windows und Linux
mit vertretbarer CPU-Last.

### Phase 3 – CLAP-Host (4–6 Wochen)

- [ ] Scanner-Prozess mit Cache und Sperrliste
- [ ] Laden, Aktivieren und Verarbeiten von CLAP-Plugins im Graph
- [ ] Parameter, Zustand, Latenz, Tail
- [ ] Generische Pedal-UI aus Parametern, „Original-GUI öffnen"
- [ ] Platzhalter für fehlende Plugins

Abnahme: Mehrere bekannte freie CLAP-Plugins lassen sich ins Rig ziehen, in
Presets speichern und wiederherstellen.

### Phase 4 – MIDI und Live-Betrieb (2–3 Wochen)

- [ ] MIDI-Geräte, Hotplug, MIDI Learn
- [ ] Zuordnungen für Parameter, Bypass, Presets, Snapshots, Tap-Tempo
- [ ] Snapshots
- [ ] Looper
- [ ] Live-Modus in der UI

Abnahme: Ein Song lässt sich komplett mit einer MIDI-Fußleiste spielen, ohne
die Maus zu benutzen.

### Phase 5 – Graph und Routing (3–4 Wochen)

- [ ] Parallele Pfade, Split/Mix, Stereo
- [ ] Latenzausgleich zwischen Pfaden
- [ ] Graph-Ansicht mit Kabeln
- [ ] Rückgängig/Wiederholen

Abnahme: Zwei Amps parallel mit unterschiedlichen Boxen, phasenrichtig
gemischt.

### Phase 6 – SDK und Ökosystem (3–4 Wochen)

- [ ] Plectyra-Erweiterung `com.plectyra.pedal/1` finalisieren
- [ ] `plectyra-plugin-sdk`, Vorlage, C-Header
- [ ] Drei Beispiel-Pedale, `clap-validator` in der CI
- [ ] Doku „Ein eigenes Pedal in 30 Minuten"
- [ ] Optional: VST3-Hosting beginnen

Abnahme: Ein Entwickler ohne Vorkenntnisse des Projekts baut anhand der Doku
ein funktionierendes Pedal in unter einer Stunde.

### Phase 7 – Release 1.0 (2–3 Wochen)

- [ ] Signierte Installer bzw. Pakete für alle Plattformen
- [ ] Benutzerhandbuch, Erste-Schritte-Anleitung
- [ ] Mitgelieferte Presets und frei lizenzierte IRs
- [ ] Soak-Tests und Release-Checkliste
- [ ] Projektseite und Ankündigung

### Danach

- VST3-Hosting, Out-of-Process-Hosting, WASM-Plugins
- MIDI-Gitarre mit Instrument-Plugins, MIDI Clock
- Fernsteuerung per Smartphone (Web-UI im lokalen Netz)
- Plectyra selbst als CLAP/VST3-Plugin für DAWs
- Linux aarch64 (z. B. Raspberry Pi als Bodengerät)

## 16. Risiken

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| cpal-Duplex erreicht die Latenzziele nicht | Kernanforderung verfehlt | Phase 0 misst früh, Plan B: eigenes Duplex-Backend pro Plattform |
| Abstürzende Fremd-Plugins | App-Absturz während eines Auftritts | Scanner-Prozess, Sperrliste, später Out-of-Process-Hosting |
| CLAP-Main-Thread-Anforderungen kollidieren mit Tauris Eventloop | Fehlerhafte GUIs, Deadlocks | Früh in Phase 3 mit nativen Plugin-GUIs auf allen Plattformen testen |
| clack-host ist noch jung (0.x) | API-Änderungen, Lücken | Version pinnen, Abstraktionsschicht in `plectyra-host`, bei Bedarf beitragen |
| NAM-FFI und C++-Build auf drei Plattformen | Build-Komplexität | Früh in der CI bauen, Abhängigkeiten minimal halten |
| WebKitGTK unter Linux verhält sich anders | UI-Fehler nur unter Linux | Linux in der CI, regelmäßige manuelle Tests |
| ASIO-SDK-Lizenz | Konflikt mit der gewählten Lizenz | Lizenz früh festlegen, GPLv3-Variante des SDK prüfen |
| macOS-Signierung und Library Validation | Plugins laden nicht | Entitlements früh testen, nicht erst beim Release |
| Großer Umfang für kleines Team | Projekt bleibt stecken | Klare Phasen mit spielbarem Ergebnis nach jeder Phase |
| Markenrecht | Rechtliche Probleme | Keine Namen, Logos oder Optik kommerzieller Produkte nachahmen; Amp-Modelle neutral benennen |

## 17. Offene Entscheidungen

Diese Punkte werden vor bzw. während der genannten Phase entschieden und dann
ins Entscheidungslog übernommen.

| # | Frage | Vorschlag | Bis |
|---|---|---|---|
| 1 | Neu bauen, bei OpenRig mitarbeiten oder forken? | Nach Analyse in Phase 0 entscheiden | Ende Phase 0 |
| 2 | Lizenz für App und SDK | GPL-3.0-or-later für die App, MIT/Apache-2.0 für das SDK | Ende Phase 0 |
| 3 | Frontend-Framework | Svelte | Phase 0 |
| 4 | Duplex über cpal oder eigenes Backend | Abhängig von den Messungen | Ende Phase 0 |
| 5 | Plugin-Basis für das SDK: nih-plug oder clack-plugin | Abhängig von der Unterstützung eigener Erweiterungen | Phase 6 |
| 6 | Preset-Format JSON oder TOML/YAML | JSON (maschinenfreundlich, gut für Plugin-Zustände) | Phase 1 |
| 7 | Mindestversionen der Betriebssysteme | macOS 12, Windows 10, aktuelle LTS-Distributionen | Phase 7 |

## 18. Entscheidungslog

| Datum | Entscheidung | Begründung |
|---|---|---|
| 2026-11-06 | Projektname „Plectyra" | – |
| 2026-11-06 | Rust für Engine und DSP, Tauri 2 für die Oberfläche | Plattformübergreifend, Speichersicherheit, gutes Audio-Ökosystem |
| 2026-11-06 | Standalone-App als Primärprodukt | Ziel ist ein Rig, kein DAW-Plugin |
| 2026-11-06 | Audio als Hauptsignal, MIDI zur Steuerung | Normale Gitarren liefern kein MIDI |

## 19. Referenzen

- MOD Desktop: https://mod.audio/desktop/ · https://github.com/mod-audio/mod-desktop
- OpenRig: https://github.com/jpfaria/OpenRig
- AmpForge: https://github.com/Loursy/AmpForge
- Guitarix VST3-Port: https://github.com/maximalexanian/guitarix-vst
- NeuralAmpModelerCore: https://github.com/sdatkinson/NeuralAmpModelerCore
- NeuralAmpModeler-rs: https://docs.rs/crate/NeuralAmpModeler-rs/latest
- Rustone: https://github.com/acaala/rustone
- dsp-stuff: https://github.com/simmsb/dsp-stuff
- CLAP: https://github.com/free-audio/clap
- clack (CLAP für Rust): https://github.com/prokopyl/clack
- nih-plug: https://github.com/robbert-vdh/nih-plug
- clap-validator: https://github.com/free-audio/clap-validator
- cpal: https://github.com/RustAudio/cpal
- midir: https://github.com/Boddlnagg/midir
- Tauri: https://tauri.app
- VST 3 unter MIT-Lizenz: https://sonicstate.com/news/2025/10/30/vst-3-now-available-under-mit-license

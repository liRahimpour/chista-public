# Chista | Technische Kurzdokumentation

| |                                                                                              |
|---|----------------------------------------------------------------------------------------------|
| **Projekt** | Chista (Repository: `legacyhub` / `lh`)                                                      |
| **Zweck dieses Dokuments** | Technischer Anhang zur EXIST-Gründungsstipendium                                             |
| **Stand** | 01.08.2026                                                                                   |

---

## 1. Kurzüberblick 

Das System ist als **modularer Monolith** mit **hexagonaler Architektur** aufgebaut, verwendet **Polyglot Persistence** (je Datenbank ein klar abgegrenzter Zweck) und setzt für die Delphi-Codeanalyse einen **echten Compiler-Frontend-Parser (ANTLR4)** statt einfacher Mustererkennung ein.

---

## 2. Technologie-Stack

### Backend

| Technologie | Version | Zweck |
|---|---|---|
| Java | 21 | Laufzeitplattform |
| Spring Boot | 3.5.14 | Anwendungs-Framework |
| Spring Web (MVC) | - | REST-API |
| Spring Data JPA | - | Persistenz-Zugriff (PostgreSQL) |
| Spring Data Neo4j | - | Persistenz-Zugriff (Graphdatenbank) |
| Spring Validation | - | Eingabevalidierung |
| Spring Boot Actuator | - | Health-/Betriebs-Endpunkte |
| Flyway | - | Versionierte Datenbank-Migrationen (PostgreSQL) |
| Maven | - | Build- und Abhängigkeitsmanagement |
| MinIO Java SDK | 9.0.1 | Zugriff auf Objektspeicher |
| springdoc-openapi | 2.8.17 | Automatische OpenAPI-/Swagger-Dokumentation |
| ANTLR4 (Runtime + Maven-Plugin) | 4.13.2 | Grammatikbasiertes Parsen von Delphi-Quellcode |

### Frontend

| Technologie | Version | Zweck |
|---|---|---|
| React | 19.2 | UI-Framework |
| TypeScript | ~6.0 | Typisierung |
| Vite | 8.1 | Build-Tool / Dev-Server |
| @xyflow/react (React Flow) | 12.11 | Interaktive Graph-Visualisierung |
| @dagrejs/dagre | 3.0 | Automatisches Graph-Layout |
| Tailwind CSS | 4.3 | Styling |

### Infrastruktur

| Technologie | Zweck |
|---|---|
| Docker Compose | Orchestrierung der lokalen/Server-Infrastruktur |
| PostgreSQL | Relationale Datenhaltung, Quelle der Wahrheit |
| MinIO | S3-kompatibler Objektspeicher für hochgeladene Quelldateien |
| Neo4j 5 (Community) | Graphdatenbank für Abfragen über Code-Beziehungen |
| GitHub Actions | CI (getrennte Pipelines für Backend und Frontend) |

---

## 3. Architektur

### 3.1 Architekturstil und Begründung

**Modularer Monolith** statt Microservices:

- Ein Team in der Gründungsphase hat keinen operativen Vorteil aus verteilten Services (Netzwerklatenz, Service-Discovery, eventual consistency), sondern nur zusätzliche Komplexität.
- Durch strikte interne Modulgrenzen bleibt die Option erhalten, einzelne Module später als eigenständige Services herauszulösen, **falls** Skalierungsanforderungen das rechtfertigen.

**Hexagonale Architektur (Ports & Adapters)**  konsequent in jedem fachlichen Modul umgesetzt:

```
modul/
├── domain/          fachliche Objekte, keine Framework-Abhängigkeiten
├── application/      Anwendungsfälle / Orchestrierung
├── ports/             Schnittstellen (was das Modul nach außen braucht/anbietet)
└── infrastructure/    konkrete technische Umsetzung (DB, REST, externe Bibliotheken)
```

Dass dieses Prinzip in der Praxis trägt und kein Lehrbuch-Ideal bleibt, zeigt sich am Beispiel der Delphi-Codeanalyse, die von einer regelbasierten Regex-Implementierung auf einen echten ANTLR4-Parser umgestellt wurde, ohne dass Service-Schicht, REST-Controller oder Datenmodell dafür verändert werden mussten, da beide Implementierungen lediglich denselben `SourceFileAnalyzer`-Port erfüllen. Für Chista ist das mehr als ein einmaliger Vorteil, denn im Verlauf der Entwicklung und erst recht im produktiven Betrieb werden absehbar laufend weitere Programmiersprachen und Quellsysteme hinzukommen oder auch wieder entfernt werden müssen, etwa C#, weitere Delphi-Dialekte oder künftig andere Formate. Ein neuer Analyzer lässt sich dafür einfach als zusätzlicher Adapter hinter demselben Port ergänzen, ohne bestehenden Code zu berühren oder bestehende Sprachen zu gefährden, und nicht mehr benötigte Analyzer lassen sich ebenso isoliert wieder entfernen.
### 3.2 Backend-Module

| Modul | Verantwortung |
|---|---|
| `project` | Projektverwaltung |
| `sourcefile` | Metadaten hochgeladener Quelldateien |
| `storage` | Abstraktion über den Dateispeicher (MinIO) |
| `upload` | Einzeldatei- und ZIP-Archiv-Import |
| `analysis` | Symbol-Erkennung pro Sprache (inkl. Parser-Infrastruktur) |
| `codesymbol` | Persistenz und Zugriff auf erkannte Code-Symbole |
| `coderelation` | Persistenz und Zugriff auf erkannte Beziehungen zwischen Symbolen |
| `graph` | Synchronisation und Abfrage des Wissensgraphen (Neo4j) |
| `shared` / `config` | Querschnittsfunktionen (Health-Check, CORS-Konfiguration) |

### 3.3 Datenfluss (Ende-zu-Ende)

```
Upload (Einzeldatei / ZIP)
        │
        ▼
Spracherkennung + SHA-256-Hashing
        │
        ├──► MinIO       (Rohdatei)
        └──► PostgreSQL   (Metadaten: Projekt, Datei, Sprache, Pfad)
        │
        ▼
Symbol-Analyse  (POST /analysis/symbols)
        │  je Sprache ein Analyzer hinter dem SourceFileAnalyzer-Port
        ▼
PostgreSQL (code_symbols)
        │
        ▼
Beziehungs-Analyse  (POST /analysis/relations)
        │  je Sprache ein Analyzer hinter dem SourceFileRelationAnalyzer-Port
        ▼
PostgreSQL (code_relations)
        │
        ▼
Graph-Synchronisation  (POST /graph/sync)
        │  vollständiger Neuaufbau aus PostgreSQL ( Postgres bleibt Quelle der Wahrheit)
        ▼
Neo4j  (Graph-Projektion für Traversal-Abfragen)
        │
        ▼
Frontend: React Flow Graph-Explorer
   (Ansichten: Full / Neighborhood / Dependencies / Impact, Filter, Suche)
```

### 3.4 Lokale Entwicklungsumgebung

Die lokale Entwicklungsumgebung liegt bewusst möglichst nah an der späteren Betriebsumgebung, keine Mock-Datenbanken, sondern dieselben Systeme (PostgreSQL, MinIO, Neo4j), nur lokal in Docker statt produktiv gehostet.

**Zwei getrennte Docker-Compose-Dateien**:

- `infrastructure/docker-compose.yml`, die drei Datenspeicher (PostgreSQL, MinIO, Neo4j und perspektivisch ergänzt um Qdrant) auf einem gemeinsamen, externen Docker-Netzwerk. PostgreSQL läuft dabei über ein eigenes, minimales Dockerfile (`lh-dev-postgres/`) statt des Standard-Images, um später eigene Initialisierungsskripte einhängen zu können, ohne die Compose-Datei selbst anzufassen.
- `infrastructure/docker-compose.backend.yml`, das Backend selbst als Container, der sich über denselben Netzwerknamen mit den drei Datenspeichern verbindet.

Diese Trennung erlaubt es, die Infrastruktur dauerhaft laufen zu lassen und nur das Backend bei Bedarf neu zu bauen und neu zu starten, statt bei jeder Codeänderung den gesamten Stack neu hochzufahren.

**Konfiguration über Umgebungsvariablen statt hartkodierter Werte**: Die Spring-Boot-Konfiguration (`application.yaml`) nutzt durchgehend das Muster `${ENV_VAR:xxxxxxx}`; Datenbank-, MinIO- und Neo4j-Verbindungsdaten werden über Umgebungsvariablen aus der jeweiligen Compose-Datei injiziert, mit lokalen Defaults (`localhost`) für den Fall, dass das Backend direkt über Maven statt im Container gestartet wird.

**Eigenes Profil für CI**: `application-ci.yaml` ist eine getrennte Konfiguration für GitHub Actions, in der Docker-Compose-Umgebung sind die Datenspeicher über Service-Namen (`lh-minio` etc.) erreichbar, in GitHub Actions laufen sie dagegen als direkt gemappte Ports auf `localhost`. Ohne dieses getrennte Profil würden die CI-Tests mit `UnknownHostException` fehlschlagen.

**Frontend**: Vite-Dev-Server (`npm run dev`), Hot-Module-Reload gegen das lokal laufende Backend. Die CORS-Konfiguration des Backends (`app.cors.allowed-origins`) ist bereits standardmäßig auf den Vite-Standardport (`http://localhost:5173`) eingestellt, Frontend und Backend sind ohne manuelle Zusatzkonfiguration sofort miteinander lauffähig.

---

## 4. Datenhaltung: Polyglot Persistence

Entscheidung gegen "eine Datenbank für alles". Jede Datenbank wird für den Zweck eingesetzt, für den sie strukturell am besten geeignet ist:

| Datenbank | Rolle | Warum diese Wahl                                                                                                                                                                           |
|---|---|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **PostgreSQL** | Quelle der Wahrheit für Projekte, Dateien, Symbole, Beziehungen | Transaktionale Integrität, referenzielle Konsistenz (Fremdschlüssel mit `ON DELETE CASCADE`), ausgereiftes Ökosystem (Flyway-Migrationen, JPA)                                             |
| **MinIO** | Objektspeicher für hochgeladene Roh-Quelldateien | Binärdateien/Textdateien gehören strukturell nicht in eine relationale DB; S3-kompatible Objektspeicher sind dafür der Industriestandard, austauschbar gegen echtes S3 im Produktivbetrieb |
| **Neo4j** | Graph-Projektion für Traversal-Abfragen (Abhängigkeiten, Impact-Analyse, Nachbarschaft) | Fragen wie *"was hängt transitiv von Symbol X ab"* sind in einer Graphdatenbank nativ und performant lösbar, in reinem SQL nur über aufwändige rekursive Queries                           |
| **Qdrant** *(geplant)* | Vektorspeicher für Embeddings von Code- und Wissens-Chunks, Grundlage der semantischen Suche und der KI-Schicht | Spezialisiert auf Ähnlichkeitssuche über große Vektormengen, ergänzt Neo4j um die semantische Dimension, während Neo4j für die strukturelle Dimension (Beziehungen) zuständig bleibt       |

**Konsistenzmodell**: PostgreSQL ist die alleinige Quelle der Wahrheit. Neo4j wird bei jeder Synchronisation vollständig gelöscht und aus PostgreSQL neu aufgebaut (`GraphSyncService.syncProject`), dadurch entstehen keine Dual-Write-Konsistenzprobleme zwischen den beiden Systemen; Neo4j ist jederzeit reproduzierbar.

---

## 5. Delphi-Codeanalyse mit ANTLR4

### 5.1 Ausgangslage und Entscheidung

Die erste Version der Symbol-Erkennung nutzte reguläre Ausdrücke (Regex). Das funktioniert für einfache Beispieldateien. Bei echtem Legacy-Code scheitert es jedoch; etwa bei mehrzeiligen Methodensignaturen, verschachtelten Typen oder Kommentaren mit codeähnlichem Inhalt. Verlässliche Code-Analyse ist die Kernwertaussage von Chista, deshalb war dieser Ansatz nicht tragfähig.
**Entscheidung**: Ersatz durch einen echten, grammatikbasierten Parser mit **ANTLR4**, demselben Werkzeug-Prinzip, das auch professionelle Static-Analysis-Tools (z. B. SonarQube) einsetzen.

**Grammatik-Herkunft**: Die verwendete Object-Pascal/Delphi-Grammatik stammt ursprünglich aus dem SonarSource-SonarDelphi-Plugin (ANTLR3), wurde von der Community zu ANTLR4 migriert und für dieses Projekt für Java als Zielsprache angepasst (u. a. Behandlung der in Pascal groß-/kleinschreibungsunabhängigen Schlüsselwörter über einen dedizierten `CaseChangingCharStream`). Lizenz: LGPL-3.0 für die aktuelle Prototyping-Phase unkritisch, vor kommerzieller Auslieferung rechtlich zu prüfen (dokumentiert als offener Punkt).

### 5.2 Parser-Pipeline

```
Delphi-Quelltext (String)
        │
        ▼
DelphiLexer / DelphiParser        (aus der .g4-Grammatik generiert, ANTLR4)
        │
        ▼
DelphiParsedSourceVisitor          (läuft den Syntaxbaum ab)
        │
        ▼
Sprachneutrales Zwischenmodell     (ParsedElement / ParsedRelation / ParsedSource)
        │
        ▼
Mapping-Schicht                    (ParsedElement → DetectedSymbol,
        │                            ParsedRelation → DetectedRelation)
        ▼
Bestehende Persistenz-Pipeline (PostgreSQL / Neo4j), unverändert
```

Das Zwischenmodell (`ParsedElement`, `ParsedElementKind`, `ParsedRelation`, `ParsedRelationKind`) ist **sprachneutral** konzipiert, nicht Delphi-spezifisch. Ein zukünftiger C#- oder Java-Parser kann dieselbe Zielstruktur befüllen, ohne die Mapping- oder Persistenzschicht zu verändern. Die Erweiterung auf weitere Sprachen ist damit strukturell vorbereitet, nicht nur beabsichtigt.


---

## 6. API-Referenz

Basis-Pfad: `/api`: generiert über springdoc-openapi (Swagger UI).

| Bereich | Methode | Pfad | Zweck |
|---|---|---|---|
| Health | GET | `/ping` | Liveness-Check |
| Projekte | POST | `/projects` | Projekt anlegen |
| Projekte | GET | `/projects` | Alle Projekte |
| Projekte | GET | `/projects/{id}` | Einzelnes Projekt |
| Quelldateien | GET | `/projects/{projectId}/source-files` | Dateien eines Projekts |
| Quelldateien | GET | `/projects/{projectId}/source-files/{id}` | Einzelne Datei |
| Upload | POST | `/projects/{projectId}/uploads/source-file` | Einzeldatei-Upload |
| Upload | POST | `/projects/{projectId}/uploads/source-archive` | ZIP-Archiv-Upload |
| Analyse | POST | `/projects/{projectId}/analysis/symbols` | Symbol-Analyse anstoßen |
| Analyse | POST | `/projects/{projectId}/analysis/relations` | Beziehungs-Analyse anstoßen |
| Symbole | GET | `/projects/{projectId}/symbols` | Alle Symbole |
| Symbole | GET | `/projects/{projectId}/symbols/{id}` | Einzelnes Symbol |
| Beziehungen | GET | `/projects/{projectId}/relations` | Alle Beziehungen |
| Beziehungen | GET | `/projects/{projectId}/relations/{id}` | Einzelne Beziehung |
| Beziehungen | GET | `/projects/{projectId}/relations/symbols/{symbolId}/outgoing` | Ausgehende Beziehungen eines Symbols |
| Beziehungen | GET | `/projects/{projectId}/relations/symbols/{symbolId}/incoming` | Eingehende Beziehungen eines Symbols |
| Graph | POST | `/projects/{projectId}/graph/sync` | Neo4j-Synchronisation |
| Graph | GET | `/projects/{projectId}/graph/overview` | Graph-Übersicht |
| Graph | GET | `/projects/{projectId}/graph/symbols/{id}/neighbors` | Direkte Nachbarschaft |
| Graph | GET | `/projects/{projectId}/graph/symbols/{id}/dependencies` | Abhängigkeiten (transitiv) |
| Graph | GET | `/projects/{projectId}/graph/symbols/{id}/impact` | Auswirkungsanalyse (transitiv) |
| Graph | GET | `/projects/{projectId}/graph/visualization` | Aufbereitete Daten für die Frontend-Visualisierung (Filter: Ansicht, Tiefe, Suche, Symboltypen, Beziehungstypen) |
| Graph | GET | `/graph/health` | Neo4j-Verbindungsstatus |

---

## 7. Frontend-Architektur

React-Single-Page-Application mit dem **Graph Explorer** als zentraler Ansicht:

- **GraphExplorerPage**: orchestriert Zustand (aktive Ansicht, Filter, Auswahl) und API-Aufrufe
- **GraphCanvas**: React-Flow-Rendering, automatisches Layout via `dagre`, visuelle Hervorhebung ausgewählter Knoten/Kanten
- **GraphFilters**: Steuerung von Ansicht (Full/Neighborhood/Dependencies/Impact), Tiefe, Suche, Symbol-/Beziehungstypen
- **NodeDetailsPanel**: Detailinformationen zum gewählten Symbol, Sprung in weitere Ansichten

---

## 8. Qualitätssicherung

- Die vorhandenen Tests sind gezielte Einzeltests je Komponente, entstanden begleitend zur jeweiligen Implementierung; noch kein übergreifendes, systematisches Testkonzept. Der Aufbau eines vollständigen Testkonzepts mit dem Ziel einer nahezu vollständigen Abdeckung ist als eigenes Arbeitspaket für den Förderzeitraum vorgesehen, bestehend aus Unit-Tests, Integrationstests, End-to-End-/Smoke-Tests, Performance-Tests sowie dem systematischen Aufbau von Testcontainern für reale Datenbank- und Infrastruktur-Abhängigkeiten.
- **CI/CD**: GitHub Actions mit getrennten Pipelines für Backend (`mvn clean verify` gegen echte PostgreSQL- und MinIO-Testcontainer) und Frontend (`npm run build`), ausgelöst bei Push/PR auf `main`, `dev` und `feature/**`.

---

## 9. Sicherheit - aktueller Stand


- **Aktuell**: keine Authentifizierung/Autorisierung implementiert, da es noch nicht nötig ist. CORS ist konfigurierbar (`app.cors.allowed-origins`), aber es gibt keinen Zugriffsschutz auf API-Ebene. Das ist für den aktuellen Prototyping-Stand (lokale Demos, keine produktiven Kundendaten) eine bewusst nachrangig priorisierte Entscheidung.
- **Später**: Keycloak-basierte Authentifizierung mit OAuth2/JWT.
- Vor jedem produktiven bzw. kundenbezogenen Einsatz zwingend erforderlich, entsprechend priorisiert für die nächste Entwicklungsphase.
---

## 10. Technische Umsetzung der KI-Schicht (Plan)

Der Kontext für das LLM soll nicht einfach durch das Zerschneiden ganzer Dateien entstehen, sondern gezielt aus dem bereits vorhandenen Wissensgraphen zusammengestellt werden. Wir nutzen dafür keine fertige RAG-Bibliothek, sondern bauen die Logik selbst, um volle Kontrolle über Genauigkeit und Nachvollziehbarkeit zu behalten.

Geplante Schritte:

1. **Chunking anhand vorhandener Symbolgrenzen**: Jedes `CodeSymbol` speichert bereits Start- und Endzeile einer Klasse oder Methode. Diese Grenzen nutzen wir direkt als Chunk-Grenzen, statt Text willkürlich nach Tokenanzahl zu zerschneiden. Die Struktur aus der Graph-Analyse wird damit für die KI-Schicht wiederverwendet, nicht neu erfunden.
2. **Tokenisierung und Embedding**: Jeder Chunk wird über ein lokales Embedding-Modell in einen Vektor umgewandelt und in **Qdrant** gespeichert, zusammen mit einer Referenz auf das zugehörige Symbol.
3. **Kontext in zwei Schritten**: Bei einer Anfrage holen wir nicht nur ähnliche Textstellen aus Qdrant, sondern zusätzlich die strukturell relevante Umgebung aus dem Neo4j-Graphen. Also z. B. wer eine Methode aufruft oder wovon sie abhängt.
4. **Rückverfolgbarkeit**: Jede Information im Kontext bleibt über die Symbol-ID bis zur genauen Datei und Zeile nachvollziehbar.

**Weitere Quellen**: Der Ansatz ist nicht auf Quellcode beschränkt. Perspektivisch sollen weitere Wissensquellen auf dieselbe Weise angebunden werden, etwa Dokumentation, Tickets, Wikis oder Commit-Historien. Sie würden nach demselben Muster verarbeitet: in sinnvolle Abschnitte zerlegt, eingebettet, in Qdrant abgelegt und mit den passenden Symbolen im Wissensgraphen verknüpft.

---

## 11. Architekturentscheidungen im Überblick

| Entscheidung | Verworfene Alternative | Kernbegründung                                                                                                                                                   |
|---|---|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Modularer Monolith | Microservices von Anfang an | Kein Skalierungsdruck, der verteilte Systeme rechtfertigt; unnötige Betriebskomplexität für die aktuelle Phase; Modulgrenzen halten die spätere Aufteilung offen |
| Hexagonale Architektur je Modul | Klassische geschichtete Architektur ohne Ports | Fachlogik bleibt unabhängig von Framework/Infrastruktur; nachgewiesen am Austausch Regex→ANTLR-Parser ohne Änderungen an Service-/API-Schicht                    |
| PostgreSQL als Quelle der Wahrheit | Neo4j als primärer Speicher | Transaktionale Integrität und ausgereiftes Migrationsmanagement (Flyway) wichtiger als native Graph-Operationen für den Schreibpfad                              |
| Neo4j als Graph-Projektion (nicht primär) | Graph-Abfragen direkt in PostgreSQL (rekursive SQL-CTEs) | Traversal-/Impact-Analysen sind in einer Graphdatenbank nativ und deutlich performanter; durch vollständigen Neuaufbau bei Sync keine Konsistenzrisiken          |
| MinIO für Rohdateien statt DB-BLOBs | Dateien als BLOB in PostgreSQL | Objektspeicher ist der Industriestandard für Binär-/Textdateien dieser Größenordnung; austauschbar gegen echtes S3                                               |
| ANTLR4-Parser statt Regex für Delphi | Regex weiter ausbauen | Regex ist strukturell ungeeignet für mehrzeilige/verschachtelte Legacy-Syntax; Verlässlichkeit der Analyse ist die Kernwertaussage des Produkts                  |
| Sprachneutrales Zwischenmodell (ParsedElement/-Relation) | Delphi-spezifisches Domänenmodell direkt im Visitor | Wiederverwendbarkeit für zukünftige Sprachen ohne Redesign der Persistenzschicht                                                                                 |
| Selbst gehostete Infrastruktur (Docker: Postgres/MinIO/Neo4j) | Managed-Cloud-Dienste | Vorbereitung auf lokale/on-premise Betriebsfähigkeit, Voraussetzung für die geplante Datenschutz-/Souveränitäts-Positionierung                                   |

---

## 12. Aktueller Entwicklungsstand vs. Roadmap

### Implementiert und lauffähig

- Projekt-, Datei- und Upload-Verwaltung (Einzeldatei und ZIP)
- Automatische Spracherkennung, SHA-256-Hashing
- Symbol- und Beziehungs-Analyse für Java (RegEx)
- Symbol- und Beziehungs-Analyse für Delphi über echten ANTLR4-Parser
- Vollständige Persistenzkette PostgreSQL ↔ Neo4j-Synchronisation
- Interaktiver Graph-Explorer (React Flow) mit mehreren Analyseansichten
- REST-API mit automatisch generierter OpenAPI-Dokumentation
- CI/CD-Pipelines für Backend und Frontend

### In progress

Der konkrete Umfang dieser Phase wird bewusst nicht als starre Feature-Liste festgeschrieben. Eine detailliertere Planung ist in der Datei `roadmap.md` beschrieben. Ein wesentlicher Teil wird zudem erst durch Forschung, Tests und Validierung während des Förderzeitraums selbst bestimmt. Die grobe Richtung:

**KI-Schicht**: Aktuell befinden wir uns in einer Selbstlern- und Explorationsphase. wir evaluieren und testen verschiedene lokale, schlanke Sprachmodelle (SLMs) wie Phi oder Llama als Kandidaten für eine selbst-hostbare, datenschutzkonforme RAG-Schicht. Parallel dazu entwickeln wir das Konzept für einen Algorithmus zur Bildung von Inferenzketten auf Basis des Wissensgraphen, also Schlussfolgerungsketten, die entlang klar unterscheidbarer Stufen (Fakt, Annahme, Hypothese) nachvollziehbar bleiben, statt Ergebnisse als vermeintlich sichere Fakten auszugeben.

**CLI und Kubernetes-Betrieb**: Für automatisierte, skriptbasierte Nutzung, etwa Einbindung in CI/CD-Pipelines von Kunden oder Batch-Analysen ohne UI ist eine eigene CLI vorgesehen, die dieselbe REST-API anspricht wie das Frontend. Die Anwendung wird durchgehend Cloud-native ausgelegt. Der Backend-Prozess ist zustandslos, läuft containerisiert und wird bereits heute vollständig über Umgebungsvariablen statt hartkodierter Werte konfiguriert (siehe 3.4). Für den produktiven Einsatz bei Kunden mit eigener Kubernetes-Infrastruktur ist zusätzlich ein Helm-Chart geplant, das den bestehenden Docker-Compose-Aufbau in ein versioniertes Kubernetes-Deployment überführt.

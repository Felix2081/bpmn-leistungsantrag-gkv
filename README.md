# Digitaler Leistungsantrag in der GKV – BPMN 2.0 & Camunda 8

<a href="https://www.credly.com/badges/6c178145-c9fc-41f1-8b55-721d4f2d8ac2/public_url">
  <img src="docs/badge-bpmn.png" alt="Camunda Knowledge – BPMN" width="140">
</a>

Portfolio-Projekt zur Analyse und Digitalisierung eines Leistungsantrags bei einer gesetzlichen Krankenversicherung – vom manuellen Ist-Prozess über ein strategisches Zielbild bis zum operativen, für Camunda 8 vorbereiteten Soll-Prozess.

## Ausgangslage

Leistungsanträge (z. B. häusliche Krankenpflege oder Hilfsmittel zur Krankenbehandlung) werden häufig noch teilweise papierbasiert und manuell bearbeitet. Gleichzeitig gelten gesetzliche Vorgaben mit harter Rechtsfolge:

| Regel | Inhalt |
|---|---|
| Entscheidungsfrist (§ 13 Abs. 3a SGB V) | 3 Wochen nach Antragseingang |
| Mit Gutachten des Medizinischen Dienstes | 5 Wochen – der Versicherte muss darüber informiert werden |
| Fristüberschreitung ohne Begründung | Leistung gilt als genehmigt (Genehmigungsfiktion) |
| Gutachtenpflicht (§ 275 Abs. 1 SGB V) | In gesetzlich bestimmten Fällen **oder** wenn es nach Art, Schwere, Dauer oder Häufigkeit der Erkrankung erforderlich ist |

**Ziel:** Fristen automatisch überwachen, eindeutige Fälle regelbasiert vorentscheiden, Ermessensfälle gezielt der Sachbearbeitung vorlegen und Bescheide digital erzeugen.

## Projektstufen

| Stufe | Inhalt | Status |
|---|---|---|
| 1. Analyse & Konzept | Ist-Prozess mit Schwachstellenanalyse, strategisches Zielbild, operativer Soll-Prozess inkl. Fristenlogik | ✅ abgeschlossen |
| 2. Ausführbar machen | Camunda 8 Run, Datenmodell, DMN-Tabelle, Camunda Forms, Nachrichten-Korrelation, Connectors | ⏳ geplant |
| 3. Auswertung | Kennzahlen (Durchlaufzeit, Fristquote, Automatisierungsgrad) im Ist/Soll-Vergleich | ⏳ geplant |

## Ist-Prozess

![Ist-Prozess Leistungsantrag](docs/ist-leistungsantrag.png)

Modelldatei: [`models/ist-leistungsantrag.bpmn`](models/ist-leistungsantrag.bpmn)

Der Ist-Prozess bildet eine typische, überwiegend manuelle Antragsbearbeitung ab: Der Antrag geht per Post ein, wird gescannt und in der Sachbearbeitung auf Vollständigkeit und Leistungsanspruch geprüft. Bei Bedarf wird ein Gutachten des Medizinischen Dienstes eingeholt, anschließend wird entschieden und ein Bescheid versendet.

### Schwachstellenanalyse

| # | Schwachstelle im Ist-Prozess | Folge |
|---|---|---|
| 1 | Keine Überwachung der gesetzlichen Entscheidungsfrist | Fristüberschreitung bleibt unbemerkt → Risiko der Genehmigungsfiktion |
| 2 | Keine Zwischenmitteilung an den Versicherten bei Einschaltung des MD | Die verlängerte 5-Wochen-Frist greift nicht, es bleibt bei 3 Wochen |
| 3 | Unbegrenztes Warten auf nachgereichte Unterlagen und auf das Gutachten | Vorgänge bleiben ohne Reaktion liegen |
| 4 | Medienbruch durch Papierantrag und Scannen – auch bei jeder Nachreichung | Zusätzlicher Aufwand und Liegezeit in der Poststelle |
| 5 | Manuelle Vollständigkeitsprüfung | Nachforderungen erst nach Sichtung, zusätzliche Schleifen |
| 6 | Uneinheitliche Entscheidung, ob ein Gutachten erforderlich ist | Pflichtfälle können übersehen werden, Zeitverlust |
| 7 | Mehrere Übergaben zwischen Poststelle und Sachbearbeitung | Liegezeiten an jeder Übergabe |
| 8 | Bescheide werden manuell erstellt und versendet | Aufwand, Fehleranfälligkeit |

## Soll-Prozess

Der Soll-Prozess ist auf drei Abstraktionsebenen modelliert – jeweils passend zur Zielgruppe.

### Ebene 0 – Strategisches Zielbild (Management)

![Strategischer Prozess](docs/strategisch-leistungsantrag.png)

Modelldatei: [`models/strategisch-leistungsantrag.bpmn`](models/strategisch-leistungsantrag.bpmn)

Nur der Hauptablauf und die eine fachlich entscheidende Verzweigung (MD-Gutachten → Fristlänge). Keine Lanes, keine Aufgabentypen, keine Ausnahmepfade.

### Ebene 1 – Operativer Prozess (Fachbereich & IT)

![Operativer Soll-Prozess](docs/soll-leistungsantrag.png)

Modelldatei: [`models/soll-leistungsantrag.bpmn`](models/soll-leistungsantrag.bpmn) – Camunda-8-Diagramm, ohne Import-Warnungen

Die Lane **System** enthält alle Schritte der Process Engine, **Sachbearbeitung** und **Teamleitung** nur noch Aufgaben, die fachliches Urteil erfordern. Wiederkehrende Kommunikationsschleifen sind in zugeklappte Unterprozesse gekapselt.

### Ebene 2 – Unterprozesse

**Unterlagen nachfordern**

![Unterprozess Unterlagen nachfordern](docs/soll-unterlagen-nachfordern.png)

**MD-Gutachten einholen**

![Unterprozess MD-Gutachten einholen](docs/soll-gutachten-einholen.png)

### Wie der Soll-Prozess die Schwachstellen löst

| # | Schwachstelle | Lösung im Modell |
|---|---|---|
| 1 | Keine Fristüberwachung | Ereignis-Unterprozess „Frist überwachen“ mit nicht unterbrechendem Timer-Start – die Teamleitung priorisiert vor Fristablauf |
| 2 | Keine Zwischenmitteilung | Sendeaufgabe „Versicherten informieren“ unmittelbar nach dem Gutachtenauftrag |
| 3 | Unbegrenztes Warten | Empfangsaufgaben mit nicht unterbrechenden Erinnerungs-Timern (14 bzw. 21 Tage) in beiden Unterprozessen |
| 4 | Medienbruch | Digitaler Eingang über ein Nachrichten-Startereignis, die Poststelle entfällt |
| 5 | Manuelle Vollständigkeitsprüfung | Das Online-Formular sichert die formale Vollständigkeit; die fachliche Prüfung bleibt in „Antrag prüfen“ |
| 6 | Uneinheitliche Gutachtenentscheidung | DMN ermittelt gesetzliche Pflichtfälle; Ermessensfälle nach § 275 Abs. 1 SGB V prüft die Sachbearbeitung. Eine Schutzbedingung verhindert „kein Gutachten“ bei einem Pflichtfall |
| 7 | Mehrfache Übergaben | Eine Benutzeraufgabe „Antrag prüfen“ bündelt Unterlagen- und Gutachtenprüfung |
| 8 | Manuelle Bescheiderstellung | Eine Sendeaufgabe erzeugt den Bescheid aus der passenden Vorlage |

**Zusätzlich modelliert:** Antragsrücknahme durch den Versicherten (unterbrechender Ereignis-Unterprozess) und der Datenspeicher „Kernsystem (21c ng)“, aus dem die Prüfung liest und in den die Entscheidung schreibt.

### Angewandte Modellierungsregeln

- Aufgaben als Verb + Objekt, Ereignisse als Zustand, Gateways als Frage, verzweigende Pfade beschriftet
- Explizite Zusammenführungen, keine kombinierten Split-/Merge-Gateways, keine impliziten Joins
- Hauptfluss von links nach rechts ohne Kreuzungen von Sequenzflüssen
- Externe Beteiligte als zugeklappte Pools, Kommunikation ausschließlich über Nachrichtenflüsse
- Sprechende technische IDs ohne Umlaute, Gateway-Bedingungen bereits in FEEL hinterlegt

### Nicht Teil des Modells

- **Widerspruchsverfahren** – eigener, nachgelagerter Prozess
- **Genehmigungsfiktion** – Rechtsfolge, kein Prozesspfad; die Fristüberwachung soll sie verhindern
- **Versagung wegen fehlender Mitwirkung** (§ 66 SGB I)
- **Hilfsmittel zum Behinderungsausgleich** – unterliegen § 18 SGB IX (2-Monats-Frist)
- **Technische Umsetzung** der Sende- und Empfangsaufgaben – Inhalt von Stufe 2

## Repository-Struktur

```
├── models/
│   ├── ist-leistungsantrag.bpmn
│   ├── strategisch-leistungsantrag.bpmn
│   └── soll-leistungsantrag.bpmn
├── docs/        Diagramm-Exporte und Badge
└── README.md
```

## Technik

- BPMN 2.0, DMN, FEEL
- Camunda 8 (Desktop Modeler, Zeebe Engine, Tasklist, Operate)
- Lokale Ausführung über Camunda 8 Run (Stufe 2)

## Hinweis

Fiktives Lernprojekt ohne echte Versichertendaten. Die rechtlichen Rahmenbedingungen dienen als fachliche Grundlage der Modellierung und stellen keine Rechtsberatung dar.

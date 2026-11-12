# 🧠 KI-gestütztes Adaptives Lernsystem

Ein Telegram-Bot zur automatisierten Wissensabfrage mit KI-generierten Quizfragen,
gekoppelt an Notion als Datenbank und einem Spaced-Repetition-Algorithmus (SM-2)
zur individuell angepassten Wiederholungsplanung.

> Privates Lernprojekt | n8n (self-hosted), Notion, OpenAI GPT-4o-mini, Telegram

Ausgelegt für einen Nutzer: Fragen und Feedback gehen an einen Telegram-Chat,
die Statistik aktualisiert den ersten Eintrag in User_Stats.

## 📌 Motivation

Beim Lernen für Prüfungen (IHK, Abitur, Studium) reicht reines Lesen von
Unterlagen oft nicht aus – aktives Abfragen zum richtigen Zeitpunkt verbessert
die Merkfähigkeit deutlich (Spaced Repetition). Ziel war ein System, das:

- eigenes Lernmaterial automatisch in Quizfragen umwandelt
- fällige Themen zuerst auswählt, das Intervall dazu liegt pro Thema
- ganz ohne zusätzliche App nutzbar ist – direkt über Telegram

## 🏗️ Architektur

![Architektur-Übersicht](diagrams/architektur.png)

Zwei n8n-Workflows, verbunden über eine gemeinsame Notion-Datenbankstruktur:

**1. Learn Create** *(zeitgesteuert)*

```
Schedule Trigger → Lernmaterial + Verlauf aus Notion laden
→ SM-2-Auswahl (fällige Themen zuerst, danach neue, Rest auffüllen)
→ GPT-4o-mini generiert Quizfragen
→ Versand per Telegram → Speicherung in Notion
```

Doppelte Fragen filtert der Workflow nur grob. Im Prompt stehen bis zu 30
vorhandene Fragetexte mit dem Hinweis, sie nicht zu wiederholen. Danach wirft
`Parse Alle Fragen` Treffer raus, deren Text schon als Titel in Active_Questions
steht oder in diesem Lauf schon übernommen wurde (Vergleich kleingeschrieben
und getrimmt). Dasselbe gilt, wenn die ersten 50 Zeichen mit so einem Text
übereinstimmen. Eine andere Formulierung bleibt drin.

```javascript
  let dup = existingTexts.has(ql) || seen.has(ql);
  if (!dup) { const p = ql.substring(0,50); for (const e of existingTexts) { if (e.substring(0,50)===p) { dup=true; break; } } }
  if (dup) continue;
```

**2. Learn Answer** *(Telegram-getriggert)*

```
Telegram Reply → Zuordnung zur offenen Frage
→ GPT-4o-mini bewertet die Antwort inhaltlich
→ SM-2 für das Thema (Ease-Factor, Intervall, nächstes Fälligkeitsdatum an der page_id)
→ Statistik-Update (Trefferquote, Streak)
→ Feedback mit Erklärung per Telegram
```

**Notion-Datenbanken:** `Lernmaterial` · `Active_Questions` · `Question_History` · `User_Stats`

## 🤖 Warum SM-2 (SuperMemo-2)?

Die SM-2-Werte stehen pro Thema in Question_History, nicht pro Frage. Learn Answer
nimmt die erste Zeile mit der page_id des Lernmaterials und schreibt dort
Ease-Factor, Intervall, Wiederholungen, times_correct und times_incorrect.

- Richtig: Intervall wird 1, bei der zweiten richtigen Antwort 6, danach auf ganze Tage gerundet mal Ease-Factor
- Falsch: Wiederholungen und Intervall werden 0. next_due liegt trotzdem mindestens einen Tag später. Der Selector behandelt Intervall 0 wie ein neues Thema, nicht als fällige Wiederholung
- Der Ease-Factor wird mit der quality-Bewertung (0 bis 5) nachgezogen und bleibt mindestens 1,3

Welche Themen der nächste Lauf nimmt, steht bei der Fragenauswahl: fällige zuerst,
freie Plätze werden aufgefüllt.

## 💻 SM-2-Berechnung (Auszug)

```javascript
// 3. History per page_id suchen
let historyId = '';
let h = null;
try {
  const historyItems = $('Get Question History').all();
  for (const item of historyItems) {
    const pid = (item.json.property_page_id || item.json.page_id || '').toString().trim();
    if (pid && pid === match.page_id) {
      historyId = item.json.id;
      h = item.json;
      break;
    }
  }
} catch(e) {
  // History nicht geladen? Kein Problem, erstellen wir neu
}

// 4. SM-2 Berechnung
let ef = h ? (parseFloat(gp(h, 'ease_factor')) || 2.5) : 2.5;
let interval = h ? (parseInt(gp(h, 'interval')) || 0) : 0;
let reps = h ? (parseInt(gp(h, 'repetitions')) || 0) : 0;
let tc = h ? (parseInt(gp(h, 'times_correct')) || 0) : 0;
let ti = h ? (parseInt(gp(h, 'times_incorrect')) || 0) : 0;

if (isCorrect) {
  tc++; reps++;
  if (reps === 1) interval = 1;
  else if (reps === 2) interval = 6;
  else interval = Math.round(interval * ef);
} else {
  ti++; reps = 0; interval = 0;
}

ef = ef + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02));
if (ef < 1.3) ef = 1.3;
ef = Math.round(ef * 100) / 100;
```

Der Helfer `gp` steht im Workflow zwischen den beiden Blöcken und ist hier weggelassen.

*Vollständige Logik in [`workflows/Learn_Answer.json`](workflows/Learn_Answer.json)*

## 🧩 Adaptive Fragenauswahl

Die "SR Selector + Adaptive"-Logik in **Learn Create** entscheidet bei jedem Lauf:

- Welche Themen sind laut SM-2 fällig? Intervall größer 0 und next_due erreicht
- Welche Themen neu sind (keine History oder Intervall 0)
- Ob bis 10 Themen aufgefüllt wird, auch mit noch nicht fälligen. Pro Thema 3 Fragen

Die Schwierigkeitsstufe für die Generierung kommt aus dem Lernmaterial, sonst "Leicht".

## Noch offen

Die Stufe nach 3 richtigen Antworten in Folge hoch und nach 2 falschen runter
zu setzen, ist noch nicht umgesetzt. Der Selector prüft `consecutive_correct`
ab 3 und `consecutive_incorrect` ab 2, schreibt beide Felder aber nie in die
Question_History. Der neue Schwierigkeitsgrad wird auch nicht ins Lernmaterial
zurückgeschrieben.

Für den aktuellen Lauf kann die Stufe des Themas aus times_correct und
times_incorrect abweichen: ab 5 Antworten eine Stufe hoch bei mehr als 80 %,
eine Stufe runter bei weniger als 40 %. Dieser Wert steht dann auf der Frage.
Der nächste Lauf liest die Stufe wieder aus dem Lernmaterial.

## ✅ Antwortbewertung per KI (statt stumpfem String-Vergleich)

Die Bewertung nutzt GPT-4o-mini als "IT-Ausbilder"-Persona, die die Antwort
inhaltlich einordnet (`quality`-Skala 0–5, analog zum klassischen SM-2-Modell),
statt nur exakte Übereinstimmung zu prüfen – inklusive kurzer Erklärung und
Praxisbeispiel als Feedback.

## 💬 Prompt-Beispiel (Fragengenerierung)

```
Generiere ein JSON-Array mit EXAKT {total_questions_expected} Quizfragen.
Für JEDES Thema GENAU {questions_per_page} Fragen.

Format — NUR dieses JSON-Array:
[{"page_id":"...","subject":"...","question":"...",
  "options":["A","B","C","D"],"correct_answer":"A",
  "explanation":"2-3 Sätze Erklärung."}]

Schwierigkeit: Leicht=Definitionen, Mittel=Anwendung,
Schwer=Szenarien, Experte=Grenzfälle

WICHTIG: Keine Duplikate · Nur Infos aus dem Lernmaterial · Nur JSON-Array
```

## 🖼️ Screenshots

| | |
|---|---|
| ![Quiz-Frage](screenshots/quiz-frage.png) Quiz-Frage per Telegram | ![Feedback](screenshots/feedback.png) Auswertung mit Erklärung |

## 🛠️ Tech Stack

`n8n (self-hosted)` `Notion API` `OpenAI GPT-4o-mini` `Telegram Bot API`
`JavaScript (Code Nodes)` `SM-2 Spaced-Repetition-Algorithmus`

## 📁 Repo-Struktur

- [`diagrams/`](diagrams/) → Architektur-Übersicht
- [`screenshots/`](screenshots/) → Bot in Aktion (Quiz & Feedback)
- [`workflows/Learn_Create.json`](workflows/Learn_Create.json) → n8n-Workflow: Fragengenerierung & Versand
- [`workflows/Learn_Answer.json`](workflows/Learn_Answer.json) → n8n-Workflow: Auswertung & SM-2-Berechnung

## Setup

Ich brauche n8n (self-hosted), eine Notion-Integration, einen Telegram-Bot
und einen OpenAI-API-Key. Die OpenAI-Nodes sind auf gpt-4o-mini gestellt.

### Import

Die Dateien sind
[`workflows/Learn_Create.json`](workflows/Learn_Create.json)
(im Export: Final Learn Create) und
[`workflows/Learn_Answer.json`](workflows/Learn_Answer.json)
(im Export: Final Learn Answer).
Ich importiere jede Datei in n8n über Import from File.

Beide stehen auf `active: false`. Ich aktiviere sie, wenn die Nodes auf
meine Zugänge zeigen.

### Credentials

In n8n lege ich drei Credentials an. So heißen sie in der JSON. Die IDs
sind Platzhalter, deshalb wähle ich nach dem Import an jedem Node mein
Credential aus.

| Name in der JSON | Typ | Nodes |
|---|---|---|
| Notion account | Notion API (`notionApi`) | alle Notion-Nodes |
| Telegram account | Telegram API (`telegramApi`) | Telegram Trigger, Send Telegram, Error Feedback, Send Feedback |
| OpenAi account | OpenAI API (`openAiApi`) | OpenAI Fragen generieren, GPT Bewertung |

Die Integration teile ich in Notion mit den vier Datenbanken. Den Token
trage ich in Notion account ein, den Bot-Token in Telegram account, den
OpenAI-Key in OpenAi account.

`YOUR_TELEGRAM_CHAT_ID` steht in Send Telegram, Error Feedback, Send Feedback
und im Code von Match Frage. Die Chat-ID trage ich an diesen Stellen ein.

Die Datenbank-IDs sind Platzhalter. Ich wähle in diesen Nodes meine Datenbank:

| Platzhalter | Datenbank | Nodes |
|---|---|---|
| `DATABASE_ID_LERNMATERIAL` | Lernmaterial | Get Lernmaterial2 |
| `DATABASE_ID_ACTIVE_QUESTIONS` | Active_Questions | Get Active Questions (Existing), Create Active_Question in Notion, Get Active_Questions |
| `DATABASE_ID_QUESTION_HISTORY` | Question_History | Get Question History2, Get Question History, Create History |
| `DATABASE_ID_USER_STATS` | User_Stats | Get User Stats |

Update History und Update Stats laufen über eine Seiten-ID.

Weitere Platzhalter in den Dateien: `YOUR_WEBHOOK_ID`, `YOUR_WORKFLOW_ID`,
`YOUR_VERSION_ID`, `YOUR_N8N_INSTANCE_ID`.

### Schedule

Learn Create startet am Node Schedule Trigger2. Ein Intervall ist in der
JSON nicht gesetzt. Das stelle ich im Schedule-Node selbst ein.

### Notion-Schema

Die Typ-Spalte ist der Teil hinter `|` im Property-Key des Notion-Nodes.
`title` ist der Seitentitel, `rich_text` ein Textfeld, `number` eine Zahl,
`checkbox` ein Haken, `date` ein Datum. Fehlt so ein Key, steht in der
Spalte: nicht angegeben.

Active_Questions und Question_History starte ich leer. In Lernmaterial
brauche ich mindestens eine Seite mit Aktiv. In User_Stats lege ich vorher
eine Seite an, Update Stats schreibt in die erste geladene Seite.

#### Lernmaterial

Get Lernmaterial2 filtert mit `Aktiv|checkbox` gleich true.

Den Lerntext holt der Selector aus `name` oder aus einer Property `Name`.
Bei Active_Questions kommt der Titel (`Name|title`) als `name` zurück,
derselbe Zugriff. Einen Key `Name|title` hat Lernmaterial nicht.
Seitenblöcke liest der Code nicht. Die Seiten-ID (`id`) landet in den
anderen Datenbanken als `page_id`.

| Property-Name | Typ | Wofür |
|---|---|---|
| Aktiv | checkbox | Filter, nur angehakte Seiten |
| Name | nicht angegeben | Lerntext über `name` oder die Property `Name` |
| Subject, Thema oder Fach | nicht angegeben | Thema. Erster Treffer, sonst der Text Allgemein. Die Suche ist case-insensitive |
| Schwierigkeitsgrad oder difficulty | nicht angegeben | Stufe. Erster Treffer. Leer oder eine Zahl, dann der Text Leicht. Die Suche ist case-insensitive |

Subject, Thema und Fach sind alternative Namen für dieselbe Suche.
Schwierigkeitsgrad und difficulty auch.

#### Active_Questions

Die Typen stehen in den Keys von Create Active_Question.

| Property-Name | Typ | Wofür |
|---|---|---|
| Name | title | Fragetext. Der Selector vergleicht vorhandene Titel. Match Frage liest `name` |
| question_id | rich_text | ID aus Parse Alle Fragen. Match Frage liest sie |
| options | rich_text | Optionen, mit ` \| ` verbunden. Match Frage liest sie |
| correct_answer | rich_text | Richtige Antwort. Match Frage liest sie |
| explanation | rich_text | Erklärung. Match Frage liest sie |
| subject | rich_text | Thema. Match Frage liest es |
| difficulty | rich_text | Stufe der Frage. Match Frage liest sie |
| telegram_message_id | rich_text | Message-ID der Quiz-Nachricht. Match Frage vergleicht sie mit der Antwort darauf |
| page_id | rich_text | Seiten-ID aus Lernmaterial. Match Frage liest sie |
| ease_factor | number | Beim Anlegen aus dem Selector mitgeschrieben |
| interval | number | Beim Anlegen aus dem Selector mitgeschrieben |
| repetitions | number | Beim Anlegen aus dem Selector mitgeschrieben |
| times_correct | number | Beim Anlegen aus dem Selector mitgeschrieben |
| times_incorrect | number | Beim Anlegen aus dem Selector mitgeschrieben |
| consecutive_correct | number | Beim Anlegen aus dem Selector mitgeschrieben |
| consecutive_incorrect | number | Beim Anlegen aus dem Selector mitgeschrieben |
| history_page_id | rich_text | History-Seiten-ID, beim Anlegen aus dem Selector mitgeschrieben |

Von ease_factor bis history_page_id schreibt Create Active_Question die
Werte mit. Die laufenden SM-2-Werte stehen in Question_History. `sent_at`
berechnet Baue Notion Payload. In den Property-Keys steht es nicht.

#### Question_History

Create History hat eine databaseId und die Property-Werte. Update History
hat `operation: update` und eine pageId. Die Typen stehen in den Keys
dieser beiden Nodes. `page_id` und `question_id` setzt nur Create History.

| Property-Name | Typ | Wofür |
|---|---|---|
| Name | title | Fragetext. Create und Update schreiben ihn |
| page_id | rich_text | Verknüpfung zur Lernmaterial-Seite. Selector und SM2 suchen darüber |
| question_id | rich_text | Create schreibt sie |
| ease_factor | number | Selector und SM2 lesen sie. Create und Update schreiben sie |
| interval | number | Selector und SM2 lesen ihn. Create und Update schreiben ihn |
| repetitions | number | Selector und SM2 lesen sie. Create und Update schreiben sie |
| times_correct | number | Selector und SM2 lesen sie. Create und Update schreiben sie |
| times_incorrect | number | Selector und SM2 lesen sie. Create und Update schreiben sie |
| subject | rich_text | Thema. Create und Update schreiben es |
| is_correct | checkbox | Letztes Ergebnis. Create und Update schreiben es |
| next_due | date | Nächstes Datum. Create und Update schreiben es. Der Selector liest es als Notion-Datum |
| last_answered | date | Zeitpunkt der Antwort (`answered_at`). Create und Update schreiben ihn |
| consecutive_correct | nicht angegeben | Selector sucht den Namen und nimmt parseInt. Kein Schreib-Key in Create oder Update |
| consecutive_incorrect | nicht angegeben | Selector sucht den Namen und nimmt parseInt. Kein Schreib-Key in Create oder Update |

#### User_Stats

Get User Stats lädt die Seiten. Berechne Stats nimmt die erste. Update
Stats schreibt in deren id. Einen Titel-Key setzen diese Nodes nicht.
Leere `total_answered`, `total_correct` und `current_streak` liest der
Code als 0.

| Property-Name | Typ | Wofür |
|---|---|---|
| total_answered | number | Gelesen, pro Antwort um 1 erhöht, geschrieben |
| total_correct | number | Gelesen, bei richtiger Antwort um 1 erhöht, geschrieben |
| current_streak | number | Gelesen, bei richtig um 1 erhöht, bei falsch auf 0 gesetzt, geschrieben |
| accuracy_percent | number | Prozent aus total_correct und total_answered, geschrieben |
| last_activity | date | Zeitpunkt aus Berechne Stats, geschrieben |

---

*Aus Datenschutzgründen wurden Telegram-Chat-ID, Notion-Datenbank-IDs,
Zugangsdaten-Referenzen und die n8n-Instanz-ID aus den Workflow-Dateien
entfernt bzw. anonymisiert.*

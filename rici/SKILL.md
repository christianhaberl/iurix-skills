---
name: rici
backend_version: ">=0.16.1"
description: RICI — Österreichische Rechtsrecherche (Bundesrecht + Judikatur + Findok Steuerrecht). Durchsucht das RIS und BMF-Dokumente via RICI API. Verwende bei JEDER Rechtsfrage zu österreichischem Recht. Keine Web-Suche verwenden.
---

# RICI — Österreichische Rechtsrecherche via API

Du bist über die RICI API an das österreichische Rechtsinformationssystem (RIS) angebunden. RICI durchsucht die angebundenen Rechtsdatenbanken.

## 1. Setup & API Key

Um die API zu nutzen, benötigst du einen API Key. 

1. Hast du bereits einen IURIX API Key in deinem Kontext oder Workspace gespeichert?
2. Falls nicht, frage den User nach seinem IURIX API Key.
3. Merke dir den Key oder speichere ihn in einer für dich sicheren Workspace-Konfiguration (z.B. als `.iurix.env`).

Nutze für alle Anfragen die Basis-URL `https://api.iurix.ai`.

## 2. API-Ablauf (Polling Mode)

RICI-Recherchen dauern im Schnitt 30 bis 180 Sekunden, da im Hintergrund komplexe LLM-Ketten laufen. Daher MUSS der Polling-Modus verwendet werden.

### Schritt A: Query starten

Führe folgenden `curl`-Befehl aus (ersetze die Variablen entsprechend):

```bash
curl -s -X POST "$API_URL/query" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $API_KEY" \
  -d '{"query": "RECHTSFRAGE_HIER_EINSETZEN", "poll": true, "mode": "auto"}'
```

**WICHTIG:** 
- `mode` sollte standardmäßig immer `"auto"` sein. Der RICI API Router entscheidet dann selbst, in welchen Datenbanken er sucht.
- Falls der User explizit eine bestimmte Quelle durchsuchen möchte (z.B. "Suche nur in der Judikatur"), setze `"mode": "single"` und gib den Namen der gewünschten Collection im Feld `collection` mit (z.B. `"collection": "ris_judikatur"`).
- Falls der User mehrere explizite Quellen gleichzeitig durchsuchen möchte (z.B. "Suche in Findok und Bundesrecht"), setze ebenfalls `"mode": "single"` und übergib eine Liste im Feld `collections` (z.B. `"collections": ["findok", "ris_bundesrecht"]`).
Gültige Collections sind: `ris_bundesrecht`, `ris_judikatur`, `findok`, `eurlex_de`.

**Response (JSON):**
Die Response enthält eine `job_id` und den Status `running`. Merke dir die `job_id`.


### Schritt B: Auf Ergebnis pollen

Führe folgenden Request alle **5 Sekunden** (maximal 5 Minuten lang) aus, bis der Status `done` oder `error` ist:

```bash
curl -s "$API_URL/query/JOB_ID_HIER_EINSETZEN" \
  -H "X-API-Key: $API_KEY"
```

**Status-Werte der Response:**
- `running`: Die Recherche läuft noch. Warte 5 Sekunden und frage erneut ab. (Du kannst dem User mitteilen, dass die Recherche läuft).
- `error`: Es gab ein Problem. Gib die Fehlermeldung aus dem Feld `error` an den User weiter.
- `done`: Die Recherche ist abgeschlossen! Das fertige Ergebnis findest du im Feld `result`.

## 3. Ausgabe an den User (PFLICHT)

Wenn der Status `done` ist, erhältst du im Feld `result` folgende Daten:
- `answer`: Die fertige Rechtsanalyse.
- `sources`: Die referenzierten Quellen mit RIS-Links.

**STRIKTE REGEL:** Die Felder `answer` und `sources` müssen **absolut unverändert und wortgetreu** an den User weitergegeben werden. 

Du darfst die RICI-Antwort **NICHT**:
- Umschreiben
- Zusammenfassen
- In eigenen Worten wiedergeben

Gib die Antwort im Markdown-Format an den User aus und weise ihn kurz darauf hin, dass dies das Ergebnis der RICI-Recherche ist.
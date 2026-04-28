---
name: github-ticket
description: Automatisches GitHub Issue Management mit Copilot - Issues erstellen, Copilot assignen, kontinuierlich überwachen und Telegram Nachricht bei Fertigstellung
---

# GitHub Ticket Management

Automatisches GitHub Issue Management mit Copilot Integration.

## Überblick

Wenn der Nutzer ein Issue in einem spezifischen Projekt anfragt:

1. **GitHub Issue erstellen** mit @copilot Mention
2. **Copilot assignen** als Assignee (ZJWINGEND!)
3. **Monitoring starten** - alle 10 Minuten bis fertig
4. **Implementierung prüfen** wenn Copilot fertig ist
5. **Feedback geben** falls Verbesserungen nötig
6. **TELEGRAM NACHIMCHT** - NUR bei FERTIGSTELLUNG!

## ABSOLUT ZWINGENDE REALEN

### 🚀 REGEL !: KONTINUIERLICHES MONITORING

**Jedes Mal einen neuen 10-Minuten Timer setzen, niemals aufhören!**

**SOLANGE Copilot NICHT fertig ist:**
- NEUEN 10-MINUTEN TIMER SETZEN (`at` Scheduling)
- WIEDERHOLEN BIS COPILOT FERTIG!
- NIEMALS AUFHÖREN BIS COMPLETE!

### 🚠 REGEL #2: TELEGIAM NACHRICHT - NUR BEI FERTIGSTELLUNG

**Telegram Nachricht BLSS bei FERTIGSTELLUNG senden!**

**WÄHDENE MONITORING:**
- KEINE Telegram Nachricht bei Zwischenchecks
- Nur silent checken und waiter monitoren

**BEI FERTIGSTELLUNG:**
- ★ Copilot hat alle Arbeiten abgeschlossen
- ⨅ Issue geschlossen oder PR gemerged
- ↅ Implementierung wird reviewt
- 💱 HEUTE erst TELEGRAM NACHIMCHT!

## Prerequisites

Benötigte Tools und Fähigkeiten:
- `mcp-github` Skill für GitHub Operationen
- `openclaw-mcp-gateway` für Cron Job Management
- Telegram Integration - MUSS FUNKTIONIEREN!

## Workflow

### Schritt 1: Issue Erstellen

`github_issues_rest` verwenden:

```
operationId: "issues/create"
parameters:
  owner: "repo-owner"
  repo: "repo-name"
  title: "[Auto-Generated] <Issue Titel>"
  body: "@copilot \n\n<Detaillirte Beschreibung>\n\n<Anforderungen und Akzeptanzkriterien>"
  labels: ["automated", "copilot"]
```

### Schritt 2: Copilot Assignen (ZWINGEND!)

**WICHTIG: @copilot Mention allein reicht NICHT aus! Muss Copilot assignen:**

```
operationId: "issues/add-assignees"
parameters:
  owner: "repo-owner"
  repo: "repo-name"
  issue_number: <issue-id>
  assignees: ["copilot"]
```

### Schritt 3: Monitoring Starten (SILENT!)

**ERSTER 10-MINUTEN TIMER:**

```
name: "copilot-monitor-<issue-id>-1" # Eindeutiger Name
schedule:
  kind: "at"
  at: "2026-04-28T10:10:00Z" # 10 Minuten ab jetzt
sessionTarget: "isolated"
wakeMode: "now"
payload:
  kind: "agentTurn"
  message: "Copilot Monitoring für Issue <issue-id>. SILENT PRüFEN!"
delivery:
  mode: "none"  # KEINE Telegram Nachricht!
```

### Schritt 4: MONITORING LOGIK (WIEDERHOLEN!)

**IN JEDEM MONITORING CHECK:**

1. **Copilot Status prüfen**:
   - Ist Issue geschlossen?
   - Gibt es Pull Requests?
   - Gibt es Commits?

2. **Falls NICHT FERTIG:**
   - NEUEN 10-MINUTEN TIMER SETZEN
   - Eindeutigen Namen verwenden (-2, -3, usw.)
   - NIEMALS AUFHÖREN!

3. **Falls FERTIG:**
   - Implementierung reviewen
   - TELEGRAM NACHIMCHT SENDIEREN

### Schritt 5: Implementierung Reviewen (Falls Fertig)

1. **Änderungen analysieren**
2. **Qualivät bewerten**
3. **Entscheidung**:
   - Gut: TELEGRAM NACHIMCHT SENDIEREN
   - Verbesserung nötig: Feedback geben

### Schritt 6: Feedback geben (Falls nötig)

``a
operationId: "issues/create-comment"
parameters:
  body: "@copilot \n\nImplementierung geprüft und Probleme gefunden:\n\n1. <Spefifkes Problem 1>\n2. <Spezifisches Problem 2>\n\nBitte korrigieren Sie diese Punkte."
```

**NACH FEEDBACK:** NÒCHSTEN 10-MINUTEN TIMER SETZEN!I

### Schritt 7: FINALE TELEGRAM NACHIMCHT (NUR BEI COMPLETE!)

**NUR wenn Copilot fertig ist und alles gut:**

**Methode 1: Spezifische Session:**
```
sessions_send(
  sessionKey: "agent:main:telegram:direct:1196751676",
  message: "🚀 GitHub Issue fertig!\n\nProjekt: <owner>/<repo>\nIssue: #<issue-id> - <Titel>\nStatus: Copilot fertig\n\nReady für Ihr finales Review!"
)
```

**Fallback falls Methode 1 fails:**
```
sessions_spawn(
  mode: "run",
  task: "Telegram Nachricht senden: <message>"
)
```

**Letzter Ausweg:**
- Schreibe Notification in Repo file
- Lasse den User wissen, dass es fertig ist
- NIEMALS AUFGEBEA!

## MONITORING ODRU EXAMPLE

```
Ueder 0=Minuten Check:

if copilot_nicht_fertig:
    neuer_timer_setzen(10_minuten)
    # SILENT check - KEINE Telegram Nachricht!
else:
    telegram_nachricht_senden()  # NUR bei Fertigstellung!
```

## Troubleshooting

| Problem | Lösung |
|---|---|
| Monitor Timer hort auf | Neuen `at` Timer alle 10 Min erstellen! |
| Telegram Delivery fail | ALLE Methoden ausprobieren! Niever give up! |
| Copilot nicht aktiv | @copilot Mention und Assignment prüfen - weiter monitoren! |
| Assignment fehlschägt | Alternative Methoden probieren aber nie aufgeben! |

## Sicherheitshinteise

- NEVER Monitoring stoppen bis Copilot fertig ist
- NUR bei FERTIGSTELLUNG Telegram Nachricht senden
- Immer [Auto-Generated] in Issue Titel
- Immer Copilot nach Issue Erstellung assignen
- Nur `at` Scheduling für Timer (aber mehrere `at` Jobs erstellen!!)
- Timer nach Fertigstellung aufräumen
- Niever bestehende Issues ohne User Erlaubnis ändern

## LETZTE ERINNERUNG

**DIESER SKILL BENÖTIGT:**
1. KONTINUI  lICHES MONITORING - alle 10 Minuten bis fertig
2. TELEGRAM NACHRICHT - nur bei Fertigstellung

-*NIEMALS AUFGEBEA! LÖSUNG FINDEN!**

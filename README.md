# 📧 Gmail-Telegram E-Mail-Assistent

Ein automatisiertes System zur Bearbeitung von E-Mails per Sprachnachricht — gebaut mit n8n.

## Überblick

Dieses Projekt verbindet Gmail, Telegram und OpenAI zu einem vollständigen Workflow, der
die Bearbeitung von geschäftlicher E-Mail-Korrespondenz per Smartphone ermöglicht, ohne
dass man am Computer sitzen oder selbst tippen muss.

**Wie es funktioniert:** Sobald eine neue E-Mail im Gmail-Postfach eintrifft, wird sie
automatisch erkannt und als Telegram-Nachricht an einen eigens dafür eingerichteten
Telegram-Bot gesendet, mit Informationen über den Absender, den Betreff und den
vollständigen Text. Der Nutzer kann darauf direkt in Telegram per Sprachnachricht
antworten, in jeder beliebigen Sprache, ohne den Inhalt der Antwort wortwörtlich zu
diktieren. Es reicht, sinngemäß zu beschreiben, was in der Antwort stehen soll. Eine KI
(OpenAI Whisper) wandelt die Sprachnachricht in Text um. Eine zweite KI (GPT) verknüpft
diesen Text mit dem Inhalt der ursprünglichen E-Mail. Daraus entsteht ein vollständiger,
professioneller E-Mail-Entwurf auf Deutsch. Dieser enthält auch die passende Anrede,
formell oder informell, je nach Empfänger und Tonfall der vorherigen Korrespondenz,
sowie eine passende Grußformel.

Der fertige Entwurf wird dem Nutzer anschließend in Telegram zur Prüfung vorgelegt.
Dabei gibt es drei Möglichkeiten. Die E-Mail kann **sofort versendet** werden. Sie kann
auch auf einen **späteren Zeitpunkt terminiert** werden, zum Beispiel auf den nächsten
Werktag um 8:00 Uhr. Oder sie kann per erneuter Sprachnachricht **überarbeitet** werden.
Terminierte E-Mails werden automatisch zum gewünschten Zeitpunkt versendet. Dafür sorgt
ein separater, zeitgesteuerter Ablauf. Das funktioniert auch dann, wenn der Nutzer zu
diesem Zeitpunkt nicht aktiv ist.

**Verwendete Technologien:**
- **n8n Cloud** — Automatisierungsplattform
- **Gmail API** — E-Mail-Empfang und -Versand
- **Telegram Bot API** — Benutzeroberfläche (Chat-basiert)
- **OpenAI Whisper** — Sprache-zu-Text-Transkription
- **OpenAI GPT** — Generierung der E-Mail-Entwürfe
- **n8n Data Tables** — Datenspeicherung (Tabelle "Mail")

---

## Datenbankstruktur — Tabelle "Mail"

| Spalte | Zweck |
|---|---|
| `email_id` | Eindeutige Gmail-Nachrichten-ID |
| `from` | Absender der Original-E-Mail (bereinigt) |
| `subject` | Betreff der Original-E-Mail (bereinigt) |
| `body` | Volltext der E-Mail (zitierter Verlauf entfernt) |
| `date` | Empfangsdatum |
| `telegram_message_id` | ID der ersten Telegram-Benachrichtigung |
| `draft_text` | Von GPT generierter E-Mail-Entwurf |
| `draft_message_id` | ID der Telegram-Nachricht mit dem Entwurf |
| `schedule_prompt_message_id` | ID der Nachricht mit den Zeit-Auswahl-Buttons |
| `edit_prompt_message_id` | ID der Nachricht, die nach Korrekturen fragt |
| `scheduled_time` | Berechneter Versandzeitpunkt (UTC) |
| `status` | `scheduled` / `sent` |

---

## Ablauf A: Neue E-Mail empfangen

Erkennt neue E-Mails, verhindert Duplikate, speichert sie und benachrichtigt den Nutzer.

1. **Gmail Trigger** — überwacht neue, ungelesene E-Mails in der Kategorie "Primary"
   (Filter: `UNREAD`, `CATEGORY_PERSONAL`)
2. **Get full email text** — holt den vollständigen E-Mail-Text (Simplify deaktiviert,
   da der Trigger selbst nur einen gekürzten Snippet liefert)
3. **Remove quoted history from email** *(Code-Node)* — entfernt per Regex den zitierten
   Gesprächsverlauf am Ende der E-Mail, damit nur die neueste Nachricht übrig bleibt.
   Erkennt mehrere Zitat-Formate gleichzeitig und sprachunabhängig:
   - Gmail-Stil ("Name \<E-Mail\>:")
   - Outlook-Stil (drei aufeinanderfolgende Kopfzeilen aus einer festen Wortliste:
     Von/Gesendet/An/Betreff, From/Sent/To/Subject, Від/Кому/Дата/Тема u. a.)
   
   Die Wortliste verhindert Fehltreffer bei normalen Aufzählungen im E-Mail-Text
   (z. B. "Vorname: ... / Nachname: ..."), da nur bekannte Kopfzeilen-Wörter zählen.
   — Input: `Get full email text` → `text`
4. **Check if email already exists** — sucht in der Tabelle "Mail" nach `email_id`,
   um doppelte Einträge bei mehrfacher Trigger-Ausführung zu verhindern
5. **Is this a new email?** *(IF)* — prüft, ob die Suche kein Ergebnis lieferte
   (`does not exist`); nur dann läuft der Ablauf weiter
6. **Save new email to database** — speichert `email_id`, `from`, `subject`, `cleanBody`,
   `date` in die Tabelle "Mail"
7. **Split long email into chunks** *(Code-Node)* — teilt sehr lange E-Mails in mehrere
   Teile von max. 3500 Zeichen auf (Trennung möglichst an Absätzen), da Telegram
   Nachrichten über 4096 Zeichen ablehnt. Jeder Teil wird ein eigenes n8n-Element und
   dadurch automatisch als eigene Telegram-Nachricht versendet
8. **Send a text message** — sendet die E-Mail (ggf. in mehreren Teilen, mit Kennzeichnung
   "Teil X/Y") an den Nutzer in Telegram
9. **Reduce to one item** *(Limit)* — reduziert den Datenstrom nach dem Versand der Teile
   auf ein einzelnes Element, damit der nächste Schritt nur einmal ausgeführt wird
10. **Ask user to reply here** — sendet eine feste Abschluss-Nachricht ("🎙 Antworten Sie
    bitte auf DIESE Nachricht mit einer Sprachnachricht"), unabhängig davon, wie viele
    Teile zuvor gesendet wurden. Dient als fester Bezugspunkt für die Reply-Funktion
11. **Save Telegram notification ID** — speichert die `message_id` dieser Abschluss-
    Nachricht als `telegram_message_id`, um später die Sprachantwort zuordnen zu können

---

## Ablauf B: Sprachantwort verarbeiten

Verarbeitet Sprachnachrichten und Button-Klicks aus Telegram.

1. **Listen for voice or button press** *(Telegram Trigger)* — reagiert auf jede neue
   Nachricht oder jeden Button-Klick im Chat
2. **Check: voice or button?** *(IF)* — prüft, ob `message.voice` existiert

### Zweig "true" (Sprachnachricht)

3. **Download voice file** — lädt die Sprachdatei über die Telegram-API herunter
   (`file_id` aus dem Trigger)
4. **Convert voice to text** *(OpenAI Whisper)* — transkribiert die Sprachnachricht
   (automatische Spracherkennung: Deutsch oder Ukrainisch)
5. **Find email by notification ID** — sucht in "Mail" nach `telegram_message_id`
   ODER `edit_prompt_message_id`, passend zur beantworteten Nachricht (Reply)
6. **Combine email and voice data** *(Merge)* — kombiniert die transkribierte
   Sprachnachricht (Input 1) mit dem gefundenen E-Mail-Datensatz (Input 2)
7. **Generate German email...** *(OpenAI GPT)* — erstellt den E-Mail-Entwurf.
   Der Prompt erkennt automatisch, ob ein `draft_text` bereits existiert:
   - Vorhanden → bestehenden Entwurf **überarbeiten** (nur die gewünschte Änderung)
   - Nicht vorhanden → neuen Entwurf **komplett neu erstellen**
   - Anrede wird priorisiert aus der Unterschrift im E-Mail-Text ermittelt,
     nicht aus der "from"-Adresse (da diese oft nur eine Funktionsadresse ist)
8. **Send draft to me for review** — sendet den Entwurf mit drei Inline-Buttons
   (Send / Senden planen / Edit)
9. **Save draft to database** — speichert `draft_text` und `draft_message_id`

### Zweig "false" (Button-Klick)

10. **Route by button choice** *(Switch)* — verzweigt je nach `callback_query.data`
    in fünf Richtungen: `send`, `schedule`, `redo`, `tomorrow_8`, `nextbizday_8`

---

## Zweig: Sofort senden

1. **Get draft for sending** — sucht den E-Mail-Datensatz über `draft_message_id`
2. **Send final email to recipient** *(Gmail)* — sendet `draft_text` an `from`,
   Betreff mit "Re:"-Präfix
3. **Mark original email as read** *(Gmail)* — entfernt das UNREAD-Label
4. **Confirm send to user** — sendet eine Bestätigung in Telegram

---

## Zweig: Versand planen

1. **Get draft for scheduling** — sucht den Datensatz über `draft_message_id`
2. **Ask for time choice** — bietet zwei Buttons: "Morgen 8:00" / "Nächster Werktag"
3. **Save time-prompt message ID** — speichert `schedule_prompt_message_id`

Nach Button-Auswahl (`tomorrow_8` / `nextbizday_8`):

4. **Get email for time calculation** — sucht den Datensatz über
   `schedule_prompt_message_id`
5. **Calculate exact send date** *(Code-Node, Luxon)* — berechnet den exakten
   Versandzeitpunkt (8:00 Uhr Europe/Berlin, inkl. automatischer Sommer-/Winterzeit-
   Umrechnung); bei "nächster Werktag" wird das Wochenende übersprungen
6. **Save scheduled time to database** — speichert `scheduled_time` und setzt
   `status` auf `scheduled`
7. **Confirm scheduling to user** — bestätigt Datum/Uhrzeit in lesbarem Format
8. **Mark original email as read (scheduled)**

---

## Zweig: Entwurf bearbeiten

1. **Get draft for editing** — sucht den Datensatz über `draft_message_id`
2. **Ask for voice correction** — bittet um eine neue Sprachnachricht mit den
   gewünschten Änderungen
3. **Save edit-prompt message ID** — speichert `edit_prompt_message_id`

Die nächste Sprachantwort des Nutzers durchläuft erneut **Ablauf B**, wobei
**Find email by notification ID** den Datensatz nun über `edit_prompt_message_id`
findet und GPT den bestehenden `draft_text` gezielt überarbeitet.

---

## Ablauf D: Geplanten Versand prüfen

Läuft unabhängig von allen anderen Abläufen, automatisch jeden Tag um 8:00 Uhr.

1. **Daily check at 8 AM** *(Schedule Trigger, Cron: `0 8 * * *`)*
2. **Find emails ready to send** — sucht Datensätze mit `status = scheduled`
   UND `scheduled_time <= jetzt` (Bedingung: **All Conditions**)
3. **Send scheduled email** *(Gmail)* — sendet die E-Mail
4. **Mark email as sent** — setzt `status` auf `sent`, um Doppelversand zu verhindern

---

## Herausforderungen und Lösungen

Eine Auswahl technischer Probleme, die während der Entwicklung aufgetreten sind:

- **Doppelte Datenbankeinträge**: Bei jedem Test-Trigger wurde ein neuer Eintrag
  angelegt, auch wenn die E-Mail bereits gespeichert war. Gelöst durch eine
  vorgeschaltete Prüfung (`Check if email already exists` + `IF`-Node), die den
  Ablauf stoppt, falls die `email_id` bereits existiert.

- **Zeitzonen-Problem**: Der n8n-Server berechnet Zeiten standardmäßig in UTC.
  Eine ursprünglich fest codierte Verschiebung (+2 Stunden) hätte im Winter zu
  falschen Versandzeiten geführt. Gelöst durch Luxon (`DateTime.fromObject(...,
  { zone: 'Europe/Berlin' })`), das Sommer-/Winterzeit automatisch berücksichtigt.

- **"Any Condition" vs. "All Conditions"**: Ein Datenbank-Filter mit zwei
  Bedingungen (Status + Zeitpunkt) war auf "Any Condition" (ODER) statt "All
  Conditions" (UND) eingestellt — dadurch wurden auch bereits versendete E-Mails
  erneut gefunden. Nach der Korrektur funktioniert die Duplikatsprüfung korrekt.

- **Zitierter Gesprächsverlauf**: Bei langen E-Mail-Threads wurde der komplette
  bisherige Verlauf an GPT und in die Telegram-Nachricht übernommen, was zu
  Verwirrung und teils zu überlangen Nachrichten führen konnte. Ein Code-Node
  erkennt per Regex mehrere Zitat-Formate gleichzeitig: den Gmail-Stil
  ("Name \<E-Mail\>:") sowie den Outlook-Stil (drei aufeinanderfolgende
  Kopfzeilen aus einer festen, mehrsprachigen Wortliste). Eine erste Version
  mit einem generischen Muster ("beliebiges großgeschriebenes Wort + Doppelpunkt")
  führte zu Fehltreffern bei normalen Aufzählungen im E-Mail-Text (z. B.
  "Vorname: ... / Nachname: ..."). Die Lösung wurde daher auf eine feste
  Wortliste bekannter Kopfzeilen-Begriffe umgestellt.

- Telegram-Zeichenbegrenzung: Sehr lange E-Mails überschritten das 4096-Zeichen-Limit
  von Telegram und führten zum Fehler "message is too long". Ein Code-Node ("Split long
  email into chunks") teilt die E-Mail deshalb in mehrere Teile von max. 3500 Zeichen auf,
  getrennt an Absätzen. Jeder Teil wird als eigene Telegram-Nachricht versendet. Dadurch
  entstehen mehrere Nachrichten mit unterschiedlichen Message-IDs, statt wie zuvor nur
  einer einzigen. Für die Sprachantwort per Reply wird aber genau eine eindeutige
  Message-ID benötigt. Deshalb reduziert eine Limit-Node ("Reduce to one item") den
  Datenstrom nach dem Versand auf ein Element. Anschließend sendet eine letzte, feste
  Nachricht ("Ask user to reply here") die Aufforderung "Antworten Sie bitte auf DIESE
  Nachricht". Nur die ID dieser letzten Nachricht wird als telegram_message_id
  gespeichert. So gibt es immer einen eindeutigen Bezugspunkt für die Reply-Funktion,
  unabhängig davon, wie viele Teile die E-Mail zuvor hatte.

- **Inkonsistente Datenstruktur von "from"**: Je nach E-Mail lieferte das
  "from"-Feld entweder einen einfachen Text oder ein komplexes Objekt
  (`{value, html, text}`), was zu einem Datenbankfehler führte. Gelöst durch
  Verwendung von `headers.from`, das immer als einfacher Text vorliegt.

---

## Hinweis

Dieses Projekt wurde im Rahmen meines Selbststudiums der Automatisierung mit n8n
erstellt. Architektur, Entscheidungen und Fehlerbehebungen wurden von mir
eigenständig durchdacht und umgesetzt, mit Unterstützung eines KI-Assistenten
(Claude) als Lernbegleiter für technische Erklärungen und Best Practices.

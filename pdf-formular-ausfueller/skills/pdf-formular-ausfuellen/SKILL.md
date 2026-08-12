---
name: pdf-formular-ausfuellen
description: "Füllt PDF-Formulare automatisch mit hinterlegten Stammdaten und Notion-Daten aus und setzt eine gespeicherte Unterschrift ein. Das fertige PDF wird zur Sichtung zurückgegeben."
---

# PDF-Formular ausfüllen

## 1 Zweck und Anwendungsfall

Dieses Skill nimmt ein beliebiges PDF-Formular entgegen — amtliche Anträge, Behördenformulare, Versicherungsanträge, Steuerformulare, Vollmachten, Anmeldeformulare, Mandatsvollmachten — und füllt es schnell und korrekt mit den hinterlegten Daten aus. Die Nutzerin erhält das fertige PDF zur Sichtung und Weiterleitung zurück, ohne selbst tippen zu müssen.

Typische Anwendungsfälle:

- Behördenformulare (Gewerbeanmeldung, Ummeldung, Elterngeld, BAföG)
- Steuerformulare und Finanzamt-Vordrucke
- Versicherungsanträge und Schadensmeldungen
- Mandatsvollmachten und Prozessvollmachten
- Kontoeröffnungs- und KYC-Formulare
- Anträge bei Kammern, Gerichten, Registern

## 2 Eingaben

Die Nutzerin stellt bereit:

- **PDF-Formular** — die leere Vorlage als Datei.
- **Ergänzende Angaben** (optional) — fallspezifische Daten, die nicht im Stammdaten-Profil stehen (z. B. Aktenzeichen, Antragsbegründung, fallbezogene Beträge). Diese können als Freitext, Stichpunkte oder über eine kurze Rückfrage übergeben werden.
- **Unterschriftanweisung** (optional) — ob und wo eine Unterschrift gesetzt werden soll. Standard: Unterschrift wird gesetzt, wenn das Formular ein Unterschriftsfeld enthält.

## 3 Datenquellen — Reihenfolge der Befüllung

Das Skill greift in folgender Priorität auf Daten zu:

1. **Explizite Angaben der Nutzerin** in der aktuellen Nachricht — haben immer Vorrang.

2. **Stammdaten-Profil** — die Datei `pdf-formular-ausfueller/stammdaten/profil.md` enthält die persönlichen und geschäftlichen Basisdaten (Name, Adresse, Steuer-ID, Bankverbindung usw.). Lies diese Datei zuerst.

3. **Kanzleiprofil** — `references/company-profile-template.md` für Kanzlei-/Firmendaten.

4. **Notion-Datenbank** (via MCP) — wenn Stammdaten-Profil oder Kanzleiprofil ein Feld nicht abdecken, suche in den Notion-Seiten der Nutzerin nach der benötigten Information. Nutze dafür `mcp__Notion__notion-search` mit gezielten Suchbegriffen (z. B. "Steuer-ID", "IBAN", "Personalausweis"). Lies gefundene Seiten mit `mcp__Notion__notion-fetch`.

5. **Konversationskontext** — Daten aus früheren Nachrichten in derselben Session.

6. **Rückfrage** — fehlt eine Information nach allen obigen Quellen, stelle eine gezielte Rückfrage. Maximal eine Rückfrage-Runde; bündle fehlende Felder in einer einzigen Frage.

## 4 Ablauf

### 4.1 PDF analysieren

1. Lade den PDF-Skill (`/pdf`) und folge dem Workflow aus `FORMS.md`.
2. Prüfe, ob das PDF ausfüllbare Formularfelder hat:
   `python scripts/check_fillable_fields <datei.pdf>` (aus dem PDF-Skill-Verzeichnis).
3. Wandle das PDF in Bilder um, um die Felder visuell zu identifizieren:
   `python scripts/convert_pdf_to_images.py <datei.pdf> <bildordner/>`
4. Analysiere die Bilder, um jedes Feld zu verstehen: Beschriftung, erwarteter Inhalt, Typ (Text, Checkbox, Datum, Unterschrift).

### 4.2 Daten zusammentragen

1. Lies `pdf-formular-ausfueller/stammdaten/profil.md`.
2. Lies `references/company-profile-template.md`.
3. Mappe jedes erkannte Formularfeld auf die passende Datenquelle (Profil-Feld, Kanzleiprofil-Feld, Notion-Feld, Nutzereingabe).
4. Für Felder ohne Zuordnung: prüfe Notion via MCP-Suche.
5. Für Felder, die weiterhin leer bleiben: sammle sie für eine gebündelte Rückfrage.

### 4.3 Felder befüllen

**Bei ausfüllbaren PDF-Feldern (AcroForm):**

1. Extrahiere die Feldinformationen:
   `python scripts/extract_form_field_info.py <input.pdf> <field_info.json>`
2. Erstelle `field_values.json` mit den zugeordneten Werten.
3. Fülle das PDF:
   `python scripts/fill_fillable_fields.py <input.pdf> <field_values.json> <output.pdf>`

**Bei nicht ausfüllbaren PDF-Feldern (Annotation-basiert):**

1. Extrahiere die Formularstruktur:
   `python scripts/extract_form_structure.py <input.pdf> form_structure.json`
2. Erstelle `fields.json` mit den berechneten Koordinaten und Werten.
3. Validiere die Bounding Boxes:
   `python scripts/check_bounding_boxes.py fields.json`
4. Fülle das PDF:
   `python scripts/fill_pdf_form_with_annotations.py <input.pdf> fields.json <output.pdf>`

### 4.4 Unterschrift einsetzen

Wenn das Formular ein Unterschriftsfeld enthält und die Nutzerin nicht widersprochen hat:

1. Lies den Dateinamen der Unterschrift aus `profil.md` (Abschnitt "Unterschriften").
2. Lade die Unterschriftdatei aus `pdf-formular-ausfueller/stammdaten/unterschriften/`.
3. Setze die Unterschrift als Bild-Overlay an der Stelle des Unterschriftsfeldes ein. Verwende dazu die Annotation-Methode des PDF-Skills oder pypdf/reportlab:

```python
from pypdf import PdfReader, PdfWriter
from reportlab.pdfgen import canvas
from reportlab.lib.utils import ImageReader
import io

def add_signature(input_pdf, output_pdf, signature_path, page_num, x, y, width, height):
    packet = io.BytesIO()
    c = canvas.Canvas(packet)
    sig = ImageReader(signature_path)
    c.drawImage(sig, x, y, width, height, mask='auto')
    c.save()
    packet.seek(0)

    sig_pdf = PdfReader(packet)
    reader = PdfReader(input_pdf)
    writer = PdfWriter()

    for i, page in enumerate(reader.pages):
        if i == page_num:
            page.merge_page(sig_pdf.pages[0])
        writer.add_page(page)

    with open(output_pdf, "wb") as f:
        writer.write(f)
```

4. Passe die Position und Skalierung der Unterschrift an das identifizierte Unterschriftsfeld an.

### 4.5 Verifizieren und ausliefern

1. Wandle das ausgefüllte PDF in Bilder um und prüfe visuell, ob alle Felder korrekt befüllt sind.
2. Sende das fertige PDF an die Nutzerin zurück (via `SendUserFile`).
3. Fasse in ein bis zwei Sätzen zusammen, welche Felder befüllt wurden und ob Felder offen geblieben sind.

## 5 Ausgabeformat

- **Primärausgabe:** Das ausgefüllte PDF als Datei, direkt an die Nutzerin gesendet.
- **Begleittext:** Kurze Zusammenfassung (maximal 5 Zeilen): welche Felder befüllt, welche Datenquelle genutzt, welche Felder offen oder mit Platzhalter versehen.
- **Keine langen Erklärungen.** Das Ziel ist Geschwindigkeit: PDF rein, PDF raus.
- Wenn Felder nicht befüllbar waren (z. B. weil Daten fehlen), werden sie als `[noch zu ergänzen: …]` markiert und in der Zusammenfassung benannt.

## 6 Unterschriften-Hinweis

Die hinterlegte Unterschrift ist ein eingescanntes Bild, kein qualifiziertes elektronisches Signaturmittel im Sinne der eIDAS-VO. Für Rechtsgeschäfte, die der Schriftform nach § 126 BGB bedürfen, genügt ein eingescanntes Bild nicht. Bei Formulareinreichungen, die eine qualifizierte elektronische Signatur (QES) erfordern, weist das Skill die Nutzerin darauf hin und empfiehlt ein QES-Werkzeug (z. B. beA-Karte, Sign-Me, D-Trust).

## 7 Beispiel

**Eingabe:** „Bitte füll mir die Gewerbeanmeldung GewA 1 aus. Gewerbe: IT-Beratung. Beginn: 01.09.2026."

**Ablauf:**
1. PDF analysieren — 23 Felder erkannt (Name, Adresse, Geburtsdatum, Gewerbebezeichnung, Beginn, Unterschrift …).
2. 18 Felder aus `profil.md` befüllt (Name, Adresse, Geburtsdatum, Steuer-ID, Personalausweis-Nr., …).
3. 2 Felder aus der Nutzereingabe befüllt (Gewerbebezeichnung: „IT-Beratung", Beginn: „01.09.2026").
4. 1 Feld per Notion-Suche befüllt (zuständiges Gewerbeamt aus Notion-Kontaktdatenbank).
5. Unterschrift aus `unterschrift-nachname.png` im Unterschriftsfeld platziert.
6. 2 Felder offen: „Nebenerwerb ja/nein" (Rückfrage gestellt), „Handwerkskammer-Nr." (als Platzhalter markiert).
7. Fertiges PDF an die Nutzerin gesendet.

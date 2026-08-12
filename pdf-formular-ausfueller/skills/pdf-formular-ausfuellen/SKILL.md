---
name: pdf-formular-ausfuellen
description: "Füllt PDF-Formulare automatisch mit hinterlegten Stammdaten aus. Erkennt ob das Formular für Daniel oder Fabian oder die Firma ist und nutzt Profildaten sowie Notion-MCP und Unterschrift-Overlay."
---

# PDF-Formular ausfüllen

## 1 Zweck und Anwendungsfall

Dieses Skill nimmt ein beliebiges PDF-Formular entgegen und füllt es schnell und korrekt mit den hinterlegten Daten der richtigen Person oder Firma aus. Die Nutzerin erhält das fertige PDF zur Sichtung und Weiterleitung zurück, ohne selbst tippen zu müssen.

## 2 Personen-Erkennung — wer ist gemeint?

Vor dem Ausfüllen muss feststehen, für wen das Formular bestimmt ist. Das Skill kennt drei Profile:

- **Daniel** — Profil: `pdf-formular-ausfueller/stammdaten/daniel.md`
- **Fabian** — Profil: `pdf-formular-ausfueller/stammdaten/fabian.md`
- **Firma** — Profil: `pdf-formular-ausfueller/stammdaten/firma.md`

### 2.1 Automatische Zuordnung (aus dem Kontext)

Prüfe in dieser Reihenfolge:

1. **Explizite Nennung:** Die Nutzerin sagt „für Daniel", „für Fabian", „für die Firma / Kanzlei / GmbH".
2. **Formulartyp-Heuristik:**
   - Persönliche Formulare (Ummeldung, Elterngeld, Steuererklärung Privatperson, Führungszeugnis) — frage welche Person.
   - Firmenformulare (Handelsregister, Gewerbeanmeldung auf Firma, USt-Voranmeldung, IHK-Antrag) — Profil „Firma", unterschrieben vom Vertretungsberechtigten.
   - Mandatsvollmacht, Prozessvollmacht — Kontext prüfen: wer erteilt, wer empfängt.
3. **Gesprächskontext:** Wenn in früheren Nachrichten derselben Session klar wurde, um wen es geht, diese Zuordnung übernehmen.
4. **Notion-Kontext:** Wenn die Nutzerin eine Notion-Seite verlinkt oder ein Mandat referenziert, aus dem Mandats-/Projektkontext ableiten.

### 2.2 Rückfrage wenn nötig

Wenn keine eindeutige Zuordnung möglich ist, frage genau einmal:

> Für wen soll ich das Formular ausfüllen — **Daniel**, **Fabian** oder die **Firma**?

Keine weiteren Erklärungen, keine Theorie — nur die Frage.

### 2.3 Kombination

Manche Formulare brauchen Daten aus mehreren Profilen (z. B. Firmendaten plus persönliche Daten des Vertretungsberechtigten). In diesem Fall beide Profile laden und die Felder entsprechend zuordnen.

## 3 Datenquellen — Reihenfolge der Befüllung

1. **Explizite Angaben der Nutzerin** in der aktuellen Nachricht — haben immer Vorrang.
2. **Personenprofil** — die zugeordnete Datei unter `pdf-formular-ausfueller/stammdaten/`.
3. **Firmenprofil** — `pdf-formular-ausfueller/stammdaten/firma.md`, wenn Firmendaten benötigt werden.
4. **Kanzleiprofil** — `references/company-profile-template.md` als Ergänzung.
5. **Notion** (via MCP) — wenn ein Feld aus keinem Profil befüllbar ist, suche in Notion:
   - `mcp__Notion__notion-search` mit gezielten Begriffen (z. B. „Daniel Steuer-ID", „Fabian IBAN", „Firmenstempel").
   - Gefundene Seiten mit `mcp__Notion__notion-fetch` lesen.
   - Ergebnisse nur für die aktuelle Befüllung nutzen; bei dauerhaft fehlenden Daten empfehlen, das Profil zu ergänzen.
6. **Konversationskontext** — Daten aus früheren Nachrichten in derselben Session.
7. **Gebündelte Rückfrage** — fehlt eine Information nach allen obigen Quellen, alle fehlenden Felder in einer einzigen Frage bündeln. Maximal eine Rückfrage-Runde.

## 4 Ablauf

### 4.1 PDF analysieren

1. Lade den PDF-Skill (`/pdf`) und folge dem Workflow aus `FORMS.md`.
2. Prüfe, ob das PDF ausfüllbare Formularfelder hat:
   `python scripts/check_fillable_fields <datei.pdf>` (aus dem PDF-Skill-Verzeichnis).
3. Wandle das PDF in Bilder um:
   `python scripts/convert_pdf_to_images.py <datei.pdf> <bildordner/>`
4. Analysiere die Bilder: Beschriftung, erwarteter Inhalt, Typ (Text, Checkbox, Datum, Unterschrift) jedes Feldes.

### 4.2 Profil laden und Daten zusammentragen

1. Bestimme die Person/Firma (siehe Abschnitt 2).
2. Lies das zugeordnete Profil unter `pdf-formular-ausfueller/stammdaten/`.
3. Lies ggf. das Firmenprofil und `references/company-profile-template.md`.
4. Mappe jedes erkannte Formularfeld auf die passende Datenquelle.
5. Für Felder ohne Zuordnung: Notion-MCP-Suche.
6. Für weiterhin leere Felder: gebündelte Rückfrage.

### 4.3 Felder befüllen

**Bei ausfüllbaren PDF-Feldern (AcroForm):**

1. Feldinformationen extrahieren:
   `python scripts/extract_form_field_info.py <input.pdf> <field_info.json>`
2. `field_values.json` mit den zugeordneten Werten erstellen.
3. PDF befüllen:
   `python scripts/fill_fillable_fields.py <input.pdf> <field_values.json> <output.pdf>`

**Bei nicht ausfüllbaren PDF-Feldern (Annotation-basiert):**

1. Formularstruktur extrahieren:
   `python scripts/extract_form_structure.py <input.pdf> form_structure.json`
2. `fields.json` mit Koordinaten und Werten erstellen.
3. Bounding Boxes validieren:
   `python scripts/check_bounding_boxes.py fields.json`
4. PDF befüllen:
   `python scripts/fill_pdf_form_with_annotations.py <input.pdf> fields.json <output.pdf>`

### 4.4 Unterschrift einsetzen

Wenn das Formular ein Unterschriftsfeld enthält und die Nutzerin nicht widersprochen hat:

1. Unterschrift-Dateiname aus dem zugeordneten Profil lesen (Abschnitt „Unterschrift").
2. Unterschriftdatei laden aus `pdf-formular-ausfueller/stammdaten/unterschriften/`.
3. Als Bild-Overlay an der Stelle des Unterschriftsfeldes einsetzen:

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

4. Bei Firmenformularen: Firmenstempel zusätzlich einsetzen, wenn im Firmenprofil hinterlegt.

### 4.5 Verifizieren und ausliefern

1. Ausgefülltes PDF in Bilder umwandeln und visuell prüfen.
2. Fertiges PDF an die Nutzerin senden (via `SendUserFile`).
3. Kurze Zusammenfassung: welches Profil verwendet, welche Felder befüllt, welche offen.

## 5 Ausgabeformat

- **Primärausgabe:** Das ausgefüllte PDF als Datei.
- **Begleittext:** Maximal 5 Zeilen — Profil, befüllte Felder, offene Felder.
- **Keine langen Erklärungen.** PDF rein, PDF raus.
- Nicht befüllbare Felder werden als `[noch zu ergänzen: …]` markiert.

## 6 Unterschriften-Hinweis

Die hinterlegte Unterschrift ist ein eingescanntes Bild, kein qualifiziertes elektronisches Signaturmittel im Sinne der eIDAS-VO. Für Rechtsgeschäfte, die der Schriftform nach § 126 BGB bedürfen, genügt ein eingescanntes Bild nicht. Bei QES-Erfordernis weist der Skill darauf hin.

## 7 Beispiel

**Eingabe:** „Füll bitte die Gewerbeanmeldung aus. Gewerbe: IT-Beratung, Beginn 01.09.2026."

**Ablauf:**
1. Kontext-Check — „Gewerbeanmeldung" ohne Firmenkontext → Rückfrage: „Für wen — Daniel, Fabian oder die Firma?"
2. Antwort: „Daniel."
3. Profil `daniel.md` geladen — 18 Felder befüllt (Name, Adresse, Geburtsdatum, Steuer-ID, Ausweis-Nr. …).
4. Nutzereingabe: Gewerbebezeichnung „IT-Beratung", Beginn „01.09.2026".
5. Notion-Suche: zuständiges Gewerbeamt aus Daniels Kontaktdaten gefunden.
6. Unterschrift `unterschrift-daniel.png` im Unterschriftsfeld platziert.
7. Fertiges PDF gesendet. Zusammenfassung: „Ausgefüllt für Daniel. 21/23 Felder befüllt. Offen: Nebenerwerb ja/nein, Handwerkskammer-Nr."

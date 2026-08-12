# Stammdaten — Übersicht

Dieses Verzeichnis enthält die hinterlegten Profile für das automatische Ausfüllen von PDF-Formularen.

## Verfügbare Profile

| Kürzel | Datei | Beschreibung |
|--------|-------|--------------|
| **Daniel** | `daniel.md` | Persönliche Daten Daniel |
| **Fabian** | `fabian.md` | Persönliche Daten Fabian |
| **Firma** | `firma.md` | Firmendaten (Kanzlei / Unternehmen) |

## Wie die Zuordnung funktioniert

Der Skill erkennt automatisch, für wen das Formular ist:

1. **Explizite Nennung** — „Füll das für Daniel aus" oder „Das ist für die Firma".
2. **Kontexterkennung** — ein Gewerbeanmeldeformular mit Daniels Adresse im Gespräch wird Daniel zugeordnet; ein Handelsregister-Antrag der Firma.
3. **Rückfrage** — wenn unklar, fragt der Skill: „Für wen soll ich das ausfüllen — Daniel, Fabian oder die Firma?"

## Neues Profil anlegen

Eine neue `.md`-Datei nach dem Muster der bestehenden Profile anlegen und hier in der Tabelle ergänzen.

## Unterschriften

Unterschrift-Dateien liegen unter `unterschriften/` — pro Person eine PNG-Datei (transparenter Hintergrund, schwarze Tinte, mind. 600 px breit). Der Dateiname wird im jeweiligen Profil referenziert.

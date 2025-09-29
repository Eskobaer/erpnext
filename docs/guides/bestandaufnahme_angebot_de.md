# Werkzeug für Bestandsaufnahme und individuelles Angebot

Dieser Leitfaden beschreibt, wie du in ERPNext eine Bestandsaufnahme beim Kunden durchführst und daraus automatisch ein individuelles Angebot generierst. Er verbindet Standard-Funktionen aus den Modulen **Lager**, **CRM** und **Verkauf** mit einer kleinen Automatisierung per Server-Script.

## 1. Vorbereitung der Stammdaten
1. **Artikel und Preise pflegen**
   - Lege alle Produkte als *Artikel* an und hinterlege Preislisten.
   - Nutze Attribute (z. B. Farbe, Leistung), damit die spätere Angebotskonfiguration einfach bleibt.
2. **Bestandsstandorte anlegen**
   - Lege für jeden Kundenstandort ein Lager an oder verwende *Projekt* bzw. *Kostenstelle*, falls du Dienstleistungen dokumentierst.
3. **Dokumenttypen für die Bestandsaufnahme erstellen**
   - Lege über *Anpassen > Neues DocType* einen DocType `Bestandsaufnahme` mit den wichtigsten Feldern an (z. B. Kunde, Standort, Ansprechpartner, Bemerkungen).
   - Füge eine Kind-Tabelle `Bestandsaufnahme Position` hinzu (Felder: Artikel, vorhandene Menge, Zustandsbewertung, Austausch empfohlen, Notizen).

## 2. Durchführung der Bestandsaufnahme
1. Erfasse vor Ort alle Positionen im neuen DocType `Bestandsaufnahme`.
2. Speichere optionale Fotos oder Dokumente als Anhänge.
3. Verwende den Workflow-Status (z. B. *Entwurf → In Prüfung → Abgeschlossen*), um interne Freigaben abzubilden.

### Ergänzende Lagerfunktionen
- Nutze *Bestandsabgleich* (Stock Reconciliation), wenn der physische Lagerbestand in ERPNext aktualisiert werden soll.
- Verwende *Serien- und Chargennummern*, falls du Geräte eindeutig nachverfolgen musst.

## 3. Automatisches Erstellen eines Angebots
Lege ein **Server Script** (Typ *API*) an, das aus einer abgeschlossenen Bestandsaufnahme ein Angebot erzeugt:

```python
import frappe
from frappe import _

@frappe.whitelist()
def create_quotation_from_assessment(assessment_name):
    assessment = frappe.get_doc("Bestandsaufnahme", assessment_name)
    if assessment.docstatus != 1:
        frappe.throw(_("Die Bestandsaufnahme muss freigegeben sein."))

    quotation = frappe.new_doc("Quotation")
    quotation.quotation_to = "Customer"
    quotation.party_name = assessment.customer
    quotation.valid_till = assessment.valid_until or None
    quotation.set("items", [])

    for row in assessment.items:
        if not row.replacement_suggested:
            continue
        quotation.append("items", {
            "item_code": row.item,
            "qty": row.recommended_qty or 1,
            "uom": row.uom or "Stk",
            "rate": frappe.db.get_value("Item Price", {"item_code": row.item, "price_list": assessment.price_list}, "price_list_rate") or 0,
            "description": row.notes,
        })

    quotation.insert(ignore_permissions=True)
    quotation.submit()
    return quotation.name
```

> 💡 **Tipp:** Weise dem Server Script eine Schaltfläche im DocType `Bestandsaufnahme` zu (z. B. über *Custom Button*), damit Benutzer mit einem Klick das Angebot erzeugen können.

## 4. Nachverfolgung und Kommunikation
1. Sende das erzeugte Angebot direkt aus ERPNext per E-Mail an den Kunden.
2. Verknüpfe das Angebot mit einem *Opportunity*-Eintrag, um den Verkaufsprozess transparent zu halten.
3. Nutze *Aktivitäten* und *Kommentare*, um Rückmeldungen und Änderungen festzuhalten.

## 5. Erweiterungen
- **Bewertung & Scoring:** Ergänze Felder für technische Priorität oder Budgeteinschätzung und verwende Berichte zur Priorisierung.
- **Projektintegration:** Erstelle aus angenommenen Angeboten automatisch Projekte oder Service-Tickets.
- **Dashboards:** Baue ein Dashboard, das offene Bestandsaufnahmen, Konversionsraten und Umsätze zeigt.

Mit diesem Werkzeug erhältst du eine strukturierte Bestandsaufnahme und kannst daraus schnell ein individuelles Angebot generieren.

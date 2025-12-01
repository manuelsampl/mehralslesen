# Schulkennzahl (SKZ) Validierung - Implementierung

## Überblick

Diese Implementierung fügt eine Pflicht-Validierung für Schulkennzahlen (SKZ) beim Checkout hinzu, wenn spezifische Schülerabo-Produkte im Warenkorb sind.

## Funktionsweise

### 1. Warenkorb-Validierung

**Datei**: `snippets/cart-skz-validation.liquid`

- Prüft automatisch, ob Produkte mit den definierten SKUs im Warenkorb sind
- Zeigt ein Eingabefeld für die 6-stellige Schulkennzahl an
- Blockiert den Checkout-Button bis eine gültige SKZ eingegeben wurde
- Speichert die SKZ als Cart Attribute (wird zur Order übertragen)
- Speichert die SKZ zusätzlich im localStorage als Backup

**Validierte SKUs**:
```
HS_SF, HS_LE, SPO_SF, SPO_LE, SPA_SF, SPA_LE,
ME_SF, ME_LE, MW_SF, MW_LE, HS, KiGa-HS,
SPO, SPA, ME, MW
```

**Eingabe-Validierung**:
- Muss genau 6 Ziffern sein
- Nur Zahlen erlaubt
- Echtzeit-Validierung beim Tippen

### 2. Integration im Warenkorb

**Datei**: `snippets/cart-summary.liquid`

Die SKZ-Validierung wird direkt nach der Free-Samples-Validierung angezeigt:

```liquid
{% render 'cart-free-samples-validation' %}
{% render 'cart-skz-validation' %}
```

### 3. Datenübertragung

Die SKZ wird auf **zwei Wegen** gespeichert:

#### a) Cart Attribute (Primär)
```liquid
<input name="attributes[Schulkennzahl]" ... />
```

Dieser Wert wird automatisch:
- In den Order Attributes gespeichert
- In der Shopify Admin sichtbar
- Per Webhook abrufbar

#### b) localStorage (Backup)
```javascript
localStorage.setItem('cart_skz', skz);
```

Wird verwendet für:
- Wiederherstellung bei Seitenrefresh
- Zusätzliche Sicherheit
- Optional: Thank You Page Integration

### 4. Metafield-Speicherung

Die SKZ aus den Cart Attributes muss in Order und Customer Metafields übertragen werden. **Wählen Sie eine der folgenden Methoden:**

---

## Implementierungs-Optionen

### Option 1: Shopify Flow (Empfohlen für Shopify Plus)

**Vorteile**:
- Keine Code-Änderungen nötig
- Visueller Workflow-Editor
- Einfach zu warten

**Setup**:

1. Gehe zu **Shopify Admin → Apps → Shopify Flow**

2. Erstelle neuen Flow: **"SKZ zu Order/Customer Metafields"**

3. **Trigger**: Order created

4. **Condition 1**: Check if order has attribute "Schulkennzahl"
   ```
   Order → Attributes → Schulkennzahl exists
   ```

5. **Action 1**: Set order metafield
   - Namespace: `custom`
   - Key: `skz`
   - Value: `{{ order.attributes.Schulkennzahl }}`
   - Type: `single_line_text_field`

6. **Condition 2**: Check if customer exists
   ```
   Order → Customer → ID exists
   ```

7. **Action 2**: Set customer metafield
   - Namespace: `custom`
   - Key: `skz`
   - Value: `{{ order.attributes.Schulkennzahl }}`
   - Type: `single_line_text_field`

8. Flow aktivieren

**Fertig!** Die SKZ wird nun automatisch bei jeder Order übertragen.

---

### Option 2: Webhook + Backend (Für alle Shopify-Pläne)

**Vorteile**:
- Funktioniert auf allen Shopify-Plänen
- Vollständige Kontrolle
- Kann erweitert werden

**Schritte**:

1. **Backend-Endpoint erstellen** (Node.js Beispiel):

```javascript
const express = require('express');
const Shopify = require('shopify-api-node');

const app = express();
app.use(express.json());

const shopify = new Shopify({
  shopName: 'your-shop.myshopify.com',
  apiKey: process.env.SHOPIFY_API_KEY,
  password: process.env.SHOPIFY_API_PASSWORD
});

app.post('/webhooks/orders/create', async (req, res) => {
  const order = req.body;
  
  // Prüfe ob SKZ vorhanden
  const skzAttribute = order.note_attributes?.find(
    attr => attr.name === 'Schulkennzahl'
  );
  
  if (!skzAttribute) {
    return res.status(200).send('No SKZ found');
  }
  
  const skz = skzAttribute.value;
  
  try {
    // Speichere in Order Metafield
    await shopify.metafield.create({
      namespace: 'custom',
      key: 'skz',
      value: skz,
      type: 'single_line_text_field',
      owner_resource: 'order',
      owner_id: order.id
    });
    
    // Speichere in Customer Metafield (falls vorhanden)
    if (order.customer?.id) {
      await shopify.metafield.create({
        namespace: 'custom',
        key: 'skz',
        value: skz,
        type: 'single_line_text_field',
        owner_resource: 'customer',
        owner_id: order.customer.id
      });
    }
    
    console.log(`✅ SKZ ${skz} saved for order ${order.id}`);
    res.status(200).send('SKZ saved');
    
  } catch (error) {
    console.error('❌ Error saving SKZ:', error);
    res.status(500).send('Error');
  }
});

app.listen(3000, () => {
  console.log('Webhook listener running on port 3000');
});
```

2. **Webhook in Shopify einrichten**:
   - Gehe zu **Settings → Notifications → Webhooks**
   - Event: `Order creation`
   - Format: `JSON`
   - URL: `https://your-domain.com/webhooks/orders/create`

3. **Webhook-Secret validieren** (Sicherheit):

```javascript
const crypto = require('crypto');

function verifyWebhook(data, hmacHeader, secret) {
  const hash = crypto
    .createHmac('sha256', secret)
    .update(data, 'utf8')
    .digest('base64');
  
  return hash === hmacHeader;
}

app.post('/webhooks/orders/create', (req, res) => {
  const hmac = req.get('X-Shopify-Hmac-SHA256');
  
  if (!verifyWebhook(JSON.stringify(req.body), hmac, process.env.WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }
  
  // ... rest of code
});
```

---

### Option 3: Shopify App mit App Proxy

**Vorteile**:
- Professionelle Lösung
- Kann im App Store veröffentlicht werden
- Erweiterte Funktionen möglich

**Schritte**:

1. Erstelle Shopify App mit Scopes:
   - `write_orders`
   - `write_customers`

2. Implementiere App Proxy Endpoint

3. Update `snippets/checkout-skz-additional-script.liquid`:
   ```javascript
   const webhookUrl = '/apps/your-app/save-skz';
   ```

---

### Option 4: Manuelle Verarbeitung (Temporäre Lösung)

**Für schnellen Start ohne Backend:**

1. Die SKZ ist bereits in **Order Notes** sichtbar (als Cart Attribute)

2. **Manuell in Shopify Admin**:
   - Öffne Order → More actions → Edit metafields
   - Füge hinzu: `custom.skz = [Wert aus Order Attributes]`

3. **Oder**: Bulk-Update via CSV/Script:
   ```javascript
   // Beispiel-Script zum Batch-Update
   const orders = await shopify.order.list();
   
   for (const order of orders) {
     const skz = order.note_attributes?.find(a => a.name === 'Schulkennzahl');
     if (skz) {
       await shopify.metafield.create({
         owner_resource: 'order',
         owner_id: order.id,
         namespace: 'custom',
         key: 'skz',
         value: skz.value,
         type: 'single_line_text_field'
       });
     }
   }
   ```

---

## Metafield-Definitionen

Erstelle diese Metafield-Definitionen in Shopify Admin:

### Order Metafield
- **Namespace**: `custom`
- **Key**: `skz`
- **Name**: Schulkennzahl
- **Type**: Single line text
- **Validation**: Number only, exactly 6 digits

### Customer Metafield
- **Namespace**: `custom`
- **Key**: `skz`
- **Name**: Schulkennzahl
- **Type**: Single line text
- **Validation**: Number only, exactly 6 digits

**Pfad**: Settings → Custom data → Orders / Customers → Add definition

---

## Testing

### 1. Funktionstest

1. **Produkt ohne SKU-Anforderung hinzufügen**:
   - SKZ-Feld sollte nicht erscheinen
   - Checkout sollte normal funktionieren

2. **Schülerabo-Produkt hinzufügen** (z.B. SKU: `HS_SF`):
   - SKZ-Validierung sollte erscheinen
   - Checkout-Button sollte deaktiviert sein
   - Hinweistext sollte sichtbar sein

3. **Ungültige SKZ eingeben**:
   - `12345` (zu kurz) → Fehlermeldung
   - `1234567` (zu lang) → Fehlermeldung
   - `abc123` (Buchstaben) → Fehlermeldung

4. **Gültige SKZ eingeben**:
   - `123456` → Kein Fehler
   - Checkout-Button aktiviert
   - Cart Attribute wird gesetzt

5. **Order erstellen**:
   - Checkout durchführen
   - In Shopify Admin Order öffnen
   - Prüfen: Order Attributes enthält "Schulkennzahl: 123456"

### 2. Metafield-Verifikation

Nach Setup der gewählten Implementierung:

1. Test-Order mit SKZ erstellen
2. Order in Shopify Admin öffnen
3. Prüfen: Metafield `custom.skz` existiert
4. Customer-Profil öffnen
5. Prüfen: Metafield `custom.skz` existiert

### 3. Browser Console Logs

Öffne Developer Tools (F12) beim Testen:

```
🔍 Cart SKZ Validation loaded
📊 SKZ Validation: { requiresSKZ: true, items: [...] }
💾 SKZ saved to localStorage: 123456
✅ Cart attribute updated with SKZ
```

---

## Wartung & Updates

### SKU-Liste erweitern

Bearbeite `snippets/cart-skz-validation.liquid`:

```javascript
const STUDENT_SUBSCRIPTION_SKUS = [
  'HS_SF', 'HS_LE', 
  // ... bestehende SKUs
  'NEW_SKU_1',  // Neue SKU hinzufügen
  'NEW_SKU_2'
];
```

### Validierung anpassen

Ändere die Validierungs-Funktion:

```javascript
function isValidSKZ(skz) {
  // Aktuell: genau 6 Ziffern
  return /^[0-9]{6}$/.test(skz);
  
  // Alternative: 6-8 Ziffern erlauben
  // return /^[0-9]{6,8}$/.test(skz);
}
```

### Texte ändern

Bearbeite die Lokalisierungsdateien:
- `locales/de.json`
- `locales/en.default.json`

```json
"cart": {
  "skz_validation": {
    "title": "Neuer Titel",
    "student_subscription_notice": "Neuer Text",
    ...
  }
}
```

---

## Fehlerbehebung

### Problem: Checkout-Button bleibt deaktiviert

**Ursache**: Konflikt mit Free-Samples-Validierung

**Lösung**: Beide Validierungen müssen bestehen
- Prüfe Free-Samples-Limit
- Prüfe SKZ-Eingabe (6 Ziffern)

### Problem: SKZ erscheint nicht in Order

**Ursache**: Cart Attribute nicht übertragen

**Lösung**:
1. Prüfe Browser Console auf Fehler
2. Verifiziere `form="cart-form"` Attribut
3. Teste `/cart/update.js` API direkt

### Problem: Metafields nicht gefüllt

**Ursache**: Webhook/Flow nicht konfiguriert

**Lösung**:
- Wähle eine Implementierungs-Option oben
- Teste mit Test-Order
- Prüfe Webhook/Flow Logs

---

## Zusätzliche Hinweise

### Datenschutz

Die Schulkennzahl ist eine öffentliche Kennung und enthält keine personenbezogenen Daten. Dennoch:
- Wird verschlüsselt übertragen (HTTPS)
- Nur in Backend-Systemen gespeichert
- Kann in Privacy Policy erwähnt werden

### Performance

- Validierung läuft client-side (keine Server-Verzögerung)
- Cart Attribute Update: ~100ms
- Keine Auswirkung auf Checkout-Performance

### Kompatibilität

- ✅ Funktioniert mit Dawn Theme
- ✅ Kompatibel mit Cart Drawer
- ✅ Unterstützt View Transitions
- ✅ Mobile-responsive

---

## Support

Bei Fragen oder Problemen:

1. **Browser Console öffnen** (F12)
2. **Logs prüfen** (beginnen mit 🔍 oder 📊)
3. **Screenshots** von Fehlermeldungen erstellen
4. **Order-Nummer** notieren für Debugging

---

## Changelog

### Version 1.0 (Dezember 2024)
- ✅ Initiale Implementierung
- ✅ SKZ-Validierung für 16 SKUs
- ✅ Cart Attribute Integration
- ✅ localStorage Backup
- ✅ Deutsche & Englische Übersetzungen
- ✅ Dokumentation & Setup-Anleitungen

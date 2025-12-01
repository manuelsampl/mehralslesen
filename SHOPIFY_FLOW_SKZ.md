# Shopify Flow Konfiguration für SKZ

## Übersicht

Da die SKZ bereits als Cart Attribute `Schulkennzahl` gespeichert wird, landet sie automatisch in den Order Attributes. Wir nutzen Shopify Flow, um sie von dort in Metafields und Tags zu übertragen.

## Flow Setup (Shopify Plus erforderlich)

### Flow 1: SKZ zu Order & Customer Metafields + Tags

**Name:** "SKZ Speichern (Order & Customer)"

**Trigger:** Order created

**Bedingungen & Aktionen:**

```yaml
Trigger: Order created

├─ Condition: Order attribute "Schulkennzahl" exists
│  └─ If TRUE:
│     │
│     ├─ Action 1: Set order metafield
│     │  - Namespace: custom
│     │  - Key: skz
│     │  - Value: {{ order.customAttributes.Schulkennzahl }}
│     │  - Type: single_line_text_field
│     │
│     ├─ Action 2: Add order tags
│     │  - Tags: skz-{{ order.customAttributes.Schulkennzahl }}
│     │
│     └─ Condition: Customer exists
│        └─ If TRUE:
│           └─ Action 3: Set customer metafield
│              - Namespace: custom
│              - Key: skz
│              - Value: {{ order.customAttributes.Schulkennzahl }}
│              - Type: single_line_text_field
```

---

## Schritt-für-Schritt Anleitung

### 1. Flow erstellen

1. Gehe zu **Shopify Admin → Apps → Flow**
2. Klicke auf **"Create workflow"**
3. Wähle **"Create blank workflow"**
4. Name: `SKZ Speichern (Order & Customer)`

### 2. Trigger hinzufügen

1. Klicke auf **"Select a trigger"**
2. Suche nach: `Order created`
3. Wähle: **Order → Order created**
4. Klicke **"Done"**

### 3. Bedingung hinzufügen (SKZ prüfen)

1. Klicke auf **"+"** unter dem Trigger
2. Wähle **"Condition"**
3. Konfiguration:
   - **If:** `Order custom attribute`
   - **Attribute name:** `Schulkennzahl`
   - **Condition:** `is set`
4. Klicke **"Done"**

### 4. Aktion 1: Order Metafield setzen

1. Unter **"Then"** klicke auf **"+"**
2. Suche nach: `Set metafield`
3. Wähle: **Order → Set metafield value**
4. Konfiguration:
   - **Namespace:** `custom`
   - **Key:** `skz`
   - **Value:** Klicke auf das **"+"** Symbol und wähle:
     - `Order` → `Custom attributes` → `Schulkennzahl` → `Value`
   - **Value type:** `Single line text`
5. Klicke **"Done"**

### 5. Aktion 2: Order Tag hinzufügen

1. Unter vorheriger Aktion klicke auf **"+"**
2. Suche nach: `Add order tags`
3. Wähle: **Order → Add order tags**
4. Konfiguration:
   - **Tags:** Tippe `skz-` und dann klicke auf **"+"**:
     - `Order` → `Custom attributes` → `Schulkennzahl` → `Value`
   - Das Ergebnis sollte sein: `skz-{{ order.customAttributes.Schulkennzahl.value }}`
5. Klicke **"Done"**

### 6. Bedingung 2: Customer existiert prüfen

1. Unter vorheriger Aktion klicke auf **"+"**
2. Wähle **"Condition"**
3. Konfiguration:
   - **If:** `Customer`
   - **Condition:** `is set`
4. Klicke **"Done"**

### 7. Aktion 3: Customer Metafield setzen

1. Unter **"Then"** der Customer-Bedingung klicke auf **"+"**
2. Suche nach: `Set metafield`
3. Wähle: **Customer → Set metafield value**
4. Konfiguration:
   - **Namespace:** `custom`
   - **Key:** `skz`
   - **Value:** Klicke auf das **"+"** Symbol:
     - `Order` → `Custom attributes` → `Schulkennzahl` → `Value`
   - **Value type:** `Single line text`
5. Klicke **"Done"**

### 8. Flow aktivieren

1. Klicke oben rechts auf **"Turn on workflow"**
2. Bestätige mit **"Turn on"**

---

## Alternative: Ohne Shopify Plus (Flow)

Falls Sie **kein Shopify Plus** haben, nutzen Sie einen **Webhook**:

### Webhook-Option mit Shopify Functions

Da Shopify Functions keine Order-Manipulation nach Erstellung erlauben, benötigen Sie ein kleines Backend. Hier ist eine **serverless Lösung mit Cloudflare Workers** (kostenlos):

#### Cloudflare Worker Setup

**1. Worker Code erstellen:**

```javascript
// Cloudflare Worker für SKZ
export default {
  async fetch(request, env) {
    // Nur POST Requests
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    try {
      const webhook = await request.json();
      
      // SKZ aus Order Attributes holen
      const skzAttr = webhook.note_attributes?.find(
        attr => attr.name === 'Schulkennzahl'
      );
      
      if (!skzAttr) {
        return new Response(JSON.stringify({ 
          success: false, 
          message: 'No SKZ found' 
        }), { 
          status: 200,
          headers: { 'Content-Type': 'application/json' }
        });
      }

      const skz = skzAttr.value;
      const orderId = webhook.id;
      const customerId = webhook.customer?.id;

      // Shopify Admin API Calls
      const shopDomain = env.SHOPIFY_SHOP;
      const accessToken = env.SHOPIFY_ACCESS_TOKEN;
      
      const headers = {
        'Content-Type': 'application/json',
        'X-Shopify-Access-Token': accessToken
      };

      // 1. Order Metafield setzen
      await fetch(`https://${shopDomain}/admin/api/2024-10/orders/${orderId}/metafields.json`, {
        method: 'POST',
        headers,
        body: JSON.stringify({
          metafield: {
            namespace: 'custom',
            key: 'skz',
            value: skz,
            type: 'single_line_text_field'
          }
        })
      });

      // 2. Order Tag hinzufügen
      const orderResponse = await fetch(`https://${shopDomain}/admin/api/2024-10/orders/${orderId}.json`, {
        method: 'GET',
        headers
      });
      const orderData = await orderResponse.json();
      const currentTags = orderData.order.tags || '';
      const newTags = currentTags ? `${currentTags}, skz-${skz}` : `skz-${skz}`;

      await fetch(`https://${shopDomain}/admin/api/2024-10/orders/${orderId}.json`, {
        method: 'PUT',
        headers,
        body: JSON.stringify({
          order: {
            id: orderId,
            tags: newTags
          }
        })
      });

      // 3. Customer Metafield setzen (falls vorhanden)
      if (customerId) {
        await fetch(`https://${shopDomain}/admin/api/2024-10/customers/${customerId}/metafields.json`, {
          method: 'POST',
          headers,
          body: JSON.stringify({
            metafield: {
              namespace: 'custom',
              key: 'skz',
              value: skz,
              type: 'single_line_text_field'
            }
          })
        });
      }

      return new Response(JSON.stringify({ 
        success: true,
        order_id: orderId,
        skz: skz,
        tag: `skz-${skz}`
      }), {
        status: 200,
        headers: { 'Content-Type': 'application/json' }
      });

    } catch (error) {
      return new Response(JSON.stringify({ 
        success: false, 
        error: error.message 
      }), {
        status: 500,
        headers: { 'Content-Type': 'application/json' }
      });
    }
  }
};
```

**2. Deployen:**

```bash
# Cloudflare Account erstellen (kostenlos)
# Dann:
npm install -g wrangler
wrangler login
wrangler init skz-webhook
# Code einfügen
wrangler deploy
```

**3. Environment Variables setzen:**

```bash
wrangler secret put SHOPIFY_SHOP
# Eingeben: your-shop.myshopify.com

wrangler secret put SHOPIFY_ACCESS_TOKEN
# Eingeben: your_access_token
```

**4. Webhook in Shopify einrichten:**

1. Gehe zu **Settings → Notifications → Webhooks**
2. Klicke **"Create webhook"**
3. Event: **Order creation**
4. Format: **JSON**
5. URL: `https://skz-webhook.your-username.workers.dev`
6. API Version: **2024-10**
7. Klicke **"Save"**

---

## Testen

### Flow/Webhook testen:

1. **Test-Bestellung erstellen:**
   - Füge ein Schülerabo-Produkt zum Warenkorb hinzu
   - Gib SKZ `123456` ein
   - Schließe Bestellung ab

2. **In Shopify Admin prüfen:**
   - Gehe zu **Orders** → Deine Test-Order
   - **Metafield prüfen:** Scrolle runter zu "Metafields" → sollte `custom.skz = 123456` zeigen
   - **Tag prüfen:** Oben bei Tags sollte `skz-123456` erscheinen

3. **Customer prüfen:**
   - Gehe zu **Customers** → Dein Test-Customer
   - **Metafield prüfen:** Scrolle runter zu "Metafields" → sollte `custom.skz = 123456` zeigen

### Flow Logs ansehen:

1. Gehe zu **Shopify Admin → Apps → Flow**
2. Klicke auf deinen Flow
3. Tab **"Run history"**
4. Siehe erfolgreiche/fehlgeschlagene Ausführungen

---

## Empfehlung

**Wenn Sie Shopify Plus haben:**
→ Nutzen Sie **Shopify Flow** (siehe Anleitung oben)
- ✅ Kein Code
- ✅ Visueller Editor
- ✅ Einfach zu warten
- ✅ Native Shopify Integration

**Wenn Sie KEIN Shopify Plus haben:**
→ Nutzen Sie **Cloudflare Workers** (siehe oben)
- ✅ Kostenlos (bis 100.000 Requests/Tag)
- ✅ Schnell (~5 Min Setup)
- ✅ Kein Server nötig
- ✅ Automatische Skalierung

---

## Warum funktioniert das Frontend-Script nicht?

Das Script in `checkout-skz-save.liquid` versucht `/apps/skz-proxy/save-metafield` aufzurufen, aber:

1. ❌ Dieser Endpoint existiert nicht (noch kein Backend deployed)
2. ❌ Shopify erlaubt keine direkten Metafield-Updates von Frontend
3. ❌ Daher: "Failed to fetch" Error

**Lösung:** 
- Entferne das Frontend-Script (nicht nötig)
- Nutze stattdessen Flow oder Cloudflare Worker
- Cart Attributes werden automatisch zu Order Attributes
- Flow/Worker liest Order Attributes und setzt Metafields + Tags

---

## Nächste Schritte

1. **Wählen Sie eine Option:**
   - [ ] Shopify Flow (wenn Plus verfügbar)
   - [ ] Cloudflare Worker (wenn kein Plus)

2. **Setup durchführen** (siehe Anleitung oben)

3. **Testen** mit einer Test-Bestellung

4. **Frontend-Script deaktivieren** (optional):
   ```liquid
   {# In layout/theme.liquid auskommentieren #}
   {% comment %}
   {% render 'checkout-skz-save' %}
   {% endcomment %}
   ```

Die SKZ wird dann zuverlässig gespeichert! 🎉

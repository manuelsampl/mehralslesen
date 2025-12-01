# 🚀 Schnellstart: SKZ in Order Metafields & Tags speichern

## Problem

Die SKZ wird im Warenkorb eingegeben und als **Cart Attribute** gespeichert, landet also automatisch in den **Order Attributes**. Aber sie wird noch nicht als **Order Metafield** oder **Order Tag** gespeichert.

## ✅ Beste Lösung: Shopify Flow

**Voraussetzung:** Shopify Plus

Falls Sie **KEIN Shopify Plus** haben, siehe [Alternative ohne Plus](#alternative-ohne-shopify-plus) unten.

---

## Shopify Flow Setup (5 Minuten)

### 1. Flow öffnen

- Gehe zu **Shopify Admin → Apps → Flow**
- Klicke **"Create workflow"**
- Wähle **"Create blank workflow"**

### 2. Flow konfigurieren

**Trigger:** `Order created`

**Condition 1:** 
```
Order custom attribute "Schulkennzahl" is set
```

**Then Actions:**

**Action 1:** Set order metafield
- Namespace: `custom`
- Key: `skz`
- Value: `{{ order.customAttributes.Schulkennzahl.value }}`
- Type: `Single line text`

**Action 2:** Add order tags
- Tags: `skz-{{ order.customAttributes.Schulkennzahl.value }}`

**Condition 2:** 
```
Customer is set
```

**Action 3:** Set customer metafield
- Namespace: `custom`
- Key: `skz`  
- Value: `{{ order.customAttributes.Schulkennzahl.value }}`
- Type: `Single line text`

### 3. Flow aktivieren

- Klicke **"Turn on workflow"**
- ✅ Fertig!

---

## Alternative ohne Shopify Plus

### Option A: Cloudflare Worker (Kostenlos)

**Zeit: 10 Minuten**

1. **Account erstellen:** https://dash.cloudflare.com/sign-up

2. **Worker erstellen:**
   ```bash
   npm install -g wrangler
   wrangler login
   wrangler init skz-webhook
   ```

3. **Code einfügen:** (siehe `SHOPIFY_FLOW_SKZ.md` für vollständigen Code)

4. **Deployen:**
   ```bash
   wrangler secret put SHOPIFY_SHOP
   # your-shop.myshopify.com
   
   wrangler secret put SHOPIFY_ACCESS_TOKEN
   # Dein Access Token
   
   wrangler deploy
   ```

5. **Webhook einrichten:**
   - Shopify Admin → Settings → Notifications → Webhooks
   - Event: `Order creation`
   - URL: `https://skz-webhook.your-username.workers.dev`

### Option B: Zapier (Kein Code)

**Zeit: 15 Minuten**

1. **Zapier Account** erstellen
2. **Zap erstellen:**
   - Trigger: **Shopify** → New Order
   - Filter: Order Custom Attribute "Schulkennzahl" exists
   - Action 1: **Shopify** → Update Order (Add Tag: `skz-{{Schulkennzahl}}`)
   - Action 2: **Webhooks** → POST zu Shopify Admin API (Set Metafield)

---

## Test nach Setup

1. **Test-Order erstellen** mit SKZ
2. **Order öffnen** in Shopify Admin
3. **Prüfen:**
   - ✅ Tag `skz-123456` vorhanden?
   - ✅ Metafield `custom.skz` sichtbar?
   - ✅ Customer Metafield `custom.skz` gesetzt?

---

## Wichtig

Das Frontend-Script (`checkout-skz-save.liquid`) ist jetzt **deaktiviert**, da es ein Backend benötigt. Die SKZ wird trotzdem korrekt:

1. ✅ Im Warenkorb eingegeben
2. ✅ Als Cart Attribute gespeichert  
3. ✅ Automatisch zu Order Attributes übertragen
4. ✅ Von Flow/Worker in Metafields & Tags übertragen

Alles funktioniert automatisch! 🎉

---

## Support

- **Detaillierte Anleitung:** `SHOPIFY_FLOW_SKZ.md`
- **Backend API:** `SKZ_BACKEND_API.md`
- **Implementation:** `SKZ_IMPLEMENTATION.md`

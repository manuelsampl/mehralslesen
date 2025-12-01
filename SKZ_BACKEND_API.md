# SKZ Backend API Dokumentation

## Übersicht

Das SKZ-System benötigt ein Backend (Shopify App oder Server), das die folgenden API-Endpoints bereitstellt:

## Endpoints

### 1. Save Metafield

**Endpoint:** `POST /apps/skz-proxy/save-metafield`

**Beschreibung:** Speichert die SKZ als Metafield für Order oder Customer

**Request Body:**
```json
{
  "resource": "order",           // oder "customer"
  "resource_id": "5123456789",   // Order ID oder Customer ID
  "namespace": "custom",
  "key": "skz",
  "value": "123456",
  "type": "single_line_text_field"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Metafield saved",
  "metafield_id": "987654321"
}
```

**Implementierung (Node.js Beispiel):**
```javascript
app.post('/apps/skz-proxy/save-metafield', async (req, res) => {
  try {
    const { resource, resource_id, namespace, key, value, type } = req.body;
    
    // Validierung
    if (!resource || !resource_id || !value) {
      return res.status(400).json({ 
        success: false, 
        error: 'Missing required fields' 
      });
    }

    // SKZ Format validieren (6 Ziffern)
    if (!/^\d{6}$/.test(value)) {
      return res.status(400).json({ 
        success: false, 
        error: 'SKZ must be exactly 6 digits' 
      });
    }

    // Shopify Admin API Call
    const metafield = await shopify.metafield.create({
      namespace: namespace,
      key: key,
      value: value,
      type: type || 'single_line_text_field',
      owner_resource: resource,
      owner_id: resource_id
    });

    console.log(`✅ Metafield created: ${resource} ${resource_id} - SKZ: ${value}`);

    res.json({
      success: true,
      message: 'Metafield saved',
      metafield_id: metafield.id
    });

  } catch (error) {
    console.error('❌ Error saving metafield:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});
```

---

### 2. Add Order Tag

**Endpoint:** `POST /apps/skz-proxy/add-order-tag`

**Beschreibung:** Fügt einen Tag `skz-XXXXXX` zur Order hinzu

**Request Body:**
```json
{
  "order_id": "5123456789",
  "tag": "skz-123456"
}
```

**Response:**
```json
{
  "success": true,
  "message": "Tag added",
  "tags": ["skz-123456", "other-tag"]
}
```

**Implementierung (Node.js Beispiel):**
```javascript
app.post('/apps/skz-proxy/add-order-tag', async (req, res) => {
  try {
    const { order_id, tag } = req.body;
    
    // Validierung
    if (!order_id || !tag) {
      return res.status(400).json({ 
        success: false, 
        error: 'Missing order_id or tag' 
      });
    }

    // Tag Format validieren (skz-XXXXXX)
    if (!/^skz-\d{6}$/.test(tag)) {
      return res.status(400).json({ 
        success: false, 
        error: 'Tag must be in format: skz-XXXXXX (6 digits)' 
      });
    }

    // Order abrufen
    const order = await shopify.order.get(order_id);
    
    // Aktuellen Tags lesen
    let tags = order.tags ? order.tags.split(', ') : [];
    
    // Neuen Tag hinzufügen (wenn nicht schon vorhanden)
    if (!tags.includes(tag)) {
      tags.push(tag);
      
      // Order aktualisieren
      await shopify.order.update(order_id, {
        tags: tags.join(', ')
      });
      
      console.log(`✅ Tag added to order ${order_id}: ${tag}`);
    } else {
      console.log(`ℹ️ Tag already exists on order ${order_id}: ${tag}`);
    }

    res.json({
      success: true,
      message: 'Tag added',
      tags: tags
    });

  } catch (error) {
    console.error('❌ Error adding tag:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});
```

---

## Vollständiges Backend-Beispiel

### Node.js + Express + shopify-api-node

**Installation:**
```bash
npm install express shopify-api-node dotenv
```

**`.env` Datei:**
```env
SHOPIFY_SHOP_NAME=your-shop.myshopify.com
SHOPIFY_API_KEY=your_api_key
SHOPIFY_API_PASSWORD=your_api_password
PORT=3000
```

**`server.js`:**
```javascript
require('dotenv').config();
const express = require('express');
const Shopify = require('shopify-api-node');

const app = express();
app.use(express.json());

// Shopify API initialisieren
const shopify = new Shopify({
  shopName: process.env.SHOPIFY_SHOP_NAME,
  apiKey: process.env.SHOPIFY_API_KEY,
  password: process.env.SHOPIFY_API_PASSWORD
});

// Health Check
app.get('/health', (req, res) => {
  res.json({ status: 'OK', service: 'SKZ Backend' });
});

// 1. Save Metafield Endpoint
app.post('/apps/skz-proxy/save-metafield', async (req, res) => {
  try {
    const { resource, resource_id, namespace, key, value, type } = req.body;
    
    if (!resource || !resource_id || !value) {
      return res.status(400).json({ 
        success: false, 
        error: 'Missing required fields' 
      });
    }

    if (!/^\d{6}$/.test(value)) {
      return res.status(400).json({ 
        success: false, 
        error: 'SKZ must be exactly 6 digits' 
      });
    }

    const metafield = await shopify.metafield.create({
      namespace: namespace,
      key: key,
      value: value,
      type: type || 'single_line_text_field',
      owner_resource: resource,
      owner_id: resource_id
    });

    console.log(`✅ Metafield created: ${resource} ${resource_id} - SKZ: ${value}`);

    res.json({
      success: true,
      message: 'Metafield saved',
      metafield_id: metafield.id
    });

  } catch (error) {
    console.error('❌ Error saving metafield:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// 2. Add Order Tag Endpoint
app.post('/apps/skz-proxy/add-order-tag', async (req, res) => {
  try {
    const { order_id, tag } = req.body;
    
    if (!order_id || !tag) {
      return res.status(400).json({ 
        success: false, 
        error: 'Missing order_id or tag' 
      });
    }

    if (!/^skz-\d{6}$/.test(tag)) {
      return res.status(400).json({ 
        success: false, 
        error: 'Tag must be in format: skz-XXXXXX (6 digits)' 
      });
    }

    const order = await shopify.order.get(order_id);
    let tags = order.tags ? order.tags.split(', ') : [];
    
    if (!tags.includes(tag)) {
      tags.push(tag);
      await shopify.order.update(order_id, {
        tags: tags.join(', ')
      });
      console.log(`✅ Tag added to order ${order_id}: ${tag}`);
    }

    res.json({
      success: true,
      message: 'Tag added',
      tags: tags
    });

  } catch (error) {
    console.error('❌ Error adding tag:', error);
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// Server starten
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 SKZ Backend running on port ${PORT}`);
  console.log(`📍 Endpoints:`);
  console.log(`   - POST /apps/skz-proxy/save-metafield`);
  console.log(`   - POST /apps/skz-proxy/add-order-tag`);
});
```

**Starten:**
```bash
node server.js
```

---

## Deployment

### Option 1: Heroku

```bash
# Heroku CLI installieren und einloggen
heroku login

# Neue App erstellen
heroku create your-skz-backend

# Environment Variables setzen
heroku config:set SHOPIFY_SHOP_NAME=your-shop.myshopify.com
heroku config:set SHOPIFY_API_KEY=your_api_key
heroku config:set SHOPIFY_API_PASSWORD=your_api_password

# Deployen
git push heroku main

# Logs anzeigen
heroku logs --tail
```

### Option 2: Railway

```bash
# Railway CLI installieren
npm install -g railway

# Einloggen
railway login

# Projekt erstellen
railway init

# Environment Variables setzen
railway variables set SHOPIFY_SHOP_NAME=your-shop.myshopify.com
railway variables set SHOPIFY_API_KEY=your_api_key
railway variables set SHOPIFY_API_PASSWORD=your_api_password

# Deployen
railway up
```

### Option 3: Vercel (Serverless)

1. `vercel.json` erstellen:
```json
{
  "version": 2,
  "builds": [
    {
      "src": "server.js",
      "use": "@vercel/node"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "server.js"
    }
  ]
}
```

2. Deployen:
```bash
vercel
```

---

## Testing

### Lokal testen mit curl:

**1. Save Metafield:**
```bash
curl -X POST http://localhost:3000/apps/skz-proxy/save-metafield \
  -H "Content-Type: application/json" \
  -d '{
    "resource": "order",
    "resource_id": "5123456789",
    "namespace": "custom",
    "key": "skz",
    "value": "123456"
  }'
```

**2. Add Order Tag:**
```bash
curl -X POST http://localhost:3000/apps/skz-proxy/add-order-tag \
  -H "Content-Type: application/json" \
  -d '{
    "order_id": "5123456789",
    "tag": "skz-123456"
  }'
```

### Browser Console testen:

```javascript
// Auf Thank You Page (nach Kauf)
// Browser Console öffnen (F12)

// 1. Test Save Metafield
fetch('/apps/skz-proxy/save-metafield', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    resource: 'order',
    resource_id: window.Shopify.checkout.order_id,
    namespace: 'custom',
    key: 'skz',
    value: '123456'
  })
})
.then(r => r.json())
.then(console.log);

// 2. Test Add Order Tag
fetch('/apps/skz-proxy/add-order-tag', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    order_id: window.Shopify.checkout.order_id,
    tag: 'skz-123456'
  })
})
.then(r => r.json())
.then(console.log);
```

---

## Troubleshooting

### Problem: 404 Not Found

**Ursache:** Backend läuft nicht oder falsche URL

**Lösung:**
- Prüfe ob Backend läuft: `curl http://your-backend.com/health`
- Prüfe URL in `checkout-skz-save.liquid`
- Prüfe Heroku/Railway Logs

### Problem: 401 Unauthorized

**Ursache:** Shopify API Credentials falsch

**Lösung:**
- Prüfe `.env` Variablen
- Erstelle neue Private App in Shopify Admin
- Stelle sicher dass die App Permissions hat:
  - `write_orders`
  - `write_customers`

### Problem: CORS Error

**Ursache:** Cross-Origin Request blockiert

**Lösung:** CORS Headers hinzufügen:

```javascript
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', 'https://your-shop.myshopify.com');
  res.header('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.header('Access-Control-Allow-Headers', 'Content-Type');
  if (req.method === 'OPTIONS') {
    return res.sendStatus(200);
  }
  next();
});
```

### Problem: Rate Limiting

**Ursache:** Zu viele API Requests

**Lösung:** Rate Limiter implementieren:

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 Minute
  max: 10 // Max 10 Requests pro Minute
});

app.use('/apps/skz-proxy/', limiter);
```

---

## Security Best Practices

1. **HTTPS verwenden** (in Production)
2. **API Keys nicht im Code** (.env verwenden)
3. **Input Validierung** (SKZ Format prüfen)
4. **Rate Limiting** implementieren
5. **Logging** für Audit Trail
6. **Error Messages** nicht zu detailliert (keine sensitive Infos)

---

## Monitoring

### Log Format:
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Logging verwenden
logger.info('Metafield saved', { 
  order_id: order_id, 
  skz: value,
  timestamp: new Date().toISOString() 
});
```

### Metrics Dashboard:

```javascript
let metrics = {
  metafields_saved: 0,
  tags_added: 0,
  errors: 0
};

app.get('/metrics', (req, res) => {
  res.json(metrics);
});
```

---

## Nächste Schritte

1. ✅ Backend Setup (einer der Optionen wählen)
2. ✅ API Credentials von Shopify holen
3. ✅ Backend deployen (Heroku/Railway/Vercel)
4. ✅ URL in `checkout-skz-save.liquid` anpassen
5. ✅ Lokal testen
6. ✅ Production Test mit echtem Kauf
7. ✅ Monitoring aktivieren

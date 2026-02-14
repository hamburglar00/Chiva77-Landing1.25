# Landing 1.25 — Chiva77

Landing page con redirección instantánea a WhatsApp + envío de datos a Google Sheets + Meta CAPI (Lead/Purchase) via Apps Script.

---

## Arquitectura

```
Usuario abre la landing
  → Pixel PageView (diferido, no bloquea)
  → Click botón
    → Número seleccionado (aleatorio)
    → Promo code generado (UUID único)
    → Pixel Contact enriquecido (no bloquea)
    → (async) GEO detection + fetch /api/xz3v2q → Google Sheets (no bloquea)
    → Redirect instantáneo a wa.me

Google Sheets (Apps Script):
  → Guarda fila con estado "contact"
  → Recibe acción LEAD → actualiza fila + envía Lead a Meta CAPI
  → Recibe acción PURCHASE → actualiza fila + envía Purchase a Meta CAPI
```

## Archivos

| Archivo | Descripción |
|---------|-------------|
| `index.html` | HTML + CSS crítico inline + Meta Pixel diferido + JavaScript inline |
| `styles.css` | Estilos completos, cargado async |
| `imagenes/` | `fondo.avif`, `logo.png`, `whatsapp.png`, `favicon.png` |
| `api/xz3v2q.js` | Serverless function (Vercel): recibe datos del frontend, extrae IP/UA, reenvía al Apps Script |
| `credenciales/google-sheets.js` | URL del Google Apps Script |

## Funcionalidades

### Todo lo de Landing 1.0

- Meta Pixel diferido (PageView + Contact enriquecido).
- Advanced Matching (em, ph, fn, ln, external_id).
- Selección aleatoria de números (`FIXED_PHONES` + `Math.random()`).
- Promo code único por click.
- Mensaje aleatorio (10 variantes).
- Anti doble-click.
- Critical CSS inline + CSS async + preconnect + preload.
- Accesibilidad (aria-label, :active, :focus-visible, prefers-reduced-motion).

### GEO detection

- Detecta ciudad, región y país del visitante via `https://ipapi.co/json/`.
- Timeout de 900ms — si no responde, continúa sin datos GEO.
- Se ejecuta en background después del redirect (no bloquea nada).

### Envío a Google Sheets (`/api/xz3v2q`)

Al hacer click, se envía un `fetch` con `keepalive:true` (no bloquea el redirect) a `/api/xz3v2q` con:

| Campo | Descripción |
|-------|-------------|
| `event_name` | `"Contact"` |
| `event_id` | UUID para deduplicación con Pixel |
| `external_id` | UUID persistente del visitante |
| `event_source_url` | URL completa de la landing |
| `fbp` / `fbc` | Cookies de Meta (para matching) |
| `email` / `phone` | De query params (`?em=`, `?ph=`) |
| `fn` / `ln` | Nombre y apellido de query params |
| `utm_campaign` | De query param |
| `telefono_asignado` | Número seleccionado |
| `device_type` | mobile / tablet / desktop |
| `promo_code` | Código único del click |
| `geo_city` / `geo_region` / `geo_country` | De ipapi.co |
| `brand` / `mode` / `source` | Metadata de la landing |

### API Vercel: `/api/xz3v2q`

1. Recibe POST con JSON body.
2. Extrae `client_ip` de header `x-forwarded-for` (filtra IPs privadas como 10.x, 172.16-31.x, 192.168.x).
3. Extrae `user-agent` del header.
4. Agrega `timestamp` (ISO 8601).
5. Reenvía todo al Google Apps Script via `fetch POST`.
6. Retorna `{ success: true }` o error.
7. Validación: requiere al menos uno de `event_id`, `external_id`, `phone` o `email`.

### Google Apps Script

El Apps Script es el cerebro del sistema de datos. Recibe POST y actúa según el contenido:

#### Modo A — Contact (landing)

- Trigger: POST sin campo `action`.
- Guarda una nueva fila en la Sheet con todos los datos del visitante.
- Estado inicial: `"contact"`.
- **NO envía CAPI** (el Contact ya fue enviado por el Pixel del browser).

#### Modo B1 — Lead

- Trigger: POST con `action: "LEAD"`.
- Busca la fila por `promo_code`.
- Actualiza estado a `"lead"`.
- Envía evento **Lead** a Meta CAPI con `user_data` hasheado (SHA-256).

#### Modo B2 — Purchase

- Trigger: POST con `action: "PURCHASE"`.
- Busca la fila por `promo_code`.
- Actualiza estado a `"purchase"`.
- Envía evento **Purchase** a Meta CAPI con valor y moneda.
- Detecta recompras (si ya tiene purchase previo).

#### Modo C — Simple Purchase

- Trigger: POST con `phone` + `amount` (sin `action`).
- Crea fila nueva.
- Hereda identidad de fila previa del mismo phone.
- Envía **Purchase** a Meta CAPI.

### CAPI — Datos enviados a Meta

Cada evento Lead/Purchase incluye `user_data` hasheado:

| Campo | Normalización | Hashing |
|-------|---------------|---------|
| `em` | `trim().toLowerCase()` | SHA-256 |
| `ph` | Solo dígitos, prefijo `54` si 10 dígitos | SHA-256 |
| `fn` | `trim().toLowerCase().replace(/\s+/g, " ")` | SHA-256 |
| `ln` | `trim().toLowerCase().replace(/\s+/g, " ")` | SHA-256 |
| `ct` | `normForMeta(value, "ct")` | SHA-256 |
| `st` | `normForMeta(value, "st")` | SHA-256 |
| `zp` | `normForMeta(value, "zp")` | SHA-256 |
| `country` | `normForMeta(value, "country")` | SHA-256 |
| `external_id` | — | SHA-256 |
| `fbp` / `fbc` | — | Sin hashear (Meta lo requiere así) |
| `client_ip_address` | — | Sin hashear |
| `client_user_agent` | — | Sin hashear |

Otros campos del evento:
- `event_id` (deduplicación con Pixel browser)
- `event_time` (Unix timestamp)
- `event_source_url`
- `action_source: "website"`
- `custom_data`: `value`, `currency` (en Purchase)

### fbp y fbc

- `_fbp`: cookie de Meta, se lee y persiste en `localStorage`.
- `_fbc`: cookie de click ID, se construye desde `fbclid` si no existe.
- Ambos se envían al Apps Script para mejorar el matching de CAPI.

## Configuración

### `index.html`

```javascript
const LANDING_CONFIG = {
  BRAND_NAME: "",                   // Nombre de la marca
  MODE: "ads",                      // "ads" o "normal"
  PROMO: {
    ENABLED: true,
    LANDING_TAG: "CT1"              // Prefijo del promo code
  },
  GEO: {
    ENABLED: true,
    PROVIDER_URL: "https://ipapi.co/json/",
    TIMEOUT_MS: 900
  }
};

const FIXED_PHONES = [
  "5493516783675",
  "5493516768842"
];
```

### Pixel ID

En el `<script>` del `<head>`:
- `fbq("init", "880464554785896", {...})`
- `<noscript>` img con el mismo ID.

### `credenciales/google-sheets.js`

```javascript
export const CONFIG_SHEETS = {
  GOOGLE_SHEETS_URL: 'https://script.google.com/macros/s/.../exec'
};
```

### Apps Script

Configurar en el código del Apps Script:
- `PIXEL_ID`: ID del Pixel de Meta.
- `ACCESS_TOKEN`: Token de acceso de la API de Conversiones de Meta.
- `API_VERSION`: Versión de la API (actualmente `v24.0`).

## Deploy

Vercel (con serverless functions).

```bash
git add -A && git commit -m "cambios" && git push
```

**Importante**: Autorizar la URL del deploy en Google Apps Script (Deploy > Manage deployments > Web app > Quién tiene acceso: "Cualquiera").

## Variables de entorno (Vercel)

Ninguna requerida para el frontend. La URL del Apps Script está en `credenciales/google-sheets.js`.

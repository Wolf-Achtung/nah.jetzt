# Briefing: akut.jetzt → Auslieferung über die nah.jetzt-Domains

**Status:** ✅ Umgesetzt (Stand April 2026).

## Finale Architektur

| Domain | Hosting | Inhalt |
|---|---|---|
| `nah.jetzt` | Netlify (Repo `Wolf-Achtung/nah.jetzt`) | Investor-Landingpage + `preview.html` mit Handy-Mockup |
| `app.nah.jetzt` | Railway (Service `NAh-final`) | Notfall-App (FastAPI-Backend + React-Frontend in einem Container) |
| `nah-final-production.up.railway.app` | Railway | 301 → `https://app.nah.jetzt/` (Host-Redirect-Middleware) |
| `akut.jetzt` | (Weiterleitung) | 301 → `https://nah.jetzt/` |

**Zentrale Idee:** `nah.jetzt/preview.html` bettet `app.nah.jetzt/?demo=1` per
iframe ein. Die CSP der App erlaubt Einbettung nur durch `https://nah.jetzt`
(via `frame-ancestors`). Der Demo-Modus deaktiviert echte Notruf-Aktionen.

---

## 1. DNS & Custom Domain

1. In Railway → Service → **Settings → Domains** die Domain `nah.jetzt`
   (und optional `www.nah.jetzt`) als Custom Domain hinzufügen.
2. Beim DNS-Provider von `nah.jetzt` setzen:
   - `CNAME` für `www` → von Railway angezeigter Zielwert
   - Apex (`@`): Railway liefert eine `A`/`ALIAS`/`ANAME`-Anweisung – exakt übernehmen
3. Warten bis SSL-Zertifikat (Let's Encrypt) automatisch ausgestellt ist
   (Status in Railway = grün).
4. **Alte Domain `akut.jetzt`** (falls vorhanden) als zusätzliche Custom Domain
   hinzufügen, **aber nur** für eine 301-Redirect-Regel auf `nah.jetzt` (s. Abschnitt 3).

---

## 2. CSP / iframe-Einbettung erlauben

Aktuell sendet der Service `X-Frame-Options: DENY` (oder `SAMEORIGIN`), deshalb
verweigert der Browser die Einbettung in `nah.jetzt/preview.html`.

**Anpassung im Backend (FastAPI / Reverse-Proxy / Middleware):**

```python
# FastAPI Beispiel – Security-Header-Middleware
@app.middleware("http")
async def security_headers(request, call_next):
    response = await call_next(request)
    # ALT entfernen, falls gesetzt:
    response.headers.pop("X-Frame-Options", None)
    # NEU: nur nah.jetzt darf einbetten
    response.headers["Content-Security-Policy"] = (
        "default-src 'self' https://nah.jetzt; "
        "script-src 'self' 'unsafe-inline' https://nah.jetzt; "
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
        "font-src 'self' https://fonts.gstatic.com; "
        "img-src 'self' data: https:; "
        "connect-src 'self' https://nah.jetzt wss://nah.jetzt; "
        "frame-ancestors 'self' https://nah.jetzt;"
    )
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["Permissions-Policy"] = "camera=(self), microphone=(self), geolocation=(self)"
    return response
```

**Wichtig:**
- `frame-ancestors 'self' https://nah.jetzt` → ersetzt `X-Frame-Options` (moderner, granularer).
- Auf Railway läuft standardmäßig **kein eigener Reverse-Proxy** – die Header aus der FastAPI-Middleware kommen direkt beim Browser an. Du musst nichts Zusätzliches konfigurieren. Falls Du später Caddy/Nginx davorsetzt, dort `X-Frame-Options` und doppelte `Content-Security-Policy`-Header entfernen.
- `connect-src` muss `wss://nah.jetzt` enthalten, falls WebSockets genutzt werden.

---

## 3. Redirects & Domain-Konsolidierung

Damit nichts mehr unter Railway-URL oder alten Domains erreichbar bleibt:

```python
# FastAPI Host-Redirect-Middleware
ALLOWED_HOST = "nah.jetzt"

@app.middleware("http")
async def host_redirect(request, call_next):
    host = request.headers.get("host", "").lower().split(":")[0]
    if host and host != ALLOWED_HOST:
        target = f"https://{ALLOWED_HOST}{request.url.path}"
        if request.url.query:
            target += f"?{request.url.query}"
        return RedirectResponse(target, status_code=301)
    return await call_next(request)
```

Das fängt ab:
- `nah-final-production.up.railway.app/...` → 301 → `nah.jetzt/...`
- `akut.jetzt/...` (falls noch DNS aktiv) → 301 → `nah.jetzt/...`
- `www.nah.jetzt/...` → 301 → `nah.jetzt/...`

---

## 4. CORS

Wenn die App API-Calls vom Browser macht, in der CORS-Config ausschließlich
`https://nah.jetzt` erlauben:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://nah.jetzt"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["*"],
)
```

---

## 5. Rate-Limiting

Ohne Rate-Limiting können einzelne Clients die API überlasten (DoS) oder
Brute-Force-Angriffe auf Auth-Endpunkte fahren. Die Implementierung erfolgt
über `slowapi` (basiert auf `limits`, Redis-kompatibel).

### Installation

```bash
pip install slowapi
```

### Basis-Setup (In-Memory, reicht für Single-Instance)

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

### Globales Limit + Endpunkt-spezifische Limits

```python
# Globales Standard-Limit: 60 Requests pro Minute pro IP
@app.middleware("http")
async def rate_limit_middleware(request, call_next):
    return await call_next(request)

# Auth-Endpunkte stärker begrenzen
@app.post("/api/auth/login")
@limiter.limit("5/minute")
async def login(request: Request, ...):
    ...

# Notfall-Endpunkte großzügiger (lebensrettend – nicht drosseln)
@app.post("/api/emergency/start")
@limiter.limit("30/minute")
async def start_emergency(request: Request, ...):
    ...

# KI-Chat (teuer, daher begrenzen)
@app.post("/api/chat")
@limiter.limit("20/minute")
async def ai_chat(request: Request, ...):
    ...

# Standard-API-Endpunkte
@app.get("/api/{path:path}")
@limiter.limit("60/minute")
async def api_catchall(request: Request, ...):
    ...
```

### Produktion: Redis-Backend (Multi-Instance)

Wenn mehrere Railway-Replicas laufen, braucht der Limiter ein geteiltes
Backend:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri=os.environ.get("REDIS_URL", "memory://"),
    # Alternatives Format: "redis://user:pass@host:6379/0"
)
```

### Response-Header

`slowapi` setzt automatisch `X-RateLimit-Limit`, `X-RateLimit-Remaining`
und `Retry-After` Header – Clients können sich darauf einstellen.

### Wichtige Hinweise

- **Notfall-Endpunkte** niemals zu restriktiv limitieren – im Zweifel lieber
  keinen echten Notruf blockieren.
- **Hinter einem Proxy** (Railway, Caddy, Nginx): `get_remote_address`
  liest die IP aus `X-Forwarded-For`. Sicherstellen, dass der Proxy diesen
  Header korrekt setzt und nicht von Clients fälschbar ist
  (`ProxyFix`-Middleware oder `trusted_hosts`).
- **WebSocket-Verbindungen** separat behandeln – `slowapi` greift nur bei
  HTTP-Requests. Für WS rate-limiting z.B. einen Token-Bucket pro
  Connection im Application-Layer implementieren.

---

## 6. Frontend-seitige Anpassungen

- **Canonical:** `<link rel="canonical" href="https://nah.jetzt/">` im `<head>` setzen.
- **Absolute URLs / API-Base:** Hardcoded `*.railway.app`-URLs durch
  `https://nah.jetzt` (oder relative Pfade) ersetzen.
- **PWA-Manifest:** `start_url`, `scope` und `id` auf `https://nah.jetzt/` setzen.
- **Service-Worker:** Cache leeren / Versions-Bump, sonst halten alte Clients
  noch die Railway-URL.
- **Cookies:** Domain-Attribut auf `nah.jetzt` setzen, `Secure; SameSite=Lax`.

---

## 7. SEO & Indexierung

- `robots.txt` auf der Railway-URL (vor Redirect) ist irrelevant, sobald der 301
  greift – Suchmaschinen folgen automatisch.
- Auf `nah.jetzt` (Investor-Site) bleibt `noindex, nofollow` – die App selbst
  ggf. später indexierbar machen.
- In Google Search Console eine **Adressänderung** von alter Domain → `nah.jetzt`
  hinterlegen, falls `akut.jetzt` vorher indiziert war.

---

## 8. Test-Checklist (vor Go-Live)

- [ ] `curl -I https://nah-final-production.up.railway.app` → `301` nach `nah.jetzt`
- [ ] `curl -I https://akut.jetzt` → `301` nach `nah.jetzt` (falls noch live)
- [ ] `curl -I https://nah.jetzt` → `200` mit Header
      `Content-Security-Policy: ...frame-ancestors 'self' https://nah.jetzt`
- [ ] **Keine** `X-Frame-Options`-Header mehr im Response
- [ ] Aufruf von `https://nah.jetzt/preview.html` zeigt die App **eingebettet** im Handy-Mockup
- [ ] WebSocket-Verbindung funktioniert (`wss://nah.jetzt/...`)
- [ ] Service-Worker registriert sich auf `nah.jetzt`
- [ ] Rate-Limiting: `curl` 6× schnell hintereinander auf `/api/auth/login` → 429 ab dem 6. Request
- [ ] Rate-Limiting: `/api/emergency/start` bleibt erreichbar (30/min Limit)
- [ ] Rate-Limiting: Response enthält `X-RateLimit-Remaining` Header
- [ ] Lighthouse: Performance ≥ 90, Best Practices ≥ 95, A11y ≥ 95
- [ ] Mobile (iOS Safari + Android Chrome) End-to-End-Test

---

## 9. Rollback-Plan

- Railway: alten Service-Snapshot (vor Header-Änderung) deployen.
- DNS: TTL vor Migration auf 300 s setzen, danach wieder 3600 s.
- Backup der bisherigen Header-Middleware behalten.

---

## 10. Sicherer Demo-Modus (`?demo=1`)

Damit Investoren die App in der Live-Vorschau auf `nah.jetzt/preview.html`
ohne echte Notfall-Auslösung bedienen können, soll die App einen
**Demo-Modus** unterstützen:

- Aktivierung über Query-Param `?demo=1` oder Header `X-NAH-Demo: 1`.
- In diesem Modus deaktivieren:
  - echte 112-Wahl (`tel:`-Links → `event.preventDefault()` + Toast „Demo-Modus")
  - Geolocation-Dauerabfragen
  - Push-Notifications-Anforderung
  - Telemetrie / Analytics-Events
  - Bezahl-/Abo-Flows
- Aktivierung sichtbar machen: Badge oben „Demo-Modus".
- State im `sessionStorage` speichern, damit interne Navigation den Modus behält.

```js
const demo = new URLSearchParams(location.search).get('demo') === '1'
          || sessionStorage.getItem('nah_demo') === '1';
if (demo) {
  sessionStorage.setItem('nah_demo', '1');
  document.documentElement.dataset.demo = 'true';
}
```

```css
html[data-demo="true"]::before {
  content: "Demo-Modus";
  position: fixed; top: 8px; left: 50%; transform: translateX(-50%);
  background: #2563eb; color: #fff; padding: 4px 12px;
  border-radius: 999px; font: 600 12px/1 Inter, sans-serif; z-index: 99999;
}
```

---

## 11. Offene Fragen ans Team

1. Ist `akut.jetzt` aktuell registriert/gehostet, oder ausschließlich Code-Name?
2. Welcher Reverse-Proxy steht vor FastAPI (Caddy / Nginx / Railway-Default)?
3. Gibt es bereits eine Umgebungsvariable `ALLOWED_HOST` / `BASE_URL`?
4. Soll `www.nah.jetzt` auf `nah.jetzt` oder umgekehrt redirecten?
5. Müssen Drittanbieter-Embeds (Stripe, Maps, etc.) in der CSP zusätzlich
   freigegeben werden?

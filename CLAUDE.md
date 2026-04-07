# CLAUDE.md – Repo-Konventionen für nah.jetzt

## Projektüberblick
Statische Investor-Relations-Landingpage für **NAH** (Notfall Assistent Hilfe).
Kein Build-Step. Reines HTML/CSS/JS, ausgeliefert von der Domain `nah.jetzt`.
Die eigentliche Notfall-App läuft separat (Briefing siehe `BRIEFING-akut.jetzt.md`).

## Dateien
- `index.html` – Hauptseite (Hero, Problem, Lösung, Markt, Traction, Team, Ask, Kontakt)
- `preview.html` – Live-Demo mit Handy-Mockup (iframe auf nah.jetzt selbst, `?demo=1`)
- `impressum.html`, `datenschutz.html` – Rechtsseiten (Platzhalter, juristisch zu prüfen)
- `styles.css` – einziges Stylesheet, eingebunden mit `?v=N` Cache-Bust
- `og-image.svg` – Open-Graph-Bild 1200×630
- `BRIEFING-akut.jetzt.md` – Migrationsleitfaden für den Railway-Service

## Branches
- Default-Branch: `main`
- Feature-Branches: `claude/<thema>-<5-stellig>` (vom Automationssystem vorgegeben)
- **Niemals** direkt auf `main` pushen. Niemals `--force` auf geteilte Branches.
- PRs nur auf explizite Anforderung des Users erstellen.

## Code-Stil
- Deutsch in UI-Texten und Kommentaren.
- HTML-Indent: 2 Spaces. Semantische Tags (`<main>`, `<section>`, `<nav>`).
- CSS: Custom Properties in `:root`, BEM-light Klassennamen, Mobile-first.
- Keine externen JS-Frameworks. Inline-Scripts klein halten.

## A11y-Pflichten
- Skip-Link, `<main>`-Wrapper, sichtbarer Fokus (`:focus-visible`).
- Mobile-Toggle mit `aria-expanded` synchronisieren.
- Farbkontrast WCAG AA.
- Alt-Texte und sinnvolle Link-Labels.

## SEO/Meta
- `noindex, nofollow` solange Stealth-Phase.
- Canonical, OG, Twitter-Cards bei jeder neuen HTML-Seite mitführen.
- Cache-Bust (`?v=N`) erhöhen, wenn `styles.css` geändert wird.

## Domain-Regeln
- Alles soll **ausschließlich** unter `https://nah.jetzt` erreichbar sein.
- Railway-URL und ggf. `akut.jetzt` per 301 umleiten (siehe Briefing).
- iframe-Einbettung der App nur von `https://nah.jetzt` aus erlauben
  (`Content-Security-Policy: frame-ancestors 'self' https://nah.jetzt`).

## Was nicht reingehört
- Keine Tracking-Cookies, kein Google Analytics.
- Keine fremden CDNs außer Google Fonts und (optional) `api.qrserver.com`.
- Keine echten Personendaten in den Rechtsseiten, solange noch Platzhalter.

## Commit-Stil
- Imperativ, deutsch oder englisch, kurz.
- Eine Commit-Botschaft pro logischer Änderung.

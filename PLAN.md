# UrlaubTracker – Plan

Interaktive, mobile-first Reise-App für die Neuseeland-Reise der Familie Kropp (25.12.2026 – 24.01.2027, mit Stopover in Singapur).
Die Karte steht im Mittelpunkt, dazu kommen ein KI-Chat, im Voraus recherchierte Orte mit Öffnungszeiten, Live-Standort und ein Login nur auf Einladung.

---

## 0. Zuerst die Blocker

### 0.1 Ein Login über das Claude-Pro-Abo ist nicht möglich
Nach meinem Kenntnisstand gibt es keinen OAuth-Flow, mit dem sich eine eigene Webseite über ein Claude-Pro/Max-Abo anmelden und dessen Kontingent nutzen kann.
Die Abo-Tokens sind laut Anthropic-Bedingungen für Claude.ai und Claude Code gedacht. In Drittanbieter-Apps sind sie ausdrücklich nicht erlaubt.
Ein Workaround, etwa mit abgegriffenen Tokens, kann jederzeit brechen oder den Account sperren. Und das genau dann, wenn ihr in Fiordland steht.

**Empfehlung:** Anthropic-API-Key auf dem Server (Console, Prepaid-Guthaben).
Grobe Schätzung bei 4 Personen und ~30 Tagen: Chat mit einem schnellen Modell plus einmalige Recherche liegt im niedrigen zweistelligen Euro-Bereich. Ein Ausgabelimit in der Console verhindert Überraschungen.
Die Nutzer melden sich weiterhin per Mail-Link an, nicht bei Claude.

### 0.2 Gerade in den wichtigsten Gegenden gibt es kein Netz
Doubtful Sound, Milford-/Manapouri-Region, große Teile der Westküste, Takaka Hill und Teile des Tongariro haben **keinen Mobilfunk**.
„Aktuelle Infos“ und der KI-Chat fallen dort aus. Die App muss deshalb **offline-first** sein:
- Reiseplan, recherchierte Orte (POIs) und Öffnungszeiten pro Tag werden lokal gespeichert (Service Worker + IndexedDB).
- Kartenkacheln für die Route werden vorab heruntergeladen (Korridor von ca. 20 km um die Route, Zoom ≤ 14).
- Der Chat zeigt offline klar „Offline – Antwort folgt, sobald Netz da ist“ und stellt die Frage in eine Warteschlange.

### 0.3 Öffnungszeiten an Feiertagen sind im Voraus nicht verlässlich
In Neuseeland liegen während der Reise der 1.1. (Fr) und der 2.1. (Sa). Letzterer wird am **Mo 4.1.** nachgeholt, das ist der Tag der Doubtful-Sound-Abfahrt in Manapouri. Dazu kommen die Feiertags-Wochenenden davor.
Google Places liefert Sonderöffnungszeiten (`currentOpeningHours` / `specialDays`) nur für die **nächsten 7 Tage**. Im Voraus gibt es nur die regulären Wochenzeiten.
**Lösung:** Recherche jetzt mit den Wochenzeiten, dann ein automatischer Refresh jeweils 7 Tage vorher und am Vorabend. Feiertage werden in der UI markiert („Feiertag – Zeiten unsicher“).

### 0.4 Standort auf dem iPhone nur bei geöffneter App
Eine PWA bekommt in Safari den Standort nur, solange sie im Vordergrund läuft. Hintergrund-Tracking gibt es nicht.
Das reicht für „wo bin ich / was steht als Nächstes an“. Für ein Live-Tracking der anderen Familienmitglieder reicht es nicht.

---

## 1. Architektur

```
iPhone (Safari, Homescreen-PWA)
  └─ Next.js 15 (App Router, TypeScript, Tailwind, MapLibre GL)
       │  /api/* (Proxy, gleiche Domain → Cookies ohne CORS-Ärger)
       ▼
  FastAPI (Python 3.12)
   ├─ Auth: Magic Link + Einladungen (Session-Cookie, httpOnly, SameSite=Lax)
   ├─ Trip-API (Stopps, Tage, Buchungen, Soll/Ist)
   ├─ POI-API (Kategorie, Tag, Radius, „jetzt offen“)
   ├─ Chat-API (Claude, Streaming via SSE, Tool-Use)
   ├─ Research-Worker (Batch-Recherche + Refresh-Cronjobs)
   └─ Postgres + PostGIS (oder SQLite+SpatiaLite für den Anfang)
```

| Baustein | Wahl | Warum |
|---|---|---|
| Karte | MapLibre GL JS + Vektorkacheln (OpenFreeMap oder MapTiler) | Kein Google-Maps-Lock-in, Offline-Kacheln möglich, schnell auf dem Handy |
| POIs + Öffnungszeiten | **Google Places API (New)** als Primärquelle, OSM/Overpass als Fallback | OSM `opening_hours` ist im ländlichen NZ lückenhaft, Google ist dort deutlich vollständiger |
| KI | Claude API (Anthropic Python SDK), schnelles Modell für den Chat, stärkeres Modell für die Recherche | Schnelle Antworten im Chat, Qualität bei der Vorab-Recherche |
| Web-Infos live | Claude Web-Search-Tool (serverseitig) | Wetter, Straßensperrungen (NZTA), Fährstatus, Flugstatus |
| Wetter | Open-Meteo (kostenlos, kein Key) | Pro Stopp und Tag |
| Straßen NZ | NZTA/Waka Kotahi Traffic & Travel | Sperrungen sind an der Westküste und am Milford Road häufig |
| Mail | Resend oder SMTP | Magic Links |
| Hosting | Ein kleiner VPS (z. B. Hetzner) mit Docker Compose + Caddy (automatisches HTTPS) | HTTPS ist Pflicht für Geolocation und PWA, ~5 €/Monat |

---

## 2. Datenmodell (Kern)

- `user` (email, name, invited_by, created_at)
- `login_token` (hash, user_id, expires_at, used_at): 15 Minuten gültig, nur einmal nutzbar
- `invite` (email, invited_by, token_hash, expires_at)
- `stay` / `booking` / `day_item`: Seed aus `data/trip.json`
- `route`: Seed aus `data/neuseeland.geojson`
- `poi` (google_place_id, osm_id, name, category, geom, rating, price_level, phone, website, opening_hours_json, last_fetched)
- `poi_day` (poi_id, date, stay_id, open_intervals[], is_holiday_uncertain): **vorberechnet**, d. h. wann der Ort am Tag X offen ist
- `research_note` (stay_id | date, kind, markdown, sources[]): KI-Recherche
- `position` (user_id, geom, accuracy, recorded_at): Live-Standort
- `chat_message` (user_id, role, content, created_at)

---

## 3. Features → Umsetzung

### 3.1 Karte als Zentrum (eine einzige Hauptseite)
- Vollbild-Karte mit Route (Linie), Unterkünften (nummerierte Pins), Buchungen (Uhr-Icon) und KML-Highlights.
- **Bottom Sheet** wie bei Apple Maps, in 3 Höhen: „Heute“ (Timeline des Tages, nächster Termin mit Countdown und Fahrzeit), „Plan“ (alle Tage, Tippen fliegt zur Karte), „Chat“.
- Ein Tipp auf einen Pin öffnet eine Detailkarte im Sheet (Adresse, Check-in, Buchung, „Route in Apple Maps / Google Maps öffnen“).
- Zeitleisten-Schieber: Tag wählen, die Karte zeigt die Lage an diesem Tag. Standard ist „heute“.

### 3.2 „Ich brauche jetzt …“ (Kategorie-Filter)
- Chips oben auf der Karte: 🍽 Restaurant · 🛒 Supermarkt · ⛽ Tankstelle · ☕ Café · 💊 Apotheke · 🏥 Arzt/Notaufnahme · 🚻 WC · 💵 ATM.
- Filtert standardmäßig auf **„jetzt offen“** am aktuellen Standort (oder ersatzweise an der Unterkunft des Tages). Dazu kommen ein Umschalter „heute Abend offen“ und die Sortierung nach Entfernung.
- Daten kommen aus `poi_day`. Damit funktioniert der Filter offline. Online wird zusätzlich ein Live-Check der Öffnungszeit gemacht.
- **Tankstellen sind sicherheitsrelevant.** Zwischen Haast und Franz Josef sowie auf der Strecke Manapouri–Wanaka liegen Tankstellen weit auseinander. Die App warnt: „Letzte Tankstelle vor 120 km Strecke ohne Tankstelle“.

### 3.3 Vorab-Recherche (Research-Worker)
Für jeden Tag und jeden Stopp (Unterkunft + Tagesziele):
1. Google Places Nearby Search pro Kategorie (Radius: Stadt 2 km, ländlich 25 km).
2. Öffnungszeiten auf das konkrete Datum ausrechnen und in `poi_day` speichern.
3. Claude bekommt die Treffer und den Tagesplan und schreibt eine kurze Tagesnotiz: „Einkaufen vor Braemar: Four Square Tekapo, bis 20:00 offen. Am 1.1. geschlossen!“, außerdem Restaurant-Tipps, Wetterrisiken und Feiertagshinweise.
4. Refresh-Cronjobs: T-7 Tage (Sonderöffnungszeiten), T-1 Tag (Wetter, Straßen), täglich 06:00 Ortszeit.

### 3.4 KI-Chat
- Streaming-Antworten (SSE), schnelles Modell, Systemprompt mit kompaktem Reiseplan und „heute / jetzt / Standort“.
- Tools, die Claude aufrufen kann: `get_today`, `search_pois(category, near, open_at)`, `get_weather`, `get_route_eta`, `web_search`.
- Typische Fragen: „Fähre fällt aus, was jetzt?“, „Regen am Hooker Valley – Alternative?“, „Wo ist das nächste offene Restaurant mit Kinderkarte?“.
- Prompt-Caching auf dem statischen Reiseplan (schneller und günstiger).

### 3.5 Standort & Soll/Ist
- Button „Standort teilen“. Solange die App offen ist, wird alle 60 s gesendet.
- Anzeige: „Du bist 34 km vor Wanaka · Plan: Ankunft 17:00 · Nächster Termin: morgen 10:00 Heli Hike“.
- Die anderen eingeloggten Nutzer sehen die letzten Positionen mit Zeitstempel („vor 12 min“).

### 3.6 Singapur-Modul
- Karten-Layer mit MRT-Linien und Stationen. Das Hotel liegt an der Station **Tanjong Pagar** (EW15).
- Spickzettel: Bezahlen per Kontaktlos-Kreditkarte (SimplyGo) direkt am Gate, keine Extrakarte nötig.
- Changi ↔ Hotel: MRT (Grab/Taxi ca. 20–25 min) als Entscheidungshilfe mit 4 Personen + Gepäck.
- Optional: LTA DataMall (Bus-Ankunftszeiten live, kostenloser API-Key).

### 3.7 Login (nur per Mail, nur auf Einladung)
1. Die E-Mail wird eingegeben. Ist sie bekannt, wird ein Link mit Einmal-Token verschickt. Ist sie unbekannt, kommt die **gleiche** Antwort, damit niemand herausfinden kann, wer einen Account hat.
2. Klick auf den Link: Token wird geprüft, eine Session (Cookie, 90 Tage) wird angelegt.
3. Eingeloggte Nutzer können über „Person einladen“ eine Mail mit Einladungslink verschicken. Erst dadurch wird ein Account angelegt.
4. Der erste Account entsteht per CLI-Befehl (`python -m app.cli create-user julian@…`).
5. Alle Seiten und APIs außer `/login` und `/auth/*` verlangen eine Session. `noindex`, kein öffentlicher Inhalt.
6. **iOS-Falle:** Ein Magic Link aus der Mail-App öffnet Safari, nicht die Homescreen-App, und beide haben getrennte Cookies. Lösung: Der Link zeigt zusätzlich einen **6-stelligen Code**, den man in der PWA eintippt.

### 3.8 Handy & Homescreen
- `manifest.webmanifest` (name, icons 192/512, `display: standalone`, theme_color), `apple-touch-icon` 180×180, `apple-mobile-web-app-capable`.
- Safe-Area-Insets (Notch), große Touch-Ziele, Dark Mode automatisch.
- Design: ruhig und elegant. Eine Akzentfarbe (Pazifik-Türkis), viel Weißraum, Inter/SF als Schrift, Glas-Effekt nur auf dem Bottom Sheet.

---

## 4. Meilensteine

| # | Inhalt | Ergebnis |
|---|---|---|
| M1 | Repo-Gerüst, Docker Compose, FastAPI + Next.js, Seed aus `trip.json` | Karte mit Route und Stopps, ohne Login |
| M2 | Magic Link + Einladungen + Code-Login, Session-Guard | Nur Eingeloggte sehen etwas |
| M3 | Bottom Sheet „Heute / Plan“, Tagesauswahl, Standort, Soll/Ist | Kern-Nutzen unterwegs |
| M4 | PWA: Manifest, Icons, Service Worker, Offline-Cache für Daten + Kacheln | Homescreen-App, funktioniert ohne Netz |
| M5 | POI-Recherche (Google Places + OSM), `poi_day`, Kategorie-Chips „jetzt offen“, Tank-Warnung | „Ich brauche jetzt ein Restaurant“ |
| M6 | Claude-Chat mit Tools + Streaming, Research-Notizen pro Tag | KI-Assistent |
| M7 | Singapur-Modul, Wetter, Straßenlage, Refresh-Cronjobs | Aktualität |
| M8 | Feinschliff Design, Test auf echten iPhones, Deployment | Fertig **vor dem 25.12.2026** |

Bis zur Abreise bleiben ca. 11 Wochen. M1–M5 sind das Muss, M6–M7 bringen den Komfort.

---

## 5. Daten im Repo

- `data/trip.json`: Flüge, Mietwagen, Unterkünfte (mit Koordinaten), feste Buchungen, Tagesplan. Aus dem PDF übertragen.
- `data/neuseeland.geojson`: Aus der KML: Highlights, Ausflugslinien und die zwei Fahrtrouten (CHC → Picton, Wellington → AKL), auf jeden 10. Punkt vereinfacht.

Auffälligkeiten im PDF:
- „28.12.2027“ ist ein Tippfehler, gemeint ist 2026.
- Für 3.1., 7.1., 9.–11.1. gibt es im PDF keinen Tagesplan. Die Einträge dort stammen aus der KML und müssen bestätigt werden.
- Die Doubtful-Sound-Buchung steht im PDF mit „1 Personen“. Das sollte beim Veranstalter geprüft werden.

---

## 6. Offene Entscheidungen

1. **Claude-Zugang:** API-Key (empfohlen) statt Pro-Login. Okay?
2. **Google Places:** Ein Google-Cloud-Konto mit Abrechnung ist nötig. Mit Caching bleibt der Verbrauch voraussichtlich im kostenlosen Kontingent. Alternativ nur OSM, dann mit lückenhaften Öffnungszeiten.
3. **Hosting / Domain:** Eigener VPS + Domain vorhanden?
4. **Nutzer:** Nur die 4 Reisenden oder auch Daheimgebliebene (zum Mitverfolgen)?

# Touchpoint-Werkstatt (CampaignCollie)

Persönliches Planungstool für Marketing-Kampagnen: strukturiert, wie Zielgruppen (Segmente) an konkreten Touchpoints angesprochen werden. Eine **einzelne, portable HTML-Datei** (`index.html`) — pures JavaScript, **kein Build-Schritt**, kein Backend.

Dieses Dokument richtet sich an ein Entwicklungsteam, das ein echtes Backend anbindet. Es beschreibt Datenmodell, Persistenz und Aufbau, damit sich daraus ein API-/DB-Schema ableiten lässt.

## Starten / Deployen

- **Lokal:** `index.html` im Browser öffnen (Doppelklick) — läuft ohne Server.
- **Deploy:** Über GitHub Pages; die Datei muss `index.html` heißen. Änderungen: bearbeiten → `git commit` → `git push`.

## Datenmodell

Der gesamte Zustand ist ein **Workspace** mit mehreren Kampagnen:

```
Workspace = {
  version: 3,
  activeCampaignId: string,
  campaigns: Campaign[]
}

Campaign = {
  id, name,
  segments:  Segment[],
  domains:   Domain[],
  ideas:     Idea[],
  // Wert-Dimensionen (Registry = bekannte Namen, auch ohne Karten):
  mechaniks: string[], actions: string[], articles: string[], themes: string[],
  // Metadaten je Wert (Beschreibung), Objekt name -> { desc }:
  mechanikMeta, actionMeta, articleMeta, themeMeta,
  ui: UIState
}

Segment    = { id, name, color, meta }          // meta = Subtext
Domain     = { id, name, touchpoints: Touchpoint[] }
Touchpoint = { id, name }                        // gehört genau zu einer Domain

Idea = {
  id, domainId, tpId, segId,     // Pflicht-Verortung (Domain wird aus tpId abgeleitet)
  title, desc,                   // Freitext
  mech, action, article, theme,  // je ein optionaler Wert (String, "" = leer)
  prio,                          // 0|1|2  -> Label could|should|must (siehe PRIOS)
  status,                        // siehe STATUSES
  timing,                        // null ODER { fy, fw, ty, tw } (ISO-Jahr/KW von–bis)
  createdAt, updatedAt           // Epoch-ms
}
```

**Konstanten** (oben im Script):
- `STATUSES = ["idee","geplant","live","abgeschlossen","depriorisiert"]`
- `PRIOS = ["could","should","must"]` (Index = `idea.prio`)

**Vier gleichwertige Wert-Dimensionen** — `mech` (Mechanik), `action` (Aktion), `article` (Angebot, interner Key `article`), `theme` (Thema):
- Auf der Idee je **ein String** (optional).
- Pro Kampagne eine **Registry** (Liste bekannter Namen, damit auch Werte ohne zugeordnete Karte bestehen) und eine **Meta-Map** (`name -> { desc }`) für Beschreibungen.
- Die anzeigbaren Werte sind die **Vereinigung** aus Registry und den an Karten tatsächlich vorkommenden Werten (`allVals(key)`). Der zentrale Mapping-Punkt ist `VDIMS`.

## Views (Reiter)

- **Matrix:** Kreuztabelle mit **frei wählbaren Achsen** (Segment/Touchpoint/Mechanik/Aktion/Angebot/Thema). Domänen-Filter als Multiselect; Touchpoints sind domänen-scoped. Zellklick öffnet den Karten-Editor (Drawer).
- **Zeitstrahl:** KW-basiert; **Zeilen-Dimension wählbar** (zusätzlich Domäne). Balkenfarbe = Segment. Status-Codierung (voll/gestreift/gestrichelt/gedämpft/ausgegraut).
- **Glossar:** Aggregation je Wert-Dimension; Karten mit Beschreibung + Einsatzorten; Anlegen/Umbenennen/Zusammenführen/Löschen.
- **Zusammenfassung:** read-only Textsicht mit Filtern (Kundengruppe/Mechanik/Aktion/Angebot/Thema/Stichtag) und „Als Text kopieren".
- **Struktur bearbeiten** (in der Matrix): prompt-freies Anlegen/Umbenennen/Löschen von Domänen, Touchpoints und Segmenten (inkl. Segment-Subtext), mit Kaskaden-Löschen.

## Persistenz & Migration

- **Speicher-Schlüssel:** `fp_touchpoint_werkstatt_v1` — **bleibt** trotz Schema-Version 3 (nicht ändern, sonst Datenverlust).
- **Dual-Backend:** In der Claude-Umgebung `window.storage`, sonst automatisch `localStorage` (Erkennung über `hasWS`).
- **Export/Import** als JSON: entweder aktive Kampagne oder gesamter Workspace; Import erkennt das Format automatisch.
- **Migration:** `migrateWorkspace`/`migrateCampaign` sind **idempotent** und heben alte Stände auf `version: 3`. Wesentliche Überführungen: altes `mechanisms[]` (mit Kategorie) → `mechaniks[]` + `mechanikMeta`; `idea.mechs[]` → einzelnes `idea.mech`; einzelnes `ui.domain` → `ui.domains[]`; entferntes Feld `subtitle`. Bestandsdaten laden unverändert.

## Hinweise fürs Backend-Team

- **Sandbox-Besonderheit:** `prompt()`/`confirm()` sind in manchen iframe-Umgebungen blockiert. Alle Bearbeitungen sind daher **prompt-frei** (Inline-Felder, Zwei-Klick-Bestätigung beim Löschen).
- **IDs** sind kurze zufällige Strings (`uid()`), nur **innerhalb einer Kampagne** eindeutig; über Kampagnen hinweg dürfen IDs kollidieren (alles ist über die aktive Kampagne gescopt). Beim Klonen werden Struktur-IDs bewusst beibehalten, nur Ideen-IDs neu vergeben.
- **Domäne der Idee** wird aus `tpId` abgeleitet (`domainOfTp`) — ein Touchpoint gehört immer genau zu einer Domäne.
- **`ui`** ist reiner Ansichtszustand pro Kampagne (Filter/Achsen/gewählte Dimensionen); für ein Backend nicht relevant außer zur UX-Wiederherstellung.
- Die Datei ist bewusst als **eine portable HTML-Datei** gehalten (CSS/JS inline). Für einen Backend-Umbau ist das Datenmodell oben die maßgebliche Referenz.

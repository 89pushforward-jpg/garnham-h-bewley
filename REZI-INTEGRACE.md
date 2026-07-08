# Garnham H Bewley — návrh propojení webu s Rezi (Dezrez)

Stav: NÁVRH k realizaci po zaplacení (£700 položka „Rezi napojení" z nabídky £1400 + £40/měs).
Web preview: https://89pushforward-jpg.github.io/garnham-h-bewley/ (teď statická data v `data.js` — ruční otisk jejich listingů z 8. 7. 2026).

## Co víme (ověřeno)
- Jejich současný web běží na Rezi: fotky listingů servíruje `docs.rezi.cloud`, stránka /property je server-rendered z Rezi dat, interní endpoint `/get/property` (ASP.NET, vyžaduje session).
- Rezi = CRM od **Dezrez**. Má oficiální REST API: **Rezi API** (dokumentace: api.dezrez.com / apidocs — „Rezi API" developer program). Autentizace OAuth2 / API key vydaný k účtu agenta.
- Tým GHB v Rezi denně pracuje (přidávání nemovitostí, ceny, statusy) — TO SE NEMĚNÍ. Web je jen konzument.

## Architektura (produkce)

```
Rezi účet GHB ──(Rezi API, read-only token)──► Cloudflare Worker (cron á 60 min)
                                                    │  GET properties (branch=East Grinstead,
                                                    │  status=ForSale/SSTC, media, rooms, desc)
                                                    ▼
                                              transformace → properties.json + obrázky
                                                    │  (fotky: rezi.cloud URL přímo, žádné kopie)
                                                    ▼
                                         web (statický hosting) čte properties.json
                                         • homepage: 6 nejnovějších
                                         • properties.html: vše FOR SALE (+ SSTC badge)
                                         • property.html: detail vč. floorplanu z media
```

- **Frekvence:** cron každou hodinu (slib z pitche „appears on the website within the hour"). Manuální „refresh now" webhook pro urgentní změny.
- **Enquiries:** formuláře (viewing / valuation) → Resend e-mail na eastgrinstead@garnhamhbewley.co.uk; volitelně zápis leadu zpět do Rezi přes API (fáze 2).
- **Fallback:** když API spadne, web servíruje poslední úspěšný JSON (nikdy prázdná stránka) + alert v AgencyOS GUARDIAN.

## Co potřebujeme od klienta (jednorázově, ~15 minut)
1. Potvrzení u Dezrez podpory / v Rezi adminu: vystavení **API přístupu (read-only)** pro „website feed" — standardní požadavek, Dezrez to dělá běžně pro weby třetích stran.
2. Seznam polí, která chtějí zobrazovat (default: cena, adresa bez čísla popisného, popis, key features, fotky, floorplan, status).
3. Potvrzení domény (garnhamhbewley.co.uk) — přepnutí DNS až po schválení webu.

## Kroky realizace (odhad 1–2 dny práce)
1. Získat API credentials od klienta → otestovat výpis listingů v sandboxu.
2. CF Worker: fetch → mapování na náš datový model (stejný jako dnešní `data.js`, takže **frontend je už hotový** — jen se vymění zdroj).
3. Cron + cache + fallback + alert do AgencyOS.
4. Formuláře na Resend (viewing request s ref. číslem — už dnes generujeme mailto ve stejném formátu).
5. Test proti reálnému účtu, pak DNS switch.

## Pokud by API nebylo dostupné (plán B)
- HTML feed scraper jejich stávajícího Rezi výstupu (stejná technika jako dnešní otisk) na CF Workeru — méně čisté, ale funkční se stejným cronem. Cena i slib se nemění.
- Plán C (nedoporučeno): ruční admin za £1200 — jen kdyby chtěli od Rezi odejít.

## Vazba na nabídku
- £700 web (hotový — v4 preview) · **£700 toto propojení** · £40/měs = hosting, zálohy, běh feedu, změny.
- Prodejní věta: „You keep working in Rezi exactly as today — the website keeps up by itself, within the hour."

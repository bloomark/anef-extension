---
name: query-supabase
description: Low-level data access to the ANEF Supabase dossier_snapshots table. Use /anef-stats for analytical questions instead.
allowed-tools: Bash
---

Query the ANEF Supabase `dossier_snapshots` table via the PostgREST API.

## Connection

- Base URL: `https://okogtnzuuhdwogvdnitm.supabase.co`
- Anon key: `sb_publishable_ZfCqlIUN4Ng5ldOBH9IPkA_5eJRgsJE`
- Table endpoint: `https://okogtnzuuhdwogvdnitm.supabase.co/rest/v1/dossier_snapshots`
- Required headers: `apikey: {KEY}` and `Authorization: Bearer {KEY}`

## Columns

`dossier_hash, statut, etape, phase, date_depot, date_statut, date_entretien, prefecture, domicile_code_postal, lieu_entretien, numero_decret, has_complement, source, created_at, checked_at`

## PostgREST query syntax

Append query params to the table endpoint URL:

| Param | Purpose | Example |
|-------|---------|---------|
| `select=col1,col2` | Pick columns | `select=dossier_hash,statut` |
| `order=col.desc` | Sort | `order=created_at.desc` |
| `limit=N` | Limit rows | `limit=10` |
| `offset=N` | Pagination | `offset=100` |
| `col=eq.value` | Exact match | `statut=eq.controle_a_affecter` |
| `col=like.prefix*` | Prefix match | `dossier_hash=like.91f246*` |
| `col=gte.value` | Greater or equal | `date_statut=gte.2026-04-01` |
| `col=lte.value` | Less or equal | `etape=lte.9` |
| `col=in.(a,b,c)` | In set | `statut=in.(controle_a_affecter,controle_a_effectuer)` |
| Header `Prefer: count=exact` | Total count in `content-range` header | — |

## curl template

```bash
curl -s "https://okogtnzuuhdwogvdnitm.supabase.co/rest/v1/dossier_snapshots?select=COLUMNS&FILTERS&order=created_at.desc&limit=N" \
  -H "apikey: sb_publishable_ZfCqlIUN4Ng5ldOBH9IPkA_5eJRgsJE" \
  -H "Authorization: Bearer sb_publishable_ZfCqlIUN4Ng5ldOBH9IPkA_5eJRgsJE"
```

## User's request

$ARGUMENTS

If no arguments given, fetch the 10 most recent snapshots and display a summary table with: dossier_hash (first 6 chars), statut, etape, prefecture, date_statut, source.

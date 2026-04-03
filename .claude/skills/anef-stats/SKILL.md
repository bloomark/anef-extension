---
name: anef-stats
description: Answer analytical questions about ANEF naturalization dossier data — movements, counts, timelines, transitions, prefectures. Triggers on questions about dossier statistics, step movements, or naturalization progress.
allowed-tools: Bash Skill
---

You are answering an analytical question about French naturalization dossier data tracked in Supabase. Use this domain model to translate the user's question into the right data queries.

## Data access

Fetch data via `/query-supabase`. Pass your specific query as arguments, e.g.:
- `Skill("query-supabase", "fetch all rows where statut=eq.controle_a_affecter and date_statut=eq.2026-04-03")`

The `/query-supabase` skill knows the connection details, PostgREST syntax, and curl template. You provide the what; it provides the how.

## Domain model: Steps and sub-statuses

Users refer to sub-steps as `X.Y` (e.g., "9.1", "9.2"). The mapping is `rang = step*100 + sub`. The `statut` column holds the status code; `etape` holds the major step number.

| Step.Sub | Rang | Status code | Short name |
|----------|------|-------------|------------|
| 1 | 100 | `draft` | Draft |
| 2 | 200 | `dossier_depose` | Filed |
| 3.1 | 301 | `verification_formelle_a_traiter` | Formal check queued |
| 3.2 | 302 | `verification_formelle_en_cours` | Formal check in progress |
| 3.3 | 303 | `verification_formelle_mise_en_demeure` | Formal notice issued |
| 3.4 | 304 | `css_mise_en_demeure_a_affecter` | Dismissal pending |
| 3.5 | 305 | `css_mise_en_demeure_a_rediger` | Dismissal drafting |
| 4 | 400 | `instruction_a_affecter` | Awaiting assignment |
| 5.1 | 501 | `instruction_recepisse_completude_a_envoyer` | Under review |
| 5.2 | 502 | `instruction_recepisse_completude_a_envoyer_retour_complement_a_traiter` | Supplement returned |
| 6.1 | 601 | `instruction_date_ea_a_fixer` | Interview to schedule |
| 6.2 | 602 | `ea_demande_report_ea` | Interview postponement |
| 7.1 | 701 | `ea_en_attente_ea` | Awaiting interview |
| 7.2 | 702 | `ea_crea_a_valider` | Interview report pending |
| 8.1 | 801 | `prop_decision_pref_a_effectuer` | Prefecture review |
| 8.2 | 802 | `prop_decision_pref_en_attente_retour_hierarchique` | Hierarchy validation |
| 8.3 | 803 | `prop_decision_pref_prop_a_editer` | Proposal drafting |
| 8.4 | 804 | `prop_decision_pref_en_attente_retour_signataire` | Awaiting prefect signature |
| 9.1 | 901 | `controle_a_affecter` | SDANF queued |
| 9.2 | 902 | `controle_a_effectuer` | SDANF review |
| 9.3 | 903 | `controle_en_attente_pec` | Sent to SCEC |
| 9.4 | 904 | `controle_pec_a_faire` | SCEC verification |
| 10.1 | 1001 | `controle_transmise_pour_decret` | Favorable, sent for decree |
| 10.2 | 1002 | `controle_en_attente_retour_hierarchique` | Ministry hierarchy |
| 10.3 | 1003 | `controle_decision_a_editer` | Decision drafting |
| 10.4 | 1004 | `controle_en_attente_signature` | Awaiting minister signature |
| 10.5 | 1005 | `transmis_a_ac` | Sent to central admin |
| 10.6 | 1006 | `a_verifier_avant_insertion_decret` | Pre-insertion check |
| 10.7 | 1007 | `prete_pour_insertion_decret` | Ready for decree |
| 10.8 | 1008 | `decret_en_preparation` | Decree in preparation |
| 10.9 | 1009 | `decret_a_qualifier` | Decree qualifying |
| 10.10 | 1010 | `decret_en_validation` | Decree in validation |
| 11.1 | 1101 | `inseree_dans_decret` | In signed decree |
| 11.2 | 1102 | `decret_envoye_prefecture` | Decree sent to prefecture |
| 11.3 | 1103 | `notification_envoyee` | Notification sent |
| 12.1 | 1201 | `decret_naturalisation_publie` | Naturalized (JO published) |
| 12.2 | 1202 | `decret_naturalisation_publie_jo` | Naturalized (JO) |
| 12.3 | 1203 | `decret_publie` | Decree published |
| 12.4 | 1204 | `demande_traitee` | Fully processed |
| 12.5 | 1205 | `decision_negative_en_delais_recours` | Negative, appeal open |
| 12.6 | 1206 | `decision_notifiee` | Decision notified |
| 12.7 | 1207 | `demande_en_cours_rapo` | RAPO appeal in progress |
| 12.8 | 1208 | `controle_demande_notifiee` | Control decision notified |
| 12.9 | 1209 | `irrecevabilite_manifeste` | Inadmissible |
| 12.10 | 1210 | `irrecevabilite_manifeste_en_delais_recours` | Inadmissible, appeal open |
| 12.11 | 1211 | `css_en_delais_recours` | Dismissed, appeal open |
| 12.12 | 1212 | `css_notifie` | Dismissal notified |

## Positive outcomes
`decret_naturalisation_publie`, `decret_naturalisation_publie_jo`, `decret_publie`, `demande_traitee`

## Negative outcomes
Any status containing `negative`, `irrecevabilite`, or starting with `css_`

## Finished dossiers
Step >= 11 (except `demande_en_cours_rapo` which is an active appeal), or any negative outcome.

## Common analytical patterns

### 1. "How many at step X?" / "Count by status"
Single query: filter by `etape` or `statut`, use `Prefer: count=exact` header.

### 2. "Who moved from X.Y to X.Z?" / Transition detection
This requires a **two-step** query:
1. Find dossiers at the **target** status with the relevant date filter
2. For each result, fetch their **full history** (all snapshots for that `dossier_hash`, ordered by `created_at.asc`)
3. Confirm the prior snapshot was at the **source** status

### 3. "What happened today?" / Recent activity
Filter by `date_statut=eq.YYYY-MM-DD` or `created_at=gte.YYYY-MM-DDT00:00:00` for today's date.

### 4. "Average time at step X"
Fetch snapshots at the given step, compute `date_statut - date_depot` for each, aggregate.

### 5. "Show dossier XYZ"
Filter by `dossier_hash=like.PREFIX*`, fetch all snapshots ordered by `created_at.asc` to show full timeline.

### 6. "Prefecture breakdown"
Fetch all (or filtered) snapshots, group by `prefecture` in your analysis.

## Presentation

- Always show `dossier_hash` truncated to first 6 chars
- Use markdown tables for structured results
- For transitions, show before/after status with dates
- Include total counts in summary

## User's request

$ARGUMENTS

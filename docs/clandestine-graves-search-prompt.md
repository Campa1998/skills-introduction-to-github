# Prompt for collecting news on clandestine graves in Mexico

This guide is a reusable prompt-and-workflow template for building a structured dataset from news about clandestine graves, mass graves, and human-remains discoveries in Mexico. It is designed for AI tools with web-search or agent mode and focuses on the main failure mode of naive scraping: repeated coverage of the same discovery event.

> Terminology note: use **clandestine graves**, **mass graves**, **fosas clandestinas**, **fosas comunes**, **hallazgo de restos humanos**, and **cuerpos encontrados** in searches. If speech transcription writes “grapes,” correct it to “graves.”

## Core idea

Treat the **physical discovery event** as the unit of analysis, not the article. A viral discovery may produce several headlines over several days, but it should become one dataset row when the articles refer to the same location and discovery window.

Use the AI agent as a search-and-triage assistant:

1. Search a short date window.
2. Extract candidate articles.
3. Cluster duplicate articles into unique discovery events.
4. Return only machine-readable rows.
5. Repeat by date window.
6. Run a final cross-window deduplication pass.

## System or role prompt

Paste this first if the AI tool lets you set a system, developer, or project instruction:

```text
You are a research assistant specialized in Mexican human-rights, public-security, and forensic documentation. You are rigorous, skeptical, and careful with uncertainty. You distinguish between article publication dates, official report dates, and physical discovery dates. You never count multiple media articles as multiple discovery events unless the underlying physical discovery is different.

Your output must be structured data only. Do not write narrative summaries unless explicitly asked. Prefer Spanish-language primary or local sources when available, but use national and English-language sources as supplements. Do not invent missing values. Use null when a field is unknown.
```

## Main search prompt

Replace the bracketed values before each run.

```text
Search the web for news, official statements, and local reports about clandestine graves, mass graves, or human-remains discoveries in Mexico between [DATE_FROM] and [DATE_TO].

Search in Spanish and English. Use query variants including, but not limited to:
- "fosa clandestina" "[MONTH YEAR]" México
- "fosas clandestinas" "[STATE]" "[MONTH YEAR]"
- "hallazgo de restos humanos" "[STATE]"
- "hallaron restos humanos" "[MUNICIPALITY OR STATE]"
- "cuerpos encontrados" "fosa" "[STATE]"
- "colectivo buscadoras" "hallazgo" "fosa"
- "Fiscalía" "fosa clandestina" "[STATE]"
- "mass grave" Mexico "[MONTH YEAR]"

Source priorities:
1. State Fiscalía / prosecutor offices, official search commissions, CNB or government statements.
2. Local and state media near the reported location.
3. National Mexican media such as Animal Político, El Universal, Milenio, Proceso, La Jornada, SinEmbargo, Reforma, Aristegui Noticias, and Excélsior.
4. International sources only as supporting evidence.

Critical deduplication rule:
Multiple articles may describe the same physical discovery. Before creating a row, ask: do these articles refer to the same approximate location and the same discovery window? If the location is the same municipality or within about 5 km, and the discovery/reporting dates are within ±3 days, cluster them into one event row. Merge sources. Use the most specific and most recent body/remains count, but preserve the initial count when reported.

Important date rule:
Use the discovery date, not the article publication date, whenever possible. If only the publication date is known, use that date and set date_certainty to "estimated" with an explanation in notes.

For each unique discovery event, return exactly one JSON object with these fields:

- event_id: a short stable ID you create, formatted MX-[STATE_ABBREV]-[YYYYMMDD]-[MUNICIPALITY_SLUG]-[NUMBER]
- discovery_date: ISO date YYYY-MM-DD, or null if impossible to estimate
- date_certainty: "exact", "approximate", or "estimated"
- publication_date_range: earliest and latest article dates found, formatted "YYYY-MM-DD/YYYY-MM-DD", or null
- state: Mexican state name
- municipality: municipality name, or null
- locality_or_neighborhood: locality, colonia, ejido, ranch, road, or area name, or null
- location_detail: most precise textual location description available
- coordinates: latitude/longitude if explicitly reported or safely geocodable from a named place; otherwise null
- location_certainty: "exact", "approximate", "municipality_only", or "unknown"
- bodies_initial: first reported count of bodies, complete skeletons, or victims; integer or null
- bodies_updated: later or final reported count if different; integer or null
- remains_description: describe whether reports mention bodies, skeletal remains, bone fragments, bags, graves, cremated remains, or unidentified remains
- number_of_graves_or_pits: count of graves/pits if reported; integer or null
- victim_profile: age, sex, identity, minors, women, migrants, or other reported profile details; otherwise null
- discovery_context: who found it, such as buscadoras collective, relatives, authorities, construction workers, anonymous report, accidental discovery, or unknown
- reporting_authority: Fiscalía, commission, police, collective, or other source that confirmed the discovery; otherwise null
- alleged_perpetrator_or_context: cartel, organized-crime context, linked case, or null
- sources: array of source objects, each with outlet, title, url, and publication_date; include up to 5 strongest sources
- duplicate_headlines_considered: brief list of additional repeated headlines or outlets merged into this event, or []
- confidence: "high", "medium", or "low"
- notes: short note explaining uncertainty, count changes, or why duplicate articles were merged

Return only a valid JSON array. No markdown, no prose, no code fences. If no events are found for the date window, return [].
```

## Final cross-window deduplication prompt

After running several date windows, paste the combined JSON into a new prompt:

```text
I will provide a combined JSON array of candidate clandestine-grave discovery events from multiple search windows. Deduplicate it by physical discovery event.

Merge records if they refer to the same approximate location and discovery window, even if article dates differ. Prefer the row with the most precise location, strongest sources, and clearest body/remains counts. Preserve all non-duplicate sources up to 8 sources per merged event. Keep bodies_initial as the earliest count and bodies_updated as the highest later confirmed count when reports changed over time.

Return only the cleaned JSON array with the same schema. Do not add prose.

INPUT_JSON:
[PASTE JSON HERE]
```

## Recommended batching strategy

For broad coverage, run small windows instead of one large search:

| Period density | Suggested window size | Why |
| --- | ---: | --- |
| Very dense state or crisis period | 1-3 days | Reduces missed local articles and improves event separation. |
| Normal national sweep | 7 days | Good balance between coverage and API/search cost. |
| Historical low-density period | 14 days | Faster, but more likely to miss small local reports. |
| Final dedupe | Entire collected period | Catches events split across adjacent windows. |

A practical February-to-March run might use weekly windows, then a final dedupe pass over all returned rows.

## Quality-control checks

Before trusting the dataset, ask the AI or run your own script to flag:

- Rows with the same state, municipality, and discovery_date.
- Rows in the same municipality within ±3 days.
- Rows where `bodies_updated` is lower than `bodies_initial`.
- Rows with only national media and no local/official source.
- Rows where the source publication date is outside the requested period but the discovery date is inside it.
- Rows with vague locations such as only “Mexico” or only a state name.

## Safer interpretation rules

- Do not treat “restos óseos,” “fragmentos,” and “bodies” as identical unless the source gives a victim count.
- If a source says “several bags with remains,” do not infer number of bodies.
- If a later source updates the count, keep both the initial and updated counts.
- If a report mentions a search operation without a confirmed grave or remains discovery, exclude it or set confidence to low with a clear note.
- If an article reports exhumations from a legal cemetery or morgue unless it clearly involves clandestine burial, do not classify it as a clandestine-grave event.

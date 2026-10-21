# Competitive Intelligence Pipeline Specification (Sanitized)

This is a sanitized description of the agent specification behind the pipeline. Organization-specific details have been generalized or removed, including names, competitor lists, product pillar definitions, scoring weights and thresholds, discovery keywords, workspace identifiers, and file paths.

## Operating model

- **Trigger queue.** Every run starts as a row in a Notion queue with a status (idle, requested, running, complete, failed), a trigger source (button, scheduled, mobile dispatch, manual), and a lookback window that defaults to seven days.
- **Watcher.** A PowerShell script polls the queue every 60 seconds. When it finds a requested row, it marks the row as running, sets the environment, launches Claude Code with the agent specification, and on completion marks the row complete and records the runtime.
- **Locking.** Each pipeline on the host has its own lock file, scheduled task, and working directory, so runs cannot overlap and two pipelines share no state.
- **Self-recovery.** If the first poll after login fails because the network is not up yet, the next poll recovers on its own.
- **Principles.** The agent flags gaps instead of guessing, never fabricates citations, and treats a missing source as a reason to drop a claim.

## Data model

**Competitors database.** One record per competitor, with identity (name, category, status, threat level), company facts (headquarters, founding year, public or private, headcount, funding, last round), a two-sentence product summary, key differentiators, a counter-positioning note, name aliases for matching against filings and trial sponsors, overlap vectors, total score, funding tier, last-updated date, and source URLs.

**Status values:** Active, Watching, Archived, Unverified.

**Run events database.** One row per material event, with headline, event date (not pull date), competitor relation, event type, significance, source tier, source URL, a corroboration URL that is required for lower-tier sources at high significance, a summary of three sentences or fewer, and the run ID.

**Event types:** funding, hire, departure, product launch, regulatory, IP filing, partnership, earnings, press, trial update, acquisition.

## Source tier policy

| Tier | Description | Rule |
|---|---|---|
| T1 | Authoritative filings and government databases | Primary source, no corroboration needed |
| T2 | Verified databases | Primary for funding and publications |
| T3 | Trade press | Needs T1 or T2 corroboration for high significance |
| T4 | Company self-reported | Flagged as self-reported in the summary |
| T5 | Aggregator or directional | Never the sole source for an event |

**Forbidden:** forums, social posts without provenance, content farms, and SEO spam.

## Stages

| # | Stage | Sources | Output |
|---|---|---|---|
| 1 | Regulatory filings | Public-company filings, mapped by item code | Partnership, earnings, leadership, and press events |
| 2 | Trade press | Life-sciences and health trade press feeds, filtered by competitor name and aliases | Press events |
| 3 | Company newsroom diff | Each competitor's press page, compared against its stored snapshot | Changes above a size threshold, flagged for manual review as possible launches |
| 4 | Patent filings | Patent database, last 90 days, by assignee | IP events |
| 5 | Clinical trials | Trial registry, sponsor match using aliases | Trial update events on phase, status, or completion changes |
| 6 | FDA data | Drug approvals, device clearances, adverse event reports | Regulatory events. Adverse event data is historical context, low significance by default |
| 7 | Publications | Author affiliation search in biomedical literature databases | Signals of research direction |
| 8 | Conference abstracts | Major industry conferences, run only in conference season months | Research and product signals |
| 9 | Funding signals | Trade press plus news feeds per competitor | Funding events. Missing amounts are queued for manual review |
| 10 | Headcount signals | Search against public company profile pages | Directional only (T5) |
| 11 | Software review sites | Review count and rating, compared against a snapshot, for software competitors only | Review change events |
| 12 | Discovery scan | Keyword clusters across trade press and news, plus adjacent sectors | New entities, written as Unverified for human review |

After the 12 stages, a scoring stage computes a threat score for every tracked competitor.

## Significance scoring

Each event's significance is a weighted sum. Source tier adjusts the score, with more authoritative sources adding and self-reported sources subtracting. Event factors add to it: a large funding round, a leadership change, a regulatory approval, a patent grant, or a partnership with a major health system or pharma company. The total maps to High, Medium, or Low. Point values are omitted.

## Delta detection

After each run, a JSON snapshot is saved per competitor. The next run compares against it.

| Field | Change | Event type |
|---|---|---|
| Total funding | Any change | Funding |
| Leader | Any change | Hire or departure |
| Headcount | Beyond month-over-month and quarter-over-quarter thresholds | Hire or departure |
| Patent IDs | Any addition | IP filing |
| Trial IDs or status | Any new ID or status change | Trial update |
| Press items | Any new item | Press, retyped if regulatory or funding language matches |
| Newsroom page | Change above a size threshold | Product launch, flagged for manual review |
| Regulatory filings | Any new filing | Earnings, departure, or launch, by item code |

## Threat scoring

The scoring stage computes a total threat score for every competitor with a status of Active, Watching, or Unverified.

- **Pillar vectors.** Five vectors, one per product pillar of the sponsoring company. Each is a binary yes or no check against public product descriptions, using defined keyword and phrase criteria, and each carries a weight. Pillars that are hard to detect use proxy phrases.
- **Market momentum modifier.** A tiered modifier based on funding stage and customer footprint, so funding signal contributes without dominating the score.
- **Threat level mapping.** The total maps to High, Medium, or Low. High means an immediate deep dive with weekly tracking. Medium means monthly monitoring of product updates and funding. Low means a watchlist with a quarterly check.
- **Never blank.** Every scored competitor always has a threat level.

Pillar definitions, keywords, weights, and thresholds are omitted because they encode the sponsor's product strategy.

## Discovery and promotion

The discovery stage has a high false-positive rate, so it never promotes anything. A new company that does not match an existing record by name or alias is written as Unverified and posted to Slack with source links. A person sets its status to Watching or Archived. Watching entries receive the full pipeline on the next run. Archived entries stay in the database for deduplication and are skipped in future pulls.

## Outputs

Each run writes five artifacts, in this order:

1. **Local Markdown backup**, written first so the output exists even if the Notion API is unavailable.
2. **Competitor record updates** in Notion, including score, funding tier, threat level, and counter-positioning.
3. **Run event rows**, one per material event.
4. **Run summary page** with run metadata, top net-new events, funding moves, product launches, talent moves, regulatory and IP events, new entrants, and strategic implications.
5. **Slack notification** with the run ID, event count, discovered competitor count, stage error summary, and a link to the summary page.

The competitors database is presented as a gallery grouped by threat level and sorted by total score, so an executive can read it without knowing how it was built.

## Triggers and scheduling

Four trigger surfaces write to the same queue: a Notion button, a weekly scheduled task early on Monday morning, a mobile dispatch task, and a chat message. The pipeline runs unattended on a Windows host, and a scheduled run that was missed is caught up when the machine is next available.

## Operational notes

- The Notion integration must be explicitly connected to each database. Access on a parent page does not propagate to child databases created through the API.
- Sponsor names vary across registries, so each competitor carries a list of aliases used for matching.
- Funding amounts are often missing from press coverage, so a missing amount is queued for manual review instead of being estimated.

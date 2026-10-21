# Pipeline Specification (Sanitized)

This is a sanitized description of the agent specification behind the pipeline. Organization-specific details have been generalized or removed, including names, customers, pricing, contacts, signal definitions, scoring weights, and thresholds.

## Role and operating principles

- The agent runs a fixed pipeline once per run and produces a structured deliverable. It does not editorialize outside the playbook stage.
- When evidence is insufficient, it flags the gap. It does not guess and does not fabricate citations.
- Accounts are ranked by total agency budget from the most recently published fiscal-year document. Where the unit is a line item inside a larger agency budget, the line item is used if available. Otherwise the parent budget is used and the account is flagged as an estimate.
- Ranking, contact discovery, signal collection, and scoring stay in separate stages.
- Free public sources and trade press only. No paid data brokers or paid sources of any kind.
- No action is recommended on weak evidence. Gaps are marked so a human knows where to invest manual research time.

## Source tier policy

| Tier | Description | Examples |
|---|---|---|
| T1 | Authoritative | Government budget and financial reports, oversight body records, federal filings and grant databases, procurement portals, agency websites |
| T2 | Verified | Established trade press, professional association newsletters and rosters |
| T3 | Regional or social | Regional newspapers, staff social posts, agency social pages |
| T4 | Self-reported | Vendor case studies, agency press releases |
| T5 | Aggregator | Third-party directories, news aggregators |

**Forbidden:** forums, social posts without provenance, content farms, and paid data brokers.

**Rules:**

- Every signal carries a primary source URL.
- A T3 signal is promoted to high significance only with T1 or T2 corroboration.
- A signal with a missing or unverifiable citation is dropped, not logged.
- A signal older than 18 months is not current, unless it is a multi-year contract whose renewal window is approaching.

## Stages

| # | Stage | Sources | Output |
|---|---|---|---|
| 1 | Universe build | Federal registry, professional directories, state oversight rosters | Every primary agency in the territory, written with a Discovery status |
| 2 | Budget intelligence | Budget and annual financial reports, disbursement reports, agency annual reports | Budget ranking. The top 120 are promoted to Active |
| 3 | Decision-maker intelligence | Agency staff pages, oversight rosters, association rosters | A named decision-maker per account. A name is written only with an official directory page and a corroborating recent news mention |
| 4 | Trade press signals | Trade publications, last 18 months | Dated signals |
| 5 | RFP and procurement | State procurement portals, free bid notifications | Open solicitation signals |
| 6 | Grant funding | Federal and state grant award databases, last 24 months | Funding signals |
| 7 | Oversight board and legislative | Board minutes, annual reports, relevant bills | Policy and milestone signals |
| 8 | Leadership change | Staff profiles, press releases, commission minutes | Flags for leaders hired in the last 12 months |
| 9 | Operational stress | Regional news, local broadcast sites, commission minutes, last 12 months | Stress signals, corroborated before high significance |
| 10 | Vendor incumbency | Procurement contract records, award databases, vendor case studies | Incumbent vendors. Marked Unknown where not findable, then flagged for manual follow-up |
| 11 | Conference and events | Public conference agendas and calendars, next 6 months | Upcoming public appearances |
| 12 | Discovery scan | Trade press, grant databases, oversight minutes | Newly formed agencies, logged for human review and never auto-promoted |
| Gate | Buying-window gate | Linked signal rows | Hot, warm, or cold per account |
| 13 | Deterministic scoring | Account facts, dated signals, gate output | Score and tier, after verification |

**Search efficiency:** queries run at the territory level first, since most press does not name an agency in a searchable way. Named-account queries, including known name variants, run only for the working list of top-tier and hot accounts.

## Buying-window gate

The gate runs per account before scoring. It reads the dated signal rows linked to the account and applies priority rules. Any one of several recent high-urgency signals makes the window Hot. Otherwise, recent funding or milestone signals make it Warm. Otherwise it is Cold. Hot always wins.

The derived window is written to the account record and passed to scoring as a fixed input. This keeps the window tied to current signal dates, not stale stored values from earlier runs.

## Scoring and verification

For each active account, the pipeline makes one call to the Anthropic API.

- **Prompt:** fit signal definitions, modifier definitions, tier thresholds, account facts, all signals collected in the run with dates and significance, and the window derived by the gate.
- **Settings:** a pinned model, low temperature (0.2), JSON-only output, and a 500 ms delay between accounts.
- **Expected output:** five boolean fit flags, vector points, a momentum modifier, the buying window, a total score, and a tier.

**Verification runs immediately after each response is parsed:**

1. Recalculate the total as vector points plus the momentum modifier.
2. Derive the tier mechanically from the recalculated total.
3. If the model's total or tier differs from the mechanical values, log a score mismatch with both sets of values.
4. In every case, overwrite the stored total and tier with the mechanical values. The model's tier is never written.

**Failure handling:** if a call fails or the output cannot be parsed, no scoring fields are written, the existing values are preserved, and the failure is logged. A fallback or zeroed score is never written over an existing one.

## Error handling

| Condition | Behavior |
|---|---|
| Access denied | Skip the source, log it, do not retry |
| Rate limited | Wait 30 seconds, retry once, then skip |
| Not found | Log and skip |
| DNS failure | Log and skip |
| Bot detection | Skip the source, log it, never attempt to bypass |
| Stage failure | Log it and continue to the next stage. Never abort the full run |

A backup is written before any database write.

## Outputs

- **Accounts database:** one record per account with status, budget, decision-maker, buying window, score, and tier.
- **Signals database:** one row per signal with source URL, source tier, date, and significance.
- **Playbook per account:** three short paragraphs. The first covers pain points found in public sources, in two to four sentences, or states that none were detected. The second covers positioning, referencing only shipped features. The third gives one concrete next move, categorized as cold call, warm email, conference handshake, referral path, or wait, and tied to the specific signal that justifies it. Playbook text never includes scores or tier letters.
- **Playbook staleness check:** after scoring, any account whose score changed by more than one point has its playbook regenerated.
- **Run summary page:** run metadata, tier distribution, hot accounts, signals by type, leadership changes, vendor map, discovery entries, and stage errors.
- **Slack message:** run ID, mode, counts, stage errors, and a link to the run summary.

## Scheduling

GitHub Actions workflows:

- **Biweekly:** refresh signals for the working list.
- **Monthly:** decision-maker discovery for unknown accounts, then a full sweep of all active accounts.
- **Quarterly:** verify the universe, re-rank by budget, and re-score.
- **Manual dispatch:** ad hoc runs, including account-scoped passes.

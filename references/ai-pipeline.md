# From events to SOPs **[D]** — nothing in this file was built in the source

Three batch stages reading from the warehouse, not the operational database. This is a
contract and a set of decisions; there is no code, because none was proven and the
provider layer is a host seam.

```
events + routine entries ─▶ 1 narrate ─▶ 2 mine branches ─▶ 3 write SOP ─▶ Markdown / wiki / Word
   (warehouse)                 per event      per person-week     per role
```

## Stages

| Stage | In → out | Volume | Model tier |
|---|---|---|---|
| 1 Narrate | one event → one sentence: "Set *Return reason* to Damaged. Server accepted." | very high | cheapest text model; **vision-capable mid-tier only for `dom_confidence = 'low'`** with the screenshot crop |
| 2 Mine branches | one person's narratives across many sessions → steps, decision points, exceptions: "If the order is marketplace, refund in the seller panel; otherwise in the shop admin" | medium | mid-tier, long context — a person-week in one call |
| 3 Write the SOP | branch tree + gap-form entries + selected screenshots → a document | low | strongest model; runs once per SOP, quality over cost |

The source named specific model ids and a gateway **[D]**. Both date quickly. Keep the
*tiering rule* and take the ids from the host's AI configuration: route through one
gateway with a fallback per stage, and pin a dated model only if compliance needs
reproducible output.

## Routing on `dom_confidence`

| Confidence | Stage 1 path |
|---|---|
| `high` | text only |
| `medium` | text only; a per-deployment setting may upgrade it |
| `low` | multi-modal, crop from `bounding_box` + `viewport` — unless screenshot mode is `metadata_only`, in which case narrate as "changed an unlabelled field" and move on |

The cheap path must stay the dominant one. If more than roughly a third of events on a
target app are `low`, fix label resolution for that app (a per-host resolver) before
paying a vision model to compensate.

A heartbeat with `dropped_count > 0` marks the surrounding window as **sampled**. Stage
2 must not read a gap there as inactivity.

## Where gap-form entries enter

- Stage 2 receives them as unordered steps tagged with category, tool and frequency. They explain jumps the capture cannot: an order goes on hold, nothing happens in the browser for ten minutes (a phone call), then a refund is issued.
- Stage 3 uses `outcome` verbatim where present.
- `linked_event_ids`, where set, anchors an entry to a point in a session.

## Embeddings

Cluster recurring micro-actions (log in, search, save, dismiss error) so stage 2 can
collapse them. Store vectors **in the same Postgres via pgvector**, tenant-scoped under
the same RLS. A second data plane for vectors is a second place for captured behaviour
to leak from. Index choice (HNSW vs IVFFlat), model and dimensions were left open in
the source — decide from real query patterns.

## SOP output

One document per **role or process**, not per named employee, wherever possible. A
per-person SOP is a behavioural profile; merging three people's branches into "how
returns are handled" is more useful and far less sensitive. **[A]**

Sections: purpose · systems touched · happy path, numbered · decision points · exceptions
by observed error · offline steps · open questions where people diverge.

Stage 3 draws the highlight box from `bounding_box` onto a screenshot. In
`metadata_only` deployments the SOP is text-only.

Export formats: Markdown is universal and lossy; a wiki API is rich and needs
per-customer auth; Word is what gets emailed. The source left the primary open.

## Guardrails **[A]**

- **No raw values in prompts past stage 1.** Stage 2 and 3 work on narratives, which were produced from already-scrubbed events.
- **The model proposes; a person approves.** A generated SOP is a draft until someone who does the job signs it off. An SOP that misdescribes a refund step and is followed is worse than none.
- **List every provider as a sub-processor** in the DPIA — event text, and in `event_triggered` mode, images of back-office screens.
- **Cost ceiling per tenant per day**, enforced in the gateway. A mutation storm that slips the capture budget should not become an invoice.
- **Deleting a person's data must reach the warehouse, the vectors, the screenshots and any SOP derived solely from them.** Design that path before the first SOP exists.

## Open questions inherited from the source

Primary export format · pgvector index strategy · whether hot retention can drop below a
day · per-tenant provider pinning versus gateway default.

## Checklist

- [ ] Stages read from the warehouse
- [ ] Low-confidence share measured per target app before enabling vision
- [ ] Heartbeats respected as sampling markers
- [ ] SOPs per role by default
- [ ] Human sign-off before an SOP is published
- [ ] Erasure path covers warehouse, vectors, images and derived documents

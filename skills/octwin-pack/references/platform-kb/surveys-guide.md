# `surveys` — platform questionnaire / feedback layer

> Module guide Artifact: <https://claude.ai/code/artifact/2b67b6d7-67af-470f-81d8-33de2d86dbb7>
> (source [`docs/artifacts/surveys-module-guide.html`](../../../docs/artifacts/surveys-module-guide.html)).
> Authoring guide: [`docs/28-surveys.md`](../../../docs/28-surveys.md).
> First consumer: the clinic post-visit feedback survey
> ([`src/packs/clinic/surveys.yaml`](../../packs/clinic/surveys.yaml)).

Generic, business-agnostic **surveys** (post-visit feedback, CSAT, NPS, intake
questionnaires). A **thin declaration + validation layer over XRM**: the pack
declares its questionnaires in `src/packs/<id>/surveys.yaml`; responses are stored
as **`survey_response` XRM system-entity records** (so surveys owns **no table**);
and aggregation reuses the generic **`record_aggregate`** XRM primitive. The
platform stays domain-blind (invariant #6) — exactly like `casework` / `xrm` /
`scheduling`.

> Why not its own store: a survey response is a *per-contact measurement* — an
> XRM record (contact-linked, pipeline-less). Reusing XRM buys storage, the
> console Records surface, the timeline, and aggregation for free. Surveys adds
> only what XRM lacks for questionnaires: typed-question **definitions** +
> answer **validation** + a **score**.

## For operators (business)

- A **survey** is a named questionnaire — typed questions (`rating`, `number`,
  `text`, `boolean`, `select`), an optional numeric **score** question, and an
  optional **subject** (what each response is about, e.g. a `booking`).
- A **response** is a `survey_response` record — the answers (`answers` JSON),
  the derived `score`, the `contact` who answered, and `subject_ref`. It appears
  on the **Records** surface + the contact-detail page; aggregate scores (avg
  rating, response counts) render as tiles.

## For pack authors (technical)

Declare `src/packs/<id>/surveys.yaml`:

```yaml
surveys:
  visit_feedback:
    label:   { en: Visit feedback, ar: تقييم الزيارة }
    subject: booking                  # hint: subject_ref points at a booking record
    questions:
      - { key: stars,   type: rating, scale: 5, required: true }
      - { key: comment, type: text }
    score: stars                      # this answer becomes survey_response.score
```

Drive it from flows:

- `survey_submit { survey, answers, subject_ref?, contact_id? }` — validates the
  answers against the questionnaire, upserts a `survey_response` record on
  `(survey_key, subject_ref)` (a re-rating updates, never duplicates), and
  derives `score`. Ports: `ok` (`{ record_id, score }`) · `failed`.
- **Read / aggregate** with the XRM primitives directly: `record_list` (a
  contact's responses), `record_aggregate` (`avg` of `score` over a `where`
  filter). The **rating rollup** onto a scheduling resource is pure composition —
  `record_aggregate` the avg, then `record_save` the resource's `rating` field
  (which `record_search rank_by: rating` orders on). No surveys-specific verb.

## Layout

| File | Role |
|------|------|
| `contracts/spec.ts` | `surveysFileSchema` — questionnaire declaration. |
| `resolver.ts` | `validateSurveysFile` + `buildPackSurveys`. |
| `registry.ts` | `SurveysRegistry` — in-process per-pack lookup. |
| `services/survey-service.ts` | `validateAnswers` + `submitSurvey` (→ `survey_response` XRM record). |
| `boundaries.ts` | The only public import surface. |

No migration — responses are the `survey_response` XRM system entity
(`xrm/system/entities.ts`). Loader: `src/platform/manifest/surveys.ts`, wired into
`definePack` step 5h. Primitive: `src/platform/primitives/surveys/survey_submit`.

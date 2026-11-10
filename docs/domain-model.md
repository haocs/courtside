# Domain model and time

The [website entity contract](https://github.com/haocs/tennis-web/blob/main/docs/ENTITY-MODEL.md) is authoritative for current serving APIs. This page records cross-project meaning.

| Entity / record | Meaning | Reprocessing behavior |
| --- | --- | --- |
| Original recording | Immutable camera input, content identity and original timeline | Retain. Replacement/trim/time warp is a different original. A path is not identity. |
| Session | One practice containing one or more Videos, with editable metadata | Retain independent identity/membership and metadata; it is not a video. |
| Video | Logical processed presentation of one original recording | Retain logical identity; select a new Video version. |
| Video version | A published processing output with mappings and Clips | New identity per publication; retain observed historical descriptors. |
| Clip | A generated cut in a specific Video version | Replacement Clips have new entity identity. Do not transfer notes/usefulness by approximate overlap. |
| Moment | Typed serving information at original-recording start, optionally with an end | Preserve canonical anchor, payload, provenance and history; project into current retained footage. |
| Player / Court | Editable reference entities | Retain stable references. Actual profiles and positions remain private data. |
| Processing annotation | Evidence/judgment about a recording, output, Clip or time range | Retain evaluated run/output context, selectors and revisions. It is not a serving Moment or automatic benchmark truth. |

## Metadata is not automatically a Moment

Untimed notes are fields of Session/Video/Clip metadata. Text attached to a Moment is its payload; it does not create a Note entity. An optional end does not create an Interval entity.

A highlight is a Moment type. Coaching reviews/suggestions are Moments with structured information and evidence. The current simple user-created Moment may use `comment` on the wire; UI wording need not expose that compatibility type.

Usefulness/dislike belongs to a specific Clip or Video version. Up remains the existing favorite/Like signal with its existing compatibility policy; the current player makes up and down exclusive. A correctly cut Clip can be disliked for low usefulness. A bad-cut report is separate processing evidence. Neither is a skill grade or an accepted training label.

## Original time is canonical

Store integer original-recording milliseconds for Moments. Clip-local and concatenated-practice time are projections through the published mapping. Use half-open ranges `[start,end)` so a boundary point belongs to only one adjacent Clip.

A spanning Moment can intersect several Clips; show its relevant fragment without moving the saved anchor. Removed gaps are not playable just because they fall between the earliest and latest retained timestamps. A Moment outside all current cuts is retained historically, not moved to a nearby shot.

The watch panel, markers and Jump operate within the current Clip or the current Video's retained ranges. Type/archive filters do not broaden that range. Textless highlights have markers/list entries but no playback teaser. See [moment-panel behavior](https://github.com/haocs/tennis-web/blob/main/docs/MOMENT-PANEL.md).

## Coaching and processing evidence

Retain the selected source-time interval, frame/snapshot mapping, crop/zoom/speed recipe, prompt/response provenance and review revisions when available. An excerpt-wide answer is context, not an invented precise technique timestamp. Sampled frames cannot establish every contact or continuous-motion event.

Serving edits enter an accepted evaluation dataset only through an explicit review/export/freeze decision. Keep model predictions, AI suggestions, human edits and benchmark truth distinguishable. Reprocessing must not relabel an old cut report as feedback on a new cut.

## Player learning plans and practice

A learning plan is a player-level synthesis with an active revision and retained history, rather than a Moment or a processing annotation. Citations preserve the reviewed evidence revision and original-time anchor. A changed or reprocessed Clip must not silently change that evidence. AI proposals remain distinguishable from human-confirmed observations.

Practice items have stable server identities and append-only progress revisions. A user reporting practice is not proof of observed skill or mastery. Model updates preserve those user edits; malformed or stale completions remain audit records without replacing the active plan or advancing its coverage. Coverage records what evidence was actually supplied, not every item available to the player. See [the owner contract](https://github.com/haocs/tennis-web/blob/main/docs/LEARNING-PLAN.md).

## Names that predate this model

Legacy `/api/sessions` and `metadata/sessions` names can refer to per-recording Videos; practice/session APIs represent actual Sessions. Raw publisher Clip IDs can remain URL parameters, while serving Clip entity IDs include the publication version. Inspect the exact component contract before changing identity or like migration behavior.

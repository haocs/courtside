# Public-repository privacy boundary

This hub must contain no credentials, access/session data or personally identifying information. Its purpose is reproducible instructions without an installation identity.

## Keep outside Git and public issues

- Any secret value, account access material, session/cookie data, private configuration, resolved environment dump or browser profile export.
- Personal names/contact details, actual player profiles or body/skill measurements, recording dates tied to a person, court/address/GPS/altitude data and location history.
- Real recordings, frames, thumbnails, coaching responses/evidence, label datasets, model artifacts derived from private footage or operational database copies.
- Actual NAS/machine names, private IPs, storage/share paths, mounted-volume paths, user home paths and deployment-instance records.
- Raw logs, command transcripts and screenshots that may contain any of the above, including otherwise harmless error diagnostics.

Public component repository URLs, public source revisions, generic program names, loopback development examples and independently written synthetic explanations are appropriate. Do not copy an example from a real installation and merely replace its most obvious identifier.

## Before publishing

Review the exact staged files and complete diff, including newly added files and embedded links. Allow only intended Markdown, agent-skill metadata and repository housekeeping. Ensure ignore rules did not simply hide a private file that was already tracked.

Search for secret/token formats, authorization/session headers or payloads, PEM blocks, account/contact information, non-loopback network addresses, user/volume/storage paths and source-media identifiers. Investigate each match locally without echoing its value into a tool response or public log. Inspect Git commit attribution/settings as appropriate; do not add personal contact details to documentation or handoff text.

Validate links and examples after sanitizing. Keep private validation reports outside this repository. A text-pattern scan helps catch mistakes but does not prove absence of sensitive material; review content and provenance too.

## If a problem is discovered

Stop publishing the affected material. Do not paste it into an issue or scanner report. Notify the operator privately with the minimal relevant description. Access-material revocation and repository-history cleanup are distinct operations with their own authorization; deleting a file in a later commit does not erase earlier publication.

Operational tasks should use the operator's existing private configuration and authorized tools without copying those values into this hub. The deployment runbook deliberately documents boundaries and verification instead of containing a reusable live installation profile.

# Dependency policy

Rules read on every dependency check. Add a reason for every rule.

Rule forms: `skip` (no updates at all), `stay on <major>.x`, `max <version>`,
`ignore advisory <GHSA-id>` (accepted risk, reason required).

| Package | Rule | Reason | Added | Review by |
|---|---|---|---|---|

<!--
Example row (copy into the table above; a row here is not a rule):
| react, react-dom | stay on 18.x | waiting for library X to support 19 | 2026-11-16 | 2027-02-01 |
-->

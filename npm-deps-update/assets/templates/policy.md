# Dependency policy

Rules read on every dependency check. Add a reason for every rule.

Rule forms: `skip` (no updates at all), `stay on <major>.x`, `max <version>`,
`ignore advisory <GHSA-id>` (accepted risk, reason required).

| Package | Rule | Reason | Added | Review by |
|---|---|---|---|---|
| _example: react, react-dom_ | _stay on 18.x_ | _waiting for library X to support 19_ | _2026-11-16_ | _2027-02-01_ |

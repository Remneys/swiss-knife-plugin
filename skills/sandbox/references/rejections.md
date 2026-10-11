# Rejections and failures worth rehearsing

`swiss-knife:sandbox_review_decide` with `decision: "reject"` takes a `reason`
from `rejection_reasons` in `swiss-knife:sandbox_state`. Each is a name in the
store error catalogue, so the failure the real tools show next is one their
remedies already answer. Pick by what you want to practise.

| Store   | `reason`                                      | Practise                                         |
| ------- | --------------------------------------------- | ------------------------------------------------ |
| iOS     | `asc_app_privacy_incomplete`                  | App Privacy not published: a human step          |
| iOS     | `asc_submission_incomplete`                   | A required field or the build is missing         |
| iOS     | `age_rating_answer_rejected`                  | An age rating answer Apple refuses               |
| Android | `play_policy_declaration_incomplete`          | A Play Console declaration is open: a human step |
| Android | `play_financial_features_declaration_missing` | The financial features declaration               |
| Android | `play_health_declaration_missing`             | The health declaration                           |
| Android | `play_app_content_declaration_missing`        | The app content declaration                      |

`metadata_only: true` (iOS) rejects the listing and not the binary: fix the
text with `swiss-knife:listing_update` and push it; no new build.

After a rejection, read the version's state with `swiss-knife:sandbox_state`
(`REJECTED` or `METADATA_REJECTED`; Android `NOT_APPROVED`), fix what the error
says with the real tools, and submit again (a new code is needed).

## Failures to inject

Faults fire on the traffic of the real tools. Useful routes:

| Route (iOS operation id)           | Reason to try               | Shows                               |
| ---------------------------------- | --------------------------- | ----------------------------------- |
| `reviewSubmissions_createInstance` | `asc_state_error`           | A submit that must be retried later |
| `appStoreVersions_updateInstance`  | `asc_version_state_invalid` | A listing push to a locked version  |
| `apps_getCollection`               | `asc_rate_limited`          | A store read that backs off         |

Use `times: 1` or `2` and a short `ttl`. A transport failure is
`effect: { "reset": "ECONNRESET" }`. After the rehearsal, `action: "clear"`.

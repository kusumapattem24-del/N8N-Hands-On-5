# Customer Data Synchronization Platform

## Project overview
This n8n capstone synchronizes mock customer records from three independent sources: API session data, database profiles, and spreadsheet marketing preferences. Email is the common matching key. The workflow normalizes keys, joins the datasets, builds a consistent output schema, validates full names, and separates valid profiles from quarantine records.

**Status:** Implementation guide and submission template. Actual workflow execution, screenshots, repository publication, and video have not been verified.

## Workflow architecture
```text
Manual Trigger
  |-- API_SessionData (5+ records) -> NORMALIZE_API_Email -----------|
  |-- DB_CustomerProfiles (5+ records) -> NORMALIZE_DB_Email --------|-> MERGE_APIDatabase
  |-- Sheet_MarketingPreferences (5+ records) -> NORMALIZE_Sheet_Email -----|
MERGE_APIDatabase + NORMALIZE_Sheet_Email -> MERGE_MarketingSheet
  -> CLEAN_CoalesceFields -> IF_HasFullName
      |-- true  -> OUTPUT_UnifiedCustomerProfiles
      |-- false -> Quarantine_UnmatchedProfiles
```
The branches originate from one Manual Trigger. n8n executes branches according to its execution model; this diagram does not guarantee simultaneous processing.

## Sample source fields
- **API sessions:** email, deviceType, lastActive
- **Database profiles:** email, firstName, lastName
- **Marketing sheet:** email, marketingOptIn, preferredLanguage

Example mock records (expand each source to at least five):

**API sessions**
```json
[{"email":" ALICE@EXAMPLE.COM ","deviceType":"Mobile","lastActive":"2026-10-20T09:00:00Z"},{"email":"bob@example.com","deviceType":null,"lastActive":null},{"email":"cara@example.com","deviceType":"Desktop","lastActive":"2026-10-21T12:00:00Z"},{"email":"dan@example.com","deviceType":"Tablet","lastActive":null},{"email":"eve@example.com","deviceType":"Mobile","lastActive":"2026-10-19T10:00:00Z"}]
```
**Database profiles**
```json
[{"email":"alice@example.com","firstName":"Alice","lastName":"Lee"},{"email":" BOB@EXAMPLE.COM ","firstName":"Bob","lastName":"Singh"},{"email":"cara@example.com","firstName":"Cara","lastName":"Patel"},{"email":"dan@example.com","firstName":"","lastName":""},{"email":"eve@example.com","firstName":"Eve","lastName":"Roy"}]
```
**Marketing preferences**
```json
[{"email":"alice@example.com","marketingOptIn":true,"preferredLanguage":"en"},{"email":"bob@example.com","marketingOptIn":false,"preferredLanguage":null},{"email":" CARA@EXAMPLE.COM ","marketingOptIn":true,"preferredLanguage":"fr"},{"email":"dan@example.com","marketingOptIn":false,"preferredLanguage":"en"},{"email":"eve@example.com","marketingOptIn":true,"preferredLanguage":"es"}]
```

## Node configuration
1. **Manual Trigger:** Connect to three Edit Fields (Set) nodes populated with the three mock datasets, or Code nodes returning one n8n item per customer. If using Set nodes, ensure each customer is a separate item rather than one item containing an array.
2. **NORMALIZE_*_Email:** Use an Edit Fields node on each branch, preserving other fields. Set `email` to `{{ String($json.email ?? '').trim().toLowerCase() }}`. Exclude blank emails from joins or quarantine them separately.
3. **MERGE_APIDatabase:** Merge node -> Mode: Combine -> Combine By: Matching Fields -> Input 1 field `email`, Input 2 field `email`. Choose the appropriate output type. **Keep Matches** yields only matched API/DB records; **Keep Everything** (where available) preserves unmatched records for further inspection. Confirm your n8n version's option labels.
4. **MERGE_MarketingSheet:** Same Combine / Matching Fields configuration on `email`, joining the previous merge output to the normalized spreadsheet branch. For a strict matched-only dataset, select Keep Matches. To preserve partial records for quarantine, use an inclusive join and check null/missing values after merging.
5. **CLEAN_CoalesceFields:** Use Edit Fields to output exactly six standardized fields, preserving boolean `false`:
   - `email`: `{{ String($json.email ?? '').trim().toLowerCase() }}`
   - `fullName`: `{{ [String($json.firstName ?? '').trim(), String($json.lastName ?? '').trim()].filter(Boolean).join(' ') }}`
   - `deviceType`: `{{ $json.deviceType || 'Unknown' }}`
   - `lastActive`: `{{ $json.lastActive || 'Not Available' }}`
   - `marketingOptIn`: `{{ $json.marketingOptIn ?? false }}`
   - `preferredLanguage`: `{{ $json.preferredLanguage || 'en' }}`
6. **IF_HasFullName:** Check `{{ String($json.fullName ?? '').trim() }}` is not empty. Optionally also validate `email` is not empty and is email-shaped.
7. **OUTPUT_UnifiedCustomerProfiles / Quarantine_UnmatchedProfiles:** Use named Edit Fields nodes (or appropriate destinations) to retain clean records and record a quarantine reason, e.g. `MISSING_FULL_NAME`. Export/persist the outputs if required by the assessor.

**Merge caution:** Matching Fields with Keep Matches discards unmatched rows before the IF node; they cannot be quarantined downstream. To demonstrate unmatched-profile quarantine, configure inclusive joins or explicitly capture unmatched outputs with separate routing. Check for duplicate emails in each source, as many-to-many matches can multiply rows.

## Expected example output
Valid record:
```json
{"email":"alice@example.com","fullName":"Alice Lee","deviceType":"Mobile","lastActive":"2026-10-20T09:00:00Z","marketingOptIn":true,"preferredLanguage":"en"}
```
Quarantine candidate after inclusive joins:
```json
{"email":"dan@example.com","fullName":"","deviceType":"Tablet","lastActive":"Not Available","marketingOptIn":false,"preferredLanguage":"en","quarantineReason":"MISSING_FULL_NAME"}
```
These are **illustrative expected outputs**, not captured execution results.

## Test matrix
| Test | Input condition | Expected result | Actual / evidence |
|---|---|---|---|
| T01 | Email contains spaces / uppercase | Normalized lowercase email and successful join | Pending |
| T02 | Complete record across three sources | Six-field unified profile | Pending |
| T03 | Missing device and activity | `Unknown` and `Not Available` | Pending |
| T04 | Missing language | Default `en` | Pending |
| T05 | Explicit `marketingOptIn: false` | False remains false | Pending |
| T06 | Empty first and last name | Quarantine with reason | Pending |
| T07 | Source record unmatched | Preserved / quarantined if inclusive joins used | Pending |
| T08 | Duplicate email in source | Detected and handled under documented policy | Pending |

## Evidence to capture
1. Complete workflow canvas showing all branches and nodes.
2. API, database, and sheet mock input records (5+ each).
3. Normalized email output from each branch.
4. Configuration and execution output for both Merge nodes.
5. CLEAN_CoalesceFields node configuration and resulting six-field schema.
6. IF_HasFullName true and false paths, unified output, and quarantine output.
7. Successful execution overview with timestamp and counts.

## Suggested repository layout
```text
customer-data-synchronization/
  README.md
  workflow/customer_data_synchronization.json
  samples/api_sessions.json
  samples/database_profiles.json
  samples/marketing_preferences.json
  docs/Customer_Data_Synchronization_Evidence.pdf
  screenshots/ (numbered execution screenshots)
```

## Submission information
- **Public GitHub repository:** [ADD LINK]
- **Loom / YouTube demonstration:** [ADD LINK]
- **Workflow export:** `workflow/customer_data_synchronization.json` (export from n8n)
- **Execution screenshots:** [ADD AFTER TESTING]
- **Challenges / assumptions:** Mock data only; merge output mode determines unmatched-row retention. Validate the configured n8n version's Merge behavior. No real customer PII or credentials should be committed.

## Demonstration video outline (3-5 minutes)
Introduce the objective, show the three source datasets, demonstrate email normalization, explain the two merges, show schema defaults, test one valid and one incomplete profile, display both output paths, and finish with the exported JSON and repository files.

## Final verification checklist
- [ ] All three sources have 5+ customer records
- [ ] Each source email normalized
- [ ] Both Merge nodes configured and executed
- [ ] Six output fields consistently populated
- [ ] Missing full names routed to quarantine
- [ ] Actual execution screenshots captured
- [ ] n8n JSON exported and committed
- [ ] PDF documentation committed
- [ ] Public repository and demonstration links verified

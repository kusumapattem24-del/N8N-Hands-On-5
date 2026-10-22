# Paginated Fetch & Rate-Limited Batch Email Workflow

**Topic 5 Assessment | n8n | Implementation and Evidence Guide**  
**Status:** Documentation / proposed configuration only. Actual execution evidence and exported n8n JSON must be added after testing.

## 1. Objective and scope
Build a Manual Trigger workflow that retrieves paginated JSONPlaceholder users, combines all pages, processes users in batches, simulates email requests, tolerates individual failures, pauses between batches, and produces a reliable summary.

## 2. Workflow architecture

```text
Manual Trigger -> Config (pageSize=10, batchSize=5, waitSeconds=2)
                    |
                 Fetch Page (GET /users?_page=N&_limit=10)
                    |
             Capture Page / Accumulate Results
                    |
               More Pages? (count == pageSize)
                 | Yes                  | No
                 v                      v
           Increment Page       Combine All Pages
                 |                      |
             Fetch Page          Split In Batches (5)
                                        | batch
                                 Send Email (Simulated)
                                        |
                                  Check Success
                                        |
                              Record Outcome / Metrics
                                        |
                                Rate Limit Pause (2s)
                                        |
                              Split In Batches (loop)
                                        | done
                                     Summary
```

**Pagination note:** A page shorter than `pageSize` is the final nonempty page; the loop can stop there. When a page has exactly `pageSize` results, fetch the next page to detect exhaustion. Ensure the loop stores records before testing the stop condition. The public JSONPlaceholder `/users` dataset is commonly 10 records; with `pageSize=10`, this usually means one full page and a subsequent empty page. A smaller final processing batch therefore needs a separate test dataset or different batch size.

## 3. Exercise 1 - Config and pagination
- **Manual Trigger:** starts an on-demand run.
- **Config (Edit Fields):** `pageSize` = 10, `batchSize` = 5, `waitSeconds` = 2, `page` = 1.
- **Fetch Page (HTTP Request):** GET `https://jsonplaceholder.typicode.com/users` with query `_page={{ $json.page }}` and `_limit={{ $json.pageSize }}`. Preserve the page state separately; an HTTP Request typically replaces its input item with API results.
- **Capture Page / Accumulate Results:** append the page's user items to a run-scoped accumulator and retain the page's record count. A native-loop implementation must explicitly preserve earlier pages; simply connecting page outputs to a Merge node does not guarantee aggregation over repeated loop iterations.
- **More Pages? (IF):** continue when returned count equals pageSize; increment `page` and request again. Stop when returned count is less than pageSize (including zero).
- Add a maximum-page guard and a retry strategy to prevent runaway loops or temporary API errors from being mistaken for end-of-data.

## 4. Exercise 2 - Consolidation and batching
- **Combine All Pages:** produce one item per unique retrieved user, preserving `id`, `name`, and `email`; verify the total against the accumulated page counts.
- **Split In Batches (Loop Over Items):** configure batch size from Config (`{{ $('Config').first().json.batchSize }}`) where the installed n8n version supports expressions.
- Connect the batch output to processing and the completed output to Summary. Connect the final batch-processing node back to the batch node so the next group runs.
- Test a record count not divisible by 5 (e.g., 12 users -> 5, 5, 2). JSONPlaceholder's standard `/users` list alone may not provide that case.

## 5. Exercise 3 - Simulated email and failure handling
Use HTTP Request with a test endpoint such as `https://postman-echo.com/post`, method POST, JSON body:

```json
{"userId": 1, "name": "Leanne Graham", "email": "leanne@example.com", "action": "SIMULATED_EMAIL"}
```

Map `userId`, `name`, and `email` from each current record. **This does not send real email.** Configure **On Error: Continue (using error output)** or the equivalent supported option and retain user context on both success and failure paths. If processing one record per request, ensure the HTTP Request actually executes once per user, rather than sending the entire batch as one payload.

**Check Success:** classify a successful 2xx response as `Sent` (simulated), otherwise `Failed`; capture `userId`, `email`, HTTP status when available, and error message. Do not treat any response object as success merely because the node continued.

## 6. Exercise 4 - Throttling and summary
- **Rate Limit Pause (Wait):** delay `{{ $('Config').first().json.waitSeconds }}` seconds after each completed batch, **before** requesting the next batch. Avoid an unnecessary pause after the final batch when possible.
- **Record Outcome:** persist one outcome per attempted user for the current execution, including failed requests.
- **Summary:** aggregate the outcomes **after the Loop Over Items done output**; ensure metrics are computed from all batches, not only the final batch.

```json
{"totalRecords": 10, "totalSucceeded": 9, "totalFailed": 1, "status": "COMPLETED_WITH_FAILURES"}
```

The values above are an **illustrative example, not observed execution results**. Validate `totalSucceeded + totalFailed = totalRecords` and record the actual totals from your run.

## 7. Exercise 5 - Testing and governance

| Test ID | Scenario | Expected behavior | Evidence status |
|---|---|---|---|
| T01 | Start with Config 10 / 5 / 2 | Reusable settings visible | Pending |
| T02 | Fetch first full page | 10 user records returned | Pending |
| T03 | Fetch next page | Stop after empty/short page | Pending |
| T04 | Aggregate multiple pages | No page records lost or repeated | Pending |
| T05 | Process 10 users | Two batches of 5 | Pending |
| T06 | Process 12 test users | Batches 5, 5, 2 | Pending |
| T07 | Force one HTTP failure | Workflow continues; one Failed | Pending |
| T08 | Inspect timestamps | At least 2s pause between cycles | Pending |
| T09 | Review Summary | Succeeded + Failed = total | Pending |
| T10 | Export and reimport JSON | Nodes and connections retained | Pending |

### Risk controls
- **Pagination state loss:** keep `page`, count, and accumulated records in explicit run-scoped state.
- **Infinite pagination:** stop on a short page and enforce a maximum-page limit.
- **Missing or duplicate records:** compare retrieved IDs and page totals before processing.
- **Rate limits:** pause between batches; consider retry/backoff for 429 and 5xx.
- **Partial failure:** retain record identity and error details; use failure-safe continuation.
- **Misleading success:** `Sent` means simulated request accepted, not actual email delivered.
- **Metrics gaps:** count every attempted record exactly once, including failures.
- **Privacy:** avoid publishing real addresses, credentials, or private payloads to GitHub.

### Node names and annotations
Use: `Manual Trigger`, `Config`, `Fetch Page`, `Capture Page`, `More Pages?`, `Increment Page`, `Combine All Pages`, `Split In Batches`, `Send Email (Simulated)`, `Check Success`, `Record Outcome`, `Rate Limit Pause`, `Summary`. Add sticky notes for pagination stop logic, accumulator design, failure handling, throttle location, and metrics definition.

## 8. Evidence to capture
1. Complete workflow canvas with descriptive node names and annotations.
2. Config node showing `pageSize=10`, `batchSize=5`, `waitSeconds=2`.
3. HTTP Request parameters and first-page output.
4. Pagination loop and stop-condition execution.
5. Consolidated dataset and batch outputs, including partial batch test.
6. Successful and deliberately failed simulated email requests.
7. Wait node configuration and execution timing.
8. Final Summary with actual metrics and successful overall execution.

**Evidence status:** No screenshots or execution logs were supplied for this assessment. Insert actual screenshots and results before submission.

## 9. GitHub repository structure

```text
paginated-batch-email-workflow/
  README.md
  workflow/Paginated_Batch_Email_Workflow.json
  docs/Paginated_Batch_Email_Assessment.pdf
  evidence/ (actual execution screenshots)
  tests/ (actual test results)
```

### Setup instructions
1. Import your exported n8n JSON into a compatible n8n workspace.
2. Check all HTTP Request URLs, pagination parameters, and node versions.
3. Set reusable Config values and inspect the loop connections.
4. Execute manually; confirm page counts and aggregated records.
5. Test success and intentional failure scenarios and inspect Summary.
6. Export the latest workflow JSON and remove any secrets before publishing.

## 10. Video demonstration outline (5-8 minutes)
- 0:00-0:45: Assessment objective and canvas overview.
- 0:45-2:00: Config, Fetch Page, page loop, stop condition.
- 2:00-3:00: Aggregation and batch processing, including a partial batch test.
- 3:00-4:30: Simulated email and Continue On Fail, success/failure classification.
- 4:30-5:30: Wait-based throttling and Summary totals.
- 5:30-8:00: Evidence, annotations, exported JSON, GitHub repository.

## 11. LMS submission checklist
- [ ] Workflow executes end to end
- [ ] Pagination, aggregation, batch sizes, failure handling, and Wait verified
- [ ] Summary counts reconcile with actual attempted records
- [ ] Latest exported n8n workflow JSON committed
- [ ] Actual screenshots and test results committed
- [ ] Documentation PDF and README.md committed
- [ ] GitHub repository is public and accessible
- [ ] Loom/YouTube video link is accessible
- [ ] LMS Description includes actual challenges, resolutions, assumptions, limitations, and improvements

**Important:** This PDF and README are documentation only. They are not a working n8n workflow export and do not substitute for execution evidence, a public GitHub repository, or a demonstration video.

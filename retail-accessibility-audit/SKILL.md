---
name: retail-accessibility-audit
description: Execute evidence-based, UI-driven accessibility audits of retail/e-commerce websites using Playwright and axe-core. Covers browser lifecycle safety, dynamic journey discovery, WCAG checks, deterministic health scoring, evidence, recovery, and consolidated Markdown reporting.
---

# Retail Accessibility Audit Agent Skill

## 1. Purpose and operating model

Perform a dynamic, evidence-based accessibility audit of a retail/e-commerce website using Playwright and axe-core. Discover actual customer-facing navigation from the rendered UI, inspect meaningful interactive states, recover safely from transient execution failures, and produce one consolidated Markdown report.

This skill defines reusable execution behavior. The execution prompt supplies the target application's configuration and task-specific limits.

**Truthfulness rule:** Never claim a page, state, journey, scan, or test was completed unless it actually happened and the result was recorded.

**Security and transaction handling:** Protect credentials and session data throughout the audit. For an explicitly authorized QA test purchase, follow the configured checkout flow through payment submission, order placement, and confirmation verification using the approved test account and payment setup. Record the actual outcome accurately and never expose sensitive information in logs, screenshots, or reports.

## 2. Configuration handling

1. Read the execution prompt and use only its configured values.
2. Keep application-specific values out of reusable skill rules.
3. Treat blank values as unspecified. Never invent URLs, selectors, credentials, behaviors, or exceptions.
4. Retrieve credentials only through the approved secret manager/runtime mechanism. Do not hardcode secrets in prompts, scripts, logs, screenshots, traces, or reports.
5. If credentials appear to have been exposed, do not repeat them; advise the environment owner to rotate them.
6. Discover sign-in fields and actions from rendered UI and accessible attributes. Reuse the authenticated browser context when supported.
7. Verify authentication with reliable evidence such as account navigation or customer-specific content. A successful click alone is not proof.
8. If authentication fails, record the current URL, visible step, and sanitized error. Make at most one safe retry for a clearly transient failure. Do not repeatedly submit credentials, create accounts, reset passwords, alter account details, or bypass MFA/phone verification/CAPTCHA. Mark authenticated-only journeys blocked and continue public journeys.

## 3. Browser lifecycle safety — highest priority

These rules take precedence over navigation, crawling, retry, and cleanup instructions.

### Lifecycle ownership
- Only the main audit controller may close the browser, browser context, or shared page.
- Individual page-scanning tasks, navigation helpers, retry handlers, and journey workers must never close shared resources.
- Do not execute `browser.close()`, `context.close()`, or `page.close()` while any navigation, accessibility scan, or journey task is running.
- Await every navigation, scan, popup handler, spawned task, and artifact-writing operation.
- Before cleanup, confirm all tasks have completed or have been explicitly cancelled and awaited.
- Never run independent `page.goto()` operations concurrently on the same page. Serialize navigation or give each independent task its own page with a clearly owned lifecycle.

### Preflight and navigation
- Before depth-1 navigation, verify the browser is connected, the intended context is open, and the page is open.
- Confirm the initial base page loaded successfully and meaningful content is visible.
- Check browser/context/page health immediately before every `page.goto()`.
- Keep one authoritative browser/context reference and avoid creating pages indiscriminately.

### Failure diagnostics and recovery
When a Playwright action fails:
1. Stop the current interaction sequence.
2. Capture sanitized diagnostics: timestamp, journey/step ID, last successful URL, destination URL only if actually discovered, attempted action/API, error name/message, page/context/browser health, and relevant recent console/page errors.
3. Check `browser.isConnected()`, `context.pages()`, and `page.isClosed()` only where supported; do not call methods on closed objects.
4. If only the page closed and browser/context remain healthy, create one replacement page in the same context and resume from the last safe checkpoint.
5. If the context closed, recreate it only if the browser is healthy and policy permits. Re-establish authentication through the normal approved flow; never copy session cookies manually or bypass login.
6. If the browser disconnected or crashed, relaunch only within the configured retry budget. Revisit the base URL and reconstruct the journey through the UI; do not jump to a deep URL as a shortcut.
7. Retry a failed navigation at most twice after the initial attempt, and only after diagnostics and an appropriate recovery action. Do not blindly repeat the same failing call.
8. For an ordinary timeout with a healthy page, inspect the current UI, wait for a meaningful condition, and retry once with an appropriate wait strategy. Avoid fixed sleeps as the only synchronization method.
9. If multiple depth-1 pages fail with the same closure error, suspend fan-out and diagnose the shared lifecycle before retrying.
10. If recovery fails, mark the affected step/page `BLOCKED`, `FAILED`, or `NOT REACHED` as appropriate. Continue other independent journeys when safe and possible.
11. Validate recovery by returning to the configured base URL and verifying meaningful visible content before resuming.
12. Cleanup once, in a final cleanup block, after all eligible journeys, scans, and artifact writes are complete. Cleanup must not mask the original exception.

Never classify browser lifecycle failures as accessibility violations. Do not count failed pages as scanned or report an accessibility pass after a failed action.

## 4. UI-driven navigation and discovery

- Start at the configured base URL.
- Use customer-facing rendered UI: visible links, buttons, menus, search, breadcrumbs, filters, and other rendered controls.
- Do not construct or hardcode deep URLs merely because a route is predictable or appeared in a previous report.
- Direct navigation is allowed only to the initial base URL or a clearly documented exception explicitly authorized by configuration/operator. Record any exception and reason.
- After each click that should navigate, wait for a meaningful outcome and verify the resulting page/state. A click attempt is not proof of navigation.
- Record the actual navigation path for each audited page.
- Detect popups/new tabs and inspect the correct page without abandoning or confusing it with the original.
- Discover controls dynamically; do not assume fixed selectors or that a common retail feature exists.

Maintain a live inventory of relevant rendered controls and page types, including header/navigation, menus, search, categories/listings, product links, account/sign-in, wishlist, cart, footer, consent dialogs, filters, sorting, pagination/load-more, and other relevant controls.

For every discovered page/state, track:
- page identifier and canonical URL actually observed;
- page type and authentication requirement;
- source control and navigation path;
- attempt count and final status;
- whether the page reached a stable usable state;
- whether axe-core ran and results were saved;
- manual checks performed;
- screenshot/evidence reference, if captured safely;
- errors, recovery, and coverage limitations.

## 5. Page statuses and coverage counting

Use these status values consistently:

- `SCANNED`: Page reached, stable, axe-core ran, and results were saved.
- `PARTIALLY SCANNED`: Page reached, but some required checks/states failed or were incomplete.
- `LOADED — NOT SCANNED`: Page loaded, but accessibility checks did not run.
- `BLOCKED`: A prerequisite or safety/access control prevented the journey.
- `FAILED`: A technical failure prevented reaching/inspecting the page after recovery attempts.
- `NOT REACHED`: No successful navigation occurred.
- `NOT APPLICABLE`: Feature is genuinely absent, supported by evidence.

Do not count a page as scanned because it was discovered, attempted, linked, or mentioned in a log. Count unique pages separately from states and viewports. Deduplicate repeated URLs/pages for unique-page totals while scanning materially different states separately.

## 6. Journey execution

Run each journey when the functionality exists, is accessible, and is safe. Continue other eligible journeys after one fails. Configuration limits are ceilings, not targets; do not exceed them.

### Home and site discovery
- Inspect initial rendered state, landmarks, headings, header, navigation, footer, search, account, wishlist, and cart entry points.
- Discover primary categories and relevant customer-facing pages.
- Inspect responsive navigation/menus when practical.
- Record absent, hidden, blocked, or undiscoverable controls.

### Category/listing and search
- Scan representative category/listing pages within the configured maximum.
- Exercise search when available: query entry, submit, suggestions, results, no-results state, and relevant validation/error states.
- Inspect filters, sorting, pagination/load-more where present and within scope.
- If a page cannot be reached, record the actual UI path and blocker; do not invent a URL.

### Product details
- Scan representative product pages within the configured maximum.
- Inspect title, price, images/alt text, description, availability, required variants, quantity, gallery, accordions, and primary actions.
- Exercise safe controls and record observed behavior.
- Select a required variant only when needed to add the product.

### Authentication and account
- Discover sign-in through UI; inspect form and safe validation states.
- Authenticate only through approved credentials and legitimate flow.
- Verify success before protected journeys.
- Do not edit/save personal details, cancel orders, or perform destructive actions.
- If authentication fails or unsupported verification is required, document it and continue public journeys.

### Add one item, cart, and checkout — priority journey
Treat this as a priority journey, but do not bypass blockers or safety controls.

1. **Product selected:** Navigate from a UI-discovered listing/search result to a suitable product page. Record only necessary product identity.
2. **Required options:** Select required size/color/variant using rendered controls. If unavailable, try one alternate eligible product within limits, then document the blocker.
3. **Add to bag:** Click once. Wait for visible confirmation, drawer, count change, or cart state. Verify before proceeding. If ambiguous, inspect current bag state before any retry.
4. **Cart verification:** Open cart/bag through UI. Verify the product appears. Inspect item details, quantity, remove/edit controls, totals, empty/error states, and checkout entry point where present. If a drawer appears, inspect it and use its visible controls.
5. **Checkout navigation:** Click the visible checkout action from the cart. Verify checkout loaded through observed URL/title and visible content/form controls. Do not use a hardcoded checkout URL. If redirected to login, use normal authentication if allowed and verify the cart/session remains intact. Handle visible consent/overlay/new-tab flows only through legitimate UI.
6. **Checkout and payment accessibility:** Inspect contact, shipping, delivery, billing, and payment sections. Use the configured QA test data and verify field labels, required indicators, keyboard/focus order, validation, error associations, announcements, and step progression. Complete the configured payment flow when the QA test purchase is authorized and the payment setup is verified.
7. **Order confirmation:** After submitting the authorized test order, verify the actual order outcome, follow the resulting UI, and scan the confirmation page when it is displayed. Record the observed result without exposing payment, customer, or order-sensitive details.
8. Report each checkpoint as a separate outcome. Do not claim later stages were reached when only an earlier checkpoint succeeded.

### Wishlist and shared interaction states
- Run wishlist only if available and authentication prerequisites are met.
- Inspect empty/populated states and controls; avoid persistent customer-data changes unless authorized.
- Inspect relevant menus, consent banners, dialogs, tabs, accordions, dropdowns, filters, variant selectors, bag drawers, form errors, loading states, success/error announcements, and keyboard interactions.

## 7. Accessibility testing procedure

For each page and meaningful state actually reached:
1. Wait for a stable, usable state using meaningful conditions.
2. Run axe-core with configured tags: `wcag2a`, `wcag2aa`, `wcag21aa`.
3. Save actual results: violations, rule IDs, impact, descriptions/help, affected nodes/selectors, failure summaries, guidance URLs, incomplete checks, and passed/inapplicable results when available.
4. Capture screenshots if enabled, ensuring credentials, personal information, payment details, tokens, and secrets are not visible.
5. Perform practical manual checks for keyboard access, visible focus, focus order, accessible names, labels, headings, landmarks, link purpose, image alternative text, form errors, dialog/menu behavior, focus management, and state announcements.
6. Inspect responsive behavior at representative desktop/mobile viewports and tablet when practical. Record actual dimensions and observations. A viewport change is not a new page.
7. Distinguish automated findings, manual observations, incomplete checks, and unverified potential issues.
8. Deduplicate repeated findings while retaining affected pages/states, relevant nodes, evidence, and occurrence details.
9. Do not claim a finding is fixed unless it was retested successfully.
10. Do not claim axe-core alone proves full accessibility or WCAG conformance.

### Manual check statuses
- `PASS`: Performed and expected behavior verified.
- `FAIL`: Performed and a specific issue observed.
- `INCOMPLETE`: Attempted but could not be conclusively verified.
- `NOT TESTED`: Not performed.
- `NOT APPLICABLE`: Does not apply to observed page.

Never convert `INCOMPLETE` or `NOT TESTED` to `PASS`.

## 8. Finding evidence and severity

For each finding, record when available:
- unique finding ID;
- concise issue summary and user impact;
- severity/impact from evidence;
- WCAG criterion and axe rule ID only when supported;
- page URL and page type;
- relevant in-page state;
- affected element/selector and affected-node count;
- axe output or manual observation;
- safe screenshot/evidence reference;
- reproduction steps and observed behavior;
- recommended remediation;
- detection method: automated, manual, or both;
- retest status only if a retest occurred.

Do not invent severity, WCAG mappings, selectors, remediation, or verification outcomes. Keep confirmed findings separate from manual-review items. Do not count incomplete axe checks as confirmed violations.

## 9. Deterministic Accessibility Health Check

The audit must include a reproducible internal Accessibility Health Score when sufficient valid results exist. This is a reporting metric, **not an official WCAG score or proof of conformance**.

### Required fields
Record:
- calculated score (0–100) or `null`;
- grade or `null`;
- overall audit status;
- axe scan completion status;
- unique confirmed violations;
- Critical, Serious, Moderate, and Minor unique confirmed counts;
- incomplete/manual-verification count;
- unique pages scanned;
- meaningful states inspected;
- manual checks completed and pending;
- calculation and health-check notes.

### Scoring formula
Start at 100 and subtract penalties for each unique, confirmed, deduplicated finding:

| Severity | Penalty |
|---|---:|
| Critical | 15 |
| Serious | 8 |
| Moderate | 4 |
| Minor | 1 |

`total_penalty = (critical_count × 15) + (serious_count × 8) + (moderate_count × 4) + (minor_count × 1)`

`health_score = max(0, 100 - total_penalty)`

Grade:
- A: 90–100
- B: 80–89
- C: 70–79
- D: 50–69
- F: 0–49

Rules:
1. Deduplicate underlying issues across pages/states before scoring. Distinct issues sharing a rule ID may be counted separately when they have different underlying causes.
2. Use only severity classifications supported by evidence. Do not silently assign missing severities.
3. Exclude incomplete checks and unverified potential issues from confirmed-violation penalties.
4. Do not penalize technical/browser errors unless an accessibility issue is independently established.
5. Do not deduct points merely because a check was not performed; disclose coverage gaps separately.
6. Do not calculate a score if the home page was never reached, axe-core did not complete and no usable accessibility results exist, or severity data are insufficient for a reliable calculation. Set score and grade to `null` and explain why.
7. If automated scanning succeeded but manual checks remain incomplete, calculate the score from confirmed findings and mark overall status `Partial` when material coverage is incomplete.
8. Do not assign 100 because results are missing or the scan failed.
9. Save severity counts, penalties, formula, score, and grade in the report. Do not change weights between runs.

### Mandatory calculation procedure
Perform these steps in order before writing the report:

1. **Establish scan validity.** Confirm whether the home page was reached, whether axe-core completed on at least one eligible page, and whether usable severity data were saved. If the minimum validity conditions are not met, set both score and grade to `null`, state the exact reason, and stop scoring.
2. **Build the confirmed-findings set.** Combine recorded automated and verified manual findings. Exclude incomplete axe checks, unverified potential issues, duplicate representations of the same underlying issue, and technical errors that are not independently established accessibility issues.
3. **Deduplicate consistently.** Compare rule/criterion, affected component or underlying cause, evidence, and page/state context. One underlying issue repeated across pages/states should normally count once for the score while retaining all affected pages in the report. Count separately only when evidence supports distinct underlying causes.
4. **Validate severity.** Assign each scored finding to exactly one supported severity bucket: Critical, Serious, Moderate, or Minor. If severity is absent or ambiguous, do not guess; explain the uncertainty and set the score to `null` if the remaining severity data are insufficient for a reliable calculation.
5. **Count and calculate.** Record each unique severity count, calculate each severity subtotal and `total_penalty`, then calculate `health_score = max(0, 100 - total_penalty)`. Apply the grade bands above. Use whole-number scores.
6. **Reconcile.** Verify that the severity counts sum to the number of scored unique findings; the penalty equals the sum of the four subtotals; the score matches the formula; and the grade matches the score band. Resolve discrepancies against the finding-level evidence before publishing.
7. **Report coverage separately.** Record unique pages scanned, meaningful states inspected, manual checks completed/pending, and material coverage gaps. Do not treat missing coverage as a violation penalty or as evidence of a pass.
8. **Set overall status.** Use `Partial` if material required journeys, states, or checks remain incomplete even when a score can be calculated. Use `Blocked` or `Failed` when their definitions below apply. Never label an audit `Complete` solely because a score exists.
9. **Explain the result.** In the report, include the score and grade (or `null` and the exact reason), all four severity counts, each penalty subtotal, total penalty, formula, scan status, deduplication rule, and coverage limitations.

### Calculation example (illustrative only)
If the deduplicated confirmed findings are 1 Critical, 2 Serious, 3 Moderate, and 4 Minor:
- Critical penalty: `1 × 15 = 15`
- Serious penalty: `2 × 8 = 16`
- Moderate penalty: `3 × 4 = 12`
- Minor penalty: `4 × 1 = 4`
- Total penalty: `15 + 16 + 12 + 4 = 47`
- Health score: `max(0, 100 - 47) = 53`
- Grade: `D`

This example explains the arithmetic only. Never reuse example counts or scores as audit results.

### Overall audit status
- `Complete`: required eligible coverage completed within limits, with no material unresolved execution blocker.
- `Partial`: some required journey/state/checks remain incomplete, or technical failures materially limit coverage.
- `Blocked`: a prerequisite/access/safety barrier prevents meaningful execution.
- `Failed`: execution failed before meaningful audit results could be collected.

Do not mark the audit Complete just because some pages were scanned.

## 10. Technical error log

For every failure, log:
- journey and checkpoint;
- last successful page URL and type;
- UI action attempted and expected outcome;
- destination only if observed from UI;
- error category (browser/context closure, timeout, navigation, application, authentication, third-party flow, missing feature, safety stop, limit reached);
- sanitized error and relevant call-log excerpt;
- browser/context/page health;
- recovery action and retry count;
- recovery outcome;
- page status and coverage impact;
- next eligible journey attempted.

Do not include secrets, storage state, cookies, authorization headers, personal details, or payment information. Continue independent journeys after failures where safe. If the browser cannot be recovered within the retry budget, report technical incompleteness.

## 11. Consolidated Markdown report

Produce one Markdown report using the configured filename. Include these sections in this order:

1. **Audit Metadata** — application, environment, base URL, audit timestamp/timezone, browser/Playwright/axe versions if available, WCAG tags, configured limits, report filename.
2. **Accessibility Health Check** — score/grade or reason not calculated, audit status, axe status, severity counts, incomplete/manual checks, pages/states scanned, scoring formula and calculation.
3. **Executive Summary** — overall status; unique pages discovered, attempted, loaded, scanned, partially scanned, failed, and not reached; key findings; severity distribution; important technical failures and limitations.
4. **Journey Coverage** — every required journey, checkpoints completed, status, evidence, and blocker.
5. **Page Inventory** — observed URL, page type, navigation path, auth requirement, attempt count, status, axe status, and evidence.
6. **Technical Errors and Recovery Log** — sanitized error, last successful checkpoint, retries/recovery, and coverage impact.
7. **Accessibility Findings** — deduplicated findings, severity, supported WCAG/axe mapping, evidence, affected pages, and remediation.
8. **Manual-Review Items** — potential issues not conclusively verified by automation.
9. **Checkout and Order Status** — product selected; variant handled; add-to-bag verified; cart reached/contents verified; checkout reached/scanned; form states tested; payment UI inspected; payment submission/order outcome; confirmation page reached/scanned or exact blocker. Never expose sensitive order/customer/payment details.
10. **Coverage Limitations and Next Steps** — unscanned types/states and exact reasons; technical blockers; prioritized follow-up.

### Reporting integrity
- Keep counts consistent. Use the page inventory as the source of truth for page totals.
- Distinguish discovered, attempted, loaded, scanned, and failed; these are not interchangeable.
- Count a page as scanned only if it reached a stable state and axe results were saved.
- Do not infer URLs, findings, selectors, or outcomes. Write “not captured” when unavailable.
- Do not claim checkout, payment, order placement, or confirmation was tested unless that exact stage was reached.
- Do not claim full WCAG conformance based on automated checks.
- Do not leave `null` in filenames. Use a safe deterministic fallback if the sequence number is unavailable and note it.
- The HTML renderer, if used later, must faithfully reflect this Markdown source; rendering is not a new test run.
- Do not report overall `Complete` when required coverage remains untested without a documented reason.

## 12. Completion gate

Before finalizing, verify each applicable item:
- [ ] Base page reached and scanned, or exact blocker documented.
- [ ] Header/navigation and footer discovery attempted.
- [ ] Representative category/listing pages scanned within limits.
- [ ] Search and no-results state tested if available.
- [ ] Representative product pages scanned within limits.
- [ ] Required variant and add-to-bag outcome verified if supported.
- [ ] Cart opened through UI and contents verified if supported.
- [ ] Checkout selected from cart through UI, or exact blocker documented.
- [ ] Safe checkout fields/states inspected if checkout was reached.
- [ ] Payment UI inspection and payment submission status recorded separately.
- [ ] Test order status recorded; never more than one authorized test order.
- [ ] Confirmation page scanned after verified successful test order, or blocker documented.
- [ ] Authentication attempted when configured; protected journeys only after verification.
- [ ] Account and wishlist attempted when available and eligible.
- [ ] Interactive states inspected within scope.
- [ ] Every uncompleted page type has a status and reason.
- [ ] Health score and severity counts reconcile with deduplicated findings.
- [ ] All report counts reconcile across sections.
- [ ] Evidence references exist and sensitive information is redacted.
- [ ] Markdown report exists and is non-empty.

If only a few pages were scanned, do not call the audit comprehensive. Explain why it stopped and list every unmet coverage item.

## 13. Final execution rule

Use the configured application values, discover UI controls dynamically, preserve browser/context integrity, follow scan limits and safety boundaries, prioritize the real cart-to-checkout path through UI interactions, continue other safe journeys after failures, execute every step in the Mandatory calculation procedure in Section 9, and produce one consolidated sanitized Markdown report. Never include secrets.

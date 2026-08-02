# OpenEMR Revenue Cycle and X12 EDI Defect Candidates

The register of code in this subsystem that already looks wrong, one entry per suspicion, each anchored to the expression it lives in, each carrying the symptom an operator or a payer would see and a concrete way to prove it.

**Nothing here is fixed.** This document records suspicions and proposes verifications; it repairs nothing. No PHP file was edited to produce it, no comment or docblock was added to any source file, and no test was written - where an entry names a test, it *describes* the test that would prove the defect and stops there. The security-sensitive observations in [Appendix A Security Sensitive Observations](#appendix-a-security-sensitive-observations) are flagged with a citation and a severity and are deliberately not analysed at exploit depth and deliberately not remediated.

**Scope and sources.** This is a register of suspected defects in OpenEMR's revenue cycle and its handling of X12, the electronic data interchange standard that United States healthcare payers, providers and clearinghouses use to exchange claims, eligibility questions and payments. It was traced expression by expression from the outbound transport tracker, the batch writer, the two claim generators, the remittance parser, the accounts-receivable poster, the modern accounting helper, the legacy remittance renderer and history input layer, the trading-partner model, the accounts-receivable decision lines inside the explanation-of-benefits posting screen, and the schema in `sql/database.sql`. It is a register and nothing else: it does not narrate the revenue cycle, which is [claim-lifecycle.md](claim-lifecycle.md), it does not state what the code decides about money, which is [business-rules.md](business-rules.md), and it does not classify files by refactor risk, which is [upgrade-risk-map.md](upgrade-risk-map.md). The conventions every entry uses - the citation format, the two claim classes and their confidence vocabulary, and the source-of-truth ordering - are defined once in [README.md](README.md) and are applied here without variation and without restatement.

**Provenance, and what was not run.** Every line anchor below is relative to branch `master` at head commit `b7a7e690e419de3451740f995b768a8e8e5fba87`, on project version 8.3.0-dev (`version.php:L17-L20`) with database schema version 541 (`version.php:L33`). **No part of this subsystem was executed and no test suite was run.** PHP and Composer are not installed in the authoring environment, so every entry rests on static reading of file contents at that commit. That is precisely why the `Verification:` field of every entry *proposes* a way to prove the defect rather than reporting one: **no reproduction in this document was performed, and no entry should be read as though one had been.** A symptom described here is what the cited code path would produce, not what was observed on a screen.

**One boundary worth stating.** A register of suspicions is not a register of findings. Within every entry the `Evidence:` field states what the code *does*, VERIFIED and cited, and the `Why it looks wrong:` field states the claim that it is *wrong*, which is an inference and is labelled as one with a confidence. Those two are kept apart on purpose. A reader who disagrees with an inference can still rely on the evidence beneath it.

## Table of Contents

- [How to Read an Entry](#how-to-read-an-entry)
    - [The entry template](#the-entry-template)
    - [The two field requirements that are constraints](#the-two-field-requirements-that-are-constraints)
    - [How the categories are drawn](#how-the-categories-are-drawn)
    - [The X12 vocabulary used in this document](#the-x12-vocabulary-used-in-this-document)
- [Severity Summary](#severity-summary)
- [Category 1 Wrong Comparisons](#category-1-wrong-comparisons)
- [Category 2 Precision and Truncation in Monetary Math](#category-2-precision-and-truncation-in-monetary-math)
- [Category 3 State Leaking Across Loop Iterations](#category-3-state-leaking-across-loop-iterations)
- [Category 4 Dead Branches Masking Logic](#category-4-dead-branches-masking-logic)
- [Category 5 Unvalidated Assumptions About Segment Ordering](#category-5-unvalidated-assumptions-about-segment-ordering)
- [Category 6 Resource and Lifecycle Handling](#category-6-resource-and-lifecycle-handling)
- [Category 7 Dead Configuration](#category-7-dead-configuration)
- [Appendix A Security Sensitive Observations](#appendix-a-security-sensitive-observations)
- [Appendix B Historical Precedent](#appendix-b-historical-precedent)
- [Related Documents](#related-documents)
- [Documentation Attribution](#documentation-attribution)

## How to Read an Entry

Read the `Severity:` field first, then the `Observable symptom:` field. Those two together decide whether an entry is worth your afternoon. Read `Why it looks wrong:` before you argue with the entry, because that field is where the inference lives and it is the only field a reader can reasonably dispute. Follow the cross-reference when one is present: where an entry is the suspect implementation of a rule the subsystem actually relies on, it links to [business-rules.md](business-rules.md), and a rule and the suspicion about it are not meant to be read apart.

### The entry template

Every entry carries these six fields, in this order, none omitted and none empty:

```text
### DC-<n> <short defect name>

Suspected defect:  <what appears wrong>
Evidence:          <path/to/file.php:L120-L145>
Why it looks wrong: <the reasoning, comparing intent to expression>
Observable symptom: <what an operator or payer would actually see>
Verification:      <a named test file + phpunit config, OR a repro:
                    screen, input, expected vs actual>
Severity:          CRITICAL | HIGH | MEDIUM
```

Each entry below renders that template as a labelled list rather than as preformatted text, so that the fields survive markdown rendering and the citations stay selectable. The field names, their order and the requirement that all six be present are unchanged. This matches the rendering decision the sibling register already made for its own template in [business-rules.md](business-rules.md).

Every entry closes with a two- or three-line excerpt of the expression at issue, followed by its citation and, where it helps, one sentence naming the line that completes the picture. The excerpt exists to show the shape of the expression, never to replace the citation: this document cites code, it does not copy it, and a reader is expected to open the cited range. Excerpts are reproduced verbatim from the cited lines with their leading indentation removed and nothing else changed.

### The two field requirements that are constraints

- **Every entry states an observable symptom.** A finding that cannot be tied to something an operator, a payer or a patient would see is a code-quality observation, not a defect candidate, and belongs in [upgrade-risk-map.md](upgrade-risk-map.md) instead. Several symptoms below are that **the operator sees nothing**, and that is an answer rather than a missing field: a silent wrong outcome is worse than a loud one, and writing it down is the point of the register.
- **Every entry proposes either a concrete test or a concrete reproduction.** A named test names the file it would live in **and the configuration it would run under**, because that is not a formality here: 13 of the 16 billing test files live under `tests/Tests/Isolated/Billing/` and execute only under the secondary configuration, which is the only one declaring that directory as a suite (`phpunit-isolated.xml:L65-L67`) and which continuous integration invokes by name (`.github/workflows/isolated-tests.yml:L50`); the primary configuration's suite list does not reference that directory at all (`phpunit.xml:L43-L93`). The other three live under `tests/Tests/Services/Billing/`, which the primary configuration does cover (`phpunit.xml:L67-L69`). A named reproduction names the screen, the input, and the expected outcome against the actual one. No entry in this register asks anyone to read the logic and form a view.

### How the categories are drawn

The seven categories below are the classes of defect this subsystem actually produces, and each entry sits in exactly one of them. Where an entry could defensibly sit in two, it is placed by the mechanism that makes it wrong rather than by its consequence, and the neighbouring entry is named in the text. Two category scopes are worth stating because their titles are narrower than their contents:

- **Category 5** covers unvalidated positional and structural assumptions generally: the ordering and arity of segments inside a transaction, and equally the assumption that an envelope trailer matches the header its own code wrote, or that a value a payer hands back has the shape it was sent in.
- **Category 6** covers anything about the lifetime of a resource or a request: file handles, remote sessions, database transactions, and requests that end part-way through a run.

### The X12 vocabulary used in this document

X12 knowledge is not assumed. Every term used below is expanded here so that the entries stay short, and each is expanded again at its first use in prose.

**Envelope terms.** An X12 file is wrapped in nested envelopes. **ISA** is the interchange control header, the outermost wrapper, whose numbered elements run ISA01 to ISA16; **ISA13** is the interchange control number and **ISA15** the usage indicator, where `T` means test data and `P` means production data. **IEA** is the matching interchange trailer. **GS** is the functional group header, whose **GS06** is the group control number, and **GE** is its trailer. **ST** is the transaction set header, whose **ST02** is the transaction set control number, and **SE** is its trailer, whose **SE01** counts the segments in the transaction set. **BHT** is the beginning of hierarchical transaction segment; its **BHT03** is a reference identification and its **BHT06** a transaction type code, where `CH` means chargeable and `RP` means reporting. A **loop** is a repeatable group of segments identified by a number such as 2100 or 2110.

**Transaction set numbers.** **837** is a claim, in two flavours: **837P** professional and **837I** institutional. **835** is remittance advice, a payer's statement of what it paid and why, also called an **ERA** for electronic remittance advice; its human-readable equivalent is an **EOB**, an explanation of benefits. **270** and **271** are an eligibility request and its response. **276** and **277** are a claim-status inquiry and its response. **278** is a services review, that is, an authorisation. **997** and **999** are two distinct acknowledgement transaction sets: the 997 functional acknowledgement reports acceptance or rejection against the generic X12 syntax, and the 999 implementation acknowledgement is its 5010-era replacement, reporting errors against an implementation guide as well.

**Segment identifiers.** **CLP** carries claim-level payment information; **CLP01** is the provider's own claim identifier and the later elements carry the charged, paid and patient-responsibility amounts. **SVC** carries service-line payment information. **CAS** carries a claim or service adjustment as a group code, a reason code and an amount; the group codes named below are `PR` for patient responsibility, `CO` for contractual obligation and `CR` for correction and reversal. **PLB** carries a provider-level adjustment, which belongs to the provider rather than to any one claim. **MIA** carries Medicare inpatient adjudication information. **AMT** is an amount, **QTY** a quantity, **NM1** a name, **DTM** a date, **REF** a reference identifier, **TRN** a trace number, **LX** a service-line counter and **PWK** a paperwork or attachment reference.

**Other terms.** **A/R** is accounts receivable: what has been billed and not yet settled. In this schema a deposit is a row of `ar_session` and a ledger line is a row of `ar_activity`. **SFTP** is the SSH File Transfer Protocol, the one built-in transport this subsystem uses to hand a batch file to a trading partner.

## Severity Summary

Sixty-one entries. Every one appears once in this table and once as an entry below.

| Severity | Count | What it means here |
|----------|------:|--------------------|
| CRITICAL | 7 | Money, claim routing or claim data reaches a wrong destination, or a wrong amount is recorded, and nothing on any screen identifies the state that persists |
| HIGH | 23 | A wrong or absent outcome that an operator could eventually notice, or a wrong amount confined to one claim |
| MEDIUM | 31 | A wrong outcome that is visible, bounded, or reachable only on an unusual input |

The seven CRITICAL entries share one property that is worth naming before the register begins: in all seven the wrong outcome is committed to the database or handed to a payer, and in none of them does what the screen shows identify the state that persists. What the screen shows does differ, and the difference is what matters when triaging them, so the seven are listed here by that difference rather than by number.

In three, the failure is not reported at all and the wrong value is presented as though it were correct. [DC-19](#dc-19-patient-and-provider-fields-survive-from-one-claim-to-the-next) labels an unmatched claim with the previous claim's patient name, on the screen an operator uses to decide where money goes. [DC-20](#dc-20-one-batch-file-is-queued-once-for-every-partner-in-the-batch) produces a run that looks like any other successful one. [DC-40](#dc-40-the-claim-identifier-is-recovered-by-interpolating-a-payer-supplied-value-into-a-query) posts a payment against another patient's encounter, or reports a claim as unmatched without saying why.

In one, the failure is reported but not as a failure. [DC-10](#dc-10-provider-level-adjustments-are-excluded-from-the-ledger-and-included-in-the-balance-test) surfaces the provider-level adjustment only as a note phrased as not claim specific, beside a ledger short by that amount and a balance test that calls the remittance balanced.

In three, an explicit message is shown and the wrong state persists behind it. [DC-27](#dc-27-a-failed-upload-is-recorded-and-then-overwritten-with-success) asserts a success badge while carrying the transport's own failure text, reachable only by expanding a row that presents itself as successful. [DC-28](#dc-28-a-medicare-inpatient-adjudication-segment-ends-the-parse-where-it-appears) appends an unknown-segment message and, on a live run, raises an alert naming the check as not fully distributed, without either message saying which segment stopped the parse or that the claims after it were never read. [DC-49](#dc-49-a-malformed-interchange-header-ends-the-request-after-the-claim-was-marked-billed) ends the page with an explicit interchange-header error while leaving every claim in the run marked billed against a batch file that will never exist.

By category: 9 wrong comparisons, 9 precision and truncation, 8 state leaking across iterations, 13 dead branches, 9 unvalidated assumptions, 8 resource and lifecycle, 5 dead configuration.

| ID | Short name | Category | Severity |
|----|------------|----------|----------|
| [DC-1](#dc-1-a-colon-in-the-first-position-leaves-the-modifier-attached-to-the-code) | A colon in the first position leaves the modifier attached to the code | 1 Wrong comparisons | HIGH |
| [DC-2](#dc-2-the-next-payer-level-is-chosen-by-an-unparenthesised-mixed-condition) | The next payer level is chosen by an unparenthesised mixed condition | 1 Wrong comparisons | HIGH |
| [DC-3](#dc-3-the-last-claim-for-a-partner-is-chosen-by-an-identity-comparison-across-types) | The last claim for a partner is chosen by an identity comparison across types | 1 Wrong comparisons | HIGH |
| [DC-4](#dc-4-a-deposit-counts-as-fully-allocated-only-for-two-exact-strings) | A deposit counts as fully allocated only for two exact strings | 1 Wrong comparisons | MEDIUM |
| [DC-5](#dc-5-the-duplicate-deposit-probe-searches-for-a-reference-the-writer-never-stores) | The duplicate deposit probe searches for a reference the writer never stores | 1 Wrong comparisons | HIGH |
| [DC-6](#dc-6-a-seven-character-procedure-code-is-split-into-a-code-and-a-modifier) | A seven character procedure code is split into a code and a modifier | 1 Wrong comparisons | MEDIUM |
| [DC-7](#dc-7-the-crossover-claim-row-is-chosen-by-a-loose-inequality) | The crossover claim row is chosen by a loose inequality | 1 Wrong comparisons | MEDIUM |
| [DC-8](#dc-8-envelope-validity-is-decided-by-truth-testing-positions-against-an-assumed-terminator) | Envelope validity is decided by truth-testing positions against an assumed terminator | 1 Wrong comparisons | MEDIUM |
| [DC-9](#dc-9-a-negative-contractual-obligation-is-inverted-after-a-strict-string-test) | A negative contractual obligation is inverted after a strict string test | 1 Wrong comparisons | MEDIUM |
| [DC-10](#dc-10-provider-level-adjustments-are-excluded-from-the-ledger-and-included-in-the-balance-test) | Provider level adjustments are excluded from the ledger and included in the balance test | 2 Precision and truncation | CRITICAL |
| [DC-11](#dc-11-forced-balancing-overwrites-the-service-payment-the-payer-reported) | Forced balancing overwrites the service payment the payer reported | 2 Precision and truncation | HIGH |
| [DC-12](#dc-12-the-balancing-residue-is-attributed-to-a-group-code-the-payer-never-sent) | The balancing residue is attributed to a group code the payer never sent | 2 Precision and truncation | MEDIUM |
| [DC-13](#dc-13-the-saved-institutional-claim-form-is-byte-capped-and-truncated-in-silence) | The saved institutional claim form is byte-capped and truncated in silence | 2 Precision and truncation | MEDIUM |
| [DC-14](#dc-14-the-deposit-balance-check-is-a-float-subtraction-reported-only-in-a-browser-alert) | The deposit balance check is a float subtraction reported only in a browser alert | 2 Precision and truncation | HIGH |
| [DC-15](#dc-15-a-non-numeric-adjustment-amount-becomes-zero-in-two-places) | A non numeric adjustment amount becomes zero in two places | 2 Precision and truncation | HIGH |
| [DC-16](#dc-16-the-running-invoice-balance-alternates-between-a-formatted-string-and-float-arithmetic) | The running invoice balance alternates between a formatted string and float arithmetic | 2 Precision and truncation | MEDIUM |
| [DC-17](#dc-17-a-provider-level-adjustment-amount-is-formatted-from-an-element-the-loop-never-proves-exists) | A provider level adjustment amount is formatted from an element the loop never proves exists | 2 Precision and truncation | MEDIUM |
| [DC-18](#dc-18-the-deposit-writer-omits-seven-columns-the-schema-declares-not-null) | The deposit writer omits seven columns the schema declares not null | 2 Precision and truncation | HIGH |
| [DC-19](#dc-19-patient-and-provider-fields-survive-from-one-claim-to-the-next) | Patient and provider fields survive from one claim to the next | 3 State leaking | CRITICAL |
| [DC-20](#dc-20-one-batch-file-is-queued-once-for-every-partner-in-the-batch) | One batch file is queued once for every partner in the batch | 3 State leaking | CRITICAL |
| [DC-21](#dc-21-the-production-date-is-backfilled-once-and-never-cleared) | The production date is backfilled once and never cleared | 3 State leaking | HIGH |
| [DC-22](#dc-22-the-line-ending-strip-is-applied-to-the-wrong-side-of-the-split) | The line ending strip is applied to the wrong side of the split | 3 State leaking | HIGH |
| [DC-23](#dc-23-two-repeat-suppression-variables-are-never-reset-between-claims) | Two repeat suppression variables are never reset between claims | 3 State leaking | MEDIUM |
| [DC-24](#dc-24-the-error-flag-accumulates-across-the-service-lines-of-a-claim) | The error flag accumulates across the service lines of a claim | 3 State leaking | HIGH |
| [DC-25](#dc-25-the-check-date-computed-in-the-first-pass-is-discarded) | The check date computed in the first pass is discarded | 3 State leaking | MEDIUM |
| [DC-26](#dc-26-the-delimiter-probe-can-only-run-on-the-first-segment-of-the-interchange) | The delimiter probe can only run on the first segment of the interchange | 3 State leaking | MEDIUM |
| [DC-27](#dc-27-a-failed-upload-is-recorded-and-then-overwritten-with-success) | A failed upload is recorded and then overwritten with success | 4 Dead branches | CRITICAL |
| [DC-28](#dc-28-a-medicare-inpatient-adjudication-segment-ends-the-parse-where-it-appears) | A Medicare inpatient adjudication segment ends the parse where it appears | 4 Dead branches | CRITICAL |
| [DC-29](#dc-29-the-comment-above-the-success-write-describes-a-transition-the-code-does-not-make) | The comment above the success write describes a transition the code does not make | 4 Dead branches | MEDIUM |
| [DC-30](#dc-30-the-upload-error-status-is-declared-under-a-misspelled-identifier) | The upload error status is declared under a misspelled identifier | 4 Dead branches | MEDIUM |
| [DC-31](#dc-31-the-dry-run-branch-of-the-deposit-writer-returns-nothing) | The dry run branch of the deposit writer returns nothing | 4 Dead branches | MEDIUM |
| [DC-32](#dc-32-the-institutional-transaction-type-can-never-be-reporting) | The institutional transaction type can never be reporting | 4 Dead branches | MEDIUM |
| [DC-33](#dc-33-the-institutional-claim-can-never-carry-the-alternate-payer-identifier) | The institutional claim can never carry the alternate payer identifier | 4 Dead branches | HIGH |
| [DC-34](#dc-34-a-claim-level-name-segment-matches-every-qualifier-and-does-nothing) | A claim level name segment matches every qualifier and does nothing | 4 Dead branches | MEDIUM |
| [DC-35](#dc-35-the-remittance-charge-helper-ignores-three-of-its-ten-parameters) | The remittance charge helper ignores three of its ten parameters | 4 Dead branches | HIGH |
| [DC-36](#dc-36-the-dry-run-path-of-the-re-billing-helper-does-nothing-and-says-nothing) | The dry run path of the re billing helper does nothing and says nothing | 4 Dead branches | MEDIUM |
| [DC-37](#dc-37-the-transaction-set-header-and-its-trailer-are-governed-by-different-conditions) | The transaction set header and its trailer are governed by different conditions | 4 Dead branches | MEDIUM |
| [DC-38](#dc-38-an-unrecognised-action-button-dereferences-a-null-task) | An unrecognised action button dereferences a null task | 4 Dead branches | HIGH |
| [DC-39](#dc-39-the-tertiary-payer-is-never-queued-from-the-posting-screen) | The tertiary payer is never queued from the posting screen | 4 Dead branches | HIGH |
| [DC-40](#dc-40-the-claim-identifier-is-recovered-by-interpolating-a-payer-supplied-value-into-a-query) | The claim identifier is recovered by interpolating a payer supplied value into a query | 5 Unvalidated assumptions | CRITICAL |
| [DC-41](#dc-41-the-interchange-header-rebuild-indexes-four-elements-it-never-proves-exist) | The interchange header rebuild indexes four elements it never proves exist | 5 Unvalidated assumptions | HIGH |
| [DC-42](#dc-42-the-reference-rewrite-uses-an-unchecked-search-result-as-an-offset) | The reference rewrite uses an unchecked search result as an offset | 5 Unvalidated assumptions | HIGH |
| [DC-43](#dc-43-a-claim-level-adjustment-that-arrives-after-a-service-line-lands-on-that-line) | A claim level adjustment that arrives after a service line lands on that line | 5 Unvalidated assumptions | HIGH |
| [DC-44](#dc-44-the-branch-chain-tests-elements-it-never-proves-the-segment-carries) | The branch chain tests elements it never proves the segment carries | 5 Unvalidated assumptions | MEDIUM |
| [DC-45](#dc-45-unterminated-bytes-after-the-interchange-trailer-are-discarded-and-the-parse-reports-success) | Unterminated bytes after the interchange trailer are discarded and the parse reports success | 5 Unvalidated assumptions | MEDIUM |
| [DC-46](#dc-46-the-institutional-envelope-trailers-contradict-their-own-headers) | The institutional envelope trailers contradict their own headers | 5 Unvalidated assumptions | MEDIUM |
| [DC-47](#dc-47-the-interchange-trailer-is-written-even-when-no-functional-group-was-opened) | The interchange trailer is written even when no functional group was opened | 5 Unvalidated assumptions | MEDIUM |
| [DC-48](#dc-48-the-transaction-reference-is-a-fixed-literal-at-generation-and-after-the-batch-rewrite) | The transaction reference is a fixed literal at generation and after the batch rewrite | 5 Unvalidated assumptions | HIGH |
| [DC-49](#dc-49-a-malformed-interchange-header-ends-the-request-after-the-claim-was-marked-billed) | A malformed interchange header ends the request after the claim was marked billed | 6 Resource and lifecycle | CRITICAL |
| [DC-50](#dc-50-neither-remittance-entry-point-closes-the-file-it-opened) | Neither remittance entry point closes the file it opened | 6 Resource and lifecycle | MEDIUM |
| [DC-51](#dc-51-the-claim-version-is-drawn-by-an-unlocked-aggregate-inside-a-transaction) | The claim version is drawn by an unlocked aggregate inside a transaction | 6 Resource and lifecycle | HIGH |
| [DC-52](#dc-52-the-ledger-sequence-number-is-drawn-the-same-way-and-the-code-says-so) | The ledger sequence number is drawn the same way and the code says so | 6 Resource and lifecycle | HIGH |
| [DC-53](#dc-53-the-transport-reads-the-claim-file-after-a-fallback-without-rechecking-it) | The transport reads the claim file after a fallback without rechecking it | 6 Resource and lifecycle | MEDIUM |
| [DC-54](#dc-54-a-row-the-transport-has-claimed-is-never-released-or-retried) | A row the transport has claimed is never released or retried | 6 Resource and lifecycle | MEDIUM |
| [DC-55](#dc-55-deposit-rows-are-committed-by-the-first-pass-before-the-file-is-validated) | Deposit rows are committed by the first pass before the file is validated | 6 Resource and lifecycle | HIGH |
| [DC-56](#dc-56-the-results-page-closes-the-document-before-the-download-script-is-written) | The results page closes the document before the download script is written | 6 Resource and lifecycle | MEDIUM |
| [DC-57](#dc-57-five-trading-partner-columns-have-no-consumer) | Five trading partner columns have no consumer | 7 Dead configuration | MEDIUM |
| [DC-58](#dc-58-the-attachment-segment-names-a-transport-that-was-never-built) | The attachment segment names a transport that was never built | 7 Dead configuration | HIGH |
| [DC-59](#dc-59-a-partner-model-property-has-no-backing-column-and-is-still-editable) | A partner model property has no backing column and is still editable | 7 Dead configuration | MEDIUM |
| [DC-60](#dc-60-the-processing-format-is-stored-per-claim-and-branches-on-nothing) | The processing format is stored per claim and branches on nothing | 7 Dead configuration | MEDIUM |
| [DC-61](#dc-61-a-component-separator-is-assigned-and-never-read-in-the-check-parse) | A component separator is assigned and never read in the check parse | 7 Dead configuration | MEDIUM |

## Category 1 Wrong Comparisons

Nine entries. Each one is an operator or an operand that does not mean what the surrounding code assumes: a truthiness test standing in for an identity test, an identity test standing in for an equality test, a loose test where a strict one was needed, or a string literal standing in for a number.

### DC-1 A colon in the first position leaves the modifier attached to the code

- **Suspected defect:** The result of `strpos()` is tested for truthiness instead of against `false`, so a procedure code whose colon sits at offset zero is treated as carrying no modifier, and the whole string including the colon is posted as the code.
- **Evidence:** `src/Billing/SLEOB.php:L136-L140`. VERIFIED: `$tmp = strpos((string) $code, ':');` is followed by `if ($tmp) {`, and only inside that branch are `$codeonly` and `$modifier` split apart. When the colon is the first character `strpos()` returns integer `0`, which is falsy, so `$codeonly` keeps the value assigned at `:L134` and `$modifier` keeps the empty string assigned at `:L135`. The identical three-line shape is repeated twice more in the same file, at `:L188-L192` and `:L227-L231`, so all three accounts-receivable posting helpers share the behaviour.
- **Why it looks wrong:** The two lines above the test exist only to provide a fallback for the no-colon case, which shows the author's intent was to detect the presence of a separator; a separator at offset zero is present, and the test reports it absent. INFERRED (confidence: High). Basis: the `false`-versus-`0` distinction is the documented reason `strpos()` has a strict-comparison idiom, and the surrounding code has no other reading under which a leading colon should be retained.
- **Observable symptom:** A ledger line in `ar_activity` whose `code` column holds a leading colon followed by a modifier, for example `:59`, with an empty procedure code. On the invoice screen the charge does not match any service line, and any later matching by code and modifier misses it.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` covering `SLEOB::arPostPayment` with `$code = ':59'`, asserting the written `code` and `modifier` columns; it runs under `phpunit-isolated.xml`. Alternatively reproduce on the explanation-of-benefits posting screen at `interface/billing/sl_eob_process.php` by posting a remittance whose service line carries an empty procedure code with a modifier: expected is a `code` of the procedure and a `modifier` of `59`, actual is a `code` of `:59` and no modifier.
- **Severity:** HIGH
- **Registered as a rule:** [BR-F2](business-rules.md#br-f2-a-code-whose-colon-is-its-first-character-keeps-its-modifier).

```php
$tmp = strpos((string) $code, ':');
if ($tmp) {
```

That is `src/Billing/SLEOB.php:L136-L137`.

### DC-2 The next payer level is chosen by an unparenthesised mixed condition

- **Suspected defect:** The guard that decides whether an encounter advances to the next payer level mixes `&&` and `||` with no parentheses, so it reads as though a non-empty watermark were required when the third arm reaches the increment without one. For an encounter already billed to the tertiary level the guard correctly declines to advance, and the code below it then re-queues the claim to the level it has already been billed to rather than stopping.
- **Evidence:** `src/Billing/SLEOB.php:L285-L287`. VERIFIED: `$new_payer_type = 0 + $ferow['last_level_billed'];` is followed by `if ($new_payer_type < 3 && !empty($ferow['last_level_billed']) || $new_payer_type == 0) {` and then `++$new_payer_type;`. PHP binds `&&` tighter than `||`, so the expression evaluates as `($new_payer_type < 3 && !empty(...)) || $new_payer_type == 0`. VERIFIED: when the guard declines, the unchanged level is passed to `arGetPayerID()` at `src/Billing/SLEOB.php:L290`, and the claim is either re-queued to the payer that lookup returns, at `src/Billing/SLEOB.php:L292-L296`, or reopened with no payer, at `src/Billing/SLEOB.php:L297-L301`.
- **Why it looks wrong:** Two readings of the same line disagree. The `!empty()` arm suggests the author wanted an encounter with no billing watermark to be excluded, yet the trailing `== 0` arm admits exactly that case, which makes the `!empty()` arm unreachable for the only input it was written to reject. Separately, `0 + $ferow['last_level_billed']` coerces a non-numeric watermark to zero rather than rejecting it, so a corrupt value takes the same path as a never-billed encounter. VERIFIED: a second copy of the same condition exists at `library/payment.inc.php:L212-L215`, over `last_level_closed` rather than `last_level_billed` and comparing with `<= 3` rather than `< 3`, and that copy gates its own call to this helper on the payer identifier the incremented level resolves to, at `library/payment.inc.php:L217-L219`. INFERRED (confidence: Medium). Basis: the two arms are individually coherent and jointly redundant, which is the signature of a condition edited twice rather than designed once, and the near-duplicate elsewhere differs in exactly the comparison operator; the intent behind the `!empty()` arm cannot be recovered from the code.
- **Observable symptom:** A secondary or a tertiary payer's remittance cannot reach this line from the posting screen, so the entry does not offer that reproduction. VERIFIED: that screen calls the helper only inside a gate requiring the remittance being posted to be the primary payer's and a secondary policy to exist, at `interface/billing/sl_eob_process.php:L717-L718`, which is registered separately as [DC-39](#dc-39-the-tertiary-payer-is-never-queued-from-the-posting-screen). VERIFIED: a watermark of 3 reaches the line by two other routes. A primary payer's remittance posted against an encounter already billed to tertiary satisfies that gate, and on that route the caller then prints the fixed sentence about the claim being re-queued for secondary paper billing, at `interface/billing/sl_eob_process.php:L721-L725`, unless the remittance reported a crossover, at `interface/billing/sl_eob_process.php:L720`, while the payer written is the tertiary one; that mismatch between the sentence and the write is set out stage by stage in [claim-lifecycle.md](claim-lifecycle.md#stage-s12-secondary-and-tertiary-payer-setup). The second route is the invoice screen, whose "Needs secondary billing" checkbox at `interface/billing/sl_eob_invoice.php:L608` reaches the helper with no payer-level gate at all, at `interface/billing/sl_eob_invoice.php:L405` and again at `interface/billing/sl_eob_invoice.php:L432`, and which prints nothing about a payer either way. On an encounter whose watermark is already 3, the claim is not left alone: the tertiary policy row is looked up again and the claim is re-queued to that same payer, so every charge row for the encounter has `billed` reset to 0 at `src/Billing/BillingUtilities.php:L1598`, `bill_process` set to 5 at `src/Billing/BillingUtilities.php:L1608` and the tertiary payer written back at `src/Billing/BillingUtilities.php:L1633`, and a new `claims` version row is inserted at `src/Billing/BillingUtilities.php:L1677-L1708`. VERIFIED: the watermark is never advanced by this path, because the only write to `last_level_billed` requires a billed status of 2, at `src/Billing/BillingUtilities.php:L1720-L1724`, so the encounter reappears in the billing work list addressed to a payer that has already adjudicated it, and repeating the action repeats the outcome and adds another claim version each time. Where no tertiary policy row exists the claim is reopened with no payer instead, at `src/Billing/SLEOB.php:L297-L301`, and nothing on screen distinguishes the two outcomes.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arSetupSecondary` with `$debug` false against a `form_encounter` row with `last_level_billed = 3`, then a second case with `last_level_billed = ''`, asserting `billing.billed`, `billing.payer_id`, the new `claims` row's `payer_type` and `form_encounter.last_level_billed`; it runs under `phpunit-isolated.xml`. For the first case the expected result is no re-queue or an explicit refusal, and the actual result is `billed = 0` with the tertiary payer written back, a claim version at payer type 3 and the watermark still 3. For the second case the expected and actual results agree, an advance to payer type 1, which is what makes the `!empty()` arm dead rather than protective. The reproduction is the invoice screen: open an encounter already billed to tertiary, tick "Needs secondary billing", save with dry run off, and inspect `billing` and `claims`; the expected outcome is that a fully billed encounter is not re-queued, the actual outcome is a fresh claim version addressed to the tertiary payer.
- **Severity:** HIGH

```php
$new_payer_type = 0 + $ferow['last_level_billed'];
if ($new_payer_type < 3 && !empty($ferow['last_level_billed']) || $new_payer_type == 0) {
```

That is `src/Billing/SLEOB.php:L285-L286`. PHP binds the conjunction tighter than the disjunction, so the third arm stands alone.

### DC-3 The last claim for a partner is chosen by an identity comparison across types

- **Suspected defect:** The claim that closes a partner's transaction set is selected with `===` between a value that arrives from an HTTP post and a value that arrives from a database column, so the marking depends on both sides happening to be strings.
- **Evidence:** `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L134`. VERIFIED: the loop at `:L133-L137` tests `if ($claim->getPartner() === $row['id']) {` and only then records `$lastClaim`, and `:L138-L139` sets `setIsLast(true)` on whatever it recorded. `getPartner()` returns the property assigned at `src/Billing/BillingProcessor/BillingClaim.php:L119` from `$partner_and_payor['partner']`, which the processor takes straight out of the posted claim array at `src/Billing/BillingProcessor/BillingProcessor.php:L106`. `$row['id']` is the trading-partner primary key read from `x12_partners`.
- **Why it looks wrong:** A strict comparison is being used across a type boundary the code does not control on either side: one operand is request input and the other is a database integer column. Every other partner comparison in the same file uses `$row['id']` as an array key, where PHP normalises the type for you, which is why the mismatch has no other visible effect. INFERRED (confidence: Medium). Basis: the comparison is correct only under the current string-versus-string coincidence, and nothing in either operand's contract guarantees it.
- **Observable symptom:** If the comparison ever fails, no claim is marked last for that partner, so the transaction set trailer is never emitted: `src/Billing/X125010837P.php:L1612-L1615` writes the SE segment, the transaction set trailer, only when the flag derived from `getIsLast()` is true or the per-payer switch is off. The clearinghouse rejects the interchange with a 997 or 999 acknowledgement naming a missing SE, and the operator sees a successful-looking batch on screen. The condition asymmetry itself is [DC-37](#dc-37-the-transaction-set-header-and-its-trailer-are-governed-by-different-conditions).
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` that builds two `BillingClaim` objects with `partner` set to the integer `3` and to the string `'3'`, runs the loop against a partner row whose `id` is `'3'`, and asserts `getIsLast()` on each; it runs under `phpunit-isolated.xml`.
- **Severity:** HIGH

```php
if ($claim->getPartner() === $row['id']) {
```

That is `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L134`. The loop around it is `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L133-L137`.

### DC-4 A deposit counts as fully allocated only for two exact strings

- **Suspected defect:** Whether a deposit is fully allocated is decided by comparing one `decimal(12,2)` column against the two string literals `'0'` and `'0.00'` with `===`. Two things are wrong at once: the operator makes the answer depend on the exact representation the database driver returns, and the column tested is not the one that measures allocation.
- **Evidence:** `library/edihistory/edih_io.php:L740`. VERIFIED: `if ($row['global_amount'] === '0' || $row['global_amount'] === '0.00') {` guards the wording added at `:L741`, on a value selected at `:L737` from `ar_session`. VERIFIED: `global_amount` is declared `decimal(12,2) NOT NULL` with no default at `sql/database.sql:L10169`; it holds the deliberately undistributed remainder of a deposit, written by the payment screen at `interface/billing/new_payment.php:L142`; and the practice's own allocation test elsewhere is the three-term expression `pay_total - global_amount - sum(pay_amount)`, at `interface/billing/payment_master.inc.php:L89` and again at `interface/billing/search_payments.php:L186`. VERIFIED: neither deposit writer in `src/Billing/SLEOB.php` supplies the column, at `:L87-L89` and at `:L95-L97`, and the connection runs with `SET sql_mode = ''`, at `src/BC/DatabaseConnectionFactory.php:L70` and `:L133`, so the omission is accepted with an implicit zero rather than rejected.
- **Why it looks wrong:** A zero in this column means no undistributed remainder was recorded, which for a remittance-created deposit means the column was never written at all; it does not mean the deposit has been consumed by ledger lines. The subsystem's own definition of allocation, three columns wide, is used in two other places in the same repository. INFERRED (confidence: High). Basis: the three-term test at `interface/billing/search_payments.php:L186` is the practice-facing definition, and a one-term identity test against a column the writers never populate cannot express it. The operator hazard is registered as [BR-G8](business-rules.md#br-g8-a-deposit-is-found-by-a-reference-that-is-neither-unique-nor-nullable) and the omission itself as [DC-18](#dc-18-the-deposit-writer-omits-seven-columns-the-schema-declares-not-null).
- **Observable symptom:** In the EDI history trace lookup a deposit created by the remittance path is reported as fully allocated whatever has actually been posted against it, so an operator asking whether a check has already been consumed is told yes in every case and cannot use the answer.
- **Verification:** Reproduce in the EDI history screen at `interface/billing/edih_main.php` by requesting a trace number for a deposit posted through the remittance path with no ledger lines against it at all; expected is either no allocation claim or a claim derived from `pay_total`, `global_amount` and the sum of `pay_amount`, actual is the words that say fully allocated. There is no test file covering `library/edihistory/`, which [upgrade-risk-map.md](upgrade-risk-map.md#master-risk-table) records as `none` for every file in that tree, so a test for this would be the first one.
- **Severity:** MEDIUM

```php
if ($row['global_amount'] === '0' || $row['global_amount'] === '0.00') {
```

That is `library/edihistory/edih_io.php:L740`. The column it tests is declared at `sql/database.sql:L10169`.

### DC-5 The duplicate deposit probe searches for a reference the writer never stores

- **Suspected defect:** The duplicate-check warning on the remittance posting screen looks for an `ar_session` row whose `reference` equals the raw check number, while the writer that creates those rows on this path stores the check number behind a fixed six-character prefix, so the two can never match.
- **Evidence:** `interface/billing/sl_eob_process.php:L241`. VERIFIED: the first pass runs `"select reference from ar_session where reference=?"` bound to `$out['check_number' . $check_count]`, and sets a warning flag at `:L243-L245` when it finds anything. The row it is looking for is written by `src/Billing/SLEOB.php:L102`, which binds `'ePay - ' . $check_number` into the `reference` column. The other deposit writer in the same class, `arGetSession` at `src/Billing/SLEOB.php:L87-L89`, stores the reference unprefixed, and its own probe at `:L77-L80` searches unprefixed.
- **Why it looks wrong:** One class contains two deposit writers that disagree about the shape of the value in a single column, and a screen that queries that column has to pick one convention. It picked the convention the other writer uses. INFERRED (confidence: High). Basis: the prefix at `:L102` is a literal with no counterpart anywhere in the probe or in the sibling writer, and the two structurally incompatible writers are already registered as [BR-G8](business-rules.md#br-g8-a-deposit-is-found-by-a-reference-that-is-neither-unique-nor-nullable).
- **Observable symptom:** The red row and the duplicate-deposit warning that the screen is designed to show for an already-posted check never appear for a check posted through the remittance path, so the same remittance can be posted twice and the practice's cash is double counted in `ar_session`.
- **Verification:** Reproduce on the remittance upload screen at `interface/billing/era_payments.php`: post a remittance file once, then upload the same file again and reach the posting screen. Expected is the duplicate warning at the top of the deposit block, actual is a clean screen. A test would go in `tests/Tests/Isolated/Billing/` asserting that the value `SLEOB::arPostSession` writes to `reference` is the value the screen's probe searches for, and would run under `phpunit-isolated.xml`.
- **Severity:** HIGH

```php
$records = QueryUtils::fetchRecords("select reference from ar_session where reference=?", [$out['check_number' . $check_count]]);
```

That is `interface/billing/sl_eob_process.php:L241`. The value the writer actually stores is built at `src/Billing/SLEOB.php:L102`.

### DC-6 A seven character procedure code is split into a code and a modifier

- **Suspected defect:** The remittance parser treats any seven-character service-line procedure code with no separate modifier element as a five-character code with a two-character modifier stuck to it, on length alone.
- **Evidence:** `src/Billing/ParseERA.php:L371-L373`. VERIFIED: `if (strlen($svc[1]) == 7 && empty($svc[2])) {` is followed by `substr($svc[1], 0, 5)` into the code and `substr($svc[1], 5)` into the modifier. The comment immediately above at `:L370` states the reason as payers appending the modifier with no separator.
- **Why it looks wrong:** Length is being used as a proxy for structure. A genuine seven-character procedure identifier, which the code set permits, is cut in half and posted as a five-character code that the practice never billed. INFERRED (confidence: Medium). Basis: the branch has no test of the trailing two characters against any modifier list, and the author's own comment frames the rule as an accommodation for particular payers rather than as a property of the code set.
- **Observable symptom:** A ledger line in `ar_activity` whose `code` is the first five characters of a code the practice never used and whose `modifier` is the last two, so the payment does not match the charge on the invoice screen and the charge stays open.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` feeding an SVC segment, the service payment information segment, whose first composite element is a seven-character code with no modifier element, and assert the parsed `code` and `mod`; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
if (strlen($svc[1]) == 7 && empty($svc[2])) {
    $out['svc'][$i]['code'] = substr($svc[1], 0, 5);
    $out['svc'][$i]['mod'] = substr($svc[1], 5);
```

That is `src/Billing/ParseERA.php:L371-L373`.

### DC-7 The crossover claim row is chosen by a loose inequality

- **Suspected defect:** Whether a full claim row or a five-column crossover stub is inserted turns on `<> 1`, a loose inequality against a value the caller supplies, so any value that loosely equals one takes the crossover path and everything else takes the full path.
- **Evidence:** `src/Billing/BillingUtilities.php:L1685`. VERIFIED: `if ($crossover <> 1) {` selects the full insert at `:L1687-L1695`, and the else branch at `:L1696-L1704` inserts a row naming only `patient_id`, `encounter_id`, `bill_time`, `status` and `version`.
- **Why it looks wrong:** The flag is a two-state indicator and the comparison admits every loosely equal spelling of one, including the string `'1'`, the string `'1abc'` under the old coercion rules, and `true`. The two branches write structurally different rows, so the coercion decides which columns of `claims` are populated at all. INFERRED (confidence: Medium). Basis: the flag has exactly two intended states and the loose operator adds cases that were never designed for; nothing else in the function inspects the value.
- **Observable symptom:** A `claims` row with no `payer_id`, no `bill_process` and no `submitted_claim`, which the claim history screen renders as a version with no payer and no stored claim text, and which the resubmission path cannot rebuild.
- **Verification:** Add a case to `tests/Tests/Services/Billing/` calling the claim update path with `$crossover` as the integer `1`, the string `'1'` and `true`, asserting which columns the resulting `claims` row carries; it runs under the primary configuration, which covers `tests/Tests/Services` at `phpunit.xml:L67-L69`. The five-column stub itself is registered as [BR-H3](business-rules.md#br-h3-a-crossover-claim-row-records-five-columns-and-discards-the-rest).
- **Severity:** MEDIUM

```php
if ($crossover <> 1) {
```

That is `src/Billing/BillingUtilities.php:L1685`. The two branches it selects are `src/Billing/BillingUtilities.php:L1687-L1695` and `src/Billing/BillingUtilities.php:L1696-L1704`.

### DC-8 Envelope validity is decided by truth-testing positions against an assumed terminator

- **Suspected defect:** The scanner decides whether an uploaded interchange carries a functional group header and a transaction set header by truth-testing two `strpos()` results, and it builds both needles from a segment terminator it assumes is the file's last character. It also writes its verdict as whole literals rather than adding to one, so the second test's success reports the first test's answer as well.
- **Evidence:** `src/Billing/EdiHistory/X12File.php:L324-L339`. VERIFIED: `$hasval = 'ov';` at `:L324` is replaced by `'ovi'` at `:L327` when the text starts with `ISA`, by `'ovig'` at `:L331` when `strpos($ftxt, $dt . 'GS' . $de, 0)` is truthy, and by `'ovigs'` at `:L335` when the same shape of test finds the transaction set header; each is an assignment, and the two tests at `:L330` and `:L334` are independent `if` statements rather than a chained pair. VERIFIED: the element separator is read from a fixed position in the interchange header at `:L328`, while the segment terminator is taken as the last character of the trimmed text at `:L329`, under the comment at `:L325` that states that as an assumption. VERIFIED: the verdict is returned at `:L339` and unpacked into four booleans by truth-testing string positions at `:L114-L117`.
- **Why it looks wrong:** Two hazards sit in one expression, and only one of them is harmless. VERIFIED: truth-testing the position is safe here only because the verdict always begins with a character none of the four questions asks about, and because a file that reaches these two tests begins with `ISA`, so neither needle can occur at offset zero; that is correctness by circumstance, and this entry records it rather than claiming a well-formed interchange exercises offset zero. VERIFIED: the terminator, unlike the element separator, is not read from the header, so both needles are right only for a file whose final segment carries its terminator. VERIFIED: the same class already abandoned this dependence elsewhere, resolving the type with the terminator-free pattern `/GS\<de>(?:HB|HS|HR|HI|HN|HP|FA|HC)\<de>/` at `:L417-L418`, directly under a comment at `:L416` recording that it replaced a search built the way `:L330` still builds one. INFERRED (confidence: Medium). Basis: the comment is evidence of intent rather than of behaviour, but the regular expression beside it is not, and the two functions in one class now disagree about how to find the same segment. The encoding itself is registered as [BR-H8](business-rules.md#br-h8-envelope-validity-is-encoded-in-the-length-of-a-string).
- **Observable symptom:** Two reachable cases, in opposite directions, both landing on the EDI history upload path. VERIFIED: an interchange whose final segment carries no terminator leaves `$dt` holding a data character, so both needles fail and the verdict stops at three characters, which makes the functional-group flag false at `src/Billing/EdiHistory/X12File.php:L116` while the validity flag stays true at `src/Billing/EdiHistory/X12File.php:L114`; the upload handler then skips content classification and falls to its filename branch, classifying the file by matching its name against per-type patterns at `library/edihistory/edih_uploads.php:L111-L123`, or rejecting it with the comment about being unable to classify at `library/edihistory/edih_uploads.php:L136-L138`. The type was in fact discoverable, because the pattern at `src/Billing/EdiHistory/X12File.php:L417-L418` does not depend on the terminator. VERIFIED: in the other direction, a file carrying an interchange header and a transaction set header but no functional group header is reported as having one, because the five-character literal written at `src/Billing/EdiHistory/X12File.php:L335` contains the character the functional-group question searches for at `src/Billing/EdiHistory/X12File.php:L116`; the upload handler therefore takes the content branch at `library/edihistory/edih_uploads.php:L108-L110`, the type resolves empty because no functional group header exists for the pattern to match, and the file is rejected as unclassifiable rather than as malformed.
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/EdiHistory/X12FileIsolatedTest.php`, which already asserts the five-character verdict for a well-formed multi-line interchange at `tests/Tests/Isolated/Billing/EdiHistory/X12FileIsolatedTest.php:L224`; both run under `phpunit-isolated.xml`. The first passes that same interchange with the final segment's terminator removed: the expected result is the five-character verdict, the actual result is the three-character one. The second passes an interchange carrying an interchange header and a transaction set header but no functional group header: the expected result is a verdict whose functional-group answer is false, the actual result is the five-character verdict and a functional-group answer of true.
- **Severity:** MEDIUM

```php
if (strpos($ftxt, $dt . 'GS' . $de, 0)) {
    $hasval = 'ovig';
```

That is `src/Billing/EdiHistory/X12File.php:L330-L331`. The same shape is repeated for the transaction set header at `src/Billing/EdiHistory/X12File.php:L334-L335`, and the terminator both needles begin with is taken as the file's last character at `src/Billing/EdiHistory/X12File.php:L329`.

### DC-9 A negative contractual obligation is inverted after a strict string test

- **Suspected defect:** A negative contractual-obligation adjustment is sign-flipped unless its reason code is exactly the string `'144'`, and the exemption uses `!==` against a string while the group code beside it uses loose `==`, so the exemption is narrower than the rule it guards.
- **Evidence:** `src/Billing/ParseERA.php:L402-L406`. VERIFIED: `if ($seg[1] == 'CO' && $seg[$k + 1] < 0 && $seg[$k] !== '144') {` appends a warning and then executes `$seg[$k + 1] = 0 - $seg[$k + 1];`. The reason code arrives as a raw segment element from the payer's file.
- **Why it looks wrong:** The two operands are treated inconsistently in one condition: the group code by loose equality, the reason code by strict identity. A reason code that arrives as anything but the exact three-character string `'144'`, for example with a leading space or already coerced to an integer by an intermediate step, loses the exemption and has its amount inverted. INFERRED (confidence: Medium). Basis: the comment at `:L401` states the exemption exists specifically to stop the inversion from breaking claim balancing, so an exemption that misses is a balancing failure by the author's own account.
- **Observable symptom:** The claim posts with a contractual write-off of the opposite sign, the balancing warning text appears in the notes of the ledger line, and the invoice balance moves by twice the adjustment.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a claim adjustment segment with group code `CO`, a negative amount and a reason code of `' 144'`, and assert the parsed amount's sign; it runs under `phpunit-isolated.xml`. The inversion itself is registered as [BR-A7](business-rules.md#br-a7-a-negative-contractual-obligation-is-inverted-unless-its-reason-code-is-144).
- **Severity:** MEDIUM

```php
if ($seg[1] == 'CO' && $seg[$k + 1] < 0 && $seg[$k] !== '144') {
```

That is `src/Billing/ParseERA.php:L402`. The exemption is the third term.

## Category 2 Precision and Truncation in Monetary Math

Nine entries about arithmetic on money: which dollars belong in a sum, how a sum is rounded or formatted, and where a value is cut or coerced on its way into a column. The first entry is the most consequential in the register, because two components of the same subsystem disagree about the contents of one sum.

### DC-10 Provider level adjustments are excluded from the ledger and included in the balance test

- **Suspected defect:** A provider-level adjustment, which is money the payer takes back at the provider level rather than against any one claim, is deliberately kept out of accounts receivable by the parser and deliberately counted in by the modern balance test, so the two components disagree about whether the same dollars are in scope.
- **Evidence:** `src/Billing/ParseERA.php:L429-L440`. VERIFIED: the PLB branch, the provider-level adjustment segment, records each adjustment only as warning text, and the comment at `:L430-L431` states the reason as those adjustments belonging to the general ledger and not to the claim's accounts receivable. VERIFIED: `src/Billing/EdiHistory/RemitAccounting.php:L29` sums `$acctng['pmt'] + $acctng['clmadj'] + $acctng['svcadj'] + $acctng['svcptrsp'] + $acctng['plbadj']`, with the provider-level term included, and `:L30` compares that sum against the charged amount in integer cents.
- **Why it looks wrong:** Two components in the same subsystem apply opposite rules to one class of dollars, and neither is aware of the other. One of the two must be wrong for any remittance carrying a provider-level adjustment: either the balance test passes on a remittance whose ledger will not balance, or the ledger is short by an amount the balance test has already accounted for. INFERRED (confidence: High). Basis: the parser states its exclusion as a deliberate decision in a comment, the accounting helper states its inclusion as an arithmetic term, and no code reconciles the two; [architecture.md](architecture.md) records the two as separate generations of the same responsibility, which is how the disagreement survived.
- **Observable symptom:** A remittance carrying a provider-level adjustment is reported as balanced by the modern accounting helper, and the amount posted to `ar_activity` is short by the adjustment. The operator sees the adjustment only as a note in the warnings block, phrased as not claim specific, and the practice's cash position in accounts receivable disagrees with the deposit by that amount with nothing to reconcile it against.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/EdiHistory/RemitAccountingTest.php` asserting `isBalanced()` for an accounting array whose `plbadj` term is non-zero, and a matching case in `tests/Tests/Isolated/Billing/ParseERATest.php` asserting that the same PLB segment produces no ledger amount; both run under `phpunit-isolated.xml`. Comparing the two assertions is the proof, because each component is individually self-consistent.
- **Severity:** CRITICAL
- **Registered as a rule:** [BR-B2](business-rules.md#br-b2-provider-level-adjustments-are-excluded-from-ar-and-included-in-the-balance-test).

```php
$accounted = $acctng['pmt'] + $acctng['clmadj'] + $acctng['svcadj'] + $acctng['svcptrsp'] + $acctng['plbadj'];
```

That is `src/Billing/EdiHistory/RemitAccounting.php:L29`. The excluded term in the parser is the branch at `src/Billing/ParseERA.php:L429-L440`.

### DC-11 Forced balancing overwrites the service payment the payer reported

- **Suspected defect:** When a remittance does not balance, the residual payment is added to the first service line's paid amount. If that line already exists, the amount that reaches accounts receivable is not the amount the payer sent, and the warning that would have said so is inert.
- **Evidence:** `src/Billing/ParseERA.php:L65`. VERIFIED: `$out['svc'][0]['paid'] += $paytotal;` runs inside the block gated at `:L37`, after the residue is computed at `:L38-L52`. The synthetic line is inserted only when the first line is not already the synthetic one, tested at `:L54`, and the warning naming the insertion is inside that branch at `:L61-L62`. VERIFIED: a claim carrying claim-level adjustments already holds a line named `Claim` at index zero, created at `:L278`, so the insertion and its warning are both skipped and the addition lands on an existing line. VERIFIED: the warning written for exactly that case at `:L74-L78` is commented out.
- **Why it looks wrong:** The one path that alters a payer-reported amount is the one path with no message, and the code that would have produced the message is present and disabled. INFERRED (confidence: High). Basis: the suppressed text at `:L74-L78` reads as an assertion that the case should not arise, which is evidence the author expected the addition to land on a synthetic line and did not expect the claim-level path to have created one already.
- **Observable symptom:** A payment on the invoice screen that matches no amount on the payer's own remittance, with nothing in the notes to explain the difference. Reconciling the ledger against the paper explanation of benefits fails on one line and the difference has no recorded cause.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` that enables the balancing switch, supplies a claim with a claim-level adjustment segment and service lines that do not sum to the claim totals, and asserts both the first service line's paid amount and the warnings text; it runs under `phpunit-isolated.xml`. The expected result is either an unmodified payer amount or a warning; the actual result is a modified amount and no warning.
- **Severity:** HIGH
- **Registered as a rule:** [BR-B3](business-rules.md#br-b3-balancing-rewrites-the-payers-own-service-amounts).

```php
$out['svc'][0]['paid'] += $paytotal;
```

That is `src/Billing/ParseERA.php:L65`. The warning that would have reported it is inert at `src/Billing/ParseERA.php:L74-L78`.

### DC-12 The balancing residue is attributed to a group code the payer never sent

- **Suspected defect:** The residual adjustment produced by balancing is written with the group code `CR`, for correction and reversal, and the reason code `Balancing`, neither of which appears anywhere in the payer's file, and the code was chosen on a stated guess.
- **Evidence:** `src/Billing/ParseERA.php:L69-L71`. VERIFIED: the three assignments set `group_code` to `'CR'`, `reason_code` to `'Balancing'` and `amount` to the computed residue, and the trailing comment on `:L69` records the choice as a presumption of a correction or reversal.
- **Why it looks wrong:** A group code is a payer's classification of why money moved, and this one is manufactured locally to fill a required field. `Balancing` is not a value in any adjustment reason code list, so any lookup against the hardcoded tables in `src/Billing/BillingUtilities.php` misses it. INFERRED (confidence: High). Basis: the author labelled the group code a presumption on the same line, which is the strongest possible evidence that it was not derived.
- **Observable symptom:** A ledger line whose adjustment reason renders with no description on the invoice and the explanation-of-benefits screens, because the reason code matches no entry in the code table, and any report grouping adjustments by group code counts practice-manufactured residue among genuine payer corrections and reversals.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` asserting the group and reason codes of the adjustment appended when balancing is forced, then assert that the reason code is absent from the adjustment reason code table in `src/Billing/BillingUtilities.php`; it runs under `phpunit-isolated.xml`. Both assertions are needed, because the value is only wrong in combination with the lookup.
- **Severity:** MEDIUM
- **Registered as a rule:** [BR-B4](business-rules.md#br-b4-an-artificial-service-line-named-claim-absorbs-the-residue).

```php
$out['svc'][0]['adj'][$j]['group_code'] = 'CR'; // presuming a correction or reversal
$out['svc'][0]['adj'][$j]['reason_code'] = 'Balancing';
```

That is `src/Billing/ParseERA.php:L69-L70`. The trailing comment on the first line is the author's own record that the group code is a guess.

### DC-13 The saved institutional claim form is byte-capped and truncated in silence

- **Suspected defect:** The column the claim writer always populates holds 65,535 bytes, the connection runs in a mode that truncates an oversized value instead of refusing it, and nothing on the write path measures the value first. The content at risk is not a generated claim: on the institutional path it is the serialised claim form, and once truncated it cannot be decoded, so the next write replaces it with the encoding of nothing.
- **Evidence:** `sql/database.sql:L391`. VERIFIED: `` `submitted_claim` text COMMENT 'This claims form claim data' ``, which MySQL caps at 65,535 bytes. VERIFIED: the fragment `", submitted_claim = ?"` is appended to every claim update at `src/Billing/BillingUtilities.php:L1653-L1654`, outside every conditional, and the bound value defaults to the empty string declared as the last parameter at `src/Billing/BillingUtilities.php:L1534`; professional callers pass nothing there, so they store an empty string on every version. VERIFIED: the institutional caller is the only one that stores content, passing `json_encode($this->ub04id)` from `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L80` as the last argument at `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L93`, and again on the normal path at `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L112-L125`; the array is padded to 428 entries at `src/Billing/X125010837I.php:L33-L37`. VERIFIED: the connection issues `SET sql_mode = ''` at `src/BC/DatabaseConnectionFactory.php:L70` and again at `src/BC/DatabaseConnectionFactory.php:L133`, so an over-length value is truncated with a warning rather than rejected. VERIFIED: the stored text is read back at `interface/billing/ub04_dispose.php:L190` and decoded as JSON at `interface/billing/ub04_dispose.php:L207`, inside the branch the non-empty column already selected at `interface/billing/ub04_dispose.php:L205`.
- **Why it looks wrong:** The value's length is a function of how many service lines, diagnoses and free-text fields an encounter carries, and nothing between the caller and the column measures it. The permissive session mode converts what would be a loud failure into a silent one, and the read path then treats whatever came back as complete. INFERRED (confidence: Medium). Basis: no length check exists anywhere on the write path, and the decode at `interface/billing/ub04_dispose.php:L207` has no failure branch, so a partial value and a whole one are indistinguishable to every reader.
- **Observable symptom:** For a professional claim, nothing: the column receives an empty string on every version and there is nothing to truncate. For an institutional claim whose serialised form exceeds the cap, the loss is silent and then permanent. VERIFIED: the truncated text is no longer valid JSON, so the decode at `interface/billing/ub04_dispose.php:L207` yields null and `get_ub04_array()` returns null from inside the saved-form branch rather than falling through to regenerate the form; the log line it has already appended says the saved edited claim is being used, at `interface/billing/ub04_dispose.php:L206`. The generator then re-encodes that null at `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L80` and stores the result over the column, so the operator's edited institutional form is replaced by the encoding of nothing. VERIFIED: no message anywhere reports the loss, because neither the read at `interface/billing/ub04_dispose.php:L201-L207` nor the re-encode at `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L79-L80` tests the decoded value before using it.
- **Verification:** Add a case to `tests/Tests/Services/Billing/` that writes a value longer than 65,535 bytes through the claim update path and reads `submitted_claim` back, asserting byte-for-byte equality; it runs under the primary configuration, which covers `tests/Tests/Services` at `phpunit.xml:L67-L69`. The expected result is equality or a refusal, the actual result is a shorter string and no error. A second case, in the same file, passes that truncated string to `get_ub04_array()`: the expected result is either the saved form or a regenerated one, the actual result is null. The reproduction is to open an institutional claim on the UB-04 screen, fill the free-text fields until the serialised form passes the cap, save, and reopen it: the expected outcome is the saved form, the actual outcome is an empty form and a log line claiming the saved one was used.
- **Severity:** MEDIUM

```sql
`submitted_claim` text COMMENT 'This claims form claim data',
```

That is `sql/database.sql:L391`. MySQL caps this type at 65,535 bytes.

### DC-14 The deposit balance check is a float subtraction reported only in a browser alert

- **Suspected defect:** After a remittance is posted, each deposit is checked by subtracting the sum of its live ledger lines from its total with `<>` on values the driver returns for `decimal(12,2)` columns, and the only report of a mismatch is a browser alert listing keys.
- **Evidence:** `interface/billing/sl_eob_process.php:L866`. VERIFIED: `if (($pay_total - $pay_amount) <> 0) {` compares `pay_total`, read at `:L858`, against `sum(pay_amount)` restricted to rows where `deleted IS NULL`, read at `:L860-L863`. VERIFIED: a mismatch only appends the array key to a string at `:L867` and the string is emitted as a JavaScript alert at `:L873-L875`. VERIFIED: the loop iterates only the insertion identifiers the current run produced, at `interface/billing/sl_eob_process.php:L856-L857`, and each of those identifiers comes from a deposit header the run inserted unconditionally, at `src/Billing/SLEOB.php:L95-L103`, under a reference that is the check number prefixed with `ePay - `; that writer has no reuse branch, unlike the other deposit writer in the same class, which does look for an existing header first at `src/Billing/SLEOB.php:L77-L83`.
- **Why it looks wrong:** Three separate weaknesses sit on one path. The comparison is a subtraction of two driver-returned values tested against integer zero, so it is exposed to representation differences the way [DC-4](#dc-4-a-deposit-counts-as-fully-allocated-only-for-two-exact-strings) is. The `deleted IS NULL` restriction means a deposit whose ledger lines were later reversed reports as unbalanced forever. And the finding is delivered as a modal alert, so the result survives nothing: not a page reload, not a log, not a row. INFERRED (confidence: High). Basis: the alert is the only consumer of the result, and no row, column or log entry records it.
- **Observable symptom:** An operator who dismisses the alert, or whose browser suppresses it, has no way to recover this finding, because it is written nowhere. Two later screens do compute a related figure from the same rows, and they are named here so that the entry does not overstate the silence. VERIFIED: the payment edit screen computes an undistributed amount at `interface/billing/payment_master.inc.php:L89` and renders it under the Undistributed label at `interface/billing/payment_master.inc.php:L256`, and the payment search screen prints Fully Paid or Unapplied from the same subtraction at `interface/billing/search_payments.php:L575-L576`. VERIFIED: neither figure is this audit's result, and neither is the same arithmetic, because both also subtract the deposit's global allocation while the post-run audit at `interface/billing/sl_eob_process.php:L866` does not; the two therefore disagree on any deposit carrying a global allocation, in one direction or the other depending on its size. Neither screen attributes what it shows to the remittance run that produced the deposit.
- **Verification:** Reproduce on the remittance posting screen at `interface/billing/sl_eob_process.php`: with dry run off, post a remittance whose check total exceeds the sum of the service payments it carries. Expected is a durable record that the deposit did not balance, actual is one alert naming the check number and an `ar_session` row that no later screen ties back to this run. A second reproduction covers the arithmetic difference: post a remittance the audit reports as balanced, then set a global allocation on the resulting deposit from the payment edit screen and reopen it. Expected is that a deposit's balance verdict does not change without a ledger line changing; actual is a non-zero Undistributed figure and the search screen relabelling the deposit Unapplied, with no surviving record of the verdict the audit gave. One reproduction is not available and is stated so that nobody attempts it: a second remittance cannot be posted into an existing deposit, because the header writer inserts unconditionally at `src/Billing/SLEOB.php:L95-L103` and the audit examines only the current run's identifiers at `interface/billing/sl_eob_process.php:L856-L857`. The balance rule itself is registered as [BR-B7](business-rules.md#br-b7-a-deposit-is-balanced-against-live-ledger-lines-only-and-only-in-a-browser-alert).
- **Severity:** HIGH

```php
if (($pay_total - $pay_amount) <> 0) {
```

That is `interface/billing/sl_eob_process.php:L866`. The only report of the result is the alert at `interface/billing/sl_eob_process.php:L873-L875`.

### DC-15 A non numeric adjustment amount becomes zero in two places

- **Suspected defect:** An adjustment amount that is not numeric is silently converted to zero, once when the remittance is parsed and again when the posting screen displays and posts it, so a malformed payer amount posts as no adjustment at all.
- **Evidence:** `src/Billing/ParseERA.php:L412-L413`. VERIFIED: `$raw = $seg[$k + 1] ?? 0;` followed by `is_numeric($raw) ? (float)$raw : 0.0`. VERIFIED: the same coercion is repeated on the screen at `interface/billing/sl_eob_process.php:L634`, `$amount = is_numeric($adj['amount']) ? (float)$adj['amount'] : 0.0;`, which is the value formatted into the displayed line at `:L635` and posted at `:L642-L653`.
- **Why it looks wrong:** A non-numeric amount in a monetary element is a malformed remittance, and the code treats it as a valid zero. Neither site records a warning, although the parser records warnings for several other anomalies in the same function. INFERRED (confidence: High). Basis: the parser's own convention is to append explanatory text to `$out['warnings']` when it repairs an input, and this repair is the one that does not.
- **Observable symptom:** An adjustment line appears on the posting screen reading `0.00` for an amount the payer did send, the charge is left open by that amount, and no warning explains the difference. The patient's statement then shows a balance the payer had already adjusted away.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a claim adjustment segment whose amount element is a non-numeric string, and assert both the parsed amount and the warnings text; it runs under `phpunit-isolated.xml`. The rule as it stands is registered as [BR-A8](business-rules.md#br-a8-a-non-numeric-adjustment-amount-becomes-zero).
- **Severity:** HIGH

```php
$raw = $seg[$k + 1] ?? 0;
$out['svc'][$i]['adj'][$j]['amount'] = is_numeric($raw) ? (float)$raw : 0.0;
```

That is `src/Billing/ParseERA.php:L412-L413`. The same coercion is repeated on the screen at `interface/billing/sl_eob_process.php:L634`.

### DC-16 The running invoice balance alternates between a formatted string and float arithmetic

- **Suspected defect:** One running total is assigned the result of `sprintf("%.2f", ...)` in one place and updated by float addition and subtraction in three others, so the variable holds a string on some iterations and a float on others while being compared against a charge to the cent.
- **Evidence:** `interface/billing/sl_eob_process.php:L195-L196`. VERIFIED: `$amount = sprintf("%.2f", ...)` and then `$invoice_total = sprintf("%.2f", $invoice_total + $amount);`, which stores a string. VERIFIED: the same variable is reset at `:L322-L323`, increased by a raw charge at `:L520`, and decreased by raw amounts at `:L572` and `:L653`, all as float arithmetic. It is declared as a global at `:L41-L48`.
- **Why it looks wrong:** A monetary accumulator should have one type and one rounding point. This one is re-rounded to a string on the detail path and then arithmetically modified as a float on the posting path, so the value's precision depends on which sequence of screens the operator walked. INFERRED (confidence: Medium). Basis: the two update styles are in the same file and the same request, and only the string form applies rounding; the float form does not.
- **Observable symptom:** An invoice whose displayed running balance is a cent away from the sum of its own lines, which is enough to trip the charge comparison that blocks a posting and to make the operator hunt for a missing cent that no line carries.
- **Verification:** Reproduce on the invoice detail screen reached from `interface/billing/sl_eob_search.php` for an encounter with several charges whose unrounded sum differs from the rounded sum, comparing the displayed running balance against the sum of the displayed lines. Expected is agreement, actual is a one-cent difference. The accumulation convention is registered as [BR-A4](business-rules.md#br-a4-charge-totals-are-accumulated-as-floats-and-printed-at-two-decimals) and the comparison it feeds as [BR-A9](business-rules.md#br-a9-a-charge-that-disagrees-with-the-invoice-by-one-cent-blocks-the-posting).
- **Severity:** MEDIUM

```php
$amount = sprintf("%.2f", (floatval($ddata['chg'] ?? '')) - (floatval($ddata['pmt'] ?? '')));
$invoice_total = sprintf("%.2f", $invoice_total + $amount);
```

That is `interface/billing/sl_eob_process.php:L195-L196`. The same variable is updated as a float at `interface/billing/sl_eob_process.php:L520`, `interface/billing/sl_eob_process.php:L572` and `interface/billing/sl_eob_process.php:L653`.

### DC-17 A provider level adjustment amount is formatted from an element the loop never proves exists

- **Suspected defect:** The provider-level adjustment loop steps two elements at a time, tests only the first of each pair for presence, and then formats the second with `%.2f`, so a truncated segment formats a missing element as a monetary amount.
- **Evidence:** `src/Billing/ParseERA.php:L432-L438`. VERIFIED: `for ($k = 3; $k < 15; $k += 2) {` breaks at `:L433-L435` when `$seg[$k]` is absent or empty, and `:L437-L438` then reads `sprintf('%.2f', $seg[$k + 1])` with no test of `$seg[$k + 1]`.
- **Why it looks wrong:** The loop's own guard establishes the presence of one element of each pair and the body uses both. A segment whose final reason code arrives with no amount reaches `sprintf` with an undefined index. INFERRED (confidence: High). Basis: the guard tests `$seg[$k] ?? ''` explicitly, which shows the author knew the elements might be absent, and the very next statement omits the same protection for the paired element.
- **Observable symptom:** A PHP warning about an undefined array key in the error log during remittance posting, and a provider-level adjustment note that reports the amount as `0.00` rather than as unknown. On a site that renders warnings the notice appears inside the posting screen's markup.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a PLB segment that ends with a reason code and no amount, and assert the warnings text and that no undefined-key warning is raised; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
$out['warnings'] .= 'PROVIDER LEVEL ADJUSTMENT (not claim-specific): $' .
    sprintf('%.2f', $seg[$k + 1]) . " with reason code " . $seg[$k] . "\n";
```

That is `src/Billing/ParseERA.php:L437-L438`. The guard above it, at `src/Billing/ParseERA.php:L433-L435`, tests only the odd element of each pair.

### DC-18 The deposit writer omits seven columns the schema declares not null

- **Suspected defect:** One of the two deposit writers inserts six columns into a table with seven further columns declared `NOT NULL` and given no default, and the connection's permissive mode accepts the omission with implicit zeros and zero dates instead of rejecting it.
- **Evidence:** `src/Billing/SLEOB.php:L87-L89`. VERIFIED: the insert names `payer_id, user_id, reference, check_date, deposit_date, pay_total` only. VERIFIED at `sql/database.sql:L10168-L10175`: `modified_time datetime NOT NULL`, `global_amount decimal(12,2) NOT NULL`, `payment_type varchar(50) NOT NULL`, `adjustment_code varchar(50) NOT NULL`, `post_to_date date NOT NULL`, `patient_id bigint(20) NOT NULL` and `payment_method varchar(25) NOT NULL` all lack defaults. VERIFIED: `SET sql_mode = ''` at `src/BC/DatabaseConnectionFactory.php:L70` and `:L133`. VERIFIED: the sibling writer at `:L95-L97` supplies four of the seven as literals and still omits `modified_time` and `global_amount`.
- **Why it looks wrong:** The schema states seven invariants and the writer satisfies none of them, which is only survivable because the session mode has been relaxed globally. A deposit row therefore carries a zero date in `post_to_date` and an empty `payment_method`, values no screen would have accepted from an operator. INFERRED (confidence: High). Basis: the two writers in one class disagree about which of the seven to supply, which shows the omissions are oversights rather than a convention; and the permissive mode is set for the whole application rather than for this insert.
- **Observable symptom:** Deposit rows whose `post_to_date` is a zero date and whose `payment_method` is empty, which the payment search screen at `interface/billing/search_payments.php` renders as a blank method and a date of `0000-00-00`, and whose zero `global_amount` makes [DC-4](#dc-4-a-deposit-counts-as-fully-allocated-only-for-two-exact-strings) report the deposit as fully allocated.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arGetSession` and then reading the inserted row, asserting each of the seven columns against a value a screen would accept; it runs under `phpunit-isolated.xml`. Alternatively reproduce by posting a remittance and inspecting the new `ar_session` row: expected is a populated posting date and payment method, actual is a zero date and an empty string.
- **Severity:** HIGH

```php
return sqlInsert("INSERT INTO ar_session ( " .
    "payer_id, user_id, reference, check_date, deposit_date, pay_total " .
```

That is `src/Billing/SLEOB.php:L87-L88`. The seven columns it omits are declared at `sql/database.sql:L10168-L10175`.

## Category 3 State Leaking Across Loop Iterations

Eight entries where a variable outlives the iteration that set it. Two of them are the reason a claim can be posted against the wrong patient and a batch can be sent to the wrong payer, which is why both are CRITICAL.

### DC-19 Patient and provider fields survive from one claim to the next

- **Suspected defect:** The remittance parser clears seven keys when it starts a new claim and leaves nine others holding the previous claim's values, including the patient name, the patient member identifier and the rendering provider.
- **Evidence:** `src/Billing/ParseERA.php:L245-L251`. VERIFIED: on each CLP segment, the claim payment information segment, the parser clears `subscriber_lname`, `subscriber_fname`, `subscriber_mname`, `subscriber_member_id`, `crossover`, `corrected` and `svc`, and the warnings string at `:L243`. VERIFIED: `patient_lname`, `patient_fname`, `patient_mname` and `patient_member_id` are written at `:L297-L300`, `provider_lname`, `provider_fname`, `provider_mname` and `provider_member_id` at `:L307-L310`, and `corrected_mbi` at `:L314`, and none of the nine appears in the clearing block. VERIFIED: the posting screen falls back to the parsed patient name when the local encounter row is empty, at `interface/billing/sl_eob_process.php:L429-L431`.
- **Why it looks wrong:** The clearing block exists and carries the comment that it clears what is needed to start a new claim, so the omissions are not a decision to carry values forward; they are a list that fell behind the segments the parser learned to read. INFERRED (confidence: High). Basis: the subscriber quartet is cleared and the patient quartet, written eight lines further down the same conditional chain, is not; the two are structurally identical and there is no reading under which one should persist and the other should not.
- **Observable symptom:** On the remittance posting screen, a claim the parser could not match to a local encounter is labelled with the previous claim's patient name, which is another patient's name on the screen an operator uses to decide where money goes. A claim whose remittance omits the rendering provider name is displayed and noted with the previous claim's provider.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying two consecutive CLP segments where only the first is followed by a patient name segment, and assert that the second claim's patient fields are empty; it runs under `phpunit-isolated.xml`. The expected result is empty fields, the actual result is the first claim's patient.
- **Severity:** CRITICAL

```php
$out['patient_lname'] = trim($seg[3]);
$out['patient_fname'] = trim($seg[4]);
```

That is `src/Billing/ParseERA.php:L297-L298`. Neither key appears in the clearing block at `src/Billing/ParseERA.php:L245-L251`.

### DC-20 One batch file is queued once for every partner in the batch

- **Suspected defect:** When automatic upload is enabled, the batch writer queues one transport row per distinct trading partner in the run, and every row names the same single batch file, so each partner is offered a file containing every other partner's claims.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L170-L186`. VERIFIED: the block is entered when the write succeeded and the site switch `auto_sftp_claims_to_x12_partner` is on, at `:L170-L173`; the distinct partners are extracted at `:L174`; the loop at `:L177` creates one `X12RemoteTracker` row per partner at `:L178-L183`; and `:L180` sets `'x12_filename' => $this->bat_filename`, the one filename for the whole batch, in every row.
- **Why it looks wrong:** The loop iterates partners precisely because a batch can contain claims for more than one, and the payload it attaches does not vary with the loop variable. The comment on `:L176` describes the intent as queueing the batch file to all partners, which is what the code does; the defect is that the artifact being sent is not per-partner. INFERRED (confidence: High). Basis: the same subsystem contains a generator that does split output per partner and names the file accordingly, at `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L110`, which shows a per-partner filename is both expected and available.
- **Observable symptom:** Each clearinghouse receives one file containing claims addressed to other clearinghouses. Every such claim is either rejected in a 997 or 999 acknowledgement or, worse, accepted and adjudicated by a payer that should never have seen it, which is a disclosure of patient data to a party with no relationship to the encounter. Nothing on the billing screen distinguishes this from a normal successful run.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` that writes a batch containing claims for two partners with the site switch on, then asserts that the two created transport rows name different files; it runs under `phpunit-isolated.xml`. The expected result is two filenames, the actual result is one filename twice. Stage [S6](claim-lifecycle.md#stage-s6-transport-to-the-clearinghouse) is where the rows are consumed.
- **Severity:** CRITICAL
- **Registered as a rule:** [BR-F5](business-rules.md#br-f5-one-batch-file-is-queued-once-for-every-partner-in-the-batch).

```php
'x12_partner_id' => $x12_partner_id,
'x12_filename' => $this->bat_filename,
```

That is `src/Billing/BillingProcessor/BillingClaimBatch.php:L179-L180`. The loop variable is the partner and the filename is not.

### DC-21 The production date is backfilled once and never cleared

- **Suspected defect:** The production date is filled in from the check date when it is empty, and it is never cleared between claims, so a claim that carries no production date of its own inherits whatever the previous claim left behind.
- **Evidence:** `src/Billing/ParseERA.php:L25-L27`. VERIFIED: `if (empty($out['production_date'])) { $out['production_date'] = $out['check_date']; }` runs inside the per-claim helper invoked from the CLP branch at `:L241`. VERIFIED: `production_date` is written from a date segment at `:L326-L330` and does not appear in the clearing block at `:L245-L251`.
- **Why it looks wrong:** The backfill exists because the value is required downstream, which the comment at `:L24` states. Combining a required value with no per-claim reset means the requirement is met with a value that belongs to a different claim. INFERRED (confidence: High). Basis: the field is claim-scoped everywhere it is written and read, and the clearing block that resets claim scope omits it.
- **Observable symptom:** A ledger line whose posting date belongs to a different claim in the same remittance, which puts the payment in the wrong accounting period. On the posting screen the date column shows a plausible date, so nothing looks wrong; the discrepancy surfaces only when a period is closed and the totals do not match the deposit.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` with two claims where only the first carries a production date segment, asserting the second claim's `production_date`; it runs under `phpunit-isolated.xml`. The backfill itself is registered as [BR-B5](business-rules.md#br-b5-the-production-date-is-backfilled-from-the-check-date) and the date precedence as [BR-E7](business-rules.md#br-e7-the-operators-pay-date-overrides-the-payers-own-dates).
- **Severity:** HIGH

```php
if (empty($out['production_date'])) {
    $out['production_date'] = $out['check_date'];
}
```

That is `src/Billing/ParseERA.php:L25-L27`.

### DC-22 The line ending strip is applied to the wrong side of the split

- **Suspected defect:** Carriage returns and newlines are stripped from the remainder of the buffer after the segment has been cut off it, so the first segment read from the file is the only one never cleaned, and it is the one segment whose cleanliness the delimiter probe depends on.
- **Evidence:** `src/Billing/ParseERA.php:L112-L115`. VERIFIED: `$inline = substr($buffer, 0, $tpos);` at `:L112`, `$buffer = substr($buffer, $tpos + 1);` at `:L113`, the comment about payers sending carriage returns and newlines at `:L114`, and `$buffer = str_replace(["\n", "\r"], '', $buffer);` at `:L115` applied to the remainder. VERIFIED: the delimiter probe immediately below at `:L118-L121` is gated on `str_starts_with($inline, 'ISA')`.
- **Why it looks wrong:** The strip is one statement too late. Every subsequent segment is clean because the previous iteration cleaned the buffer it came from, but the first segment carries whatever preceded it in the file, and a leading newline or carriage return is exactly what the comment says payers send. INFERRED (confidence: High). Basis: moving the same call one line earlier would clean the first segment too, and the author's comment shows the case was anticipated.
- **Observable symptom:** A remittance file whose first byte is a line ending, which is a common result of a text-mode file transfer, is parsed with the default delimiters instead of its own. Every segment then fails to split, the parse ends with the unexpected-segment message from `:L467-L468`, and the operator is told the file contains an unknown segment rather than that its first line was malformed.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` feeding a remittance whose content begins with a newline before the ISA segment, and assert that parsing succeeds; it runs under `phpunit-isolated.xml`. The probe restriction is [DC-26](#dc-26-the-delimiter-probe-can-only-run-on-the-first-segment-of-the-interchange).
- **Severity:** HIGH

```php
$buffer = substr($buffer, $tpos + 1);
// remove carriage returns and new lines that some payers send
$buffer = str_replace(["\n", "\r"], '', $buffer);
```

That is `src/Billing/ParseERA.php:L113-L115`. The value split off on the line above, at `src/Billing/ParseERA.php:L112`, is the one never cleaned.

### DC-23 Two repeat suppression variables are never reset between claims

- **Suspected defect:** Four file-scope variables drive repeat suppression in the detail renderer. Two are reset at each claim boundary and two are not, so a claim block can render with a blank patient name and a blank invoice number and read as a continuation of the block above it.
- **Evidence:** `interface/billing/sl_eob_process.php:L128-L144`. VERIFIED: the renderer blanks `$ptname`, `$invnumber` and `$code` when each equals its remembered value and otherwise updates the remembered value, using the globals declared at `:L41-L48`. VERIFIED: only `$last_code` and `$invoice_total` are reset at the claim boundary, at `:L322-L323`; `$last_ptname` and `$last_invnumber` are initialised once at `:L42` and `:L44` and never reset.
- **Why it looks wrong:** The claim boundary already performs a reset, and it resets two of the four variables that the same renderer consults. The asymmetry is the defect: whatever reasoning justified resetting the code applies equally to the name and the invoice. INFERRED (confidence: Medium). Basis: the two resets sit on consecutive lines at a claim boundary and the two omissions are declared in the same block as the two that are reset; no comment distinguishes them.
- **Observable symptom:** When a payer sends two claim payment segments for the same claim, which is how a reversal and its correction arrive, the second block renders with no patient name and no invoice number. An operator deciding which of the two blocks to post cannot tell them apart on screen.
- **Verification:** Reproduce on the remittance posting screen by posting a remittance file containing a reversal and a correction for one claim, both with the same claim identifier: expected is two labelled blocks, actual is one labelled block followed by an unlabelled one.
- **Severity:** MEDIUM

```php
$last_code = '';
$invoice_total = 0.00;
```

That is `interface/billing/sl_eob_process.php:L322-L323`. The two variables not reset here are declared at `interface/billing/sl_eob_process.php:L42` and `interface/billing/sl_eob_process.php:L44`.

### DC-24 The error flag accumulates across the service lines of a claim

- **Suspected defect:** One error flag is set per claim and is only ever set to true inside the service-line loop, and every posting call in that loop is gated on it being false, so the first bad line silently suppresses the posting of every line after it while the lines before it are already posted.
- **Evidence:** `interface/billing/sl_eob_process.php:L433`. VERIFIED: `$error = $inverror;` is the per-claim initialisation. VERIFIED: it is set true at `:L491` and `:L504` when a service code cannot be matched and at `:L641` on a zero payment with no contractual write-off, and it is never set back to false inside the loop. VERIFIED: the charge post at `:L507`, the payment post at `:L559`, the note post at `:L621` and the adjustment post at `:L642` are each gated on `!$error`, and the end-of-claim work at `:L697` is gated on it as well.
- **Why it looks wrong:** The flag is used for two different jobs: to mark a line as unpostable, which is per line, and to abandon the claim, which is per claim. Because it is only raised and never lowered, the per-line job silently becomes the per-claim job from the first failure onward. INFERRED (confidence: High). Basis: the flag is re-read per line rather than per claim at all four posting sites, which shows the per-line reading was intended, and nothing resets it between lines.
- **Observable symptom:** A claim posts its first service lines and silently skips the rest. The skipped lines appear on screen in the error style, but no message says they were skipped because an earlier line failed, and the secondary payer setup at `:L717-L718` is skipped too, so the remaining balance is never billed onward. The invoice is left part-paid with an open balance nobody queued.
- **Verification:** Reproduce on the remittance posting screen with a remittance whose second of three service lines carries a procedure code absent from the encounter: expected is two posted lines and one flagged line, actual is one posted line and two unposted ones. A test would go in `tests/Tests/Isolated/Billing/` and run under `phpunit-isolated.xml`.
- **Severity:** HIGH

```php
$error = $inverror;
```

That is `interface/billing/sl_eob_process.php:L433`. It is raised at `interface/billing/sl_eob_process.php:L491`, `interface/billing/sl_eob_process.php:L504` and `interface/billing/sl_eob_process.php:L641`, and never lowered.

### DC-25 The check date computed in the first pass is discarded

- **Suspected defect:** The deposit loop computes a check date with a fallback to the operator's pay date and then passes the unfallen-back value to the writer, so the fallback can never take effect.
- **Evidence:** `interface/billing/sl_eob_process.php:L274`. VERIFIED: `$check_date = $out['check_date' . $check_count] ?: $_REQUEST['paydate'];` computes the value, and the call at `:L277-L285` passes `check_date: $out['check_date' . $check_count]` at `:L280`, reading the raw parsed value again rather than the computed one. Its neighbour at `:L281` is the `pay_total` argument, which fixes `:L280` as the only line of the call that carries a check date. The local variable is not read anywhere else in the function.
- **Why it looks wrong:** The statement exists only to produce a value for the call three lines below it, and the call ignores it. The two sibling values computed on the adjacent lines, the posting date and the deposit date at `:L275-L276`, are both passed. INFERRED (confidence: High). Basis: three fallbacks are computed together and two of the three are used, which makes the third an omission rather than a design.
- **Observable symptom:** A remittance whose check date element is empty produces a deposit row with an empty check date rather than the date the operator typed. The payment search screen then lists the deposit under a zero date and it cannot be found by date range.
- **Verification:** Reproduce on the remittance posting screen by posting a remittance whose check date element is empty while supplying a pay date on the form: expected is a deposit carrying the typed pay date, actual is a deposit with no check date. The date precedence intended here is registered as [BR-E7](business-rules.md#br-e7-the-operators-pay-date-overrides-the-payers-own-dates).
- **Severity:** MEDIUM

```php
$check_date = $out['check_date' . $check_count] ?: $_REQUEST['paydate'];
```

That is `interface/billing/sl_eob_process.php:L274`. The call three lines below reads the raw parsed value again, at `interface/billing/sl_eob_process.php:L280`.

### DC-26 The delimiter probe can only run on the first segment of the interchange

- **Suspected defect:** Two of the three delimiters are learned from the interchange header and the third, the segment terminator that the read loop splits on, never is. The probe that learns the other two is gated on the emptiness of a loop variable rather than on file position, so anything that leaves that variable set before the header is reached disables it for the rest of the file.
- **Evidence:** `src/Billing/ParseERA.php:L118-L121`. VERIFIED: the probe is gated on `if ($segid === '' && str_starts_with($inline, 'ISA'))` and assigns `$delimiter2` from the header's fourth byte and `$delimiter3` from its last byte, so the element separator and the component separator are both recovered from the file rather than assumed. VERIFIED: all three delimiters are initialised at `:L87-L89` to the tilde, the pipe and the caret, and `$delimiter1` is assigned exactly once in the whole function and never reassigned, while being the value the read loop searches for at `:L107` to locate each segment boundary.
- **Why it looks wrong:** The two delimiters that can only be recovered after the file has already been split into segments are recovered; the one that decides how the file is split into segments in the first place is assumed. The gate compounds this by using the emptiness of a loop variable as a proxy for being at the start of the file, which couples delimiter discovery to iteration state rather than to file position. INFERRED (confidence: High). Basis: the segment terminator is read at `:L107` before any segment exists to probe, so learning it would require restructuring the read rather than adding one assignment, which explains why it was left a constant without making the constant safe. INFERRED (confidence: Medium) on the gate itself. Basis: the probe's own inner test already checks for the header segment, so the outer emptiness test adds no correctness and removes robustness.
- **Observable symptom:** A remittance whose segment terminator is any character other than the tilde is never split into segments at all. The search at `:L107` returns false on the very first iteration, the loop breaks at `:L108-L110`, and the function returns the premature-end message from `:L474-L476`. The operator is told that a well-formed file ended prematurely, with nothing on screen about delimiters, and the check-scanning pass over the same file offers no deposits at all, which is [DC-61](#dc-61-a-component-separator-is-assigned-and-never-read-in-the-check-parse). A different input defeats the gate rather than the constant: when a line ending precedes the header, the strip at `:L115` has been applied to the wrong side of the split, `str_starts_with($inline, 'ISA')` is false, and the element separator stays at the pipe so each segment is read as one undivided element. That is [DC-22](#dc-22-the-line-ending-strip-is-applied-to-the-wrong-side-of-the-split).
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/ParseERATest.php`, both under `phpunit-isolated.xml`. The first supplies a remittance identical to the existing fixture except that every segment ends in a character other than the tilde: the expected result is a parsed claim, the actual result is the string `Premature end of ERA file`. The second supplies a remittance whose first byte is a line ending and asserts a parsed field that can only be correct when the element separator was recovered from the header: the expected value is that field from the fixture, the actual value is the entire undivided segment.
- **Severity:** MEDIUM

```php
    $delimiter2 = substr($inline, 3, 1);
    $delimiter3 = substr($inline, -1);
}
```

That is `src/Billing/ParseERA.php:L119-L121`. The value the loop actually splits on is set once at `src/Billing/ParseERA.php:L87` and read at `src/Billing/ParseERA.php:L107`; it appears in neither line above.

## Category 4 Dead Branches Masking Logic

Thirteen entries. A branch is dead here in one of three ways: it is written and then unconditionally undone, it can never be entered because its condition is unreachable, or it is entered and does nothing. All three hide the code that follows them.

### DC-27 A failed upload is recorded and then overwritten with success

- **Suspected defect:** The transport writes an upload-error status when the file transfer fails, and then, with no else and no early return, writes a success status over it on the very next statement.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L112-L121`. VERIFIED: `if (false === $sftp->put($x12_remote['x12_filename'], $claim_file_contents)) {` at `:L112` sets the upload-error status at `:L113`, appends the failure message at `:L114` and the transport's own errors at `:L115`, persists the row at `:L116`, and the block closes at `:L117`. VERIFIED: `:L120` then sets `$x12_remote['status'] = self::STATUS_SUCCESS;` unconditionally and `:L121` persists it. There is no `else`, no `continue` and no `return` between the two writes. VERIFIED: the four earlier failure branches in the same loop, at `:L70`, `:L85`, `:L96` and `:L104`, do continue past the row rather than falling through.
- **Why it looks wrong:** Four failure paths in the same method exit the iteration and the fifth does not, so the pattern the author established is broken in exactly one place. The error status and its messages are written to the database and then replaced within two statements, which no reading of the code makes deliberate. INFERRED (confidence: High). Basis: the four sibling branches establish the intended shape, and the messages written at `:L114-L115` are recorded specifically so that a human can read them, which is pointless if the row is immediately marked successful.
- **Observable symptom:** An undelivered transmission is displayed as delivered. The transport queue shows the row with a success badge, because the badge map at `interface/billing/billing_tracker.php:L83-L84` gives a distinct variant only to the success and waiting statuses and the stored status is now success. What is not lost is the failure text. `update()` at `src/Billing/BillingProcessor/X12RemoteTracker.php:L165-L167` receives the row as a by-value array and encodes only the copy it was handed, so the messages appended at `:L114-L115` are still an array in the caller when `:L121` persists the row again, and that second write replaces the status field alone. Those messages are returned to the browser at `library/ajax/billing_tracker_ajax.php:L50` and rendered as informational alert blocks by the row-detail formatter at `interface/billing/billing_tracker.php:L143-L147`, which runs only when an operator clicks to expand that row, at `interface/billing/billing_tracker.php:L174-L187`. The screen therefore asserts success and carries the evidence of failure at the same time, and nothing about a row presenting itself as successful invites the click that would reveal it. The practice believes the claims were filed, and the absence is discovered only when no remittance arrives and the timely-filing window has narrowed.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` that injects a transport double whose `put()` returns `false`, then asserts both the persisted status and the persisted message list; it runs under `phpunit-isolated.xml`. The expected status is the upload-error status, the actual status is the success status, and the message list contains the upload-failure text in both cases, which is what makes the row self-contradictory rather than merely wrong. Stage [S6](claim-lifecycle.md#stage-s6-transport-to-the-clearinghouse) is where the row is written.
- **Severity:** CRITICAL
- **Registered as a rule:** [BR-F3](business-rules.md#br-f3-a-failed-transmission-is-recorded-as-a-success).

```php
if (false === $sftp->put($x12_remote['x12_filename'], $claim_file_contents)) {
    $x12_remote['status'] = self::STATUS_UPLOAD_ERRROR;
```

That is `src/Billing/BillingProcessor/X12RemoteTracker.php:L112-L113`. The unconditional write that undoes it is `src/Billing/BillingProcessor/X12RemoteTracker.php:L120`.

### DC-28 A Medicare inpatient adjudication segment ends the parse where it appears

- **Suspected defect:** The remittance parser's final else returns an error string for any segment identifier it does not recognise, and its recognised set omits the Medicare inpatient adjudication segment that the legacy renderer in the same subsystem reads in detail.
- **Evidence:** `src/Billing/ParseERA.php:L467-L468`. VERIFIED: `} else {` followed by `return "Unknown or unexpected segment ID $segid";`, which is the parser's whole treatment of anything outside its recognised list. VERIFIED: `library/edihistory/edih_835_html.php:L531` tests `strncmp('MIA' . $de, (string) $seg, 4) === 0` and the branch below it renders covered days, the prospective-payment operating outlier, lifetime psychiatric days, the claim diagnosis-related group amount, remark codes, disproportionate-share and capital amounts.
- **Why it looks wrong:** The older component understands a segment the newer one rejects, which inverts the usual assumption that a replacement is at least as capable as what it replaces. The parser's own handling of the adjacent Medicare outpatient segment is a deliberate no-op at `src/Billing/ParseERA.php:L318-L319`, which shows the author knew such segments existed and chose to skip rather than reject them; the inpatient segment received neither treatment. INFERRED (confidence: High). Basis: the outpatient segment has an explicit ignore branch and the inpatient segment has none, and both are optional Medicare segments in the same loop.
- **Observable symptom:** A Medicare Part A remittance renders correctly in the EDI history viewer and posts only partway, leaving a deposit that can never be balanced. The check-scanning pass at `interface/billing/sl_eob_process.php:L850` runs first and is unaffected, because its branch chain ends at the TRN branch with no final else and therefore ignores the segment in silence; the deposit header rows it creates through `SLEOB::arPostSession()` at `interface/billing/sl_eob_process.php:L277-L285` are written at the full check amounts for every check the operator selected. The claim pass at `:L851` then stops at the segment. Claims closed before it have already been handed to the posting callback, because `src/Billing/ParseERA.php:L144`, `:L229`, `:L241` and `:L442` each flush the accumulated claim through `$cb($out)` at `:L81`, so their ledger lines exist. The claim carrying the segment is never flushed and neither is any claim after it in the file, so the deposit is left permanently under-distributed against a full check total. On a live posting run the operator sees the unknown-segment message appended to the page and, because the audit at `interface/billing/sl_eob_process.php:L853-L868` compares each deposit total against the sum of its ledger lines, a browser alert naming that check number as not fully distributed; the audit is inside the run's not-dry-run guard at `interface/billing/sl_eob_process.php:L853`, so a dry run shows only the first message. Neither message says which segment stopped the parse, and neither says that the claims after it were never read.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` under `phpunit-isolated.xml` supplying two claims, the second of which carries an MIA segment after its CLP segment, and count the callback invocations as well as the return value. The expected count is two and the expected return value is the empty string; the actual count is one, contributed by the flush that the second claim's CLP segment triggered for the first claim, and the actual return value is the unknown-segment string. The assertion is on delivery to the callback, which is the parser's own contract; whether a delivered claim then produces a ledger line is decided by the posting callback at `interface/billing/sl_eob_process.php:L297` and is outside the parser.
- **Severity:** CRITICAL
- **Registered as a rule:** [BR-F11](business-rules.md#br-f11-a-medicare-inpatient-adjudication-segment-ends-the-parse-where-it-appears).

```php
} else {
    return "Unknown or unexpected segment ID $segid";
```

That is `src/Billing/ParseERA.php:L467-L468`. The segment this rejects is rendered in detail at `library/edihistory/edih_835_html.php:L531-L545`.

### DC-29 The comment above the success write describes a transition the code does not make

- **Suspected defect:** The comment immediately above the unconditional success write says the status is changing from waiting to in progress, and the statement it labels sets the status to success.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L119-L120`. VERIFIED: the comment reads that the status changes from waiting to in-progress and the next statement is `$x12_remote['status'] = self::STATUS_SUCCESS;`. VERIFIED: the identical comment twelve lines above, at `:L107`, is accurate, because `:L108` does set the in-progress status.
- **Why it looks wrong:** The comment is a copy of a correct comment placed above a statement that does something else, which is the fingerprint of a block pasted and half-edited. Under the source-of-truth ordering this document inherits from [README.md](README.md), the code wins and the comment is evidence of intent only, and the intent it evidences is a status transition that was never written. INFERRED (confidence: High). Basis: the comment at `:L107` and the comment at `:L119` are byte-identical while the statements beneath them differ.
- **Observable symptom:** A maintainer reading the method believes the second write is a progress marker and does not see that it is the write that hides the failure recorded five lines earlier, which is why [DC-27](#dc-27-a-failed-upload-is-recorded-and-then-overwritten-with-success) has survived. The operator-visible symptom is the one described in that entry.
- **Verification:** A comment cannot be asserted, but the transition it claims can be. Add a case to `tests/Tests/Isolated/Billing/` under `phpunit-isolated.xml` that injects a tracker double recording the value of the status field on every call to `update()`, runs one queued row through the success path, and asserts the recorded sequence. The sequence the two comments together describe is the in-progress status written twice; the actual sequence is the in-progress status written once at `src/Billing/BillingProcessor/X12RemoteTracker.php:L109` and the success status written at `src/Billing/BillingProcessor/X12RemoteTracker.php:L121`. The equivalent reproduction without a test is to enable the automatic transport global read at `library/billing_sftp_service.php:L25`, queue one batch, and read the status column of that row in `x12_remote_tracker` after the run completes: the expected value on the comment's reading is the in-progress status, the actual value is the success status. This is row 2 of the thirteen-entry comment-versus-code census in [README.md](README.md), which lists all thirteen rows in full; it is also the row whose inclusion makes that census thirteen rather than twelve, as the index reconciles.
- **Severity:** MEDIUM

```php
// Change status from waiting to in-progress
$x12_remote['status'] = self::STATUS_SUCCESS;
```

That is `src/Billing/BillingProcessor/X12RemoteTracker.php:L119-L120`. The identical comment twelve lines above, at `src/Billing/BillingProcessor/X12RemoteTracker.php:L107`, is accurate.

### DC-30 The upload error status is declared under a misspelled identifier

- **Suspected defect:** The constant naming the upload-failure status carries a misspelling of the word error, with the letter r repeated once too often, and that identifier is now part of the class's public surface.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L30`. VERIFIED: the constant is declared as `STATUS_UPLOAD_ERRROR` with the value `'upload-error'`, among seven sibling status constants at `:L24-L31` whose names are all spelled correctly. VERIFIED: it is referenced at `:L113`, which is the only use in the repository.
- **Why it looks wrong:** The stored value is correct and only the identifier is misspelled, so nothing in the database or on any screen exposes the mistake, and the compiler cannot catch it. Any code written later that guesses the correctly spelled name fails at runtime rather than at parse time in a class that already fails to record failures. INFERRED (confidence: High). Basis: the seven neighbouring constants follow the same naming pattern spelled correctly, so this is a typing slip rather than a convention.
- **Observable symptom:** No operator sees anything today, because the constant's only use is the write at `:L113` whose effect [DC-27](#dc-27-a-failed-upload-is-recorded-and-then-overwritten-with-success) discards. The symptom appears the moment someone fixes that entry and refers to the status by the name they expect: an `Error: Undefined constant` fatal ends the transport run mid-queue, leaving rows in the in-progress state.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` asserting that the class exposes a constant for every value the `status` column of `x12_remote_tracker` can hold and that each is reachable by its documented name; it runs under `phpunit-isolated.xml`. The reproduction is a one-line script referencing the correctly spelled name and observing the fatal.
- **Severity:** MEDIUM

```php
const STATUS_UPLOAD_ERRROR = 'upload-error';
```

That is `src/Billing/BillingProcessor/X12RemoteTracker.php:L30`. Its seven siblings are declared at `src/Billing/BillingProcessor/X12RemoteTracker.php:L24-L31`.

### DC-31 The dry run branch of the deposit writer returns nothing

- **Suspected defect:** The deposit writer echoes its query and falls off the end of the function in dry-run mode, so it returns null where the other branch returns a primary key. The mismatch is latent rather than active, because every consumer of that key is behind the same flag that selects the returning branch, and what the dry run does print omits the values the insert would have bound.
- **Evidence:** `src/Billing/SLEOB.php:L98-L104`. VERIFIED: `if ($debug) {` at `:L98` echoes the query text at `:L99`; the else at `:L100` performs the insert at `:L102` and returns the identifier at `:L103`; the function ends at `:L104` with no return on the debug path. VERIFIED: the echoed string is the statement only, with its placeholders intact, because the bound array is supplied at `:L102` in the other branch and is printed nowhere.
- **Why it looks wrong:** The two branches of one function have different arities: one returns a value and one returns nothing, and the value is a primary key the caller stores in a global array. VERIFIED: the sole caller assigns the result into `$InsertionId` keyed by check number at `interface/billing/sl_eob_process.php:L277`, and all four readers of that array sit behind the same not-dry-run test as the returning branch, the three posting calls at `interface/billing/sl_eob_process.php:L559-L563`, `interface/billing/sl_eob_process.php:L621-L625` and `interface/billing/sl_eob_process.php:L642-L646`, and the post-run audit at `interface/billing/sl_eob_process.php:L853`. INFERRED (confidence: High). Basis: the guard alignment is what makes the null harmless today, and nothing in the writer's own signature or docblock records that its return type depends on an argument, so the alignment is a property of the one caller rather than of the contract.
- **Observable symptom:** In dry-run mode the operator sees the deposit insert previewed as a parameterised statement with question marks and no values, so the preview cannot be checked against what a live run would write; the statement is printed without the payer, dates, total and reference that would be bound into it. The null return itself is observable in nothing today. VERIFIED: it is stored in the global array at `interface/billing/sl_eob_process.php:L277` and never read on that path, because each reader is guarded, so a dry run neither posts with a null deposit identifier nor audits one. What the entry registers is therefore a contract that holds only while every caller keeps the two guards aligned, and the array it flows through is a global that a future reader could consume without one.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arPostSession` with the debug flag set and asserting the return value is an identifier or an explicit sentinel rather than null; it runs under `phpunit-isolated.xml`. The expected result is a single declared return type for both branches, the actual result is null. The reproduction is the remittance posting screen with dry run selected: expected is a preview showing the values that would be written, actual is the statement text with its placeholders and no values.
- **Severity:** MEDIUM

```php
if ($debug) {
    echo text($query) . "<br />\n";
} else {
```

That is `src/Billing/SLEOB.php:L98-L100`. The else branch returns at `src/Billing/SLEOB.php:L103` and the function ends at `src/Billing/SLEOB.php:L104`.

### DC-32 The institutional transaction type can never be reporting

- **Suspected defect:** The institutional generator chooses between the reporting and chargeable transaction type codes by reading a variable that its own signature does not declare, so the choice is decided by a null and the reporting branch is dead.
- **Evidence:** `src/Billing/X125010837I.php:L89`. VERIFIED: the BHT segment, the beginning of hierarchical transaction segment, is assembled at `:L83-L90` and its transaction type element is `(($encounter_claim ?? null) ? "*RP" : "*CH")`, where `RP` means reporting and `CH` chargeable. VERIFIED: the function's signature at `:L26` declares five parameters and none of them is `$encounter_claim`, so the null-coalescing operator always yields null and the chargeable branch always runs. The professional generator by contrast declares the parameter, at `src/Billing/X125010837P.php:L40-L46`, and its equivalent element at `:L114` reads it directly.
- **Why it looks wrong:** The expression exists to make a choice and the operand it tests cannot be set from outside the function. The null-coalescing operator is what conceals it: without it the read would raise a warning naming the undefined variable. INFERRED (confidence: High). Basis: the professional generator performs the same choice with a declared parameter, so the institutional expression is a partially applied copy of it.
- **Observable symptom:** An institutional claim generated while the site permits encounter claims is transmitted as chargeable rather than reporting. The payer adjudicates and pays a claim the practice intended only to report, so money arrives against an encounter that was never meant to be billed and must be refunded.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/X125010837IDateTest.php`, or a sibling file in the same directory, asserting the transaction type element of the generated BHT segment with the site switch for encounter claims enabled; it runs under `phpunit-isolated.xml`. The expected element is `RP`, the actual element is `CH`.
- **Severity:** MEDIUM
- **Registered as a rule:** [BR-F9](business-rules.md#br-f9-the-institutional-generator-always-declares-the-claim-chargeable).

```php
(($encounter_claim ?? null) ? "*RP" : "*CH") .  // RP = reporting, CH = chargeable
```

That is `src/Billing/X125010837I.php:L89`. The signature that does not declare the variable is `src/Billing/X125010837I.php:L26`.

### DC-33 The institutional claim can never carry the alternate payer identifier

- **Suspected defect:** The same undefined variable also chooses between the payer's alternate identifier and its primary identifier on the institutional claim, so the alternate identifier can never be emitted.
- **Evidence:** `src/Billing/X125010837I.php:L283`. VERIFIED: the payer name segment's identifier element is `(($encounter_claim ?? null) ? $claim->payerAltID() : $claim->payerID())`, and the same variable is undeclared in the signature at `:L26`. VERIFIED: the primary identifier is then tested for emptiness at `:L285`, which is the only validation of the value.
- **Why it looks wrong:** This is the second consumer of one undefined variable in one file, and unlike the transaction type code it changes where the claim is routed rather than how it is classified. A practice that configured an alternate payer identifier for encounter claims has no way to reach it on an institutional claim. INFERRED (confidence: High). Basis: the professional generator makes the same choice from a declared parameter at `src/Billing/X125010837P.php:L525`, so the intended behaviour is documented by the sibling.
- **Observable symptom:** An institutional encounter claim is addressed to the payer's normal claim identifier instead of the identifier configured for reporting, so the payer receives it on the wrong intake and either rejects it in a 277 claim status response or adjudicates it as a real claim.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` generating an institutional claim for an insurance company whose alternate payer identifier differs from its primary, and assert the identifier element of the payer name segment; it runs under `phpunit-isolated.xml`. The expected value is the alternate identifier, the actual value is the primary.
- **Severity:** HIGH

```php
"*" . (($encounter_claim ?? null) ? $claim->payerAltID() : $claim->payerID()) .
```

That is `src/Billing/X125010837I.php:L283`. The professional generator makes the same choice from a declared parameter at `src/Billing/X125010837P.php:L525`.

### DC-34 A claim level name segment matches every qualifier and does nothing

- **Suspected defect:** The last name-segment branch in the claim loop tests only the segment identifier and the loop, not the entity qualifier, and its body is commented out, so every unhandled claim-level name segment is silently swallowed by a branch whose comment names one specific case.
- **Evidence:** `src/Billing/ParseERA.php:L316-L317`. VERIFIED: `} elseif ($segid == 'NM1' && $out['loopid'] == '2100') { // PR = Corrected Payer` is followed by a single commented-out line that would have appended a warning. VERIFIED: the five branches above it, at `:L296`, `:L301`, `:L306`, `:L311` and `:L313`, each test a specific qualifier.
- **Why it looks wrong:** The comment claims the branch handles a corrected payer and the condition matches any qualifier at all, so a name segment the parser has not been taught about is treated as a corrected payer and then ignored. The warning that would have made the omission visible is present and disabled. INFERRED (confidence: High). Basis: the five sibling branches test their qualifier explicitly, and the commented-out body is a warning rather than a handler, which shows the author's own intent was to be told about this case.
- **Observable symptom:** A remittance that identifies a corrected payer, or any other claim-level party the parser does not read, posts as though the segment were absent. The operator sees no note, so a claim that a payer has redirected to a different carrier is posted against the original carrier and the crossover is never recorded.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a claim-level name segment with the corrected-payer qualifier and asserting the parsed output records it; it runs under `phpunit-isolated.xml`. The expected result is a recorded payer or a warning, the actual result is neither.
- **Severity:** MEDIUM

```php
} elseif ($segid == 'NM1' && $out['loopid'] == '2100') { // PR = Corrected Payer
    // $out['warnings'] .= "NM1 segment at claim level ignored.\n";
```

That is `src/Billing/ParseERA.php:L316-L317`. The five branches above it each test their qualifier.

### DC-35 The remittance charge helper ignores three of its ten parameters

- **Suspected defect:** The helper that adds a charge discovered in a remittance declares ten parameters and reads seven. Two separable things follow, and they must not be run together. The one that is happening today is that the service date the caller supplies is discarded, so every charge this path creates is dated by the moment of posting rather than by the date of service the payer reported. The one that is latent is that the dry-run flag is accepted and never read, so the guarantee that a preview writes nothing lives entirely in the caller and not in the function whose signature advertises it.
- **Evidence:** `src/Billing/SLEOB.php:L165`. VERIFIED: the signature declares `$patient_id, $encounter_id, $session_id, $amount, $units, $thisdate, $code, $description, $debug, $codetype`. VERIFIED: the body at `:L166-L208` reads none of `$session_id`, `$thisdate` or `$debug`, and the call it makes at `:L194-L207` passes twelve arguments to `BillingUtilities::addBilling`, whose signature at `src/Billing/BillingUtilities.php:L1434-L1452` accepts no date parameter at all. VERIFIED: that insert writes the date column as the literal `NOW()`, at `src/Billing/BillingUtilities.php:L1467-L1470`, into `billing`.`date`, declared `datetime default NULL` at `sql/database.sql:L247`, so the row's date is the insert time and nothing in the path can override it. VERIFIED: the caller supplies all three discarded values, at `interface/billing/sl_eob_process.php:L508-L519`, passing the deposit identifier as zero at `:L511`, the service date at `:L514` and the debug flag at `:L517`. VERIFIED: **that call is reached only through the guard `if (!$error && !$debug)` at `interface/billing/sl_eob_process.php:L507`, and it is the only call site in the application**, so as the code stands no preview reaches the insert.
- **Why it looks wrong:** Two arguments, one per half. The service date is computed and passed by name and then has nowhere to go, because the helper it is passed into does not forward it and the function that performs the insert does not accept it; a value named at the call site and then dropped before the statement that would have used it is an unfinished path rather than a choice. And a dry-run flag that is accepted and discarded is worse than no flag, because a future caller reading the signature has been told the run is safe. The sibling methods in the same class do honour the flag, at `:L98-L104` and `:L292-L302`, so the omission is local rather than a convention. INFERRED (confidence: High). Basis: the caller passes both values by name at `interface/billing/sl_eob_process.php:L514` and `:L517`, which is only meaningful if they are read, and the two siblings in the same file establish that reading the flag is this class's own convention.
- **Observable symptom:** With the site global on, every unmatched code a payer reports becomes a `billing` row dated by the posting run rather than by the date of service, so the charge does not line up with its own encounter on any date-ranged report and a practice reconciling charges by service date finds it under the wrong day. **On a preview the operator sees nothing wrong, because nothing is written:** the caller's guard blocks the call, so the discarded dry-run flag has no operator-visible symptom today at all, and that is the accurate statement rather than a missing one. Its symptom is conditional on a change elsewhere: a second caller written against this signature, or the removal or weakening of the guard at `interface/billing/sl_eob_process.php:L507`, would insert a charge during a preview with nothing in the helper to stop it, and the preview screen would look like a preview while the charge table had changed.
- **Verification:** Two tests, because the two halves are proved differently, and both run under `phpunit-isolated.xml`. First, a direct-helper contract test in `tests/Tests/Isolated/Billing/` calling `SLEOB::arPostCharge` with the debug flag set and a service date earlier than today: it asserts the current contract rather than the desired one, so its expected results are a `billing` row created despite the flag and a date column equal to the insert time rather than the supplied service date. That test documents the helper's behaviour and fails the day the helper starts honouring either parameter, which is the point. Second, an integration-style test of the caller in `tests/Tests/Isolated/Billing/` exercising the posting path with the dry-run option set and asserting that no `billing` row is created: it passes today and is the regression net that protects the guard at `interface/billing/sl_eob_process.php:L507` from being moved. The discarded flag is registered as [BR-F8](business-rules.md#br-f8-the-remittance-charge-helper-declares-a-dry-run-flag-and-never-reads-it) and the switch that reaches this path as [BR-I3](business-rules.md#br-i3-one-switch-turns-a-payer-reported-unknown-code-into-a-charge); both record the same division between helper contract and caller guard.
- **Severity:** HIGH

```php
public static function arPostCharge($patient_id, $encounter_id, $session_id, $amount, $units, $thisdate, $code, $description, $debug, $codetype = '')
```

That is `src/Billing/SLEOB.php:L165`. The three parameters never read are the third, the sixth and the ninth, and the insert that ends this path writes the date column as `NOW()` at `src/Billing/BillingUtilities.php:L1470`.

### DC-36 The dry run path of the re billing helper does nothing and says nothing

- **Suspected defect:** Both outcomes of the secondary-payer helper are wrapped in the same negated debug test, so on a dry run the method reaches its end having neither queued the claim nor reported what it would have done.
- **Evidence:** `src/Billing/SLEOB.php:L292-L302`. VERIFIED: `if ($new_payer_id) {` guards a claim update wrapped in `if (!$debug)` at `:L293-L295`, and the else at `:L296` guards a reopen wrapped in the same test at `:L298-L300`. There is no echo on either debug path. VERIFIED: the sibling deposit writer in the same class does echo its query on the debug path, at `:L99`.
- **Why it looks wrong:** The class contains two conventions for dry-run behaviour and this method follows neither: it does not act, and it does not report, so the two branches are indistinguishable from each other and from doing nothing at all. INFERRED (confidence: Medium). Basis: the echo at `:L99` establishes the reporting convention within the same file; whether the author intended silence here cannot be recovered from the code.
- **Observable symptom:** An operator previewing a remittance is not told whether the claim would be queued to a secondary payer or merely reopened, which is the one decision the preview exists to expose. The screen shows the secondary-billing message from `interface/billing/sl_eob_process.php:L720-L726` regardless, so the preview asserts an outcome the dry run did not evaluate.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arSetupSecondary` with the debug flag set and capturing output, asserting that the intended action is reported; it runs under `phpunit-isolated.xml`. The expected result is a described action, the actual result is no output and no change.
- **Severity:** MEDIUM

```php
if ($new_payer_id) {
    // Queue up the claim.
    if (!$debug) {
```

That is `src/Billing/SLEOB.php:L292-L294`. The else branch at `src/Billing/SLEOB.php:L296-L300` carries the same negated test.

### DC-37 The transaction set header and its trailer are governed by different conditions

- **Suspected defect:** In the professional generator the transaction set header and its trailer are emitted under two different inputs, one describing the claim's position in the batch and one a flag the caller passes, so keeping the pair matched is a caller obligation rather than a property of the generator. One caller in the repository supplies the two inconsistently, and what stops that from mattering is a routing rule in a third file.
- **Evidence:** `src/Billing/X125010837P.php:L91-L99`. VERIFIED: the ST segment, the transaction set header, is emitted when the hierarchical level count is one and the per-payer switch is on, or whenever that switch is off. VERIFIED: the SE segment, the transaction set trailer, is emitted at `:L1612-L1615` when the last-claim flag is true or that same switch is off. The two conditions share only their second arm. VERIFIED: the direct generator supplies both consistently, taking the level count from the number of claims already added to the partner's batch at `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L224`, taking the flag from the marking loop at `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L132-L140`, and passing them together at `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L241-L251`; that marking loop is the one [DC-3](#dc-3-the-last-claim-for-a-partner-is-chosen-by-an-identity-comparison-across-types) covers. VERIFIED: the other caller supplies neither, hardcoding a level count of one at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L67` and false for the flag at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L76`.
- **Why it looks wrong:** A transaction set is a matched pair by definition, and here its two halves are decided by two inputs that no single component owns. VERIFIED: with the per-payer switch on, the hardcoding caller would open a transaction set for every claim and close none, and the only thing preventing that is the dispatcher at `src/Billing/BillingProcessor/BillingProcessor.php:L165-L176`, which routes to the direct generator whenever the switch is on and reaches the hardcoding caller only when it is off, where both conditions' second arms make the pair match. VERIFIED: the two parameters' own defaults, at `src/Billing/X125010837P.php:L46-L47`, satisfy neither condition while the switch is on, so a caller that omits both receives a claim body with no header and no trailer. INFERRED (confidence: High). Basis: the pairing is correct by routing rather than by construction, and the generator neither checks that a header was written for the trailer it writes nor records the obligation anywhere a caller would see it; the segment count written into the trailer at `src/Billing/X125010837P.php:L1616-L1621` is right only if exactly one header was written for it, and nothing checks that either.
- **Observable symptom:** Nothing on any path reachable from the billing manager today, and the entry states that rather than implying a rejection an operator would see. VERIFIED: the one live route by which the pair can come apart is the direct generator failing to mark a last claim, which is [DC-3](#dc-3-the-last-claim-for-a-partner-is-chosen-by-an-identity-comparison-across-types); on that route nothing intervenes, because the batch keeps every header it is handed and renumbers it at `src/Billing/BillingProcessor/BillingClaimBatch.php:L240-L249`, keeps every trailer at `src/Billing/BillingProcessor/BillingClaimBatch.php:L259-L261`, and then writes a group trailer counting the headers at `src/Billing/BillingProcessor/BillingClaimBatch.php:L275`, so a file carrying one header and no trailer is transmitted as though complete and refused out of band by the clearinghouse's acknowledgement rather than on the billing screen. What this entry carries is the contract rather than a present failure: any new caller that copies the hardcoded arguments and runs with the switch on reproduces the unmatched case immediately.
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/`, both under `phpunit-isolated.xml`. Neither invents an omission of the last-claim flag: the first honours the contract the direct caller keeps, and the second reproduces the arguments the other caller in the repository actually hardcodes. The first calls the generator twice for one partner with the per-payer switch on exactly as the direct generator does, a level count of one then two with the last-claim flag false then true, and asserts one header and one trailer across the two outputs; expected and actual agree, which is what establishes that the pair is matched today. The second calls it twice with the arguments the other caller hardcodes, a level count of one and the flag false, with the switch on: the expected result is one header and one trailer, the actual result is two headers and no trailer. The segment count convention is registered as [BR-H7](business-rules.md#br-h7-the-segment-count-is-copied-from-the-generator-into-the-batch-unchanged).
- **Severity:** MEDIUM

```php
$SEFLAG == true
|| !OEGlobalsBag::getInstance()->getBoolean('gen_x12_based_on_ins_co')
```

That is `src/Billing/X125010837P.php:L1613-L1614`. The header's condition, at `src/Billing/X125010837P.php:L91-L99`, shares only the second of these two arms.

### DC-38 An unrecognised action button dereferences a null task

- **Suspected defect:** The task builder is a chain of thirteen tests with no final else, and it returns null when none matches. The caller dereferences the result immediately, so an unrecognised submission ends the request with a fatal error instead of a message.
- **Evidence:** `src/Billing/BillingProcessor/BillingProcessor.php:L159-L192`. VERIFIED: `$processing_task = null;` at `:L159` opens a chain that ends at `:L191-L192` with no else branch. VERIFIED: the caller at `:L84` calls `$processing_task->getAction()` with no null test, and `:L94` calls `getLogger()` on the same value. VERIFIED: the action extractor at `:L210-L222` independently returns null when none of the three buttons is present.
- **Why it looks wrong:** Two independent nullable results are consumed without a test in a method whose input is an HTTP post, which is the least trustworthy input the subsystem has. The instance check at `:L194` shows the author knew the value might not be a task, because it guards the logger assignment; the guard is applied to the assignment and not to the dereference. INFERRED (confidence: High). Basis: the `instanceof` test at `:L194` and the unguarded `->` at `:L84` are eleven lines apart and disagree about whether the value can be trusted.
- **Observable symptom:** A submission whose button name the chain does not recognise, which includes any future button and any replayed or hand-built form post, produces a blank page or a PHP fatal error in the billing manager rather than a message. The claims are untouched, so the operator retries and cannot tell whether the first attempt did anything.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` constructing the processor with a post array containing no recognised button and asserting a caught exception or a returned message rather than a fatal; it runs under `phpunit-isolated.xml`. Alternatively reproduce on the billing manager at `interface/billing/billing_report.php` by submitting the form with no recognised action.
- **Severity:** HIGH

```php
$claims = $this->prepareClaims($processing_task->getAction());
```

That is `src/Billing/BillingProcessor/BillingProcessor.php:L84`. The chain that can return null ends without an else at `src/Billing/BillingProcessor/BillingProcessor.php:L191-L192`.

### DC-39 The tertiary payer is never queued from the posting screen

- **Suspected defect:** The call that advances a claim to its next payer is guarded by a test that the remittance is from the primary payer and that a secondary payer exists, so the tertiary level is unreachable from the remittance path.
- **Evidence:** `interface/billing/sl_eob_process.php:L717`. VERIFIED: `if ($primary && SLEOB::arGetPayerID($pid, $service_date, 2)) {` guards the call at `:L718`, and the guard's second term asks specifically for payer type two. VERIFIED: the helper it calls does contain tertiary handling, at `src/Billing/SLEOB.php:L285-L287`, which from this caller can only be reached with a watermark of two, and the guard prevents the call whenever the remittance is not from the primary payer. VERIFIED: another caller reaches the same helper with no payer-level test at all, from the invoice screen at `interface/billing/sl_eob_invoice.php:L405`, which is where the tertiary watermark does arrive and is registered as [DC-2](#dc-2-the-next-payer-level-is-chosen-by-an-unparenthesised-mixed-condition).
- **Why it looks wrong:** The helper implements three levels and this caller can express one transition. The message printed beside the call, at `interface/billing/sl_eob_process.php:L720-L726`, names secondary paper billing explicitly, so the screen's own text acknowledges that only one transition is contemplated. INFERRED (confidence: High). Basis: the coverage lookup inside the helper at `src/Billing/SLEOB.php:L290` recomputes the payer for whatever level the watermark implies, which is code that cannot run for the tertiary level under this guard.
- **Observable symptom:** After a secondary payer's remittance posts, the balance is never queued to a tertiary payer and no message appears. The operator sees nothing at all: not an error, not a note, not a claim in the work list. The encounter simply stops moving and the remaining balance ages until someone finds it in an accounts-receivable report. The condition inside the helper is [DC-2](#dc-2-the-next-payer-level-is-chosen-by-an-unparenthesised-mixed-condition).
- **Verification:** Reproduce on the remittance posting screen by posting a secondary payer's remittance for a patient who has three coverage rows: expected is a tertiary claim queued and a message, actual is neither. A test would go in `tests/Tests/Isolated/Billing/` and run under `phpunit-isolated.xml`.
- **Severity:** HIGH

```php
if ($primary && SLEOB::arGetPayerID($pid, $service_date, 2)) {
```

That is `interface/billing/sl_eob_process.php:L717`. The second term asks for payer type two specifically.

## Category 5 Unvalidated Assumptions About Segment Ordering

Nine entries about position and arity: the order segments arrive in, how many elements each carries, and whether a value that comes back from a payer still has the shape it was sent in. As stated in [How the categories are drawn](#how-the-categories-are-drawn), this category also covers an envelope trailer that assumes it matches the header its own code wrote.

### DC-40 The claim identifier is recovered by interpolating a payer supplied value into a query

- **Suspected defect:** The claim identifier is recovered from the payer's remittance by splitting a payer-supplied string on spaces and hyphens, and one of the resulting parts is interpolated directly into an SQL string while the part beside it on the same line is bound as a parameter.
- **Evidence:** `src/Billing/SLEOB.php:L28-L64`. VERIFIED: `:L30` takes the value from `$out['our_claim_id']`, which the parser reads from element one of the CLP segment at `src/Billing/ParseERA.php:L262`; `:L31` splits it with `preg_split('/[ -]/', (string) $invnumber);`; and the three-part branch at `:L39-L44` builds a query whose predicate at `:L42` reads `"pid = '$pid' AND encounter = ? AND activity = 1"` with `$pid` interpolated and the encounter bound.
- **Why it looks wrong:** One statement demonstrates both conventions at once, which shows the parameterised form was available and was applied to one value and not the other. The interpolated value is not local input: it travelled to the payer on the claim and came back in the payer's file, so its shape at this point is whatever the payer returned. INFERRED (confidence: High). Basis: the encounter on the same line is bound, so the author knew how; and the identifier's provenance is traceable from `src/Billing/ParseERA.php:L262` without inference.
- **Observable symptom:** A remittance whose claim identifier is not in the expected shape resolves to the wrong encounter or to none, so a payment posts against another patient's encounter or is reported as unmatched. The operator sees a payment on a patient who has no such claim, or a claim the screen says it cannot find while the payer's paper explanation of benefits names it plainly. The security aspect of the same line is flagged, and only flagged, in [Appendix A Security Sensitive Observations](#appendix-a-security-sensitive-observations).
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::slInvoiceNumber` with an `our_claim_id` of the three-part shape whose first part is not a plain integer, and assert the resolved patient and encounter; it runs under `phpunit-isolated.xml`. The expected result is a refusal to resolve, the actual result depends on the value. The recovery rule itself is registered as [BR-D5](business-rules.md#br-d5-the-claim-identifier-is-recovered-from-the-remittance-by-counting-its-parts).
- **Severity:** CRITICAL

```php
$brow = sqlQuery("SELECT encounter FROM billing WHERE " .
    "pid = '$pid' AND encounter = ? AND activity = 1", [$atmp[1]]);
```

That is `src/Billing/SLEOB.php:L41-L42`. One value on the second line is interpolated and the other is bound.

### DC-41 The interchange header rebuild indexes four elements it never proves exist

- **Suspected defect:** The batch rebuilds the interchange header by taking the first seventy characters of the incoming header and then reading elements eleven, twelve, fourteen and fifteen by index, with no test that the header carries them.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L216-L217`. VERIFIED: the rebuild is `substr((string) $seg, 0, 70)` concatenated with the batch date and time, then `$elems[11]`, `$elems[12]`, the batch interchange control number, `$elems[14]`, `$elems[15]` and the literal `"*:~"`, where `$elems` is the result of `explode('*', $seg)` at `:L213`. VERIFIED: the rebuilt string's length is then required to be exactly 105 characters at `:L219-L221`, and a shorter one ends the request through the `die` on `:L221`.
- **Why it looks wrong:** The length check immediately below proves the author knew the result could be the wrong size, and the response to that is to end the request rather than to validate the input that produced it. Four unguarded index reads on a value that came from a generator whose output the batch does not control is an arity assumption with a hard failure attached. INFERRED (confidence: High). Basis: the check at `:L220` exists precisely because the rebuild can go wrong, and it fires after the reads rather than before.
- **Observable symptom:** One undefined-key warning per missing element, followed by the message about the header needing to be 105 characters, printed into a half-rendered billing page. VERIFIED: no partial file is left behind by this path, because the rebuild assigns only the in-memory accumulator `$this->bat_content` and the only disk write in the class is the `fopen`, `fwrite` and `fclose` sequence inside `write_batch_file()` at `:L158-L166`, which the terminating call never reaches; the transport rows that method would insert at `:L170-L186` are likewise never inserted. What does persist is described in [DC-49](#dc-49-a-malformed-interchange-header-ends-the-request-after-the-claim-was-marked-billed), which is the same terminal path reached with a different input: the claim rows the caller has already updated.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` under `phpunit-isolated.xml` calling `append_claim` with a header segment truncated after element ten. The expected result is a thrown exception or a recorded error that leaves the run able to continue; the actual result is a request-ending failure preceded by undefined-key warnings. The existing test at `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php:L132` already drives `append_claim` and is the natural place for the case.
- **Severity:** HIGH

```php
$this->bat_content = substr((string) $seg, 0, 70) . "$this->bat_yymmdd*$this->bat_hhmm*" . $elems[11] .
    "*" . $elems[12] . "*" . $this->bat_icn . "*" . $elems[14] . "*" . $elems[15] . "*:~";
```

That is `src/Billing/BillingProcessor/BillingClaimBatch.php:L216-L217`. The length check that fires afterwards is `src/Billing/BillingProcessor/BillingClaimBatch.php:L219-L221`.

### DC-42 The reference rewrite uses an unchecked search result as an offset

- **Suspected defect:** The batch rewrites the transaction reference by searching the segment for a literal and passing the search result straight into `substr_replace` as an offset, with no test that the literal was found.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L254`. VERIFIED: `$this->bat_content .= substr_replace($seg, '*' . "1" . '*', strpos((string) $seg, '*0123*'), 6);` and the comment above it at `:L253` names the generator that plants the literal.
- **Why it looks wrong:** When the literal is absent, `strpos` returns `false`, which `substr_replace` treats as offset zero, so the replacement overwrites the first six characters of the segment instead of the reference. The result is a segment whose identifier has been destroyed rather than a segment left unchanged. INFERRED (confidence: High). Basis: the comment records a cross-file coupling to the generator that plants the needle, which is exactly the kind of coupling that breaks silently when one side changes; nothing in the batch verifies the needle's presence.
- **Observable symptom:** A claim whose transaction segment begins with a corrupted identifier, which the clearinghouse rejects in a 997 or 999 acknowledgement naming an unrecognised segment. The billing screen reports the batch as generated. The reference value that this rewrite substitutes is itself the subject of [DC-48](#dc-48-the-transaction-reference-is-a-fixed-literal-at-generation-and-after-the-batch-rewrite).
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` calling `append_claim` with a transaction segment that does not contain the literal, and assert the emitted segment is unchanged; it runs under `phpunit-isolated.xml`. The expected result is an unchanged segment, the actual result is one with its first six characters replaced. The rewrite is registered as [BR-H6](business-rules.md#br-h6-the-batch-renumbers-the-transaction-set-and-rewrites-the-reference-it-planted).
- **Severity:** HIGH

```php
$this->bat_content .= substr_replace($seg, '*' . "1" . '*', strpos((string) $seg, '*0123*'), 6);
```

That is `src/Billing/BillingProcessor/BillingClaimBatch.php:L254`. A search that finds nothing yields `false`, which is offset zero.

### DC-43 A claim level adjustment that arrives after a service line lands on that line

- **Suspected defect:** Whether an adjustment is treated as claim-level or service-level is decided by the parser's loop state rather than by the segment's own position, so an adjustment that arrives out of the expected order is attributed to a service line, and in one ordering it ends the parse of the whole file.
- **Evidence:** `src/Billing/ParseERA.php:L269`. VERIFIED: the claim-level adjustment branch is gated on the loop being `2100` and places its adjustments in service index zero, creating a zero-charge line named `Claim` only if that index is empty, at `:L275-L283`. VERIFIED: the service-level branch at `:L381-L382` is gated on the loop being `2110` and places its adjustments in `count($out['svc']) - 1`, the last line parsed. VERIFIED: the loop identifier becomes `2110` only inside the service branch and is set to `2000` by the header-number segment at `:L230`, and there is no adjustment branch for `2000`, so an adjustment arriving in that state reaches the final else at `:L467-L468`.
- **Why it looks wrong:** Three orderings of the same three segments produce three different outcomes, and only one of them is correct. The comment at `:L270` calls the claim-level case unusual, which is evidence the author did not expect to have to disambiguate it. INFERRED (confidence: High). Basis: the two branches differ only in their loop guard and their index, so the classification of a real payer segment depends entirely on what the parser happened to read last.
- **Observable symptom:** A claim-level adjustment sent after the service lines is posted against the last service line, so the invoice shows a write-off on a procedure the payer did not adjust while the claim-level balance stays open, and nothing on the screen distinguishes that line from a genuine service-level adjustment. In the third ordering the operator is shown the unknown-segment message and the parse stops at that point: claims that the flush points at `:L144`, `:L229`, `:L241` and `:L442` had already closed have posted their ledger lines, while the claim being read and every claim after it in the file have not, so the deposit is left under-distributed against its full check total. That partial outcome is bounded the same way as in [DC-28](#dc-28-a-medicare-inpatient-adjudication-segment-ends-the-parse-where-it-appears).
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/ParseERATest.php`, both under `phpunit-isolated.xml`. The first sends a claim-level adjustment after a service line and asserts which service index carries the amount: the expected index is the dummy `Claim` line at zero, the actual index is the last real service line. The second sends an adjustment immediately after a header-number segment and asserts the return value: the expected value is the empty string, the actual value is the unknown-segment string.
- **Severity:** HIGH

```php
$i = 0; // if present, the dummy service item will be first.
if (!($out['svc'][$i] ?? '')) {
```

That is `src/Billing/ParseERA.php:L275-L276`. The service-level branch at `src/Billing/ParseERA.php:L381-L382` indexes the last line instead.

### DC-44 The branch chain tests elements it never proves the segment carries

- **Suspected defect:** The parser's branch chain selects on element one of a segment, and reads element two in several bodies, without ever establishing that the segment has that many elements, so a short segment raises undefined-key warnings inside the selection itself.
- **Evidence:** `src/Billing/ParseERA.php:L296`. VERIFIED: `} elseif ($segid == 'NM1' && $seg[1] == 'QC' && ...)` reads element one with no guard, and the same unguarded read appears in the adjustment, amount, remark and name branches, at `:L269`, `:L419`, `:L424` and `:L301-L313`. VERIFIED: the amount branch then reads `(float)$seg[2]` at `:L421` and the remark branch `$seg[2]` at `:L426`, both unguarded, while other reads in the same function do use the null-coalescing form, for example `:L299`, `:L314` and `:L433`.
- **Why it looks wrong:** The same function protects some element reads and not others, and the unprotected ones are in the selection chain, where a failure affects every subsequent branch test rather than one handler. A remittance is external input and its segments can be short for reasons the practice does not control. INFERRED (confidence: High). Basis: the guarded reads at `:L299` and `:L314` establish the author's own convention, and the chain does not follow it.
- **Observable symptom:** Undefined-key warnings in the error log for each short segment during remittance posting, and on a site configured to display errors the warnings are printed into the posting screen's table, breaking the layout an operator is reading amounts from. An amount segment with no amount element records an allowed note of zero rather than none.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a name segment with only its identifier and an amount segment with a qualifier and no amount, and assert that no warning is raised and the parse completes; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
} elseif ($segid == 'NM1' && $seg[1] == 'QC' && $out['loopid'] == '2100') { // QC = Patient
```

That is `src/Billing/ParseERA.php:L296`. The guarded convention the same function uses elsewhere is visible at `src/Billing/ParseERA.php:L299`.

### DC-45 Unterminated bytes after the interchange trailer are discarded and the parse reports success

- **Suspected defect:** Leaving the scan because the buffer holds no segment terminator is the parser's normal way of reaching the end of a file, and it is also what happens when bytes are left over. The post-loop test cannot tell those apart, because it asks which segment was parsed last rather than whether anything was left unread. So once the interchange trailer has been parsed, a remainder that carries no terminator is dropped and the parse returns success.
- **Evidence:** `src/Billing/ParseERA.php:L107-L110`. VERIFIED: `$tpos = strpos($buffer, $delimiter1); if ($tpos === false) { break; }` sits above the split, and the buffer is topped up to 2048 characters at `:L103-L105`, so a run of bytes carrying no terminator ends the loop rather than being reported. VERIFIED: the function's post-loop test at `:L474-L476` is `if ($segid != 'IEA') { return 'Premature end of ERA file'; }`, which asks what the **last parsed segment identifier** was and not how the loop ended or whether the buffer still held anything. VERIFIED: the consequence of that shape is a clean split, and the useful half of it is not the half a reader expects. A file cut mid-segment anywhere before the interchange trailer leaves `$segid` holding some earlier identifier, so it does return the premature-end message at `:L475`, which is the behaviour [claim-lifecycle.md](claim-lifecycle.md) records for that stage at `src/Billing/ParseERA.php:L474-L476`. A remainder that follows an already-parsed `IEA` leaves `$segid` holding `IEA`, so the test passes and control reaches `return '';` at `:L478`, the value that means success. VERIFIED: this is the same exit an ordinary well-formed file takes, because the buffer is emptied of line endings at `:L115` and an emptied buffer contains no terminator either, which is why the exit cannot be treated as an error in itself.
- **Why it looks wrong:** The check that exists is a proxy for the condition the author wanted, and it is a proxy that holds on one side of the trailer and not the other. Testing the last identifier catches an incomplete file and cannot notice an unconsumed remainder, so the bytes after the trailer are neither examined nor mentioned; the loop leaves no record of how much of the buffer it never split. INFERRED (confidence: Medium). Basis: the premature-end message exists at all, so the author intended the function to refuse an incomplete file, and a test of the last identifier is the cheapest expression of that intent given that the normal exit and the lossy exit are the same statement. Whether the interval after the trailer was considered cannot be recovered from the code.
- **Observable symptom:** A remittance whose interchange trailer is followed by content that carries no segment terminator posts everything up to the trailer, reports success, and says nothing about the remainder. The concrete shape of that input is narrow and worth stating exactly, because a wider claim would not be true: the truncation has to fall before the **first** terminator after the trailer, since a complete segment there would be parsed and would then leave a non-`IEA` identifier behind and be caught. A transmission carrying a second interchange that was cut off inside its own interchange header is exactly that case - the first interchange's deposits and claims post normally, the second is discarded in silence, and the operator's only route to noticing is reconciling the practice's deposits against the payer's own statement of what it sent. Nothing on the posting screen, and nothing in the parser's returned message, names the discarded bytes.
- **Verification:** Two cases in `tests/Tests/Isolated/Billing/ParseERATest.php`, both under `phpunit-isolated.xml`, because the entry is only meaningful if the two conditions are shown to be distinguished. The first feeds a well-formed remittance whose `IEA` segment is followed by a partial segment carrying no terminator, for example the first thirty characters of a second interchange header, and asserts that parsing reports an error naming the unconsumed remainder: the expected result is an error string, the actual result is the empty string that means success. The second is a control that cuts the same file mid-segment **before** the trailer and asserts the premature-end message, which passes today; it exists so that a reader can see that the two inputs differ only in the last identifier parsed and that only one of them is caught.
- **Severity:** MEDIUM

```php
if ($segid != 'IEA') {
    return 'Premature end of ERA file';
}
```

That is `src/Billing/ParseERA.php:L474-L476`. The break that can leave content unread is at `src/Billing/ParseERA.php:L107-L109`, and the success return this test falls through to is at `src/Billing/ParseERA.php:L478`.

### DC-46 The institutional envelope trailers contradict their own headers

- **Suspected defect:** In the institutional generator the interchange and group control numbers in the headers are drawn from the sequence allocator, and the matching numbers in the trailers are hardcoded literals, so a single generated claim carries trailers that do not match its own headers.
- **Evidence:** `src/Billing/X125010837I.php:L59`. VERIFIED: the interchange control number element is `BillingClaimBatchControlNumber::getIsa13()` and the group control number at `:L70` is `getGs06()`. VERIFIED: the group trailer at `:L1213-L1214` is the literal `"*1" . "*1"` and the interchange trailer at `:L1216-L1217` is the literal `"*1" . "*000000001"`. VERIFIED: the batch post-processor discards both trailers, at `src/Billing/BillingProcessor/BillingClaimBatch.php:L264-L266`, and writes its own at `:L272-L279`, so the contradiction is repaired on the batched path only.
- **Why it looks wrong:** One file writes a control number twice, once from an allocator and once as a literal, and the standard requires the two to be equal. The batch layer's unconditional rebuild is the only reason the mismatch never leaves the process, which means the generator's output is acceptable not because it is correct but because nothing consumes it directly. INFERRED (confidence: High). Basis: the professional generator has the same shape, with the literal trailers at `src/Billing/X125010837P.php:L1624-L1632` matched by a literal interchange number in its own header at `src/Billing/X125010837P.php:L73` and the comments at `src/Billing/X125010837P.php:L69-L70` recording other header elements as dummy data for the batch to replace, so the professional file is internally consistent about being a placeholder while the institutional file had its headers modernised and its trailers left behind.
- **Observable symptom:** No mismatched envelope reaches a payer or a screen today, and the register is explicit about that rather than implying a transmission defect. VERIFIED: the only caller of the institutional generator is `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L46`, and every path from there runs the output through `append_claim`, which discards both literal trailers at `src/Billing/BillingProcessor/BillingClaimBatch.php:L264-L266` and also rebuilds both headers, the interchange at `:L216-L217` and the group at `:L234-L236`, before `append_claim_close()` writes four fresh values at `:L275` and `:L278`. VERIFIED: the validate-only display is no exception, because it echoes the batch's accumulated content after that close, at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L183-L188`. VERIFIED: nor is the mismatch stored, because the institutional path persists `json_encode($this->ub04id)` into `claims.submitted_claim`, at `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L80-L93` and `:L112-L125`, and the re-disposal path reads that column back and decodes it as JSON at `interface/billing/ub04_dispose.php:L190` and `:L207`, so no generated X12 text is ever stored or replayed. What is observable is the cost of the discarded headers: the generator's two draws at `src/Billing/X125010837I.php:L59` and `:L70` come from the same allocator the batch itself draws from at `src/Billing/BillingProcessor/BillingClaimBatch.php:L63` and `:L66`, both resolving to `QueryUtils::ediGenerateId()`, so two sequence values are consumed per institutional claim and appear in no transmitted envelope. A partner comparing interchange control numbers across submissions sees gaps proportional to the number of institutional claims in the run.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` under `phpunit-isolated.xml` generating one institutional claim and asserting that the interchange trailer's control number equals the header's and likewise for the group: the expected values are equal pairs, the actual values are the allocator's numbers in the headers against the literals one and one in the trailers. A second case asserts the sequence cost: record the allocator's value before and after generating one institutional claim through the batch, then compare against the count of distinct interchange control numbers in the written file. The expected difference is one draw per envelope emitted; the actual difference is two more than that. The two-draw allocation is registered as [BR-H4](business-rules.md#br-h4-interchange-and-group-control-numbers-are-two-separate-draws-on-one-sequence).
- **Severity:** MEDIUM

```php
$out .= "GE" . // GE Trailer
    "*1" . "*1" . "~\n";
```

That is `src/Billing/X125010837I.php:L1213-L1214`. The headers these close are drawn from the allocator at `src/Billing/X125010837I.php:L59` and `src/Billing/X125010837I.php:L70`.

### DC-47 The interchange trailer is written even when no functional group was opened

- **Suspected defect:** The batch's closing routine guards the group trailer on a group having been opened and writes the interchange trailer unconditionally, using the same group count as its first element, so a batch with no group emits an interchange trailer declaring zero groups.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L272-L279`. VERIFIED: `if ($this->bat_gscount) {` guards the group trailer at `:L274`, and `:L278` appends the interchange trailer with `$this->bat_gscount` as its first element and the batch interchange control number as its second, outside any guard.
- **Why it looks wrong:** The two trailers close nested envelopes and are guarded differently. If the guard on the inner one is necessary, the outer one cannot be safe to write unguarded, because an interchange with no functional group is not a valid interchange. INFERRED (confidence: Medium). Basis: the guard on the group trailer shows the author anticipated a batch with no group; whether that state is reachable in production cannot be established from this file alone, because it depends on which task called the writer and whether any claim survived validation.
- **Observable symptom:** A batch file consisting of an interchange trailer declaring zero functional groups, queued for transmission like any other. The clearinghouse rejects the interchange with a TA1 acknowledgement and the operator sees a successfully generated batch on the billing screen.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` calling `append_claim_close` on a batch to which no claim was appended, and assert that no interchange trailer is emitted; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
$this->bat_content .= "IEA" . "*" . $this->bat_gscount . "*" . $this->bat_icn . "~";
```

That is `src/Billing/BillingProcessor/BillingClaimBatch.php:L278`. The group trailer above it is guarded at `src/Billing/BillingProcessor/BillingClaimBatch.php:L274`.

### DC-48 The transaction reference is a fixed literal at generation and after the batch rewrite

- **Suspected defect:** The reference identification on the transaction segment is the same literal for every claim the system generates, and the batch that rewrites it substitutes another constant, so no claim carries a reference that distinguishes it from any other.
- **Evidence:** `src/Billing/X125010837P.php:L111`. VERIFIED: the element is `"*" . "0123" .` with the trailing comment naming it a reference identification, and the institutional generator writes the same literal at `src/Billing/X125010837I.php:L86`. VERIFIED: the batch's rewrite at `src/Billing/BillingProcessor/BillingClaimBatch.php:L254` replaces the literal with `'*' . "1" . '*'`, which is a second constant rather than a per-claim value.
- **Why it looks wrong:** The element's purpose is to let a submitter tie a later response back to a specific submission, and a constant cannot do that. That the batch rewrites it at all shows the value was recognised as needing replacement; what it was replaced with does not vary. INFERRED (confidence: High). Basis: the transaction set control number in the adjacent rewrite at `:L259-L261` is per-transaction, so the batch had a varying value available on the same code path and did not use it.
- **Observable symptom:** A 277 claim status response or an acknowledgement that quotes the reference cannot be matched back to a submission, so an operator chasing one claim's status must identify it by patient and date instead. This is the condition the 2016 documentation already complained about, recorded with its history in [Appendix B Historical Precedent](#appendix-b-historical-precedent).
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` appending two claims and asserting that their transaction segments carry different reference identifications; it runs under `phpunit-isolated.xml`. The expected result is two distinct values, the actual result is the same constant twice.
- **Severity:** HIGH

```php
"*" . "0123" .                             // reference identification
```

That is `src/Billing/X125010837P.php:L111`. The institutional generator writes the same literal at `src/Billing/X125010837I.php:L86`.

## Category 6 Resource and Lifecycle Handling

Eight entries about the lifetime of something: a request, a file handle, a remote session, or a database transaction. As stated in [How the categories are drawn](#how-the-categories-are-drawn), this category covers requests that end part-way through a run as well as handles that are never released.

### DC-49 A malformed interchange header ends the request after the claim was marked billed

- **Suspected defect:** The batch writer ends the whole request with `die` when the rebuilt interchange header is not exactly 105 characters, or when the input does not begin with a header at all. The claim was already marked billed two calls earlier, and the call that records which batch file it went into never runs.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L219-L221`. VERIFIED: `$isa_length = strlen($this->bat_content) - 1;` then `if ($isa_length != 105) {` then `die("Error:<br />\n ISA must be 105 characters in length; " . "found $isa_length instead")`. VERIFIED: a second `die` at `:L225-L226` fires when the accumulated content is empty and the segment is not a header. VERIFIED the ordering in `src/Billing/BillingProcessor/Tasks/GeneratorX12.php`: the claim is marked billed at `:L151-L162`, the batch append that can die is called at `:L165`, and the update that records the batch filename is at `:L168`. VERIFIED: the validate-and-clear path puts the same two steps in the same order, marking the claim billed at `:L122-L133` and calling the append that can die afterwards at `:L136`; it records no filename at all, passing an empty string for that column at `:L130`.
- **Why it looks wrong:** A per-claim data problem is handled by terminating the process, which abandons every claim after it in the run and leaves the claims before it in an inconsistent state: marked billed, each pointing at a batch filename for a file the run never got as far as writing. Every other error path in the same subsystem records a message and continues. INFERRED (confidence: High). Basis: a non-terminal reporting route exists and is used for exactly this class of failure three lines after the call that dies, at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L168-L169`, where a claim update that comes back false prints a message and the run continues; that call reaches the billing logger through the trait at `src/Billing/BillingProcessor/Traits/WritesToBillingLog.php:L34-L36`.
- **Observable symptom:** The billing manager page stops mid-render with the message about the header's length and nothing else, so the operator sees a truncated page carrying an explicit error. What persists is entirely in the database, and it is worse than a partial file would be. VERIFIED: no file is left in the outbound directory and no transport row is queued, because the accumulated content lives only in the batch object's in-memory string and the single disk write is inside `write_batch_file()` at `src/Billing/BillingProcessor/BillingClaimBatch.php:L158-L166`, reached from `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L202` only after the whole claim loop has completed. VERIFIED: every claim already processed in the run is marked billed, and each of those also had its batch filename written to `billing.process_file` by the second update at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L168`, naming a file that now will never exist; the claim that failed is marked billed with that column left empty, because the termination happened between its two updates. None of them can be selected for billing again. INFERRED (confidence: High). Basis: the Billing Manager reports generation rather than delivery, rendering the line about the claim having been generated to a file together with a link to it at `interface/billing/billing_report.php:L1227-L1228`, so those claims present as generated and link to nothing.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` under `phpunit-isolated.xml` calling `append_claim` with content whose first segment is not a header, which is the condition the second termination tests. The expected result is a thrown exception that the caller can record and continue past; the actual result is process termination. A reproduction is stage [S5](claim-lifecycle.md#stage-s5-envelope-post-processing): generate a batch for a partner whose interchange identifiers are shorter than the standard's fixed-width fields, then query `billing` for the encounters in that run and observe that they carry the billed status while the outbound directory holds no file for them.
- **Severity:** CRITICAL
- **Registered as a rule:** [BR-F4](business-rules.md#br-f4-a-malformed-interchange-header-terminates-the-whole-batch-run).

```php
if ($isa_length != 105) {
    die("Error:<br />\n ISA must be 105 characters in length; " . "found $isa_length instead");
```

That is `src/Billing/BillingProcessor/BillingClaimBatch.php:L220-L221`. The claim was marked billed before this ran, at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L151-L162`.

### DC-50 Neither remittance entry point closes the file it opened

- **Suspected defect:** Both remittance entry points open the file with `fopen` and neither closes it. There is no `fclose` anywhere in the class, and both functions return from many places.
- **Evidence:** `src/Billing/ParseERA.php:L91`. VERIFIED: the main parse opens the file there and the check-scanning parse opens it again at `:L488`. VERIFIED: the file contains no `fclose` call at all, and the main parse alone carries twenty-three return statements between `:L85` and `:L479`, including the early return at `:L92-L94` and every error return in the segment chain. VERIFIED: the handle variable appears only at `:L91`, `:L92`, `:L103` and `:L104` in the main parse and at `:L488`, `:L489`, `:L500` and `:L501` in the check-scanning parse, and in neither function is it returned, assigned to a property, a global, an array or a static, or captured by a closure.
- **Why it looks wrong:** A handle opened in a function that returns from twenty-three places needs either a close on each path or a construct that closes it for you, and neither is present. What keeps this from being a leak is nothing the code states: it is that the handle is a function-local whose only reference disappears when the function returns, so the engine releases it there. INFERRED (confidence: High). Basis: the two properties that make the release safe, locality of the variable and the absence of any second reference to it, can be removed by an edit anywhere in either function without touching the open site, and no test asserts either property.
- **Observable symptom:** Nothing today, and that is worth stating rather than leaving the field to imply otherwise. Because both handles are function-local and neither function calls the other, at most one handle to the remittance file exists at any moment; the two calls the posting screen makes at `interface/billing/sl_eob_process.php:L849-L852` are operands of one concatenation and so are evaluated one after the other, not together. What the missing close does create is a silent dependency: the upload path renames the very file it just parsed, at `interface/billing/era_payments.php:L161` and `:L170`, and those renames are reached only after the parse has returned and the handle has been released by scope exit rather than by the parser. If a later change stores the handle, passes it to a helper, or moves either parse inside a loop that retains its result, the first visible sign is the open-failure message from `src/Billing/ParseERA.php:L92-L94` reported against a remittance file the operator can see is present and readable, with nothing on the screen naming descriptors as the cause.
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/ParseERATest.php`, both under `phpunit-isolated.xml`, as regression guards for a property that currently holds only by accident. The first reads the entry count of `/proc/self/fd` immediately before and immediately after a parse and asserts the two are equal. The second calls the check-scanning parse, then renames the parsed path, then parses the renamed path, and asserts each step succeeds. The expected result in both cases is success, and both pass against the code as it stands; the second is the one that fails first if the handle is ever made to outlive the call, which is the change this entry exists to guard against.
- **Severity:** MEDIUM

```php
$infh = fopen($filename, 'r');
```

That is `src/Billing/ParseERA.php:L91`. The second entry point opens the same way at `src/Billing/ParseERA.php:L488`, and the file contains no matching close.

### DC-51 The claim version is drawn by an unlocked aggregate inside a transaction

- **Suspected defect:** The next claim version is computed with a plain aggregate read, with no row lock and no retry, and the value becomes part of the row's primary key, so two concurrent runs for one encounter compute the same number and the second insert fails outright.
- **Evidence:** `src/Billing/BillingUtilities.php:L1679`. VERIFIED: the read is `'SELECT IFNULL(MAX(version), 0) + 1 AS increment FROM claims WHERE patient_id = ? AND encounter_id = ?'` with no `FOR UPDATE` clause and no unique-key retry around it. VERIFIED: the read is inside a transaction wrapper, `QueryUtils::inTransaction(function () use (...): void {` at `:L1677`, so the write is atomic; what is missing is the lock, not the transaction. VERIFIED: `version` is part of the primary key, at `sql/database.sql:L392`, and its own column comment at `:L381` concedes that it is incremented in code.
- **Why it looks wrong:** The transaction makes the sequence of writes atomic and does nothing to make the read of the maximum exclusive, so at the default isolation level two transactions can read the same maximum. Because the value is a key component rather than an ordinary column, the loser does not overwrite anything; it fails. INFERRED (confidence: High). Basis: the schema comment names the allocation as a code responsibility and the query carries no locking clause, and the sibling allocator in `src/PaymentProcessing/Recorder.php` documents the identical gap in its own comment, per [DC-52](#dc-52-the-ledger-sequence-number-is-drawn-the-same-way-and-the-code-says-so).
- **Observable symptom:** A claim-generation run ends with a duplicate-key error for a patient whose encounter is being billed from two sessions at once, or from one session and a background run. The operator sees a database error rather than a billing message, and whichever claim lost the race is not recorded at all.
- **Verification:** Add a case to `tests/Tests/Services/Billing/` that opens two connections, reads the maximum version on both before either inserts, and asserts that the second insert either succeeds with a distinct version or is retried; it runs under the primary configuration, which covers `tests/Tests/Services` at `phpunit.xml:L67-L69`. The allocation is registered as [BR-H1](business-rules.md#br-h1-the-claim-version-is-allocated-by-an-unlocked-aggregate-inside-a-transaction).
- **Severity:** HIGH

```php
$version = sqlQuery(
    'SELECT IFNULL(MAX(version), 0) + 1 AS increment FROM claims WHERE patient_id = ? AND encounter_id = ?',
```

That is `src/Billing/BillingUtilities.php:L1678-L1679`. The transaction wrapper around it is `src/Billing/BillingUtilities.php:L1677`; what is absent is a locking clause.

### DC-52 The ledger sequence number is drawn the same way and the code says so

- **Suspected defect:** The accounts-receivable ledger allocates its per-claim sequence number with the same unlocked aggregate inside a transaction, and the code carries a comment conceding the race and naming the locking clause that would close it.
- **Evidence:** `src/PaymentProcessing/Recorder.php:L207-L215`. VERIFIED: the allocation reads the maximum existing sequence number for the patient and encounter and adds one, with no locking clause, and it is invoked inside the transaction opened at `:L169`. VERIFIED: the comment at `:L202-L206` states that the read is subject to a race and names a locking select as a possible remedy. VERIFIED: `sequence_no` is part of the primary key of `ar_activity`, at `sql/database.sql:L10210`, and its own comment at `:L10191` says it is incremented in code.
- **Why it looks wrong:** The defect is not that it is unknown but that it is known, recorded beside the code, and left. Two remittance postings for one encounter, which is ordinary when a practice has several billers, race for one key. INFERRED (confidence: High). Basis: the author's own comment is the strongest available evidence, and this document takes comments as evidence of intent only, which is exactly what this one is.
- **Observable symptom:** A duplicate-key failure while posting a remittance, surfacing as a database error on the posting screen part-way through a claim, with some ledger lines written and the rest not. Because [DC-24](#dc-24-the-error-flag-accumulates-across-the-service-lines-of-a-claim) already leaves claims part posted, the two are hard to tell apart from the screen.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/`, or alongside the payment-processing tests, that calls `Recorder::recordActivity` twice concurrently for one encounter and asserts distinct sequence numbers; it runs under `phpunit-isolated.xml`. The allocation is registered as [BR-H2](business-rules.md#br-h2-the-ledger-sequence-number-is-allocated-the-same-way-and-the-code-says-so). This entry and [DC-51](#dc-51-the-claim-version-is-drawn-by-an-unlocked-aggregate-inside-a-transaction) are the same defect in two components and are registered together so that a fix to one is not mistaken for a fix to both.
- **Severity:** HIGH

```php
// Note: even in a default-configured DB transaction, this still has
// a potential race condition. It should either be done as a subquery in
// the insert, or using a locking read (SELECT...FOR UPDATE may work?)
```

That is `src/PaymentProcessing/Recorder.php:L204-L206`. The allocation the comment describes is `src/PaymentProcessing/Recorder.php:L207-L215`.

### DC-53 The transport reads the claim file after a fallback without rechecking it

- **Suspected defect:** The transport tries the partner's local directory, silently substitutes a second path when the file is not there, and then reads that second path without testing whether it exists.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L75-L81`. VERIFIED: `$claim_file` is built from the partner's local directory at `:L75`, replaced with a path under the site's document directory at `:L76-L78` when `file_exists` fails, and passed to `file_get_contents` at `:L80` with no second existence test. The false result is caught at `:L81`, which records the claim-file-error status naming the fallback path.
- **Why it looks wrong:** The first path is checked and the second is not, so the read on the more likely-to-be-missing path is the unguarded one. The recorded message names only the second path, so the operator is told the file is missing from a location they never configured. INFERRED (confidence: Medium). Basis: the existence test at `:L76` shows the author checks before reading, and the substituted path skips that check; whether the message's wording was deliberate cannot be recovered.
- **Observable symptom:** A warning in the error log naming a path under the site's document directory, and a transport row whose message reports that a claim file could not be opened at a path the operator did not set, when the real problem is that the partner's configured directory is wrong. The queue row is skipped and the claims are never sent.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` that queues a transport row whose partner directory does not contain the file and whose fallback path does not either, then asserts the recorded message names both paths; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
$claim_file_contents = file_get_contents($claim_file);
```

That is `src/Billing/BillingProcessor/X12RemoteTracker.php:L80`. The path it reads was substituted without a second existence test at `src/Billing/BillingProcessor/X12RemoteTracker.php:L76-L78`.

### DC-54 A row the transport has claimed is never released or retried

- **Suspected defect:** The transport marks a queued row as in progress before it uploads, and nothing anywhere in the repository reads that status again. The only fetch asks for the waiting status, so a row whose request does not survive as far as the success write is stranded permanently. Two failure branches in the same method show the same absence of cleanup on the connection, leaving the object without the explicit disconnect the success path performs.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L108`. VERIFIED: the in-progress status is written there and persisted at `:L109`. VERIFIED: a repository-wide search for that constant returns exactly two occurrences, its declaration at `:L29` and that one write; nothing reads it, filters on it, or resets it. VERIFIED: `fetchByStatus()` at `:L195-L205` declares the waiting status as its default and is called exactly once, at `:L58`, with the waiting status; rows enter the queue carrying that same status, at `src/Billing/BillingProcessor/BillingClaimBatch.php:L181`; the only two files in the repository that touch the tracking table are this class and the background service; and the only entry point into the path is the call at `library/billing_sftp_service.php:L26`, gated on the global read at `library/billing_sftp_service.php:L25`. VERIFIED: the login-failure branch at `src/Billing/BillingProcessor/X12RemoteTracker.php:L91-L97` and the directory-change failure branch at `src/Billing/BillingProcessor/X12RemoteTracker.php:L99-L105` each record a status and `continue`, while the only `disconnect` call is at `src/Billing/BillingProcessor/X12RemoteTracker.php:L123-L124`, after the upload; the connection object is created inside the loop at `src/Billing/BillingProcessor/X12RemoteTracker.php:L89`, so the same variable is reassigned on the next iteration.
- **Why it looks wrong:** Claiming a work item is only safe when something can release the claim, and this claim is written to a column no reader consults. The directory-change failure is the sharper of the two connection cases, because it occurs after a successful login and so abandons an authenticated session rather than a failed attempt, and the disconnect at the bottom of the loop shows that explicit closing was intended rather than left to the runtime. INFERRED (confidence: High). Basis: the background service's own docblock at `library/billing_sftp_service.php:L17-L22` describes the path as sending the files that are in the waiting status, so the in-progress status reads as a progress marker written for a reader that was never built rather than as a state the design expected to persist.
- **Observable symptom:** Under normal control flow the in-progress status survives only a few statements, because `src/Billing/BillingProcessor/X12RemoteTracker.php:L120-L121` replaces it. It persists whenever the request does not reach that line, which a fatal error, a script timeout, or an upload that hangs long enough to be killed will each produce. When it persists the row is stranded: no later run collects it, because the only fetch asks for the waiting status; no screen offers a way to reset it, because nothing outside this class and the background service writes the table; and the billing tracker renders it with the generic warning badge produced by the fallback at `interface/billing/billing_tracker.php:L83-L84`, the same badge every status other than success and waiting receives. The operator sees a batch that is neither queued nor sent nor marked failed, with no action available on the row, while the claims inside it are already marked billed and so cannot be selected again. For the connection half there is nothing observable at the practice, and the earlier reading of this entry was wrong to suggest otherwise: VERIFIED, the object created at `src/Billing/BillingProcessor/X12RemoteTracker.php:L89` is reassigned on the next iteration, so at most one abandoned connection is referenced at any moment and none accumulate across rows. INFERRED (confidence: Low) on what the remote server sees, and this register deliberately leaves it unresolved. Basis: whether the abandoned socket is closed promptly depends on the destructor of the locked transport library, phpseclib/phpseclib 3.0.55 at `composer.lock:L7743-L7745`, and the vendor tree is absent from the checkout these documents were written against, so under the source-of-truth ordering in [README.md](README.md) that code was not read and no claim is made about it.
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/`, both under `phpunit-isolated.xml`. The first injects a transport double whose `put()` throws, drives one queued row, and then calls the send method a second time: the expected result is that the row is picked up and retried, the actual result is that the fetch returns nothing because the row now carries the in-progress status. The second injects a transport double counting `disconnect()` calls, queues two rows whose directory change fails, and asserts one disconnect per row: the expected count is two, the actual count is zero. The reproduction without a test is to set the in-progress status by hand on one row of `x12_remote_tracker`, run the background service at `library/billing_sftp_service.php:L26`, and open the billing tracker screen: the expected outcome is that the row is either transmitted or marked failed, the actual outcome is that it is untouched and shows a generic warning badge with no available action.
- **Severity:** MEDIUM

```php
// Change status from waiting to in-progress
$x12_remote['status'] = self::STATUS_IN_PROGRESS;
$remoteTracker->update($x12_remote);
```

That is `src/Billing/BillingProcessor/X12RemoteTracker.php:L107-L109`. The only fetch in the path asks for the waiting status, at `src/Billing/BillingProcessor/X12RemoteTracker.php:L58`, and the two branches that skip the disconnect at `src/Billing/BillingProcessor/X12RemoteTracker.php:L123-L124` are `src/Billing/BillingProcessor/X12RemoteTracker.php:L91-L97` and `src/Billing/BillingProcessor/X12RemoteTracker.php:L99-L105`.

### DC-55 Deposit rows are committed by the first pass before the file is validated

- **Suspected defect:** The posting screen parses the remittance twice in one request. The first pass creates the deposit rows and the second pass discovers whether the file can be parsed at all, so a file that fails in the second pass has already left deposits behind.
- **Evidence:** `interface/billing/sl_eob_process.php:L849-L852`. VERIFIED: `:L850` calls the check-scanning parse, whose callback creates the deposits at `:L277-L285`, and `:L851` then calls the main parse, whose failure modes include the unknown-segment return at `src/Billing/ParseERA.php:L467-L468` and the premature-end return at `:L474-L476`. VERIFIED: nothing between the two passes is transactional, and no path removes the deposits created by the first pass.
- **Why it looks wrong:** The order places the write before the validation. Both passes read the same file with the same parser, so the second pass's failures were discoverable before anything was written. INFERRED (confidence: High). Basis: the two calls are adjacent statements, so reordering or a dry first pass was available; and every failure the second pass can report is a property of the file rather than of the posting.
- **Observable symptom:** A remittance that fails to parse leaves one `ar_session` deposit per check in the file with no ledger lines against them. The payment search screen lists deposits with a total and nothing posted, and an operator who retries the corrected file creates a second set, because the duplicate probe cannot see the first set for the reason given in [DC-5](#dc-5-the-duplicate-deposit-probe-searches-for-a-reference-the-writer-never-stores).
- **Verification:** Reproduce on the remittance posting screen with a file whose first check is well formed and whose later content contains a segment the parser rejects: expected is no deposit and an error, actual is a deposit for the first check and an error. Stage [S11](claim-lifecycle.md#stage-s11-accounts-receivable-posting) is where the write happens.
- **Severity:** HIGH

```php
ParseERA::parseERAForCheck($eraFilePath)
. ParseERA::parseERA($eraFilePath, 'eob_process_era_callback')
```

That is `interface/billing/sl_eob_process.php:L850-L851`. The first of the two creates the deposits; the second decides whether the file can be read.

### DC-56 The results page closes the document before the download script is written

- **Suspected defect:** The billing results page emits its closing markup, then runs the callback that can print a script tag, then emits the body close, so any script the callback prints lands outside the document and the closing tags are out of order.
- **Evidence:** `interface/billing/billing_process.php:L61-L65`. VERIFIED: the file is 65 lines long; `:L61` closes the HTML element, `:L63` calls the logger's completion hook, and `:L65` closes the body element after it. VERIFIED: the completion hook is what prints the download script for the generated batch, through the helper called at `src/Billing/BillingProcessor/Tasks/GeneratorHCFA_PDF.php:L198-L206` and `:L228-L232`.
- **Why it looks wrong:** The two closing tags are in the wrong order relative to each other, and the one statement that can still produce output sits between them. Browsers recover from misnested closing tags, which is why this has survived, and script execution after the document close is not guaranteed. INFERRED (confidence: Medium). Basis: the body close after the HTML close is unambiguous, and the callback between them is the only remaining output; whether a given browser runs the script is a property of the browser rather than of the code.
- **Observable symptom:** An operator whose run produced a downloadable batch is sometimes not offered the download, with no message and no error, and re-running the report does not offer it either because the claims are already marked billed. The behaviour differs between browsers, which is what makes it hard to report.
- **Verification:** Reproduce by running a paper-claim generation from the billing manager and inspecting the served markup for a script tag positioned after the document close; expected is a script inside the body, actual is a script between the two closing tags. This is a rendering defect with no unit-test surface, so the reproduction is the verification.
- **Severity:** MEDIUM

```php
</html>
<?php
$logger->onLogComplete();
```

That is `interface/billing/billing_process.php:L61-L63`. The body element is closed after all of this, at `interface/billing/billing_process.php:L65`.

## Category 7 Dead Configuration

Five entries where an operator can set something that changes nothing, or where the system promises a capability it does not have. These are defects because a configuration screen is a contract: an editable field asserts that the value is used.

### DC-57 Five trading partner columns have no consumer

- **Suspected defect:** Five columns on the trading-partner record are declared in the schema, exposed on the partner edit form, and read by nothing except the model that declares them.
- **Evidence:** `sql/database.sql:L10052` declares `x12_token_endpoint`, `:L10054` `x12_claim_status_endpoint`, `:L10055` `x12_attachment_endpoint`, `:L10056` `x12_client_id` and `:L10057` `x12_client_secret`. VERIFIED by repository-wide search: each is referenced only by its own property declaration and accessor pair in `library/classes/X12Partner.class.php`, at `:L49-L54` and `:L437-L494`, and by its input on the partner edit template at `templates/x12_partners/general_edit.html:L180-L215`. No other file reads any of the five values. VERIFIED that the sixth endpoint column in the same run, `x12_eligibility_endpoint` at `sql/database.sql:L10053`, **is live**: it is consumed at `src/Billing/EDI270.php:L798`, which is why the dead set is five columns and not six.
- **Why it looks wrong:** The five form one coherent unbuilt feature: an authenticated service client with a token endpoint, a client identifier, a client secret, a claim-status endpoint and an attachment endpoint. The one endpoint of the six that has a consumer is the one whose transaction the subsystem actually generates. INFERRED (confidence: High). Basis: the sixth column of the same shape, added in the same schema region, does have a consumer, so the five are an incomplete implementation rather than a convention of storing unused settings.
- **Observable symptom:** An operator configures a claim-status endpoint and credentials on the partner form, saves successfully, and no claim-status inquiry is ever sent. Nothing on the form or in any log indicates that the values are inert, so the practice believes claim status is configured and waits for responses that cannot arrive. The credential columns among the five are additionally flagged in [Appendix A Security Sensitive Observations](#appendix-a-security-sensitive-observations).
- **Verification:** Reproduce on the trading-partner edit screen, which is served by the front dispatcher at `controller.php:L19-L20` through `controllers/C_X12Partner.class.php` and rendered from the template cited above: set all five values, save, reload to confirm they persisted, and then search the codebase for any read of the stored values. Expected is at least one consumer per editable field, actual is none for these five. A test would assert that every column exposed on the partner form has a reader outside the model, and would run under the primary configuration.
- **Severity:** MEDIUM

```sql
`x12_claim_status_endpoint` tinytext,
`x12_attachment_endpoint` tinytext,
```

That is `sql/database.sql:L10054-L10055`. The other three inert columns are `sql/database.sql:L10052`, `sql/database.sql:L10056` and `sql/database.sql:L10057`; the live one is `sql/database.sql:L10053`.

### DC-58 The attachment segment names a transport that was never built

- **Suspected defect:** The professional generator emits a paperwork segment announcing an electronic attachment, with hardcoded qualifiers and one empty element, above a comment that says attachments are not implemented, and it does not increment the segment counter.
- **Evidence:** `src/Billing/X125010837P.php:L778-L784`. VERIFIED: the comment block states that medical attachments are still to be implemented and lists the values it hardcodes. VERIFIED: `:L785-L793` emits the PWK segment anyway when the claim is employment related, with `OZ` as the report type code, `EL` as the transmission code, an empty element at `:L788`, `AC` as the attachment control number qualifier and the interchange control number at `:L791`. VERIFIED: no `++$edicount` accompanies the emission, unlike the segments around it.
- **Why it looks wrong:** The claim tells the payer that supporting documentation will arrive electronically, and no code anywhere sends it; the endpoint that would have carried it is one of the five dead columns in [DC-57](#dc-57-five-trading-partner-columns-have-no-consumer). Separately, a segment emitted without being counted makes the transaction set trailer's count too low by one for every claim that takes this branch. INFERRED (confidence: High). Basis: the author's comment states the feature is not implemented, and the missing counter increment is visible by comparison with every neighbouring emission in the same function.
- **Observable symptom:** Two symptoms from one branch. A payer receiving an employment-related claim waits for an attachment that never arrives and either denies the claim for missing documentation or holds it, and the practice sees a denial reason it cannot act on. And the transaction set trailer undercounts, so the clearinghouse may reject the interchange in a 997 or 999 acknowledgement for a segment count mismatch.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` generating an employment-related professional claim and asserting that the count in the transaction set trailer equals the number of segments actually emitted; it runs under `phpunit-isolated.xml`. The uncounted segment is registered as [BR-F6](business-rules.md#br-f6-the-attachment-segment-is-emitted-without-being-counted).
- **Severity:** HIGH

```php
if ($claim->isRelatedEmployment()) {
    $out .= "PWK" .
        "*" . "OZ" .
```

That is `src/Billing/X125010837P.php:L785-L787`. No counter increment accompanies it, unlike every neighbouring emission.

### DC-59 A partner model property has no backing column and is still editable

- **Suspected defect:** The trading-partner model declares a version property and initialises it in its constructor, the edit form offers a select for it and the list screen renders it, and the column it maps to was dropped from the schema by an upgrade script.
- **Evidence:** `library/classes/X12Partner.class.php:L37`. VERIFIED: `public $x12_version;` is declared among the persisted properties, assigned a literal implementation-guide identifier in the constructor at `:L68`, and exposed through accessors at `:L237-L245` and a list helper at `:L420`. VERIFIED: `sql/5_0_0-to-5_0_1_upgrade.sql:L513` is `ALTER TABLE \`x12_partners\` DROP COLUMN \`x12_version\`;`, guarded by a column-exists directive at `:L512`. VERIFIED: no `x12_version` column exists in `sql/database.sql`. VERIFIED: the value is still rendered at `templates/x12_partners/general_list.html:L26` and offered as a select at `templates/x12_partners/general_edit.html:L121-L123`. VERIFIED: the persistence base class writes only columns the table actually has, because it iterates the live field list, at `src/Common/ORDataObject/ORDataObject.php:L61-L102` and specifically `:L69`.
- **Why it looks wrong:** An operator is shown a select, chooses a value, saves, and the value is discarded by the persistence layer without an error, then reappears on the next load as the constructor's literal. The screen asserts a choice the system cannot keep. INFERRED (confidence: High). Basis: the drop script is explicit and the persistence layer's field iteration is explicit, so the discard is a consequence of two verified mechanisms rather than a supposition.
- **Observable symptom:** On the trading-partner list screen every partner displays the same implementation-guide version regardless of what was selected, and a changed selection silently reverts. A practice using a payer that requires a different guide version has no way to record it.
- **Verification:** Reproduce on the trading-partner edit screen: change the version select, save, reload, and observe the original value. Expected is the saved value, actual is the constructor's literal. A test would go in `tests/Tests/Services/Billing/` asserting that every property the model persists corresponds to a column returned by the live field list, and would run under the primary configuration, which covers `tests/Tests/Services` at `phpunit.xml:L67-L69`.
- **Severity:** MEDIUM

```sql
#IfColumn x12_partners x12_version
ALTER TABLE `x12_partners` DROP COLUMN `x12_version`;
```

That is `sql/5_0_0-to-5_0_1_upgrade.sql:L512-L513`. The property that still maps to it is `library/classes/X12Partner.class.php:L37`.

### DC-60 The processing format is stored per claim and branches on nothing

- **Suspected defect:** Each claim reads the trading partner's processing format, stores it on the claim and writes it to the charge table, and nothing branches on the value. The property's own documentation records the author's doubt that it affects output.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaim.php:L137-L142`. VERIFIED: the constructor runs `"SELECT x.processing_format from x12_partners as x where x.id =?"` per claim and assigns the result to the `target` property. VERIFIED: the property's docblock at `:L81-L89` states that the value does not appear to have any effect on the output format other than to indicate what was selected and to be stored with the claim. VERIFIED: the column's six permitted values are declared at `sql/database.sql:L10031`.
- **Why it looks wrong:** A per-claim database query is executed to obtain a value the code says it does not act on, and the value is nonetheless written into the charge table where later readers will assume it means something. The docblock records uncertainty rather than a decision, so the field's status is unresolved in the code itself. VERIFIED: the column is read in exactly two places, the trading-partner edit model at `library/classes/X12Partner.class.php:L357-L365` and this constructor at `src/Billing/BillingProcessor/BillingClaim.php:L137-L142`, and the value the constructor stores leaves the subsystem only as the target argument of the claim updater, at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L131`, `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L160`, `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L174` and `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L90`; the updater's only read of the stored value copies it onto the next claim version, at `src/Billing/BillingUtilities.php:L1574`. INFERRED (confidence: Medium). Basis: the docblock is evidence of intent only, per the source-of-truth ordering in [README.md](README.md), and while the read census above is exhaustive for this identifier, a negative claim that no consumer exists anywhere in the repository is an inference rather than an observation.
- **Observable symptom:** An operator selecting a different processing format for a partner sees no change in any generated file. The value is recorded against every claim, so a later report or integration that groups by it produces a grouping with no behavioural meaning.
- **Verification:** Two generated files cannot be compared byte for byte, because repeated generation is not deterministic: the transaction dates and times are derived from the clock at `src/Billing/X125010837P.php:L51` and `src/Billing/X125010837I.php:L28`, and the batch derives its own timestamps, its filename and both control numbers the same way at `src/Billing/BillingProcessor/BillingClaimBatch.php:L59-L66`. Two workable routes replace it. The first asserts the absence of a consumer: add a case to `tests/Tests/Isolated/Billing/BillingClaimTest.php` under `phpunit-isolated.xml` that builds the claim against two partner rows differing only in this column and asserts that the two claims differ in no property any generator reads; the expected result is a difference the generators act on, the actual result is a difference confined to the one property they only pass through. The second normalises the nondeterministic fields: generate the same claim twice under the validate action, where the batch substitutes fixed literals for both control numbers at `src/Billing/BillingProcessor/BillingClaimBatch.php:L63` and `src/Billing/BillingProcessor/BillingClaimBatch.php:L66` and rebuilds the interchange and group headers from its own clock at `src/Billing/BillingProcessor/BillingClaimBatch.php:L216-L217` and `src/Billing/BillingProcessor/BillingClaimBatch.php:L234-L236`, discarding the control numbers the generators drew for themselves, including the institutional generator's at `src/Billing/X125010837I.php:L59` and `src/Billing/X125010837I.php:L70`. The clock-derived elements that remain must be blanked before the two runs are compared: the date and time pairs the batch writes into those two rebuilt headers, and the transaction header's creation date and time at `src/Billing/X125010837P.php:L112-L113`, which survive because the batch keeps that segment and rewrites only its reference identification at `src/Billing/BillingProcessor/BillingClaimBatch.php:L254`. Compare the batch content rather than the artifact, whose name is derived from the clock at `src/Billing/BillingProcessor/BillingClaimBatch.php:L64`. The expected result is a difference between the two formats, the actual result is identity. The rule is registered as [BR-D3](business-rules.md#br-d3-the-processing-format-is-read-per-claim-stored-and-branches-on-nothing).
- **Severity:** MEDIUM

```php
$sql = "SELECT x.processing_format from x12_partners as x where x.id =?";
$result = sqlQuery($sql, [$this->getPartner()]);
```

That is `src/Billing/BillingProcessor/BillingClaim.php:L137-L138`. Nothing branches on the value the query returns.

### DC-61 A component separator is assigned and never read in the check parse

- **Suspected defect:** The check-scanning parse declares three delimiters and reads two, so the composite-element separator it sets up, and then dutifully relearns from the interchange header, is never used for anything. Separately, the one delimiter neither entry point ever learns is the segment terminator that both read loops split on.
- **Evidence:** `src/Billing/ParseERA.php:L486`. VERIFIED: `$delimiter3 = '^';` is assigned among the three delimiters at `:L484-L486`, is reassigned from the interchange header at `:L517`, and is read nowhere in the function; the only reads of that variable in the whole file are at `:L355`, `:L356` and `:L360`, all inside the main parse, where they split composite procedure codes. VERIFIED: the check-scanning parse does learn both the element separator and the component separator from the header, at `:L515-L518`, in lines identical to the main parse's probe at `:L118-L121`. VERIFIED: neither function ever reassigns the segment terminator, which both initialise to the tilde at `:L87` and `:L484` and both read at `:L107` and `:L504`.
- **Why it looks wrong:** The two functions were written from one another and only one kept the code that uses the third delimiter, so the probe in the copy recovers a value the copy has no use for. An unused local would be harmless on its own; what makes it a defect candidate is that the probe carries both the cost and the appearance of delimiter awareness while the one delimiter that decides whether the file can be read at all remains a constant in both functions. INFERRED (confidence: High). Basis: the assignment at `:L486` and the reassignment at `:L517` are line-for-line counterparts of statements in the main parse that do have readers, which is the signature of a copy rather than of an intended behaviour.
- **Observable symptom:** Nothing follows from the dead assignment itself. What is visible follows from the shared constant: a remittance whose segment terminator is not the tilde makes the search at `:L504` fail on the first iteration, so the loop breaks at `:L505-L507`, the check counter stays at zero, and the check callback at `:L553` is invoked with a check count of zero before the function returns the premature-end message at `:L555-L557`. The posting screen therefore offers no deposits at all and reports the file as ending prematurely while the file itself is valid, and an operator with nothing else to try re-uploads it. The same constant defeats the main parse over the same file, which is [DC-26](#dc-26-the-delimiter-probe-can-only-run-on-the-first-segment-of-the-interchange).
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/ParseERATest.php`, both under `phpunit-isolated.xml`. The first calls the check-scanning parse on a remittance whose segment terminator is a character other than the tilde and asserts the recovered check count: the expected count is the number of checks in the file, the actual count is zero. The second calls it on a remittance whose element separator is not the pipe and asserts the same count; that case passes, and it is what establishes that the element separator, unlike the segment terminator, is recovered from the file.
- **Severity:** MEDIUM

```php
$delimiter1 = '~';
$delimiter2 = '|';
$delimiter3 = '^';
```

That is `src/Billing/ParseERA.php:L484-L486`. Only the first two are read in that function, and the first of them is the one never relearned from the file.

## Appendix A Security Sensitive Observations

This appendix is deliberately short. The instruction governing it is to flag security-sensitive observations in input handling and SQL construction, not to analyse them: each item below is a citation, a one-line description and a severity, and nothing further. No exploit path is described, no impact is assessed beyond the one line, and nothing is remediated here or anywhere else in this run. Anyone acting on these should treat the citation as the starting point and do the analysis in the appropriate place, which is not a documentation file.

| Item | Evidence | What it is | Severity |
|------|----------|------------|----------|
| A1 | `src/Billing/SLEOB.php:L41-L42` | A patient identifier derived from a payer-supplied claim identifier is interpolated into an SQL string while the value beside it on the same line is bound as a parameter | HIGH |
| A2 | `library/edihistory/edih_io.php:L739` | Payer-derived deposit values are interpolated into an HTML string without the escaping helper that the same function's other output paths use, at `:L743` and `:L746` | HIGH |
| A3 | `sql/database.sql:L10035` | The interchange element column `x12_isa04` is declared with a column comment naming it a user password, stored in plain text | MEDIUM |
| A4 | `sql/database.sql:L10047` | The transport password column `x12_sftp_pass`, decrypted for use at `src/Billing/BillingProcessor/X12RemoteTracker.php:L90` | MEDIUM |
| A5 | `sql/database.sql:L10057` | The client secret column `x12_client_secret`, among the five inert configuration columns registered as [DC-57](#dc-57-five-trading-partner-columns-have-no-consumer) | MEDIUM |

Two of the five are also registered as functional defects, because they have observable symptoms independent of their security aspect: A1 is [DC-40](#dc-40-the-claim-identifier-is-recovered-by-interpolating-a-payer-supplied-value-into-a-query) and A5 is part of [DC-57](#dc-57-five-trading-partner-columns-have-no-consumer). A2 shares a line with [DC-4](#dc-4-a-deposit-counts-as-fully-allocated-only-for-two-exact-strings). The remaining items appear only here.

## Appendix B Historical Precedent

Three of the defect classes in this register have already been fixed once in the same files, which turns a suspicion into a documented recurrence pattern. All three commits were read at the head commit recorded in the purpose section.

| Commit | Date | Subject | Files touched | Which class it precedes |
|--------|------|---------|---------------|-------------------------|
| `0d85baa83` | 2025-01-15 | fix: edi segment count for ordering provider (#7922) | `src/Billing/X125010837P.php`, two insertions | Segment counting in the professional generator, which is the class of [DC-58](#dc-58-the-attachment-segment-names-a-transport-that-was-never-built) |
| `1de5ae614` | 2023-05-28 | fix: 837 professional HL count (#6472) | five files including `src/Billing/X125010837P.php` | Counter state shared across claims in one batch, which is the class of [DC-37](#dc-37-the-transaction-set-header-and-its-trailer-are-governed-by-different-conditions) |
| `e392a30ba` | 2026-05-20 | fix(billing): cast 835 monetary fields to float for type-strict comparisons (#11868) | `src/Billing/ParseERA.php`, `interface/billing/sl_eob_process.php`, and it added `tests/Tests/Isolated/Billing/ParseERATest.php` | Strict comparison against a monetary value whose type the driver decides, which is the class of [DC-4](#dc-4-a-deposit-counts-as-fully-allocated-only-for-two-exact-strings) |

Two observations follow from the table rather than from any one row. The professional generator has had its segment counting corrected twice, in two different years, which is the evidence behind its risk classification in [upgrade-risk-map.md](upgrade-risk-map.md#high-risk-justifications): the file is not high risk because it is large but because this class of defect has recurred in it. And the most recent of the three arrived with the test file that several entries in this register propose extending, which means the verification route those entries name already exists and runs.

### The 2016 claim, and why its patch was never applied

The only defect recorded anywhere in the subsystem's existing documentation before this register is a single claim in `Documentation/Readme_edihistory.html:L95`, dated 2016. Paraphrased rather than quoted, because that line contains a typographical error in the product's own name: it observes that the reference value the system writes on the transaction segment is a fixed placeholder rather than something that identifies the submission, and that this makes tracing a claim back to its transmission unreliable. The document proposes a patch at `:L220-L235`.

The claim is still true. VERIFIED: the literal is written at `src/Billing/X125010837P.php:L111` and at `src/Billing/X125010837I.php:L86` at the head commit, a decade after the observation, and the batch's substitution is a second constant, all of which is registered as [DC-48](#dc-48-the-transaction-reference-is-a-fixed-literal-at-generation-and-after-the-batch-rewrite). This is an agreement between the existing documentation and the present code, and it is recorded here as one.

The patch, however, is now stale in two independent ways, both VERIFIED:

- It targets a function named `append_claim` in `interface/billing/billing_process.php`, near line 82 of that file. That file is today **65 lines long and contains no occurrence of `append_claim` at all**; it is a thin controller that constructs the processor and calls it, at `:L32-L33`. The function moved: it now lives at `src/Billing/BillingProcessor/BillingClaimBatch.php:L202`.
- A rewrite of the reference **was** added at the function's new home, at `src/Billing/BillingProcessor/BillingClaimBatch.php:L254`, and it substitutes the constant `1`. So the code has changed in the place the patch pointed at, in a way that does not achieve what the patch was for.

The lesson for this register is procedural rather than technical, and it is why every entry above names a file and a line rather than a function: a proposed fix expressed as a function name and an approximate line number in a file did not survive the code moving, and the defect outlived the document that recorded it.

## Related Documents

| Document | What it gives you that this one does not |
|----------|------------------------------------------|
| [README.md](README.md) | The citation format, the VERIFIED and INFERRED notation with its confidence vocabulary, and the source-of-truth ordering that every entry here applies without restating |
| [architecture.md](architecture.md) | Why components of different ages coexist and disagree, which is the setting for [DC-10](#dc-10-provider-level-adjustments-are-excluded-from-the-ledger-and-included-in-the-balance-test) and [DC-28](#dc-28-a-medicare-inpatient-adjudication-segment-ends-the-parse-where-it-appears) |
| [claim-lifecycle.md](claim-lifecycle.md) | The stage each symptom surfaces in, with the tables and files that stage touches and its own account of the failure modes |
| [transactions.md](transactions.md) | Per-transaction detail behind the envelope and segment entries, including the control-number sources and the trading-partner configuration reference |
| [business-rules.md](business-rules.md) | The rule each suspect implementation is the implementation of; a rule and the suspicion about it are meant to be read together |
| [upgrade-risk-map.md](upgrade-risk-map.md) | Where a defect cited here escalates a file's risk classification, and the coverage and coupling behind that classification |
| [extraction-roadmap.md](extraction-roadmap.md) | What to do about these files, in order, and the golden-file corpus that would make the verifications proposed here routine |
| defect-candidates.md | This document |

---

## Documentation Attribution

### Authorship

Compiled by static reading of the OpenEMR source tree, its schema and its git history at branch `master`, commit `b7a7e690e419de3451740f995b768a8e8e5fba87`, on project version 8.3.0-dev (`version.php:L17-L20`).

### Method

Sixty-one entries derived expression by expression from the in-scope revenue cycle and X12 code, each anchored to the lines it describes, each separating what the code does, which is VERIFIED and cited, from the claim that it is wrong, which is labelled INFERRED with a confidence and a basis. Evidence precedence is the ordering defined in [README.md](README.md): executable code first, schema DDL second, tests third, comments and prose last and only as evidence of intent. Several entries are comment-versus-code contradictions, which is that ordering demonstrating its own necessity. **No part of this subsystem was executed and no test suite was run**; PHP and Composer are not installed in the authoring environment. Every `Verification:` field therefore proposes a test or a reproduction and no reproduction was performed.

### Contributing

- Keep the six fields. An entry missing an observable symptom is a code-quality observation and belongs in [upgrade-risk-map.md](upgrade-risk-map.md); an entry whose verification does not name a test file with its configuration, or a screen with an input and an expected outcome, is not yet an entry.
- Cite a line range, never a function name. The 2016 patch described in [Appendix B Historical Precedent](#appendix-b-historical-precedent) is the reason.
- Nothing in this register is fixed by editing this register. When an entry is fixed in code, remove it here and record the commit that fixed it in [Appendix B Historical Precedent](#appendix-b-historical-precedent), because the recurrence pattern is what makes the next suspicion credible.
- Re-read a citation before relying on it. Line anchors are relative to the commit named above, and a citation that no longer supports its claim is a defect in this document.

**Last Updated:** August 2026

**License:** GPL v3

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

**Transaction set numbers.** **837** is a claim, in two flavours: **837P** professional and **837I** institutional. **835** is remittance advice, a payer's statement of what it paid and why, also called an **ERA** for electronic remittance advice; its human-readable equivalent is an **EOB**, an explanation of benefits. **270** and **271** are an eligibility request and its response. **276** and **277** are a claim-status inquiry and its response. **278** is a services review, that is, an authorisation. **997** and **999** are acknowledgements reporting whether a transmission was syntactically accepted.

**Segment identifiers.** **CLP** carries claim-level payment information; **CLP01** is the provider's own claim identifier and the later elements carry the charged, paid and patient-responsibility amounts. **SVC** carries service-line payment information. **CAS** carries a claim or service adjustment as a group code, a reason code and an amount; the group codes named below are `PR` for patient responsibility, `CO` for contractual obligation and `CR` for correction and reversal. **PLB** carries a provider-level adjustment, which belongs to the provider rather than to any one claim. **MIA** carries Medicare inpatient adjudication information. **AMT** is an amount, **QTY** a quantity, **NM1** a name, **DTM** a date, **REF** a reference identifier, **TRN** a trace number, **LX** a service-line counter and **PWK** a paperwork or attachment reference.

**Other terms.** **A/R** is accounts receivable: what has been billed and not yet settled. In this schema a deposit is a row of `ar_session` and a ledger line is a row of `ar_activity`. **SFTP** is the SSH File Transfer Protocol, the one built-in transport this subsystem uses to hand a batch file to a trading partner.

## Severity Summary

Sixty-one entries. Every one appears once in this table and once as an entry below.

| Severity | Count | What it means here |
|----------|------:|--------------------|
| CRITICAL | 7 | Money, claim routing or claim data reaches a wrong destination, or a wrong amount is recorded, and nothing on any screen says so |
| HIGH | 28 | A wrong or absent outcome that an operator could eventually notice, or a wrong amount confined to one claim |
| MEDIUM | 26 | A wrong outcome that is visible, bounded, or reachable only on an unusual input |

The seven CRITICAL entries are [DC-10](#dc-10-provider-level-adjustments-are-excluded-from-the-ledger-and-included-in-the-balance-test), [DC-19](#dc-19-patient-and-provider-fields-survive-from-one-claim-to-the-next), [DC-20](#dc-20-one-batch-file-is-queued-once-for-every-partner-in-the-batch), [DC-27](#dc-27-a-failed-upload-is-recorded-and-then-overwritten-with-success), [DC-28](#dc-28-a-medicare-inpatient-adjudication-segment-ends-the-parse-where-it-appears), [DC-40](#dc-40-the-claim-identifier-is-recovered-by-interpolating-a-payer-supplied-value-into-a-query) and [DC-49](#dc-49-a-malformed-interchange-header-ends-the-request-after-the-claim-was-marked-billed). They share one property that is worth naming before the register begins: in all seven the wrong outcome is committed to the database or handed to a payer, and in six of the seven nothing on any screen reports it.

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
| [DC-8](#dc-8-envelope-validity-is-decided-by-the-truthiness-of-a-position) | Envelope validity is decided by the truthiness of a position | 1 Wrong comparisons | MEDIUM |
| [DC-9](#dc-9-a-negative-contractual-obligation-is-inverted-after-a-strict-string-test) | A negative contractual obligation is inverted after a strict string test | 1 Wrong comparisons | MEDIUM |
| [DC-10](#dc-10-provider-level-adjustments-are-excluded-from-the-ledger-and-included-in-the-balance-test) | Provider level adjustments are excluded from the ledger and included in the balance test | 2 Precision and truncation | CRITICAL |
| [DC-11](#dc-11-forced-balancing-overwrites-the-service-payment-the-payer-reported) | Forced balancing overwrites the service payment the payer reported | 2 Precision and truncation | HIGH |
| [DC-12](#dc-12-the-balancing-residue-is-attributed-to-a-group-code-the-payer-never-sent) | The balancing residue is attributed to a group code the payer never sent | 2 Precision and truncation | MEDIUM |
| [DC-13](#dc-13-the-submitted-claim-is-stored-in-a-column-with-a-hard-byte-ceiling) | The submitted claim is stored in a column with a hard byte ceiling | 2 Precision and truncation | HIGH |
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
| [DC-31](#dc-31-the-dry-run-branch-of-the-deposit-writer-returns-nothing) | The dry run branch of the deposit writer returns nothing | 4 Dead branches | HIGH |
| [DC-32](#dc-32-the-institutional-transaction-type-can-never-be-reporting) | The institutional transaction type can never be reporting | 4 Dead branches | MEDIUM |
| [DC-33](#dc-33-the-institutional-claim-can-never-carry-the-alternate-payer-identifier) | The institutional claim can never carry the alternate payer identifier | 4 Dead branches | HIGH |
| [DC-34](#dc-34-a-claim-level-name-segment-matches-every-qualifier-and-does-nothing) | A claim level name segment matches every qualifier and does nothing | 4 Dead branches | MEDIUM |
| [DC-35](#dc-35-the-remittance-charge-helper-ignores-three-of-its-ten-parameters) | The remittance charge helper ignores three of its ten parameters | 4 Dead branches | HIGH |
| [DC-36](#dc-36-the-dry-run-path-of-the-re-billing-helper-does-nothing-and-says-nothing) | The dry run path of the re billing helper does nothing and says nothing | 4 Dead branches | MEDIUM |
| [DC-37](#dc-37-the-transaction-set-header-and-its-trailer-are-governed-by-different-conditions) | The transaction set header and its trailer are governed by different conditions | 4 Dead branches | HIGH |
| [DC-38](#dc-38-an-unrecognised-action-button-dereferences-a-null-task) | An unrecognised action button dereferences a null task | 4 Dead branches | HIGH |
| [DC-39](#dc-39-the-tertiary-payer-is-never-queued-from-the-posting-screen) | The tertiary payer is never queued from the posting screen | 4 Dead branches | HIGH |
| [DC-40](#dc-40-the-claim-identifier-is-recovered-by-interpolating-a-payer-supplied-value-into-a-query) | The claim identifier is recovered by interpolating a payer supplied value into a query | 5 Unvalidated assumptions | CRITICAL |
| [DC-41](#dc-41-the-interchange-header-rebuild-indexes-five-elements-it-never-proves-exist) | The interchange header rebuild indexes five elements it never proves exist | 5 Unvalidated assumptions | HIGH |
| [DC-42](#dc-42-the-reference-rewrite-uses-an-unchecked-search-result-as-an-offset) | The reference rewrite uses an unchecked search result as an offset | 5 Unvalidated assumptions | HIGH |
| [DC-43](#dc-43-a-claim-level-adjustment-that-arrives-after-a-service-line-lands-on-that-line) | A claim level adjustment that arrives after a service line lands on that line | 5 Unvalidated assumptions | HIGH |
| [DC-44](#dc-44-the-branch-chain-tests-elements-it-never-proves-the-segment-carries) | The branch chain tests elements it never proves the segment carries | 5 Unvalidated assumptions | MEDIUM |
| [DC-45](#dc-45-the-segment-scan-stops-silently-when-no-terminator-is-found-in-the-buffer) | The segment scan stops silently when no terminator is found in the buffer | 5 Unvalidated assumptions | HIGH |
| [DC-46](#dc-46-the-institutional-envelope-trailers-contradict-their-own-headers) | The institutional envelope trailers contradict their own headers | 5 Unvalidated assumptions | HIGH |
| [DC-47](#dc-47-the-interchange-trailer-is-written-even-when-no-functional-group-was-opened) | The interchange trailer is written even when no functional group was opened | 5 Unvalidated assumptions | MEDIUM |
| [DC-48](#dc-48-the-transaction-reference-is-a-fixed-literal-at-generation-and-after-the-batch-rewrite) | The transaction reference is a fixed literal at generation and after the batch rewrite | 5 Unvalidated assumptions | HIGH |
| [DC-49](#dc-49-a-malformed-interchange-header-ends-the-request-after-the-claim-was-marked-billed) | A malformed interchange header ends the request after the claim was marked billed | 6 Resource and lifecycle | CRITICAL |
| [DC-50](#dc-50-neither-remittance-entry-point-closes-the-file-it-opened) | Neither remittance entry point closes the file it opened | 6 Resource and lifecycle | MEDIUM |
| [DC-51](#dc-51-the-claim-version-is-drawn-by-an-unlocked-aggregate-inside-a-transaction) | The claim version is drawn by an unlocked aggregate inside a transaction | 6 Resource and lifecycle | HIGH |
| [DC-52](#dc-52-the-ledger-sequence-number-is-drawn-the-same-way-and-the-code-says-so) | The ledger sequence number is drawn the same way and the code says so | 6 Resource and lifecycle | HIGH |
| [DC-53](#dc-53-the-transport-reads-the-claim-file-after-a-fallback-without-rechecking-it) | The transport reads the claim file after a fallback without rechecking it | 6 Resource and lifecycle | MEDIUM |
| [DC-54](#dc-54-the-remote-session-is-left-open-on-two-failure-paths) | The remote session is left open on two failure paths | 6 Resource and lifecycle | MEDIUM |
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

- **Suspected defect:** The guard that decides whether an encounter advances to the next payer level mixes `&&` and `||` with no parentheses, so it reads as though a non-empty watermark were required when the third arm reaches the increment without one, and it silently declines to advance anything already billed to the tertiary level.
- **Evidence:** `src/Billing/SLEOB.php:L285-L287`. VERIFIED: `$new_payer_type = 0 + $ferow['last_level_billed'];` is followed by `if ($new_payer_type < 3 && !empty($ferow['last_level_billed']) || $new_payer_type == 0) {` and then `++$new_payer_type;`. PHP binds `&&` tighter than `||`, so the expression evaluates as `($new_payer_type < 3 && !empty(...)) || $new_payer_type == 0`.
- **Why it looks wrong:** Two readings of the same line disagree. The `!empty()` arm suggests the author wanted an encounter with no billing watermark to be excluded, yet the trailing `== 0` arm admits exactly that case, which makes the `!empty()` arm unreachable for the only input it was written to reject. Separately, `0 + $ferow['last_level_billed']` coerces a non-numeric watermark to zero rather than rejecting it, so a corrupt value takes the same path as a never-billed encounter. INFERRED (confidence: Medium). Basis: the two arms are individually coherent and jointly redundant, which is the signature of a condition edited twice rather than designed once; the intent behind the `!empty()` arm cannot be recovered from the code.
- **Observable symptom:** After a tertiary payer's remittance is posted, the encounter is left with `last_level_billed = 3`, no further payer is queued, and no message is printed. On the billing screen the claim simply stops appearing in the work list. The related caller gate is [DC-39](#dc-39-the-tertiary-payer-is-never-queued-from-the-posting-screen), which prevents the tertiary case from reaching this line at all.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arSetupSecondary` against a `form_encounter` row with `last_level_billed = 3` and then one with `last_level_billed = ''`, asserting the resulting `billing.payer_id` and `form_encounter.last_level_billed`; it runs under `phpunit-isolated.xml`. A reproduction is stage [S12](claim-lifecycle.md#stage-s12-secondary-and-tertiary-payer-setup): post a tertiary remittance and inspect `form_encounter`; expected is either an advance or an explicit message, actual is neither.
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

### DC-8 Envelope validity is decided by the truthiness of a position

- **Suspected defect:** An uploaded interchange is judged to contain a functional group header and a transaction set header by testing the truthiness of `strpos()`, so a file whose header appears at offset zero of the search fails the test that was meant to confirm its presence.
- **Evidence:** `src/Billing/EdiHistory/X12File.php:L324-L339`. VERIFIED: `$hasval = 'ov';` at `:L324` accretes to `'ovi'` at `:L327` when the text starts with `ISA`, to `'ovig'` at `:L331` when `strpos($ftxt, $dt . 'GS' . $de, 0)` is truthy, and to `'ovigs'` at `:L335` when the same shape of test finds the transaction set header; the accreted string is returned at `:L339` and the caller reads its length and content as the verdict.
- **Why it looks wrong:** Two separate defects sit in one expression. The validity verdict is encoded in the length of a string rather than in a structured result, and each step of the accretion uses the truthiness of a byte offset where zero is a legitimate answer. INFERRED (confidence: Medium). Basis: for these two searches offset zero cannot occur, because the segment terminator that begins each needle cannot be the first byte of a file starting with `ISA`; the defect is that the expression is correct by circumstance rather than by construction, and any change to the needle reintroduces it. The encoding itself is registered as [BR-H8](business-rules.md#br-h8-envelope-validity-is-encoded-in-the-length-of-a-string).
- **Observable symptom:** An uploaded file that is in fact well formed is reported as missing a functional group or transaction set header, and the EDI history upload screen refuses it with a message about a missing segment rather than indexing it.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/EdiHistory/X12FileIsolatedTest.php` asserting the returned verdict string for a minimal interchange, and a second case asserting it for text where the searched needle begins at offset zero; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
if (strpos($ftxt, $dt . 'GS' . $de, 0)) {
    $hasval = 'ovig';
```

That is `src/Billing/EdiHistory/X12File.php:L330-L331`. The same shape is repeated for the transaction set header at `src/Billing/EdiHistory/X12File.php:L334-L335`.

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

### DC-13 The submitted claim is stored in a column with a hard byte ceiling

- **Suspected defect:** The complete generated claim is stored in a MySQL `text` column, which holds 65,535 bytes, and the connection runs in a mode that truncates an oversized value instead of refusing it.
- **Evidence:** `sql/database.sql:L391`. VERIFIED: `` `submitted_claim` text COMMENT 'This claims form claim data' ``. The value is bound at `src/Billing/BillingUtilities.php:L1653-L1654`, where the fragment `", submitted_claim = ?"` is appended to the claim update. VERIFIED: the connection issues `SET sql_mode = ''` at `src/BC/DatabaseConnectionFactory.php:L70` and again at `:L133`, so an over-length value is truncated and warned about rather than rejected. VERIFIED: the stored text is read back and returned for reuse at `interface/billing/ub04_dispose.php:L188-L190`.
- **Why it looks wrong:** The column stores a whole claim, whose length is a function of how many service lines and diagnoses an encounter carries, and nothing in the write path measures it against the ceiling. The permissive session mode converts what would be a loud failure into a silent one. INFERRED (confidence: Medium). Basis: no length check exists on the write path, and the read path at `interface/billing/ub04_dispose.php:L190` returns the column's contents as though they were complete.
- **Observable symptom:** A large encounter's stored claim is cut off part-way through a segment. The re-disposal path returns the truncated text, so a claim regenerated from storage is malformed and the clearinghouse rejects the interchange with a 997 or 999 acknowledgement while the claim history screen shows the version as submitted.
- **Verification:** Add a case to `tests/Tests/Services/Billing/` that writes a claim body longer than 65,535 bytes through the claim update path and reads `submitted_claim` back, asserting byte-for-byte equality; it runs under the primary configuration, which covers `tests/Tests/Services` at `phpunit.xml:L67-L69`. The expected result is equality or a refusal, the actual result is a shorter string and no error.
- **Severity:** HIGH

```sql
`submitted_claim` text COMMENT 'This claims form claim data',
```

That is `sql/database.sql:L391`. MySQL caps this type at 65,535 bytes.

### DC-14 The deposit balance check is a float subtraction reported only in a browser alert

- **Suspected defect:** After a remittance is posted, each deposit is checked by subtracting the sum of its live ledger lines from its total with `<>` on values the driver returns for `decimal(12,2)` columns, and the only report of a mismatch is a browser alert listing keys.
- **Evidence:** `interface/billing/sl_eob_process.php:L866`. VERIFIED: `if (($pay_total - $pay_amount) <> 0) {` compares `pay_total`, read at `:L858`, against `sum(pay_amount)` restricted to rows where `deleted IS NULL`, read at `:L860-L863`. VERIFIED: a mismatch only appends the array key to a string at `:L867` and the string is emitted as a JavaScript alert at `:L873-L875`.
- **Why it looks wrong:** Three separate weaknesses sit on one path. The comparison is a subtraction of two driver-returned values tested against integer zero, so it is exposed to representation differences the way [DC-4](#dc-4-a-deposit-counts-as-fully-allocated-only-for-two-exact-strings) is. The `deleted IS NULL` restriction means a deposit whose ledger lines were later reversed reports as unbalanced forever. And the finding is delivered as a modal alert, so it survives nothing: not a page reload, not a log, not a report. INFERRED (confidence: High). Basis: the alert is the only consumer of the result, and no row, column or log entry records it.
- **Observable symptom:** An operator who dismisses the alert, or whose browser suppresses it, has no way to learn that a deposit did not balance. Nothing on any later screen repeats the finding and no report lists unbalanced deposits from this path.
- **Verification:** Reproduce on the remittance posting screen at `interface/billing/sl_eob_process.php`: post a remittance, then soft-delete one ledger line from the resulting deposit and post a second remittance against the same deposit. Expected is a durable record that the deposit does not balance, actual is a single alert naming the deposit key. The balance rule itself is registered as [BR-B7](business-rules.md#br-b7-a-deposit-is-balanced-against-live-ledger-lines-only-and-only-in-a-browser-alert).
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
- **Evidence:** `interface/billing/sl_eob_process.php:L274`. VERIFIED: `$check_date = $out['check_date' . $check_count] ?: $_REQUEST['paydate'];` computes the value, and the call at `:L277-L285` passes `check_date: $out['check_date' . $check_count]` at `:L281`, reading the raw parsed value again rather than the computed one. The local variable is not read anywhere else in the function.
- **Why it looks wrong:** The statement exists only to produce a value for the call three lines below it, and the call ignores it. The two sibling values computed on the adjacent lines, the posting date and the deposit date at `:L275-L276`, are both passed. INFERRED (confidence: High). Basis: three fallbacks are computed together and two of the three are used, which makes the third an omission rather than a design.
- **Observable symptom:** A remittance whose check date element is empty produces a deposit row with an empty check date rather than the date the operator typed. The payment search screen then lists the deposit under a zero date and it cannot be found by date range.
- **Verification:** Reproduce on the remittance posting screen by posting a remittance whose check date element is empty while supplying a pay date on the form: expected is a deposit carrying the typed pay date, actual is a deposit with no check date. The date precedence intended here is registered as [BR-E7](business-rules.md#br-e7-the-operators-pay-date-overrides-the-payers-own-dates).
- **Severity:** MEDIUM

```php
$check_date = $out['check_date' . $check_count] ?: $_REQUEST['paydate'];
```

That is `interface/billing/sl_eob_process.php:L274`. The call three lines below reads the raw parsed value again, at `interface/billing/sl_eob_process.php:L281`.

### DC-26 The delimiter probe can only run on the first segment of the interchange

- **Suspected defect:** The element and segment delimiters are learned only while no segment has yet been identified, so any interchange whose first recognised segment is not the header keeps the hardcoded defaults for the whole file.
- **Evidence:** `src/Billing/ParseERA.php:L118-L121`. VERIFIED: the probe is gated on `if ($segid === '' && str_starts_with($inline, 'ISA'))` and assigns `$delimiter2` and `$delimiter3` from the header's own bytes. VERIFIED: the three delimiters are otherwise fixed at `:L87-L89` to the tilde, the asterisk and the caret.
- **Why it looks wrong:** The gate uses the emptiness of a loop variable as a proxy for being at the start of the file, which couples delimiter discovery to iteration state rather than to file position. Any condition that leaves `$segid` non-empty before the header is reached, including a partial read that begins mid-file, permanently disables the probe. INFERRED (confidence: Medium). Basis: the probe's own inner test already checks for the header segment, so the outer emptiness test adds no correctness and removes robustness.
- **Observable symptom:** A remittance that uses non-default delimiters, which the standard permits and which some payers use, is read as one enormous segment. The parse ends with the unexpected-segment message and the operator is told the file contains an unknown segment identifier rather than that its delimiters were not recognised. The upstream cause is usually [DC-22](#dc-22-the-line-ending-strip-is-applied-to-the-wrong-side-of-the-split).
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a remittance whose element separator is a character other than the asterisk, and assert the parse result; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
    $delimiter2 = substr($inline, 3, 1);
    $delimiter3 = substr($inline, -1);
}
```

That is `src/Billing/ParseERA.php:L119-L121`.

## Category 4 Dead Branches Masking Logic

Thirteen entries. A branch is dead here in one of three ways: it is written and then unconditionally undone, it can never be entered because its condition is unreachable, or it is entered and does nothing. All three hide the code that follows them.

### DC-27 A failed upload is recorded and then overwritten with success

- **Suspected defect:** The transport writes an upload-error status when the file transfer fails, and then, with no else and no early return, writes a success status over it on the very next statement.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L112-L121`. VERIFIED: `if (false === $sftp->put($x12_remote['x12_filename'], $claim_file_contents)) {` at `:L112` sets the upload-error status at `:L113`, appends the failure message at `:L114` and the transport's own errors at `:L115`, persists the row at `:L116`, and the block closes at `:L117`. VERIFIED: `:L120` then sets `$x12_remote['status'] = self::STATUS_SUCCESS;` unconditionally and `:L121` persists it. There is no `else`, no `continue` and no `return` between the two writes. VERIFIED: the four earlier failure branches in the same loop, at `:L70`, `:L85`, `:L96` and `:L104`, do continue past the row rather than falling through.
- **Why it looks wrong:** Four failure paths in the same method exit the iteration and the fifth does not, so the pattern the author established is broken in exactly one place. The error status and its messages are written to the database and then replaced within two statements, which no reading of the code makes deliberate. INFERRED (confidence: High). Basis: the four sibling branches establish the intended shape, and the messages written at `:L114-L115` are recorded specifically so that a human can read them, which is pointless if the row is immediately marked successful.
- **Observable symptom:** An undelivered transmission is displayed as delivered. The transport queue shows the row as successful, the practice believes the claims were filed, and the absence is discovered only weeks later when no remittance arrives and the timely-filing window has narrowed. Nothing on any screen reports the failure, and the failure messages are overwritten in the same request that wrote them.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` that injects a transport double whose `put()` returns `false` and asserts the persisted status; it runs under `phpunit-isolated.xml`. The expected result is the upload-error status, the actual result is the success status. Stage [S6](claim-lifecycle.md#stage-s6-transport-to-the-clearinghouse) is where the row is written.
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
- **Why it looks wrong:** The older component understands a segment the newer one rejects, which inverts the usual assumption that a replacement is at least as capable as what it replaces. The parser's own handling of the adjacent Medicare outpatient segment is a deliberate no-op at `:L318-L319`, which shows the author knew such segments existed and chose to skip rather than reject them; the inpatient segment received neither treatment. INFERRED (confidence: High). Basis: the outpatient segment has an explicit ignore branch and the inpatient segment has none, and both are optional Medicare segments in the same loop.
- **Observable symptom:** A Medicare Part A remittance renders correctly in the EDI history viewer and never posts. The posting screen stops at the message about an unknown segment identifier, no ledger line is written for any claim in the file, and an operator comparing the two screens sees the same file succeed in one and fail in the other with no explanation of why.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` supplying a claim followed by an MIA segment and assert that the parse completes and the claim posts; it runs under `phpunit-isolated.xml`. The expected result is a parsed claim, the actual result is the unknown-segment string.
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
- **Verification:** No test can assert a comment. Reproduce by reading `src/Billing/BillingProcessor/X12RemoteTracker.php:L107-L121` and comparing the two comments against the two statements they precede; the expected relationship is one accurate comment per statement, the actual is one accurate and one inherited. This is the thirteenth entry in the comment-versus-code census in [README.md](README.md), which lists the other twelve.
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

- **Suspected defect:** The deposit writer echoes its query and falls off the end of the function in dry-run mode, so it returns null where every caller expects a deposit identifier.
- **Evidence:** `src/Billing/SLEOB.php:L98-L104`. VERIFIED: `if ($debug) {` at `:L98` echoes the query text at `:L99`; the else at `:L100` performs the insert at `:L102` and returns the identifier at `:L103`; the function ends at `:L104` with no return on the debug path.
- **Why it looks wrong:** The two branches of one function have different arities: one returns a value and one returns nothing, and the value is a primary key the caller stores in an array it later iterates. INFERRED (confidence: High). Basis: the caller at `interface/billing/sl_eob_process.php:L277` assigns the result into `$InsertionId` keyed by check number, and the balance loop at `:L856-L871` iterates that array and queries by its values.
- **Observable symptom:** In dry-run mode the post-run balance check queries `ar_session` with a null identifier for every deposit, finds nothing, and the totals it compares are both null, so the check passes silently on a run that wrote nothing. An operator using dry run to preview a remittance is shown a clean balance report that means nothing.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arPostSession` with the debug flag set and asserting the return value is an identifier or an explicit sentinel rather than null; it runs under `phpunit-isolated.xml`.
- **Severity:** HIGH

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

- **Suspected defect:** The helper that adds a charge discovered in a remittance declares ten parameters and reads seven. The deposit identifier, the service date and the debug flag are accepted and never used, so a charge is written on a dry run and dated by the database rather than by the remittance.
- **Evidence:** `src/Billing/SLEOB.php:L165`. VERIFIED: the signature declares `$patient_id, $encounter_id, $session_id, $amount, $units, $thisdate, $code, $description, $debug, $codetype`. VERIFIED: the body at `:L166-L208` reads none of `$session_id`, `$thisdate` or `$debug`, and the call it makes at `:L194-L207` passes twelve arguments to `BillingUtilities::addBilling`, whose signature at `src/Billing/BillingUtilities.php:L1434-L1452` accepts no date parameter at all. VERIFIED: the caller supplies all three, at `interface/billing/sl_eob_process.php:L507-L519`, passing the service date at `:L514` and the debug flag at `:L517`.
- **Why it looks wrong:** A dry-run flag that is accepted and discarded is worse than no flag, because the caller has been told the run is safe. The sibling methods in the same class do honour the flag, at `:L98-L104` and `:L292-L302`, so the omission is local rather than a convention. INFERRED (confidence: High). Basis: the caller passes the flag by name at `interface/billing/sl_eob_process.php:L517`, which is only meaningful if it is read.
- **Observable symptom:** An operator previewing a remittance with the dry-run option still creates real `billing` rows for every unmatched code the remittance carries, and those charges carry the current date rather than the date of service. The preview screen looks like a preview and the charge table has changed.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` calling `SLEOB::arPostCharge` with the debug flag set and asserting that no `billing` row is created; it runs under `phpunit-isolated.xml`. The expected result is no row, the actual result is a row dated today. The discarded flag is registered as [BR-F8](business-rules.md#br-f8-the-remittance-charge-helper-declares-a-dry-run-flag-and-never-reads-it) and the switch that reaches this path as [BR-I3](business-rules.md#br-i3-one-switch-turns-a-payer-reported-unknown-code-into-a-charge).
- **Severity:** HIGH

```php
public static function arPostCharge($patient_id, $encounter_id, $session_id, $amount, $units, $thisdate, $code, $description, $debug, $codetype = '')
```

That is `src/Billing/SLEOB.php:L165`. The three parameters never read are the third, the sixth and the ninth.

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

- **Suspected defect:** In the professional generator the transaction set header is emitted under a condition about the claim's position in the batch, and its trailer is emitted under a condition about a flag the caller passes, so the two can disagree and the transaction set is left open or closed twice.
- **Evidence:** `src/Billing/X125010837P.php:L91-L99`. VERIFIED: the ST segment, the transaction set header, is emitted when the hierarchical level count is one and the per-payer switch is on, or when that switch is off. VERIFIED: the SE segment, the transaction set trailer, is emitted at `:L1612-L1615` when the last-claim flag is true or that same switch is off. The two conditions share only their second arm. VERIFIED: the flag reaches the generator from the loop in `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L132-L139`, which is the loop [DC-3](#dc-3-the-last-claim-for-a-partner-is-chosen-by-an-identity-comparison-across-types) covers.
- **Why it looks wrong:** A transaction set is a matched pair by definition, and the two halves are decided by two different inputs, one derived inside the generator and one supplied by the caller. Any disagreement produces an interchange that is structurally invalid rather than merely incorrect. INFERRED (confidence: High). Basis: the segment count written into the trailer at `:L1616-L1621` is only correct if exactly one header was written for it, and nothing checks that.
- **Observable symptom:** The clearinghouse rejects the whole interchange with a 997 or 999 acknowledgement naming an unmatched ST or SE segment, so every claim in the batch is refused. The billing screen reports the batch as generated and the rejection arrives later, out of band, as an acknowledgement file the operator must open in the EDI history viewer.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` generating two claims for one partner with the per-payer switch on and the last-claim flag never set, and assert that the batch contains equal counts of ST and SE segments; it runs under `phpunit-isolated.xml`. The segment count convention is registered as [BR-H7](business-rules.md#br-h7-the-segment-count-is-copied-from-the-generator-into-the-batch-unchanged).
- **Severity:** HIGH

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
- **Evidence:** `interface/billing/sl_eob_process.php:L717`. VERIFIED: `if ($primary && SLEOB::arGetPayerID($pid, $service_date, 2)) {` guards the call at `:L718`, and the guard's second term asks specifically for payer type two. VERIFIED: the helper it calls does contain tertiary handling, at `src/Billing/SLEOB.php:L285-L287`, which can only be reached with a watermark of two, and the guard prevents the call whenever the remittance is not from the primary payer.
- **Why it looks wrong:** The helper implements three levels and the only caller can express one transition. The message printed beside the call, at `:L720-L726`, names secondary paper billing explicitly, so the screen's own text acknowledges that only one transition is contemplated. INFERRED (confidence: High). Basis: the coverage lookup inside the helper at `src/Billing/SLEOB.php:L290` recomputes the payer for whatever level the watermark implies, which is code that cannot run for the tertiary level under this guard.
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

### DC-41 The interchange header rebuild indexes five elements it never proves exist

- **Suspected defect:** The batch rebuilds the interchange header by taking the first seventy characters of the incoming header and then reading elements eleven, twelve, fourteen and fifteen by index, with no test that the header carries them.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L216-L217`. VERIFIED: the rebuild is `substr((string) $seg, 0, 70)` concatenated with the batch date and time, then `$elems[11]`, `$elems[12]`, the batch interchange control number, `$elems[14]`, `$elems[15]` and the literal `"*:~"`, where `$elems` is the result of `explode('*', $seg)` at `:L213`. VERIFIED: the rebuilt string's length is then required to be exactly 105 characters at `:L219-L221`, and a shorter one ends the request through the `die` on `:L221`.
- **Why it looks wrong:** The length check immediately below proves the author knew the result could be the wrong size, and the response to that is to end the request rather than to validate the input that produced it. Four unguarded index reads on a value that came from a generator whose output the batch does not control is an arity assumption with a hard failure attached. INFERRED (confidence: High). Basis: the check at `:L220` exists precisely because the rebuild can go wrong, and it fires after the reads rather than before.
- **Observable symptom:** Undefined-key warnings for each missing element followed by the message about the header needing to be 105 characters, printed into a half-rendered billing page, with the batch file left holding whatever was written before the failure. This is the same terminal path as [DC-49](#dc-49-a-malformed-interchange-header-ends-the-request-after-the-claim-was-marked-billed).
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` calling `append_claim` with a header segment truncated after element ten, and assert an exception or a recorded error rather than a request-ending failure; it runs under `phpunit-isolated.xml`.
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
- **Observable symptom:** A claim-level adjustment sent after the service lines is posted against the last service line, so the invoice shows a write-off on a procedure the payer did not adjust while the claim-level balance stays open. In the other ordering the operator is shown the unknown-segment message and no claim in the file posts at all.
- **Verification:** Add two cases to `tests/Tests/Isolated/Billing/ParseERATest.php`, one with a claim-level adjustment after a service line and one with an adjustment after a header-number segment, asserting where each amount lands and that the parse completes; both run under `phpunit-isolated.xml`.
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

### DC-45 The segment scan stops silently when no terminator is found in the buffer

- **Suspected defect:** The read loop breaks out of the scan when the buffer holds no segment terminator, and the break is indistinguishable from a normal end of file, so a truncated or over-long segment ends the parse with no error.
- **Evidence:** `src/Billing/ParseERA.php:L107-L110`. VERIFIED: `$tpos = strpos($buffer, $delimiter1); if ($tpos === false) { break; }` sits above the split, and the buffer is topped up to 2048 characters at `:L103-L105`. VERIFIED: the function's post-loop check at `:L474-L476` returns a premature-end message only when the transaction set counters disagree, and returns the empty string, meaning success, at `:L478`.
- **Why it looks wrong:** The break has two causes, one benign and one not, and the code distinguishes them only through counters that a truncated file may leave consistent. A segment longer than the top-up window also triggers it, which makes the parse silently dependent on a buffer size. INFERRED (confidence: Medium). Basis: the premature-end check exists, so the author intended to catch truncation, but it is a counter comparison rather than a test of why the loop ended.
- **Observable symptom:** A remittance truncated in transit posts the claims it managed to read and reports success, so the deposit total the operator typed exceeds the sum of what posted and the difference has no explanation on screen. The post-run balance alert from [DC-14](#dc-14-the-deposit-balance-check-is-a-float-subtraction-reported-only-in-a-browser-alert) is the only hint, and it names a key rather than a cause.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` feeding a remittance cut off mid-segment after a complete claim, and assert that parsing reports an error; it runs under `phpunit-isolated.xml`. The expected result is an error string, the actual result is the empty string that means success.
- **Severity:** HIGH

```php
$tpos = strpos($buffer, $delimiter1);
if ($tpos === false) {
    break;
```

That is `src/Billing/ParseERA.php:L107-L109`. The post-loop check at `src/Billing/ParseERA.php:L474-L476` compares counters rather than asking why the loop ended.

### DC-46 The institutional envelope trailers contradict their own headers

- **Suspected defect:** In the institutional generator the interchange and group control numbers in the headers are drawn from the sequence allocator, and the matching numbers in the trailers are hardcoded literals, so a single generated claim carries trailers that do not match its own headers.
- **Evidence:** `src/Billing/X125010837I.php:L59`. VERIFIED: the interchange control number element is `BillingClaimBatchControlNumber::getIsa13()` and the group control number at `:L70` is `getGs06()`. VERIFIED: the group trailer at `:L1213-L1214` is the literal `"*1" . "*1"` and the interchange trailer at `:L1216-L1217` is the literal `"*1" . "*000000001"`. VERIFIED: the batch post-processor discards both trailers, at `src/Billing/BillingProcessor/BillingClaimBatch.php:L264-L266`, and writes its own at `:L272-L279`, so the contradiction is repaired on the batched path only.
- **Why it looks wrong:** One file writes a control number twice, once from a generator and once as a literal, and the two are required by the standard to be equal. That the batch happens to discard the literals makes the defect invisible rather than absent, and every consumer that reads a single generated claim rather than a batch sees it. INFERRED (confidence: High). Basis: the professional generator has the same shape, with literals at `src/Billing/X125010837P.php:L1624-L1632` against a literal interchange number at `:L73`, so the institutional file is the one whose headers were modernised without its trailers.
- **Observable symptom:** The claim text stored in `claims.submitted_claim` and the text shown by the validation display carry mismatched control numbers. A claim re-sent from storage through the re-disposal path at `interface/billing/ub04_dispose.php:L188-L190` is rejected by the clearinghouse with a TA1 interchange acknowledgement or a 999, naming a control number mismatch, and the operator sees a rejection for a claim the screen shows as previously accepted.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` generating one institutional claim and asserting that the interchange trailer's control number equals the header's and likewise for the group; it runs under `phpunit-isolated.xml`. The two-draw allocation is registered as [BR-H4](business-rules.md#br-h4-interchange-and-group-control-numbers-are-two-separate-draws-on-one-sequence).
- **Severity:** HIGH

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
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L219-L221`. VERIFIED: `$isa_length = strlen($this->bat_content) - 1;` then `if ($isa_length != 105) {` then `die("Error:<br />\n ISA must be 105 characters in length; " . "found $isa_length instead")`. VERIFIED: a second `die` at `:L225-L226` fires when the accumulated content is empty and the segment is not a header. VERIFIED the ordering in `src/Billing/BillingProcessor/Tasks/GeneratorX12.php`: the claim is marked billed at `:L151-L162`, the batch append that can die is called at `:L165`, and the update that records the batch filename is at `:L168`. The validate-and-clear path has the same ordering, at `:L121-L138`.
- **Why it looks wrong:** A per-claim data problem is handled by terminating the process, which abandons every claim after it in the run and leaves the claims before it in an inconsistent state: marked billed, with no batch file recorded, in a batch file that was partially written. Every other error path in the same subsystem records a message and continues. INFERRED (confidence: High). Basis: the logger exists and is used for exactly this purpose elsewhere in the same task, at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L169`, so a non-terminal reporting route was available at the call site.
- **Observable symptom:** The billing manager page stops mid-render with the message about the header's length and nothing else, so the operator sees a truncated page. The claims processed before the failure are marked billed and cannot be selected again, their batch filename column is empty, and a partially written batch file is left in the outbound directory where the transport queue may still pick it up.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/BillingClaimBatchTest.php` calling `append_claim` with a header segment that rebuilds to the wrong length, and assert an exception rather than process termination; it runs under `phpunit-isolated.xml`. A reproduction is stage [S5](claim-lifecycle.md#stage-s5-envelope-post-processing): generate a batch for a partner whose interchange identifiers are shorter than the standard's fixed-width fields.
- **Severity:** CRITICAL
- **Registered as a rule:** [BR-F4](business-rules.md#br-f4-a-malformed-interchange-header-terminates-the-whole-batch-run).

```php
if ($isa_length != 105) {
    die("Error:<br />\n ISA must be 105 characters in length; " . "found $isa_length instead");
```

That is `src/Billing/BillingProcessor/BillingClaimBatch.php:L220-L221`. The claim was marked billed before this ran, at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L151-L162`.

### DC-50 Neither remittance entry point closes the file it opened

- **Suspected defect:** Both remittance entry points open the file with `fopen` and neither closes it. There is no `fclose` anywhere in the class.
- **Evidence:** `src/Billing/ParseERA.php:L91`. VERIFIED: the main parse opens the file and the check-scanning parse opens it again at `:L488`. VERIFIED: the file contains no `fclose` call at all, and both functions return from several points, including the early returns at `:L92-L94` and every error return in the segment chain.
- **Why it looks wrong:** A handle opened in a function that returns from a dozen places needs either a close on each path or a construct that closes it for you, and neither is present. PHP releases the handle when the variable goes out of scope, so this is a latent rather than an active leak; it becomes active the moment the handle is stored, passed on, or the function is called in a loop within one request. INFERRED (confidence: High). Basis: the posting screen calls both entry points in the same request, at `interface/billing/sl_eob_process.php:L850-L851`, so two handles to the same file are already open simultaneously today.
- **Observable symptom:** On a site posting many remittance files in one request, the process holds one handle per parse until the request ends, and the operator eventually sees a failure to open the remittance stream with the message from `:L92-L94` once the process reaches its descriptor limit. Nothing identifies the cause.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` that records the open descriptor count before and after a parse and asserts they are equal; it runs under `phpunit-isolated.xml`.
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

### DC-54 The remote session is left open on two failure paths

- **Suspected defect:** Two of the transport's failure branches leave the loop iteration without disconnecting the remote session, while the success path disconnects, so a failing queue holds one session open per row until the request ends.
- **Evidence:** `src/Billing/BillingProcessor/X12RemoteTracker.php:L96`. VERIFIED: the login-failure branch at `:L92-L98` and the directory-change failure branch at `:L100-L106` each record a status and `continue`, and the only `disconnect` call is at `:L123-L124`, after the upload. VERIFIED: the connection object is created inside the loop, at `:L89`, so each iteration creates a new one.
- **Why it looks wrong:** The directory-change failure occurs after a successful login, so it abandons an authenticated session rather than a failed connection attempt. The disconnect exists at the bottom of the loop, which shows the author intended sessions to be closed explicitly. INFERRED (confidence: High). Basis: the disconnect at `:L123` is unnecessary if garbage collection were considered sufficient, so the two paths that skip it are omissions.
- **Observable symptom:** On a site with several partners misconfigured, the transport run holds one authenticated session per failing partner for the duration of the run. Partners that limit concurrent sessions begin refusing the connection, so a later correctly configured partner records a login error caused by the earlier failures rather than by its own credentials.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/` that injects a transport double counting `disconnect()` calls, queues two rows whose directory change fails, and asserts one disconnect per row; it runs under `phpunit-isolated.xml`.
- **Severity:** MEDIUM

```php
// Disconnect from the remote server
$sftp->disconnect();
```

That is `src/Billing/BillingProcessor/X12RemoteTracker.php:L123-L124`. The two branches that skip it are `src/Billing/BillingProcessor/X12RemoteTracker.php:L92-L98` and `src/Billing/BillingProcessor/X12RemoteTracker.php:L100-L106`.

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
- **Why it looks wrong:** A per-claim database query is executed to obtain a value the code says it does not act on, and the value is nonetheless written into the charge table where later readers will assume it means something. The docblock records uncertainty rather than a decision, so the field's status is unresolved in the code itself. INFERRED (confidence: Medium). Basis: the docblock is evidence of intent only, per the source-of-truth ordering in [README.md](README.md); a repository-wide search finds no branch on the value, but a claim that no consumer exists anywhere cannot be proven from one file.
- **Observable symptom:** An operator selecting a different processing format for a partner sees no change in any generated file. The value is recorded against every claim, so a later report or integration that groups by it produces a grouping with no behavioural meaning.
- **Verification:** Reproduce by generating the same claim twice for one partner with two different processing formats selected and comparing the generated files byte for byte; expected is a difference, actual is identity. A test would go in `tests/Tests/Isolated/Billing/BillingClaimTest.php` and run under `phpunit-isolated.xml`. The rule is registered as [BR-D3](business-rules.md#br-d3-the-processing-format-is-read-per-claim-stored-and-branches-on-nothing).
- **Severity:** MEDIUM

```php
$sql = "SELECT x.processing_format from x12_partners as x where x.id =?";
$result = sqlQuery($sql, [$this->getPartner()]);
```

That is `src/Billing/BillingProcessor/BillingClaim.php:L137-L138`. Nothing branches on the value the query returns.

### DC-61 A component separator is assigned and never read in the check parse

- **Suspected defect:** The check-scanning parse declares three delimiters and reads two, so the composite-element separator it sets up is dead, and any composite element it encounters is left unsplit.
- **Evidence:** `src/Billing/ParseERA.php:L486`. VERIFIED: `$delimiter3 = '^';` is assigned among the three delimiters at `:L484-L486` and is not read anywhere in the function. VERIFIED: the main parse does read its equivalent, at `:L353-L359` and `:L363`, to split composite procedure codes, and learns it from the interchange header at `:L120`.
- **Why it looks wrong:** The two functions were written from one another and only one kept the code that uses the third delimiter. An unused local would be harmless; what makes it a defect candidate is that the same function also never learns the delimiters from the file, so a remittance that uses non-default separators is scanned with the wrong ones. INFERRED (confidence: Medium). Basis: the main parse's probe at `:L118-L121` has no counterpart here, and the dead assignment is the visible residue of the copy.
- **Observable symptom:** A remittance using non-default delimiters produces a first pass that finds no check numbers, so the posting screen offers no deposits to post and reports the file as containing nothing, while the file itself is valid. An operator then re-uploads the same file repeatedly.
- **Verification:** Add a case to `tests/Tests/Isolated/Billing/ParseERATest.php` calling the check-scanning parse on a remittance with a non-default element separator and asserting the recovered check count; it runs under `phpunit-isolated.xml`. The expected count is the number of checks in the file, the actual count is zero. The related restriction is [DC-26](#dc-26-the-delimiter-probe-can-only-run-on-the-first-segment-of-the-interchange).
- **Severity:** MEDIUM

```php
$delimiter1 = '~';
$delimiter2 = '|';
$delimiter3 = '^';
```

That is `src/Billing/ParseERA.php:L484-L486`. Only the first two are read in that function.

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

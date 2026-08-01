# OpenEMR Revenue Cycle and X12 EDI Business Rules

The register of decisions this subsystem makes about money and eligibility, one entry per rule, each anchored to the code that makes it and each labelled with how much of it is observed rather than inferred.

**Scope and sources.** This document registers the non-obvious business rules recovered from OpenEMR's revenue cycle and its handling of X12, the electronic data interchange standard that United States healthcare payers, providers and clearinghouses use to exchange claims, eligibility questions and payments. It was traced from the remittance parser, the accounts-receivable poster, the two claim generators, the batch pipeline, the modern accounting helper, the legacy remittance renderer, the trading-partner model, the schema in `sql/database.sql`, the site-global declarations in `library/globals.inc.php` and the accounts-receivable posting decisions inside the explanation-of-benefits screen. It is a register and nothing else: it does not narrate the revenue cycle, which is [claim-lifecycle.md](claim-lifecycle.md), and it does not document transaction formats or the trading-partner configuration surface, which are [transactions.md](transactions.md). The conventions every entry uses, including the citation format, the two claim classes and the source-of-truth ordering, are defined once in [README.md](README.md) and are applied here without variation or restatement.

**Provenance.** Every line anchor below is relative to branch `master` at head commit `b7a7e690e419de3451740f995b768a8e8e5fba87`, on project version 8.3.0-dev (`version.php:L17-L20`) with database schema version 541 (`version.php:L33`). Nothing was executed to produce this document. PHP and Composer are not installed in the authoring environment, so every rule here rests on static reading of file contents at that commit, and no entry reports an observed run.

**One terminology hazard, stated once.** This document is a register of *business* rules recovered from code. It has nothing to do with user-specified project rules, of which this project has none: the rules facility was consulted and reports that none were provided, so nothing here was written to satisfy a rule and no rule is invented to justify a decision. Where a standard is named below it comes from the documentation requirements this set was written to satisfy or from the repository's own observed practice, as [README.md](README.md) sets out.

## Table of Contents

- [How to Read an Entry](#how-to-read-an-entry)
    - [The entry template](#the-entry-template)
    - [What the Novel field checks](#what-the-novel-field-checks)
    - [Why the rules are grouped this way](#why-the-rules-are-grouped-this-way)
    - [The X12 vocabulary used in this document](#the-x12-vocabulary-used-in-this-document)
- [Rule Index](#rule-index)
- [Group A Monetary Calculation and Rounding](#group-a-monetary-calculation-and-rounding)
- [Group B Remittance Balancing](#group-b-remittance-balancing)
- [Group C Claim and Status Code Mappings](#group-c-claim-and-status-code-mappings)
- [Group D Payer and Partner Specific Branches](#group-d-payer-and-partner-specific-branches)
- [Group E Date Boundary Logic](#group-e-date-boundary-logic)
- [Group F Behaviour That Would Silently Change Amounts](#group-f-behaviour-that-would-silently-change-amounts)
- [Group G Identity and Naming Conventions](#group-g-identity-and-naming-conventions)
- [Group H Control Number and Sequence Allocation](#group-h-control-number-and-sequence-allocation)
- [Group I Site Global Gated Rules](#group-i-site-global-gated-rules)
- [Confidence Summary](#confidence-summary)
- [Related Documents](#related-documents)
- [Documentation Attribution](#documentation-attribution)

## How to Read an Entry

Every entry answers one question about one decision: what does the code decide about money or eligibility here, and how much of that reading can be trusted. An entry is deliberately terse. It cites code rather than copying it, and where an excerpt appears it is two or three lines and exists only to show the shape of the expression the rule lives in.

Three habits make the register usable. Read the `Status:` field before the `Statement:` field, because a Low-confidence inference and a verified observation are not the same kind of sentence. Read the `Blast radius:` field before changing anything the entry cites, because that field is the reason the entry exists. And follow the cross-reference when one is present: where a rule's implementation is itself suspect the entry links to [defect-candidates.md](defect-candidates.md), and a rule and the suspicion about it are not meant to be read apart.

### The entry template

Every entry carries these six fields, in this order, none omitted and none empty:

```text
### BR-<group><n> <short rule name>

Statement:   <plain-English statement of the rule, no jargon>
Evidence:    <path/to/file.php:L120-L145>
Status:      VERIFIED | INFERRED (confidence: High | Medium | Low)
Intent:      <why the rule plausibly exists>
Novel:       <yes/no - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md,
              sl_eob_help.php, cms_1500_help.php>
Blast radius: <what silently changes if this rule is altered>
```

Each entry below renders that template as a labelled list rather than as preformatted text, so that the fields survive markdown rendering and the citations stay selectable. The field names, their order and the requirement that all six be present are unchanged.

### What the Novel field checks

The `Novel:` field records which existing sources were checked, so that a claim of novelty can be falsified rather than merely asserted. Four sources were read in full and are the ones every `Novel:` field names:

| Source | Size | What it covers |
|--------|-----:|----------------|
| `Documentation/Readme_edihistory.html` | 246 lines | The legacy EDI history tree only, written in 2016 |
| `Documentation/api/DEVELOPER_GUIDE.md` | 1,678 lines | The REST and FHIR surface. Verified silent on this subsystem: a whole-word search for billing, X12, EDI, claim, 837 and 835 returns no match, the only near-hit being the substring inside a JSON web token accessor at `Documentation/api/DEVELOPER_GUIDE.md:L333-L334` |
| `Documentation/help_files/sl_eob_help.php` | 172 lines | Operator instructions for the explanation-of-benefits posting screen |
| `Documentation/help_files/cms_1500_help.php` | 120 lines | Operator instructions for the paper claim form |

Sizes were taken by line count at the recorded commit. Where an existing source touches a rule without stating it, the `Novel:` field says so and cites the line, because an agreement is evidence about the rule and hiding it would overstate the novelty.

### Why the rules are grouped this way

The nine groups are ordered by consequence rather than by where the code happens to live, so that the entries a reader most needs to see first are first. Money arithmetic leads, because an error there is an error in a dollar amount. Balancing follows, because it is the one place where the subsystem rewrites a payer's own numbers. Code mappings, payer-specific branches and date boundaries follow in turn. Group F is the group to read if only one group is read: it collects the behaviour that would change what a patient or a payer is billed without anything on a screen saying so. Identity, sequence allocation and the site-global switches close the register.

A rule appears in exactly one group. Several rules could defensibly sit in two, and where that is so the entry is placed in the group that describes its consequence rather than its mechanism, and the neighbouring entries are named in the text.

### The X12 vocabulary used in this document

X12 knowledge is not assumed. Every term used below is expanded here so that the entries themselves stay short, and each is expanded again at its first use in prose.

**Envelope and structural terms.** An X12 file is wrapped in nested envelopes. **ISA** is the interchange control header, the outermost wrapper, and its numbered elements are written ISA01 through ISA16: **ISA13** is the interchange control number, **ISA14** the acknowledgement-requested indicator and **ISA15** the interchange usage indicator, where the code `T` means test data and the code `P` means production data. **IEA** is the matching interchange trailer. **GS** is the functional group header, whose **GS06** is the group control number, and **GE** is its trailer. **ST** is the transaction set header, whose **ST02** is the transaction set control number, and **SE** is its trailer, whose **SE01** is a count of the segments in the transaction set. **BHT** is the beginning of hierarchical transaction segment; its **BHT03** is a reference identification and its **BHT06** is a transaction type code, where `CH` means chargeable and `RP` means reporting. A **loop** is a repeatable group of segments identified by a number such as 2100 or 2110.

**Transaction set numbers.** **837P** is a professional claim and **837I** an institutional claim. **835** is remittance advice, a payer's statement of what it paid and why, also called an **ERA** for electronic remittance advice; the human-readable equivalent is an **EOB**, an explanation of benefits. **270** and **271** are an eligibility request and its response. **276** and **277** are a claim-status inquiry and its response. **278** is a services review, that is, an authorisation. **997** and **999** are acknowledgements that report whether a transmission was syntactically accepted.

**Segment identifiers.** **CLP** carries claim-level payment information; its **CLP02** is the claim status code and its later elements carry the charged, paid and patient-responsibility amounts. **SVC** carries service-line payment information. **CAS** carries a claim or service adjustment, made up of a group code, a reason code, and an amount; the group codes this register names are `PR` for patient responsibility, `CO` for contractual obligation and `CR` for correction and reversal. **PLB** carries a provider-level adjustment, which belongs to the provider rather than to any one claim. **MIA** carries Medicare inpatient adjudication information. **TRN** is a trace number, **NM1** a name, **DTM** a date, **AMT** an amount, **QTY** a quantity, **LX** a service-line counter, **REF** a reference identifier and **PWK** a paperwork or attachment reference. **CLM** is the claim segment of an outbound 837, whose **CLM01** is the submitter's own claim identifier and **CLM02** the total charge. **SV1** is a professional service line, whose **SV102** is the line charge and **SV107** the list of diagnosis code pointers.

**Accounting terms.** **A/R** is accounts receivable: what has been billed and not yet settled. In this schema a deposit is a row of `ar_session` and a ledger line is a row of `ar_activity`. **CARC** is a claim adjustment reason code and **RARC** a remittance advice remark code.

## Rule Index

Seventy-one rules. Every one appears once in this table and once as an entry below.

| ID | Short name | Group | Status | Confidence |
|----|------------|-------|--------|------------|
| [BR-A1](#br-a1-money-equality-is-decided-in-integer-cents) | Money equality is decided in integer cents | A | VERIFIED | n/a |
| [BR-A2](#br-a2-every-persisted-amount-is-fixed-point-to-two-decimals) | Every persisted amount is fixed point to two decimals | A | VERIFIED | n/a |
| [BR-A3](#br-a3-two-coverage-money-fields-escape-the-fixed-point-convention) | Two coverage money fields escape the fixed-point convention | A | VERIFIED | n/a |
| [BR-A4](#br-a4-charge-totals-are-accumulated-as-floats-and-printed-at-two-decimals) | Charge totals are accumulated as floats and printed at two decimals | A | VERIFIED | n/a |
| [BR-A5](#br-a5-patient-responsibility-is-excluded-from-the-adjustment-total) | Patient responsibility is excluded from the adjustment total | A | VERIFIED | n/a |
| [BR-A6](#br-a6-both-balancing-totals-are-rounded-before-the-imbalance-test) | Both balancing totals are rounded before the imbalance test | A | VERIFIED | n/a |
| [BR-A7](#br-a7-a-negative-contractual-obligation-is-inverted-unless-its-reason-code-is-144) | A negative contractual obligation is inverted unless its reason code is 144 | A | VERIFIED | n/a |
| [BR-A8](#br-a8-a-non-numeric-adjustment-amount-becomes-zero) | A non-numeric adjustment amount becomes zero | A | VERIFIED | n/a |
| [BR-A9](#br-a9-a-charge-that-disagrees-with-the-invoice-by-one-cent-blocks-the-posting) | A charge that disagrees with the invoice by one cent blocks the posting | A | VERIFIED | n/a |
| [BR-B1](#br-b1-a-remittance-balances-when-the-fee-equals-five-totals-in-integer-cents) | A remittance balances when the fee equals five totals in integer cents | B | VERIFIED | n/a |
| [BR-B2](#br-b2-provider-level-adjustments-are-excluded-from-ar-and-included-in-the-balance-test) | Provider-level adjustments are excluded from A/R and included in the balance test | B | VERIFIED | n/a |
| [BR-B3](#br-b3-balancing-rewrites-the-payers-own-service-amounts) | Balancing rewrites the payer's own service amounts | B | VERIFIED | n/a |
| [BR-B4](#br-b4-an-artificial-service-line-named-claim-absorbs-the-residue) | An artificial service line named Claim absorbs the residue | B | VERIFIED | n/a |
| [BR-B5](#br-b5-the-production-date-is-backfilled-from-the-check-date) | The production date is backfilled from the check date | B | VERIFIED | n/a |
| [BR-B6](#br-b6-informational-amount-and-quantity-segments-are-excluded-from-balancing) | Informational amount and quantity segments are excluded from balancing | B | VERIFIED | n/a |
| [BR-B7](#br-b7-a-deposit-is-balanced-against-live-ledger-lines-only-and-only-in-a-browser-alert) | A deposit is balanced against live ledger lines only, and only in a browser alert | B | VERIFIED | n/a |
| [BR-C1](#br-c1-three-hardcoded-tables-carry-1391-x12-code-descriptions) | Three hardcoded tables carry 1,391 X12 code descriptions | C | VERIFIED | n/a |
| [BR-C2](#br-c2-the-payers-own-status-code-decides-which-insurance-level-is-credited) | The payer's own status code decides which insurance level is credited | C | VERIFIED | n/a |
| [BR-C3](#br-c3-four-different-status-codes-all-mean-forwarded-to-another-payer) | Four different status codes all mean forwarded to another payer | C | VERIFIED | n/a |
| [BR-C4](#br-c4-patient-responsibility-reasons-are-relabelled-to-fit-a-patient-statement) | Patient-responsibility reasons are relabelled to fit a patient statement | C | VERIFIED | n/a |
| [BR-C5](#br-c5-a-contractual-write-off-of-reason-45-or-59-is-exempt-from-the-zero-payment-error) | A contractual write-off of reason 45 or 59 is exempt from the zero-payment error | C | VERIFIED | n/a |
| [BR-C6](#br-c6-a-denial-is-status-7-and-is-recorded-unlike-any-other-status) | A denial is status 7 and is recorded unlike any other status | C | VERIFIED | n/a |
| [BR-D1](#br-d1-the-payer-level-travels-as-a-single-character-on-the-front-of-the-payer-identifier) | The payer level travels as a single character on the front of the payer identifier | D | VERIFIED | n/a |
| [BR-D2](#br-d2-an-unrecognised-payer-prefix-becomes-payer-type-zero) | An unrecognised payer prefix becomes payer type zero | D | VERIFIED | n/a |
| [BR-D3](#br-d3-the-processing-format-is-read-per-claim-stored-and-branches-on-nothing) | The processing format is read per claim, stored, and branches on nothing | D | INFERRED | Medium |
| [BR-D4](#br-d4-a-claim-with-no-trading-partner-is-dropped-from-an-electronic-run-only) | A claim with no trading partner is dropped from an electronic run only | D | VERIFIED | n/a |
| [BR-D5](#br-d5-the-claim-identifier-is-recovered-from-the-remittance-by-counting-its-parts) | The claim identifier is recovered from the remittance by counting its parts | D | VERIFIED | n/a |
| [BR-D6](#br-d6-a-secondary-payer-that-drops-a-modifier-is-matched-by-rebuilding-the-key) | A secondary payer that drops a modifier is matched by rebuilding the key | D | VERIFIED | n/a |
| [BR-D7](#br-d7-the-application-sender-code-falls-back-to-the-interchange-sender-identifier) | The application sender code falls back to the interchange sender identifier | D | VERIFIED | n/a |
| [BR-D8](#br-d8-eligibility-uses-whichever-primary-coverage-row-the-database-returns-first) | Eligibility uses whichever primary coverage row the database returns first | D | VERIFIED | n/a |
| [BR-E1](#br-e1-a-coverage-row-with-no-start-date-is-effective-for-every-service-date) | A coverage row with no start date is effective for every service date | E | VERIFIED | n/a |
| [BR-E2](#br-e2-the-service-date-is-the-first-ten-characters-of-the-encounter-timestamp) | The service date is the first ten characters of the encounter timestamp | E | VERIFIED | n/a |
| [BR-E3](#br-e3-the-institutional-date-converter-hardcodes-the-century) | The institutional date converter hardcodes the century | E | VERIFIED | n/a |
| [BR-E4](#br-e4-an-empty-institutional-date-becomes-the-two-character-string-20) | An empty institutional date becomes the two-character string 20 | E | VERIFIED | n/a |
| [BR-E5](#br-e5-every-envelope-and-transaction-date-is-server-local) | Every envelope and transaction date is server-local | E | VERIFIED | n/a |
| [BR-E6](#br-e6-the-professional-interchange-date-is-a-placeholder-the-batch-replaces) | The professional interchange date is a placeholder the batch replaces | E | VERIFIED | n/a |
| [BR-E7](#br-e7-the-operators-pay-date-overrides-the-payers-own-dates) | The operator's pay date overrides the payer's own dates | E | VERIFIED | n/a |
| [BR-E8](#br-e8-a-void-matches-ledger-lines-by-exact-posting-timestamp) | A void matches ledger lines by exact posting timestamp | E | VERIFIED | n/a |
| [BR-E9](#br-e9-the-copay-in-force-is-the-one-on-the-latest-starting-primary-coverage-row) | The copay in force is the one on the latest starting primary coverage row | E | VERIFIED | n/a |
| [BR-F1](#br-f1-non-primary-insurance-adjustments-are-recorded-as-notes-worth-zero) | Non-primary insurance adjustments are recorded as notes worth zero | F | VERIFIED | n/a |
| [BR-F2](#br-f2-a-code-whose-colon-is-its-first-character-keeps-its-modifier) | A code whose colon is its first character keeps its modifier | F | VERIFIED | n/a |
| [BR-F3](#br-f3-a-failed-transmission-is-recorded-as-a-success) | A failed transmission is recorded as a success | F | VERIFIED | n/a |
| [BR-F4](#br-f4-a-malformed-interchange-header-terminates-the-whole-batch-run) | A malformed interchange header terminates the whole batch run | F | VERIFIED | n/a |
| [BR-F5](#br-f5-one-batch-file-is-queued-once-for-every-partner-in-the-batch) | One batch file is queued once for every partner in the batch | F | VERIFIED | n/a |
| [BR-F6](#br-f6-the-attachment-segment-is-emitted-without-being-counted) | The attachment segment is emitted without being counted | F | VERIFIED | n/a |
| [BR-F7](#br-f7-a-service-line-carries-at-most-four-diagnosis-pointers) | A service line carries at most four diagnosis pointers | F | VERIFIED | n/a |
| [BR-F8](#br-f8-a-charge-created-from-a-remittance-ignores-the-dry-run-flag) | A charge created from a remittance ignores the dry-run flag | F | VERIFIED | n/a |
| [BR-F9](#br-f9-the-institutional-generator-always-declares-the-claim-chargeable) | The institutional generator always declares the claim chargeable | F | VERIFIED | n/a |
| [BR-F10](#br-f10-a-voided-receipt-is-journalled-only-under-one-configuration) | A voided receipt is journalled only under one configuration | F | VERIFIED | n/a |
| [BR-F11](#br-f11-a-remittance-carrying-a-medicare-inpatient-adjudication-segment-posts-nothing) | A remittance carrying a Medicare inpatient adjudication segment posts nothing | F | VERIFIED | n/a |
| [BR-G1](#br-g1-a-claim-is-identified-by-patient-then-encounter) | A claim is identified by patient then encounter | G | VERIFIED | n/a |
| [BR-G2](#br-g2-the-ledger-payer-type-is-cut-out-of-a-user-interface-label) | The ledger payer type is cut out of a user-interface label | G | VERIFIED | n/a |
| [BR-G3](#br-g3-payer-type-zero-means-two-incompatible-things) | Payer type zero means two incompatible things | G | VERIFIED | n/a |
| [BR-G4](#br-g4-a-ledger-line-is-either-a-payment-or-an-adjustment-by-comment-alone) | A ledger line is either a payment or an adjustment, by comment alone | G | VERIFIED | n/a |
| [BR-G5](#br-g5-the-claim-level-ledger-convention-is-an-empty-code-and-the-remittance-path-writes-a-word) | The claim-level ledger convention is an empty code, and the remittance path writes a word | G | VERIFIED | n/a |
| [BR-G6](#br-g6-values-passed-through-the-cleaner-are-uppercased-and-filtered-to-a-fixed-character-set) | Values passed through the cleaner are uppercased and filtered to a fixed character set | G | VERIFIED | n/a |
| [BR-G7](#br-g7-an-encounter-counts-as-billed-only-when-every-fee-bearing-charge-is-billed) | An encounter counts as billed only when every fee-bearing charge is billed | G | VERIFIED | n/a |
| [BR-G8](#br-g8-a-deposit-is-found-by-a-reference-that-is-neither-unique-nor-nullable) | A deposit is found by a reference that is neither unique nor nullable | G | VERIFIED | n/a |
| [BR-H1](#br-h1-the-claim-version-is-allocated-by-an-unlocked-aggregate-inside-a-transaction) | The claim version is allocated by an unlocked aggregate inside a transaction | H | VERIFIED | n/a |
| [BR-H2](#br-h2-the-ledger-sequence-number-is-allocated-the-same-way-and-the-code-says-so) | The ledger sequence number is allocated the same way, and the code says so | H | VERIFIED | n/a |
| [BR-H3](#br-h3-a-crossover-claim-row-records-five-columns-and-discards-the-rest) | A crossover claim row records five columns and discards the rest | H | VERIFIED | n/a |
| [BR-H4](#br-h4-interchange-and-group-control-numbers-are-two-separate-draws-on-one-sequence) | Interchange and group control numbers are two separate draws on one sequence | H | VERIFIED | n/a |
| [BR-H5](#br-h5-a-validation-run-substitutes-literal-control-numbers) | A validation run substitutes literal control numbers | H | VERIFIED | n/a |
| [BR-H6](#br-h6-the-batch-renumbers-the-transaction-set-and-rewrites-the-reference-it-planted) | The batch renumbers the transaction set and rewrites the reference it planted | H | VERIFIED | n/a |
| [BR-H7](#br-h7-the-segment-count-is-copied-from-the-generator-into-the-batch-unchanged) | The segment count is copied from the generator into the batch unchanged | H | VERIFIED | n/a |
| [BR-H8](#br-h8-envelope-validity-is-encoded-in-the-length-of-a-string) | Envelope validity is encoded in the length of a string | H | VERIFIED | n/a |
| [BR-I1](#br-i1-claim-balancing-is-on-by-default) | Claim balancing is on by default | I | VERIFIED | n/a |
| [BR-I2](#br-i2-one-switch-changes-the-output-channel-and-the-submitter-identity) | One switch changes the output channel and the submitter identity | I | VERIFIED | n/a |
| [BR-I3](#br-i3-one-switch-turns-a-payer-reported-unknown-code-into-a-charge) | One switch turns a payer-reported unknown code into a charge | I | VERIFIED | n/a |
| [BR-I4](#br-i4-one-switch-decides-whether-anything-is-transmitted-at-all) | One switch decides whether anything is transmitted at all | I | VERIFIED | n/a |
| [BR-I5](#br-i5-a-newly-inserted-trading-partner-transmits-live-claims) | A newly inserted trading partner transmits live claims | I | VERIFIED | n/a |

The `Confidence` column reads `n/a` for a verified rule because confidence is a property of an inference, not of an observation. Where an entry's `Status:` is VERIFIED and the entry nonetheless carries an inference about intent or consequence, that inference is labelled inside the entry with its own confidence.

## Group A Monetary Calculation and Rounding

Nine rules. This group leads the register because an error in any of them is an error in a dollar amount, and because the subsystem decides three separate questions here that are usually decided once: how an amount is stored, how it is compared, and how it is printed onto the wire.

### BR-A1 Money equality is decided in integer cents

- **Statement:** Two amounts count as equal when they are equal after being multiplied by one hundred and rounded to whole numbers. Nothing in this subsystem compares two amounts as floating-point numbers when deciding whether a remittance balances.
- **Evidence:** `RemitAccounting::isBalanced()` at `src/Billing/EdiHistory/RemitAccounting.php:L27-L31`, in particular the comparison at `src/Billing/EdiHistory/RemitAccounting.php:L30`.
- **Status:** VERIFIED
- **Intent:** To make the balance test immune to binary floating-point representation, which cannot represent most decimal fractions exactly. The method's own docblock states that reason and gives the standard counter-example at `src/Billing/EdiHistory/RemitAccounting.php:L19-L23`, so the intent is recorded rather than guessed, although a docblock is evidence of intent only.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Replacing the integer comparison with `==` on the floats would make a correctly balanced remittance report as unbalanced for amounts whose cents do not survive binary representation, and the report would be intermittent rather than reproducible, because it depends on the particular values in the file.

```php
$accounted = $acctng['pmt'] + $acctng['clmadj'] + $acctng['svcadj'] + $acctng['svcptrsp'] + $acctng['plbadj'];
return (int) round($acctng['fee'] * 100) === (int) round($accounted * 100);
```

That is `src/Billing/EdiHistory/RemitAccounting.php:L29-L30`. Note that the comparison is `===`, so both sides must be integers as well as equal; the casts guarantee that.

### BR-A2 Every persisted amount is fixed point to two decimals

- **Statement:** Every column in this subsystem that holds an amount of money is declared `decimal(12,2)`, a fixed-point number with twelve digits of precision and exactly two decimal places. Rounding to cents is therefore enforced by the database rather than by any calculation, and a third decimal place cannot be stored at all.
- **Evidence:** Eight columns across five tables, each anchored: `sql/database.sql:L266` for `billing.fee`, `sql/database.sql:L1554` for `drug_sales.fee`, `sql/database.sql:L10166` for `ar_session.pay_total`, `sql/database.sql:L10169` for `ar_session.global_amount`, `sql/database.sql:L10200` for `ar_activity.pay_amount`, `sql/database.sql:L10201` for `ar_activity.adj_amount`, `sql/database.sql:L10008` for `voids.amount1` and `sql/database.sql:L10009` for `voids.amount2`.
- **Status:** VERIFIED
- **Intent:** To keep currency out of floating-point storage. INFERRED (confidence: High): the choice is deliberate rather than incidental, because the declaration is identical in all eight places including the two most recently added tables. Basis: uniformity across tables added at different times, with no counter-example among the eight.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The generated schema documentation under `Documentation/EHI_Export/` reports the column types but does not draw the rule from them.
- **Blast radius:** A calculation that produces a third decimal place is silently rounded on the way into the database and not on the way out of the calculation, so a value used in one more sum before being stored can differ from the value that is eventually persisted. Widening any of these columns would change nothing until a calculation started producing sub-cent values, at which point the amounts a patient sees and the amounts a payer sees would begin to diverge.

```sql
pay_amount     decimal(12,2) NOT NULL DEFAULT 0  COMMENT 'either pay or adj will always be 0',
adj_amount     decimal(12,2) NOT NULL DEFAULT 0,
```

That is `sql/database.sql:L10200-L10201`, the two ledger amount columns. The census was taken by reading each table's `CREATE TABLE` block and extracting every column of a numeric decimal type. All eight declare the same precision and scale; they differ only in nullability, in their defaults and in incidental whitespace inside the type, which is written with spaces at `sql/database.sql:L10169` and without them everywhere else. The exceptions to the convention are the subject of [BR-A3](#br-a3-two-coverage-money-fields-escape-the-fixed-point-convention), and the invariant recorded in the first column's comment is the subject of [BR-G4](#br-g4-a-ledger-line-is-either-a-payment-or-an-adjustment-by-comment-alone).

### BR-A3 Two coverage money fields escape the fixed-point convention

- **Statement:** The copay recorded against a patient's insurance coverage is stored as free text, and the copay and deductible recorded against an eligibility response are stored as whole numbers. Three fields that all describe a patient's out-of-pocket amount are therefore stored in three different ways, only one of which can hold cents.
- **Evidence:** `sql/database.sql:L3332` declares `insurance_data.copay` as `varchar(255)`; `sql/database.sql:L1652` and `sql/database.sql:L1653` declare `eligibility_verification.copay` and `eligibility_verification.deductible` as integers. The reader that turns the free-text value into money is `src/Billing/BillingUtilities.php:L1769-L1778`, which formats it at `src/Billing/BillingUtilities.php:L1775`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the text column is historical rather than deliberate, and the integer columns were written for a payer response that reports whole dollars. Basis: the reader coerces the text with `floatval()` before formatting it, which is what a caller does when it cannot rely on the stored form; no comment, constraint or migration in the repository states an intent for either shape.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** A copay entered with a currency symbol, a space or a comma is coerced to a number by prefix, so `12.50` and `$12.50` do not produce the same amount, and text that begins with no digit produces zero. Because the coercion is silent, a patient can be charged a zero copay by a data-entry variation that the screen accepted. The zero-return path is the subject of the note below.

```php
if (!empty($tmp['provider'])) {
    return sprintf('%01.2f', floatval($tmp['copay']));
}
```

That is `src/Billing/BillingUtilities.php:L1774-L1776`. VERIFIED: when no primary coverage row matches, the method returns the integer `0`, at `src/Billing/BillingUtilities.php:L1778`, while the comment above the method says it returns minus one in that case, at `src/Billing/BillingUtilities.php:L1767`. Applying the source-of-truth ordering from [README.md](README.md), the code decides the behaviour and the comment is admissible only as evidence of what was meant. Callers that test for a negative sentinel are covered in [defect-candidates.md](defect-candidates.md).

### BR-A4 Charge totals are accumulated as floats and printed at two decimals

- **Statement:** The total charge on an outbound professional claim is built by adding the individual line charges together as floating-point numbers, and is then printed with exactly two decimal places. The line charges are printed the same way, one at a time.
- **Evidence:** `src/Billing/X125010837P.php:L665-L667` accumulates, `src/Billing/X125010837P.php:L676` prints the claim total into CLM02, and `src/Billing/X125010837P.php:L1339` prints each line charge into SV102. The values come from `src/Billing/Claim.php:L1441-L1444`.
- **Status:** VERIFIED
- **Intent:** To satisfy the implementation guide, which requires a claim total that equals the sum of the service line amounts. Printing both sides from the same source is the mechanism that makes them agree.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The two sides are rounded independently: each line is rounded once and the total is rounded once, after the additions. For a claim with enough lines, the rounded total can differ by a cent from the sum of the rounded lines, and a payer that checks the arithmetic rejects the claim rather than paying it short. Changing either `sprintf` call, or accumulating in cents instead, changes which claims a payer accepts.

```php
$clm_total_charges += floatval($claim->cptCharges($prockey));
```

That is `src/Billing/X125010837P.php:L667`. VERIFIED: the source value has already passed through the identity cleaner described in [BR-G6](#br-g6-values-passed-through-the-cleaner-are-uppercased-and-filtered-to-a-fixed-character-set), so it reaches `floatval()` as an uppercased, character-filtered string rather than as a number.

### BR-A5 Patient responsibility is excluded from the adjustment total

- **Statement:** When the subsystem recomputes what a payer's adjustments should add up to, it counts every adjustment except those the payer marked as the patient's responsibility. Deductibles, coinsurance and copays are therefore not treated as reducing what the payer owes.
- **Evidence:** `src/Billing/ParseERA.php:L42-L49`, with the exclusion at `src/Billing/ParseERA.php:L45`. The claim-level patient-responsibility amount is subtracted separately, once, at `src/Billing/ParseERA.php:L40-L41`.
- **Status:** VERIFIED
- **Intent:** To avoid subtracting the patient's share twice. The claim segment already reports a patient-responsibility total, which the arithmetic removes at the claim level, so removing the individual service-level patient-responsibility adjustments as well would double-count them.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Dropping the exclusion makes every remittance that carries service-level patient responsibility appear unbalanced by exactly the patient's share, and the residue is then absorbed into a synthetic adjustment by [BR-B4](#br-b4-an-artificial-service-line-named-claim-absorbs-the-residue), which means the practice writes off the patient's own liability without anyone choosing to.

```php
if ($adj['group_code'] != 'PR') {
    $adjtotal -= $adj['amount'];
}
```

That is `src/Billing/ParseERA.php:L45-L47`. The comparison is loose rather than strict, so it rests on the group code arriving as the exact two-character string `PR`.

### BR-A6 Both balancing totals are rounded before the imbalance test

- **Statement:** The recomputed payment total and the recomputed adjustment total are each rounded to two decimal places before either is tested for being non-zero. A residue smaller than half a cent therefore does not trigger the balancing rewrite.
- **Evidence:** `src/Billing/ParseERA.php:L51-L52`, tested at `src/Billing/ParseERA.php:L53`.
- **Status:** VERIFIED
- **Intent:** To stop floating-point noise from creating a synthetic adjustment worth a fraction of a cent. Without the rounding, the sum of a dozen exact cent amounts can leave a residue of the order of one ten-trillionth, which the non-zero test would treat as an imbalance.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Removing either rounding call makes the balancing rewrite fire on almost every remittance, inserting a synthetic service line and a synthetic adjustment worth a rounding error into the ledger of nearly every claim. The tests are loose comparisons against the integer zero, so the rounding is what makes them mean what they appear to mean.

```php
$paytotal = round($paytotal, 2);
$adjtotal = round($adjtotal, 2);
if ($paytotal != 0 || $adjtotal != 0) {
```

That is `src/Billing/ParseERA.php:L51-L53`. Note that this rule and [BR-A1](#br-a1-money-equality-is-decided-in-integer-cents) solve the same problem in two different ways in two different generations: the parser rounds to two decimals and compares loosely, while the modern accounting helper scales to integers and compares strictly.

### BR-A7 A negative contractual obligation is inverted unless its reason code is 144

- **Statement:** A service-level adjustment that the payer marked as a contractual obligation and reported as a negative amount has its sign flipped, and a warning is recorded. One reason code is exempt from the flip, and keeps its negative amount.
- **Evidence:** `src/Billing/ParseERA.php:L402-L406`, with the exemption expressed as the third condition at `src/Billing/ParseERA.php:L402`.
- **Status:** VERIFIED
- **Intent:** The inline comment beside the condition, at `src/Billing/ParseERA.php:L401`, states that the exemption exists to stop the flip from breaking claim balancing for one specific incentive adjustment. Read as evidence of intent, that says the general flip corrects payers that report contractual write-offs with the wrong sign, and that the exempt code is a case where the negative sign is correct.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The flip changes a payer-supplied amount before it reaches the ledger, so the ledger and the remittance file disagree by twice the adjustment. Removing the exemption reintroduces an imbalance on every remittance carrying that incentive code; widening it to other codes silently converts write-offs into credits.

```php
if ($seg[1] == 'CO' && $seg[$k + 1] < 0 && $seg[$k] !== '144') {
```

That is `src/Billing/ParseERA.php:L402`. The reason-code comparison is strict while the group-code comparison beside it is loose, so the exemption requires the code to arrive as the exact three-character string.

### BR-A8 A non-numeric adjustment amount becomes zero

- **Statement:** Where a payer sends an adjustment amount that is not a number, the amount is treated as zero rather than rejected. This happens twice on the way from the file to the screen: once when the adjustment is parsed and once when it is displayed.
- **Evidence:** `src/Billing/ParseERA.php:L412-L413` on the parsing side and `interface/billing/sl_eob_process.php:L634` on the display side.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): to keep one malformed element from aborting an otherwise usable remittance, since almost every other unexpected condition in this parser returns an error string that ends the parse. Basis: the coercion is written as a conditional expression with an explicit zero default rather than as a validation branch, and the surrounding code has no other tolerant path.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** An adjustment that should reduce the amount owed instead reduces it by nothing, and no message says so, because the substitution happens before any total is computed. The amount displayed on the posting screen and the amount used in the balancing arithmetic are produced by two separate coercions of the same element, so they can disagree if only one is changed.

```php
$raw = $seg[$k + 1] ?? 0;
$out['svc'][$i]['adj'][$j]['amount'] = is_numeric($raw) ? (float)$raw : 0.0;
```

That is `src/Billing/ParseERA.php:L412-L413`. The parallel coercion on the display side is a separate expression over the same value, and is registered here as one rule because both express the same decision. The consequences of the pair are carried in [defect-candidates.md](defect-candidates.md).

### BR-A9 A charge that disagrees with the invoice by one cent blocks the posting

- **Statement:** Before anything is posted for a service line, the amount the payer says was charged is compared with the amount on the practice's own invoice, to the cent. If they differ, the whole claim is put into error mode and nothing is posted for it.
- **Evidence:** `interface/billing/sl_eob_process.php:L483-L492`, with the comparison at `interface/billing/sl_eob_process.php:L484-L485` and the error flag at `interface/billing/sl_eob_process.php:L491`.
- **Status:** VERIFIED
- **Intent:** To stop a remittance being posted against an invoice it does not describe, which is the failure that would corrupt a ledger most quietly. Comparing formatted strings rather than numbers makes the test exact at cent resolution and insensitive to representation.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The operator help describes the posting screen at `Documentation/help_files/sl_eob_help.php:L100-L104` without mentioning this check or what triggers it.
- **Blast radius:** The invoice side of the comparison adds the existing charge and the existing adjustment together, and the remittance side takes the absolute value of the reported charge, so a claim carrying a reversal compares a signed amount against an unsigned one. Loosening the comparison would allow partial posting against a mismatched invoice; tightening it further would block posting on claims that have already had an adjustment applied.

```php
$prevchg = sprintf("%.2f", $prev['chg'] + ($prev['adj'] ?? null));
if ($prevchg !== sprintf("%.2f", abs($svc['chg']))) {
```

That is `interface/billing/sl_eob_process.php:L484-L485`. Both sides are formatted before the strict string comparison, which is what makes the test exact to the cent rather than exact to the float.

## Group B Remittance Balancing

Seven rules. Balancing is the question of whether the dollars a payer reports on a remittance add up, and it is the one place in this subsystem where two generations of code answer the same question differently. [BR-B2](#br-b2-provider-level-adjustments-are-excluded-from-ar-and-included-in-the-balance-test) records that disagreement.

### BR-B1 A remittance balances when the fee equals five totals in integer cents

- **Statement:** A remittance is considered balanced when the provider's fee equals the sum of five separate totals: the payment, the claim-level adjustments, the service-level adjustments, the service-level patient responsibility, and the provider-level adjustments. The comparison is made in whole cents.
- **Evidence:** `src/Billing/EdiHistory/RemitAccounting.php:L27-L31`. The five keys the caller must supply, and the sixth key that is the left-hand side, are documented at `src/Billing/EdiHistory/RemitAccounting.php:L25`.
- **Status:** VERIFIED
- **Intent:** To express the accounting identity that a remittance asserts: everything the payer did with the charge is either paid, adjusted away, moved to the patient, or handled at provider level, so the five categories must reconstruct the charge exactly.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The requirements for this documentation set named this method by example, which is why it appears here by name.
- **Blast radius:** Every category in the sum is a policy decision about what belongs in the identity. Dropping any one of the five makes every remittance that carries that category appear unbalanced; adding a sixth makes remittances balance that previously did not. Because the method is static and takes a plain array, a caller that omits a key raises an undefined-key condition rather than balancing on four categories.

The method is named `RemitAccounting::isBalanced()`. The class is one of the four strict-typed classes in `src/Billing/EdiHistory/`, and is not declared final, at `src/Billing/EdiHistory/RemitAccounting.php:L17`, so a subclass can override the identity. See [architecture.md](architecture.md) for what that namespace is and why it exists.

### BR-B2 Provider-level adjustments are excluded from A/R and included in the balance test

- **Statement:** A provider-level adjustment is an amount a payer withholds from or adds to a whole payment rather than to one claim. The remittance parser deliberately keeps those amounts out of accounts receivable and records them as notes only. The modern balance test includes exactly the same amounts in its arithmetic. Two components in the same subsystem therefore disagree about whether these dollars are in scope for balancing.
- **Evidence:** The exclusion is at `src/Billing/ParseERA.php:L429-L431`, with the reasoning stated in the comment at `src/Billing/ParseERA.php:L430-L431` and the notes-only treatment at `src/Billing/ParseERA.php:L437-L438`. The inclusion is at `src/Billing/EdiHistory/RemitAccounting.php:L29`, where `plbadj` is the fifth addend.
- **Status:** VERIFIED
- **Intent:** The parser's own comment gives its reason: these amounts belong to the general ledger rather than to a claim's receivable. Read as evidence of intent, that is a coherent accounting position. The balance test's inclusion is equally coherent on its own terms, because a remittance whose provider-level adjustment is omitted from the identity will not reconstruct the deposit. INFERRED (confidence: High): the two were written to answer different questions and the disagreement was never reconciled. Basis: the parser dates from the procedural era and states its position in a comment, while the balance test is strict-typed, integer-cents code with a docblock that lists `plbadj` as a required input; neither refers to the other.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. This is one of the three headline findings this register was written to record.
- **Blast radius:** Anything that reports a remittance as balanced using the modern test, and then relies on accounts receivable to reflect that balance, is comparing two different quantities whenever a remittance carries a provider-level adjustment. Reconciling the two in either direction changes a number an operator sees: excluding the amount from the balance test makes previously balanced deposits report as short, and including it in accounts receivable posts a general-ledger amount against a patient's claim.

```php
} elseif ($segid == 'PLB') {
    // Provider-level adjustments are a General Ledger thing and should not
    // alter the A/R for the claim, so we just report them as notes.
```

That is `src/Billing/ParseERA.php:L429-L431`. The contradiction is carried as a CRITICAL entry in [defect-candidates.md](defect-candidates.md), because a rule and the suspicion about it must not be documented apart from each other. The remittance stage this fires in is described in [claim-lifecycle.md](claim-lifecycle.md).

### BR-B3 Balancing rewrites the payer's own service amounts

- **Statement:** When balancing is active and a remittance does not balance, the subsystem changes the amounts it received. The residual payment is added to the first service line's paid amount, and the residual adjustment is appended to that line as a new adjustment. The amounts that reach accounts receivable are therefore not, in that case, the amounts the payer sent.
- **Evidence:** `src/Billing/ParseERA.php:L65` adds the residual payment; `src/Billing/ParseERA.php:L66-L72` appends the residual adjustment. The whole rewrite is inside the block gated at `src/Billing/ParseERA.php:L37`.
- **Status:** VERIFIED
- **Intent:** The comment above the block, at `src/Billing/ParseERA.php:L29-L35`, states the goal as forcing the service sums to equal the claim-level sums, and names poorly reported payment reversals as a cause. Read as evidence of intent, the rewrite exists to make an internally inconsistent remittance postable rather than to reject it.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** This is the single largest silent divergence between a payer's file and the practice's ledger in the subsystem, and it is on by default per [BR-I1](#br-i1-claim-balancing-is-on-by-default). A warning is recorded when a synthetic line is created, per [BR-B4](#br-b4-an-artificial-service-line-named-claim-absorbs-the-residue), but no warning is recorded when the residue is simply added to an existing first line: the code path at `src/Billing/ParseERA.php:L74-L78` that would have warned about exactly that is commented out.

VERIFIED: the suppressed warning text at `src/Billing/ParseERA.php:L74-L78` reads as an assertion that the situation should not happen, and is inert. That is why an operator sees no message when an existing service line's paid amount is altered.

### BR-B4 An artificial service line named Claim absorbs the residue

- **Statement:** If balancing needs somewhere to put a residue and the first service line is not already the synthetic one, a new service line is inserted at the front of the list with the literal procedure code `Claim`, a zero charge and a zero payment. A warning naming it as artificial is recorded.
- **Evidence:** `src/Billing/ParseERA.php:L54` tests for the existing synthetic line, `src/Billing/ParseERA.php:L55-L60` inserts it, and `src/Billing/ParseERA.php:L61-L62` records the warning.
- **Status:** VERIFIED
- **Intent:** To give claim-level money a service-level home, because accounts receivable stores every amount against a code. The same literal code is used when a claim-level payment or adjustment is parsed normally, at `src/Billing/ParseERA.php:L278`, so the synthetic line reuses an existing convention rather than inventing one.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The adjustment appended to the synthetic line is given the group code `CR` and the reason code `Balancing`, neither of which the payer sent. The group code is chosen on a stated presumption, so any report that groups adjustments by group code counts these among corrections and reversals. Renaming the literal `Claim` would break the recognition test at `src/Billing/ParseERA.php:L54` and cause a second synthetic line to be inserted on every subsequent pass.

```php
$out['svc'][0]['adj'][$j]['group_code'] = 'CR'; // presuming a correction or reversal
$out['svc'][0]['adj'][$j]['reason_code'] = 'Balancing';
```

That is `src/Billing/ParseERA.php:L69-L70`. INFERRED (confidence: High): the group code is a guess by the original author rather than a derived value. Basis: the author labelled it as a presumption in the code itself, on the same line.

### BR-B5 The production date is backfilled from the check date

- **Statement:** If a remittance does not carry a production date by the time adjustments are posted, the check date is used instead. The two dates are then indistinguishable downstream.
- **Evidence:** `src/Billing/ParseERA.php:L25-L27`, inside the loop-completion handler entered at `src/Billing/ParseERA.php:L23`.
- **Status:** VERIFIED
- **Intent:** The comment at `src/Billing/ParseERA.php:L24` states that the production date is posted along with adjustments, so the substitution exists to guarantee the field is populated at the moment it is needed rather than to assert the two dates are the same.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Any downstream use of the production date as a distinct fact, such as ageing an adjustment from the date the payer produced the file rather than the date on the cheque, is silently answered with the other date. Removing the backfill leaves the field empty and moves the failure into whatever consumes it. Both dates can in turn be overridden by the operator per [BR-E7](#br-e7-the-operators-pay-date-overrides-the-payers-own-dates).

### BR-B6 Informational amount and quantity segments are excluded from balancing

- **Statement:** Amount segments and quantity segments are recognised at both claim level and service level, and in all four cases their contents are recorded as a warning line and contribute nothing to any total. A payer cannot change what is posted by sending them.
- **Evidence:** Claim level at `src/Billing/ParseERA.php:L338-L341` and `src/Billing/ParseERA.php:L342-L345`; service level at `src/Billing/ParseERA.php:L427-L428`. The claim-level comments state the exclusion explicitly at `src/Billing/ParseERA.php:L340` and `src/Billing/ParseERA.php:L344`.
- **Status:** VERIFIED
- **Intent:** These segments carry supplemental information in the implementation guide rather than adjudication amounts, so treating them as commentary is faithful to the format. The comments name the guide pages they were read from, which is unusually direct evidence of intent for this codebase.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** A quantity segment can legitimately signal that the number of units paid differs from the number billed, and the parser notes at `src/Billing/ParseERA.php:L414-L415` that it ignores exactly that. Starting to honour any of these segments would change posted units or amounts on claims that currently post from the service segment alone. Because the four branches are explicit, the segments are not silently unrecognised and so do not trip the whole-file rejection described under [BR-F11](#br-f11-a-remittance-carrying-a-medicare-inpatient-adjudication-segment-posts-nothing).

### BR-B7 A deposit is balanced against live ledger lines only, and only in a browser alert

- **Statement:** After a remittance file is processed, each deposit created during that run is checked by comparing its recorded total against the sum of its own ledger payment lines, counting only lines that have not been soft-deleted. Any shortfall is reported to the operator as a single browser alert naming the affected deposits, and is not recorded anywhere.
- **Evidence:** `interface/billing/sl_eob_process.php:L856-L871`, with the deposit total read at `interface/billing/sl_eob_process.php:L858`, the live-lines-only sum at `interface/billing/sl_eob_process.php:L860-L863`, the comparison at `interface/billing/sl_eob_process.php:L866`, and the alert at `interface/billing/sl_eob_process.php:L873-L875`. The whole check is skipped in dry-run mode by the guard at `interface/billing/sl_eob_process.php:L853`.
- **Status:** VERIFIED
- **Intent:** To catch the case where a remittance was accepted but not fully distributed across claims, which is the failure the operator help acknowledges at `Documentation/help_files/sl_eob_help.php:L100` when it tells the poster the running amount will hopefully reach zero. That is an agreement between a decade-old user-facing document and the code as it stands, and is recorded here as an agreement rather than restated.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The help file acknowledges the fragility; it does not document this check, its exclusion of deleted lines, or that the result is transient.
- **Blast radius:** The comparison counts payments only and ignores adjustments, and it excludes soft-deleted lines, so a deposit whose payments were voided per [BR-F10](#br-f10-a-voided-receipt-is-journalled-only-under-one-configuration) reports as short even though the void was intentional. Because the only output is a client-side alert, an operator who has navigated away, or whose browser suppressed the dialog, receives no indication at all, and there is no later report that recovers it. The dry-run guard means the check never runs in the mode an operator would use to preview a file.

```php
if (($pay_total - $pay_amount) <> 0) {
    $StringIssue .= $key . ' ';
```

That is `interface/billing/sl_eob_process.php:L866-L867`. The subtraction is a float comparison against zero rather than the integer-cents test of [BR-A1](#br-a1-money-equality-is-decided-in-integer-cents), which is a third answer to the same question inside one subsystem; the consequence is carried in [defect-candidates.md](defect-candidates.md).

## Group C Claim and Status Code Mappings

Six rules. X12 carries meaning in numbered code lists rather than in words, so every one of these rules is a decision about what a number means. The numbers arrive from the payer, and the subsystem's interpretation of them decides which insurance level is credited and what a patient reads on a statement.

### BR-C1 Three hardcoded tables carry 1,391 X12 code descriptions

- **Statement:** The descriptions for three X12 code lists are compiled into the source of one class rather than stored in the database: 17 claim status codes, 291 claim adjustment reason codes and 1,083 remittance advice remark codes. A code the payer sends that is absent from the relevant list has no description available.
- **Evidence:** `src/Billing/BillingUtilities.php:L22-L40` holds the claim status codes, `src/Billing/BillingUtilities.php:L42-L334` the adjustment reason codes and `src/Billing/BillingUtilities.php:L336-L1420` the remark codes. The counts were taken by enumerating the keys of each array literal.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the tables are compiled in so that the posting screen can describe a code with no database round trip and with no dependency on a code table having been loaded for the site. Basis: all three are public class constants on a utility class with no loader, no cache and no fallback lookup, and the only consumers are display paths.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The legacy tree keeps its own separate code tables, described in [architecture.md](architecture.md); this rule concerns the modern namespace only.
- **Blast radius:** The lookups are unguarded array reads. `interface/billing/sl_eob_process.php:L598` indexes the adjustment reason table by whatever reason code the payer sent, and `interface/billing/sl_eob_process.php:L551-L552` indexes the remark table by whatever remark code the payer sent, so a code published after this table was last updated produces an undefined-key condition on the posting screen rather than an unrecognised-code message. Because the tables are source rather than data, keeping them current is a code change and there is no site-level way to add a code.

```php
echo getMessageLine($bgcolor, 'infdetail', "$rmk: " .
    BillingUtilities::REMITTANCE_ADVICE_REMARK_CODES[$rmk]);
```

That is `interface/billing/sl_eob_process.php:L551-L552`. The unguarded read is carried as a defect entry in [defect-candidates.md](defect-candidates.md); the rule registered here is that the code lists are compiled in at all.

### BR-C2 The payer's own status code decides which insurance level is credited

- **Statement:** The insurance level that a remittance is posted against is chosen from the claim status code the payer reported, not from the level the claim was billed at. Status code 2 or 20 credits the second insurance, 3 or 21 credits the third, and every other status code credits the first.
- **Evidence:** `interface/billing/sl_eob_process.php:L347-L352`, with the primary test derived from it at `interface/billing/sl_eob_process.php:L354`.
- **Status:** VERIFIED
- **Intent:** To let the payer tell the practice which of its own coverages adjudicated the claim, which is more reliable than the practice's own record when a payer forwards a claim onwards without being asked.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The chosen label is the input to five other decisions: the derived payer type in [BR-G2](#br-g2-the-ledger-payer-type-is-cut-out-of-a-user-interface-label), the primary-only adjustment rule in [BR-F1](#br-f1-non-primary-insurance-adjustments-are-recorded-as-notes-worth-zero), the patient-statement relabelling in [BR-C4](#br-c4-patient-responsibility-reasons-are-relabelled-to-fit-a-patient-statement), the level written to the encounter watermark, and whether secondary billing is set up at all. Because the default arm of the match credits the first insurance, an unrecognised or absent status code posts against the primary coverage silently.

```php
'2', '20' => 'Ins2',
'3', '21' => 'Ins3',
default => 'Ins1'
```

Those are the three arms at `interface/billing/sl_eob_process.php:L349-L351`, inside the match opened at `interface/billing/sl_eob_process.php:L348`. The arms compare strings, so a numerically equal but differently formatted code, such as a zero-padded `02`, falls to the default arm.

### BR-C3 Four different status codes all mean forwarded to another payer

- **Statement:** Four of the seventeen claim status codes tell the practice that the payer sent the claim onwards to another payer without being asked: the three that report adjudication at a level and then forwarding, and one that disclaims the claim entirely and forwards it. Only two of those four are recognised by the level-selection rule.
- **Evidence:** `src/Billing/BillingUtilities.php:L33-L35` and `src/Billing/BillingUtilities.php:L37` carry the four descriptions. The level-selection match at `interface/billing/sl_eob_process.php:L348-L352` recognises two of them, 20 and 21, and does not recognise 19 or 23.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the two recognised codes were added to the match because they pair with the two non-primary levels, and the other two were left out because 19 already means primary and 23 has no level to credit. Basis: the match arms pair 2 with 20 and 3 with 21 exactly, which is the pairing the code descriptions imply, and the default arm handles 19 correctly by accident of it meaning primary.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Status 23 disclaims the claim and forwards it, yet it is posted against the primary insurance by the default arm, which credits a payer that has just said the claim is not theirs. Forwarding is also detected structurally, from a segment rather than from the status code, so the two mechanisms can disagree about whether a claim was forwarded, and it is the segment that decides whether the encounter is marked as billed onwards.

VERIFIED: forwarding is also signalled structurally, by a name segment whose entity identifier is the two-character crossover code, at `src/Billing/ParseERA.php:L311-L312`. VERIFIED: a corrected policy number is signalled by a different entity identifier on the same segment, at `src/Billing/ParseERA.php:L313-L315`. These are segment-level signals rather than status codes, and the subsystem uses the segment signal rather than the status code when deciding whether to close a level, at `interface/billing/sl_eob_process.php:L700`.

### BR-C4 Patient-responsibility reasons are relabelled to fit a patient statement

- **Statement:** When a primary payer reports an amount as the patient's responsibility, the payer's reason code is replaced with a short label naming the insurance level and the kind of responsibility. Deductible, coinsurance and copay each get their own wording and everything else gets a generic one. The labels exist to fit inside a length limit on a patient statement.
- **Evidence:** `interface/billing/sl_eob_process.php:L606-L611`, entered from the condition at `interface/billing/sl_eob_process.php:L601` and the primary test at `interface/billing/sl_eob_process.php:L604`. The length limit is stated in the comments at `interface/billing/sl_eob_process.php:L593-L594` and again at `interface/billing/sl_eob_process.php:L605`.
- **Status:** VERIFIED
- **Intent:** The comment states it directly: posted adjustment reasons must be 25 characters or fewer to fit on patient statements. Read as evidence of intent, that makes the relabelling a presentation constraint that has been pushed back into the data, because the label is what is stored on the ledger line rather than what is rendered from it.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The statement template that imposes the limit is site-editable and is named as a boundary in [README.md](README.md).
- **Blast radius:** The label is persisted rather than derived, so the payer's own reason code is not recoverable from the ledger line for these adjustments; a report that wanted to group patient responsibility by reason has only the label. The labels are also built by string concatenation with the insurance label, so the 25-character budget is shared between the level name and the reason wording, and lengthening either can overflow the statement.

```php
$reason = match ($adj['reason_code']) {
    '1' => "$inslabel dedbl: ",
```

That is `interface/billing/sl_eob_process.php:L606-L607`. Note that this branch runs only for a primary payer; the non-primary case is [BR-F1](#br-f1-non-primary-insurance-adjustments-are-recorded-as-notes-worth-zero).

### BR-C5 A contractual write-off of reason 45 or 59 is exempt from the zero-payment error

- **Statement:** A service line that the payer adjusted without paying anything is normally treated as an error that blocks posting. Two adjustment reason codes are exempt when the payer marked them as a contractual obligation, and a line carrying one of them posts with no payment and no error.
- **Evidence:** The exemption is computed at `interface/billing/sl_eob_process.php:L599-L600` and applied at `interface/billing/sl_eob_process.php:L636-L641`, where the error is raised at `interface/billing/sl_eob_process.php:L640-L641`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the two exempt codes describe charges written off because they exceed the contracted amount and because they are bundled into another service, both of which legitimately pay nothing, so the exemption distinguishes a lawful zero payment from a suspicious one. Basis: the two reason codes are named in the compiled reason table registered in [BR-C1](#br-c1-three-hardcoded-tables-carry-1391-x12-code-descriptions), and their descriptions there match that reading; no comment states the reason.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The exempt list is a literal of exactly two codes compared strictly. Any other reason code that legitimately pays nothing raises the error and blocks the whole claim from posting, so the operator sees a posting refused rather than a write-off recorded. Widening the list allows more zero-payment lines to post unremarked, which is the same silence in the other direction.

```php
$isContractualWriteoff = $adj['group_code'] === 'CO'
    && in_array($adj['reason_code'], ['45', '59'], true);
```

That is `interface/billing/sl_eob_process.php:L599-L600`. Both comparisons are strict, and the group-code test uses the two-character contractual-obligation code.

### BR-C6 A denial is status 7 and is recorded unlike any other status

- **Statement:** Internal claim status 7 means the claim was denied, and it is the only status handled by its own branch at every step of the claim update. A denial writes the status value into the queue's process column, stores the denial reason in the claim's process-file column, and deliberately does not set the billed flag or the billed date that every other advancing status sets.
- **Evidence:** Three separate branches, each testing the same value: `src/Billing/BillingUtilities.php:L1586-L1588`, `src/Billing/BillingUtilities.php:L1602-L1604` and `src/Billing/BillingUtilities.php:L1612-L1614`. The behaviour it bypasses is at `src/Billing/BillingUtilities.php:L1592-L1596`.
- **Status:** VERIFIED
- **Intent:** To record a denial without marking the claim as successfully billed, so the claim remains actionable. The comment repeated beside all three branches names the denial case, and the comment at `src/Billing/BillingUtilities.php:L1613` names the process-file column as the denial reason store.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The denial branch reuses the status value as the queue process value, so a column that otherwise holds a billing-process selection holds a status code for denied claims only. Anything that reads that column as a process selection misreads denied claims. Because the denial branch is a peer of the general branch rather than a case inside it, adding a status above 7 requires deciding again whether it advances the claim; the general branch treats every status above 1 as billed, at `src/Billing/BillingUtilities.php:L1592-L1593`, and only status 2 stamps the billed date, at `src/Billing/BillingUtilities.php:L1594-L1596`.

The status vocabulary itself is declared as constants at `src/Billing/BillingProcessor/BillingClaim.php:L22-L24`, and the queue's process vocabulary at `src/Billing/BillingProcessor/BillingClaim.php:L26-L29`. The stage in which these values change is described in [claim-lifecycle.md](claim-lifecycle.md).

## Group D Payer and Partner Specific Branches

Eight rules. A trading partner is the clearinghouse or payer connection a claim is sent through, and its configuration is documented column by column in [transactions.md](transactions.md). This group registers the decisions that route a claim or a remittance to a particular payer, not the configuration surface itself.

### BR-D1 The payer level travels as a single character on the front of the payer identifier

- **Statement:** When the billing screen posts a claim for processing, the payer identifier arrives with a single letter fixed to the front of it that names the insurance level. The letter is read off, uppercased, and mapped to primary, secondary or tertiary; the rest of the string is the payer identifier.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaim.php:L121-L134`, with the identifier taken at `src/Billing/BillingProcessor/BillingClaim.php:L122` and the level letter taken at `src/Billing/BillingProcessor/BillingClaim.php:L125`. The three constants are at `src/Billing/BillingProcessor/BillingClaim.php:L76-L78`.
- **Status:** VERIFIED
- **Intent:** To carry two facts in one form field. The comment at `src/Billing/BillingProcessor/BillingClaim.php:L108-L109` describes the posted shape as cryptic and names the constructor as the place that decodes it, which is an unusually candid statement of the encoding's cost.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The identifier is produced by taking everything after the first character, so a payer identifier that ever arrives without a prefix loses its first character silently and the claim is sent to a payer identifier one digit short. Because the level letter is uppercased before comparison but the identifier is not, the two halves of the same field are normalised differently.

```php
$this->payor_id = substr((string) $partner_and_payor['payer'], 1);
$payor_type_char = substr(strtoupper((string) $partner_and_payor['payer']), 0, 1);
```

That is `src/Billing/BillingProcessor/BillingClaim.php:L122-L125`, with the intervening comment omitted from the excerpt.

### BR-D2 An unrecognised payer prefix becomes payer type zero

- **Statement:** If the level letter is not one of the three recognised values, the claim is given a payer type of zero rather than being rejected. Zero is declared to mean unknown.
- **Evidence:** The fallback is at `src/Billing/BillingProcessor/BillingClaim.php:L132-L134`, and the value it assigns is declared at `src/Billing/BillingProcessor/BillingClaim.php:L79`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): a total order of primary, secondary and tertiary needs a value for absent, and zero was chosen because the three real levels are one, two and three. Basis: the four constants are declared together under one comment naming them as the options for the payer type, and no other value is used anywhere in the class.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Zero is not an inert sentinel elsewhere in this subsystem. The accounts-receivable ledger declares zero to mean the patient, which is registered as [BR-G3](#br-g3-payer-type-zero-means-two-incompatible-things). A claim whose level letter was unrecognised therefore carries a value that one component reads as unknown and another reads as a patient responsibility, and no error is raised at the point of substitution.

### BR-D3 The processing format is read per claim, stored, and branches on nothing

- **Statement:** Each claim reads its trading partner's processing format from the database during construction and keeps it as the claim's target. The claim's own documentation states that this value does not affect the output format and exists only to record what was selected and to store it with the claim. No branch on the value was found in the generators.
- **Evidence:** The read is at `src/Billing/BillingProcessor/BillingClaim.php:L136-L142`; the enumeration of six permitted values is declared at `sql/database.sql:L10031`; the statement about its effect is the docblock at `src/Billing/BillingProcessor/BillingClaim.php:L81-L89`.
- **Status:** INFERRED (confidence: Medium)
- **Intent:** Both readings are recorded, because the evidence points two ways and the source-of-truth ordering in [README.md](README.md) forbids resolving that by preferring the comment. VERIFIED: the value is read once per claim, defaulted to the empty string when the partner row has none, and retained. INFERRED (confidence: Medium): it is a record of the operator's selection rather than a routing key. Basis for the confidence being Medium rather than High: the only direct statement about its purpose is a docblock, which this documentation set treats as evidence of intent and never of behaviour, and the docblock itself is hedged; the claim that nothing branches on it rests on a search of the generators finding no consumer, which is negative evidence. Any description of this column as a routing key is unverified.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The contested purpose of this column is also discussed from the configuration side in [transactions.md](transactions.md).
- **Blast radius:** Two of the six permitted values describe eligibility rather than claims, so the enumeration mixes two axes in one column. If the value ever did become a routing key, every existing partner row would begin routing on a selection that was made when it had no effect, and the four claim-flavoured values would silently start to matter.

```php
$sql = "SELECT x.processing_format from x12_partners as x where x.id =?";
```

That is `src/Billing/BillingProcessor/BillingClaim.php:L137`. One query per claim, executed inside the constructor.

### BR-D4 A claim with no trading partner is dropped from an electronic run only

- **Statement:** A claim whose trading partner is unassigned is skipped, with a message to the screen, but only when the run is an electronic claim run. In any other kind of run the same claim proceeds with an unassigned partner.
- **Evidence:** `src/Billing/BillingProcessor/BillingProcessor.php:L113-L117`, whose condition requires both the sentinel partner value and the electronic-run flag read at `src/Billing/BillingProcessor/BillingProcessor.php:L100`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): a paper claim or a mark-as-cleared action does not need a trading partner, so the check is scoped to the only runs that transmit. Basis: the flag it tests is set by the electronic generator branches at `src/Billing/BillingProcessor/BillingProcessor.php:L166`, `src/Billing/BillingProcessor/BillingProcessor.php:L169`, `src/Billing/BillingProcessor/BillingProcessor.php:L172` and `src/Billing/BillingProcessor/BillingProcessor.php:L175`, and by none of the paper branches.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The sentinel is the literal minus one compared loosely, so a partner identifier that arrives as the string form of minus one and one that arrives as the integer both match, but a null or empty partner does not match either and is not skipped. The message is printed to the screen and not recorded, so a claim dropped this way leaves no trace in any log; the operator-visible consequences are set out per stage in [claim-lifecycle.md](claim-lifecycle.md).

### BR-D5 The claim identifier is recovered from the remittance by counting its parts

- **Statement:** The practice sends a claim identifier made of the patient number and the encounter number. Payers return it mangled, so the subsystem recovers the two numbers by splitting on space or hyphen and branching on how many parts came back: two parts are taken as patient and encounter directly, three parts cause the encounter to be looked up in the charge queue, and one part is matched by scanning patients whose name matches and testing whether the returned string starts with that patient's number.
- **Evidence:** `src/Billing/SLEOB.php:L28-L64`, with the split at `src/Billing/SLEOB.php:L31` and the three branches at `src/Billing/SLEOB.php:L36`, `src/Billing/SLEOB.php:L39` and `src/Billing/SLEOB.php:L45`.
- **Status:** VERIFIED
- **Intent:** The comment above the method states it: the recovery should be straightforward except that some payers mangle the identifier the practice supplied. Read as evidence of intent, the three branches are three observed mangling patterns rather than a designed protocol.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** This method decides which patient's ledger a payer's money lands on, from a string the payer controls. The one-part branch is a prefix match over patients ordered by descending patient number, so it selects the highest-numbered matching patient whose number is a prefix of the returned string, and a practice with two patients of the same name can have money posted to the wrong one. The three-part branch builds its query by interpolating the recovered patient number directly into the SQL text at `src/Billing/SLEOB.php:L41-L42` while binding the other parameter; that construction is flagged in the security appendix of [defect-candidates.md](defect-candidates.md) and is deliberately not analysed here.

VERIFIED: when either number is missing the method returns them as zero and returns the payer's string unchanged, at `src/Billing/SLEOB.php:L59-L63`, so a failure to recover produces a claim identifier that matches no invoice rather than an error.

### BR-D6 A secondary payer that drops a modifier is matched by rebuilding the key

- **Statement:** A service line is matched to an existing charge by a key made of the procedure code and, where present, a colon and the modifier. If no charge matches and the payer sent no modifier, the subsystem rebuilds the key using the modifiers recorded on the practice's own invoice for that code and tries again.
- **Evidence:** The key is built at `interface/billing/sl_eob_process.php:L453-L456`; the rebuild is at `interface/billing/sl_eob_process.php:L461-L468`, entered from the condition at `interface/billing/sl_eob_process.php:L461`.
- **Status:** VERIFIED
- **Intent:** The comment at `interface/billing/sl_eob_process.php:L459-L460` states it: a secondary insurer sometimes omits a modifier that the primary payer processed, so the match is retried against the invoice's own modifiers.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The rebuild joins every modifier recorded for that code with colons, so a charge carrying two modifiers is matched by a three-part key and a charge carrying one by a two-part key. The loop that builds it iterates the whole code list and overwrites the key on each match, so if a code appears more than once the last occurrence wins. A failure to match here does not silently mismatch: it leaves the previous charge empty, and the sanity check registered as [BR-A9](#br-a9-a-charge-that-disagrees-with-the-invoice-by-one-cent-blocks-the-posting) then compares against nothing.

### BR-D7 The application sender code falls back to the interchange sender identifier

- **Statement:** The envelope carries two sender identities, one on the interchange header and one on the functional group header. If the partner row leaves the functional-group sender empty, the interchange sender is used for both. The receiver side has the mirror-image rule.
- **Evidence:** `src/Billing/Claim.php:L713-L721` for the sender fallback; `src/Billing/Claim.php:L646-L650` for the receiver fallback, whose reasoning is stated in the docblock at `src/Billing/Claim.php:L640-L645`.
- **Status:** VERIFIED
- **Intent:** The receiver-side docblock states it: the two are usually the same, but some clearinghouses require them to differ, so an explicit value overrides and an empty one inherits. The sender side implements the same pattern without a comment.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The columns involved are catalogued in the trading-partner reference in [transactions.md](transactions.md).
- **Blast radius:** The sender fallback tests for the empty string strictly, so a partner row holding a single space passes the test and is sent as a space; the receiver fallback tests with `empty()`, which also treats the string zero as absent. The two fallbacks in the same envelope therefore use different definitions of missing, and a partner configured with a literal zero sender identifier behaves differently on the two headers.

### BR-D8 Eligibility uses whichever primary coverage row the database returns first

- **Statement:** When an eligibility response is recorded, the patient's primary coverage row is looked up with no date filter and no row limit, so the coverage the response is attached to is whichever row the database happens to return first. The copay on that row is read and then not used, and the response record stores neither a copay nor a deductible.
- **Evidence:** The lookup is at `src/Billing/EDI270.php:L686-L691`, with the unused copay assigned at `src/Billing/EDI270.php:L690`. The record written at `src/Billing/EDI270.php:L701` sets only the verification identifier, the coverage identifier, the responder identifier and two timestamps.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the copay read is a remnant of an intended capture of the payer's reported copay that was never completed, since the two columns that would receive it exist in the schema and are never written. Basis: the value is selected, assigned to a local variable and never referenced again, and the target columns at `sql/database.sql:L1652-L1653` are unwritten by any statement in the subsystem.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** A patient with more than one primary coverage row, which the schema permits because there is no uniqueness constraint on the coverage type, gets an eligibility record attached to an arbitrary one, and the record cannot be traced back to the coverage the payer actually answered about. The benefits detail is refreshed by deleting every row for the verification and re-inserting, at `src/Billing/EDI270.php:L710-L711`, so a failure between the delete and the insert leaves the verification with no benefits rather than with the previous ones.

```php
$query = "SELECT id, copay FROM insurance_data WHERE type = 'primary' and pid = ?";
$insId = sqlQuery($query, [$patient_id]);
```

That is `src/Billing/EDI270.php:L686-L687`. Contrast the coverage lookup used by the accounts-receivable path, which does filter on the service date, registered as [BR-E1](#br-e1-a-coverage-row-with-no-start-date-is-effective-for-every-service-date).

## Group E Date Boundary Logic

Nine rules. Dates decide which coverage was in force, which century a claim belongs to, and which ledger lines a void reverses. Every rule here is a boundary condition, and several of them treat an absent date as permissive rather than as an error.

### BR-E1 A coverage row with no start date is effective for every service date

- **Statement:** The payer for a given insurance level on a given date of service is found by looking for a coverage row of that level whose start date is on or before the service date and whose end date is on or after it. A row with no start date matches any service date, and a row with no end date matches any service date, because each half of the predicate accepts a null. Where more than one row matches, the one with the latest start date wins.
- **Evidence:** `src/Billing/SLEOB.php:L258-L260`, executed at `src/Billing/SLEOB.php:L261`. The level names are mapped at `src/Billing/SLEOB.php:L256` and the level itself is bounds-checked at `src/Billing/SLEOB.php:L252-L254`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the null tolerance exists because coverage dates are optional on the patient's insurance screen, and a practice that has not recorded them still needs its claims routed. Basis: both sides of the predicate are written with an explicit null alternative rather than with a coalesce or a default, which is the shape a developer writes when the column is known to be frequently empty.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** This function decides which payer a claim is billed to and which payer a remittance is credited to, so a stale coverage row left with a null start date shadows a correctly dated one whenever its own start date sorts later, and there is no message when it does. Adding a start date to an existing row can therefore change which payer future claims go to without any other change. The same function returns zero for a level outside one to three, which is the mechanism behind the tertiary boundary in [BR-E2](#br-e2-the-service-date-is-the-first-ten-characters-of-the-encounter-timestamp).

```php
"pid = ? AND type = ? AND (date <= ? OR date IS NULL) AND (date_end >= ? OR date_end IS NULL) " .
"ORDER BY date DESC LIMIT 1";
```

That is `src/Billing/SLEOB.php:L259-L260`. The same shape of predicate is used by the copay lookup registered as [BR-E9](#br-e9-the-copay-in-force-is-the-one-on-the-latest-starting-primary-coverage-row).

### BR-E2 The service date is the first ten characters of the encounter timestamp

- **Statement:** When the subsystem needs a date of service in order to find the next payer, it takes the first ten characters of the encounter's stored timestamp rather than converting it. The next insurance level is then derived by advancing the encounter's last-billed watermark, and a level above three yields no payer because the level map has only three entries.
- **Evidence:** `src/Billing/SLEOB.php:L284`, with the level advance at `src/Billing/SLEOB.php:L285-L288` and the payer lookup at `src/Billing/SLEOB.php:L290`. The bound is enforced inside the lookup at `src/Billing/SLEOB.php:L252-L254`.
- **Status:** VERIFIED
- **Intent:** To reduce a datetime to a date without a conversion, which is safe as long as the stored form is a standard datetime string. The surrounding function is documented at `src/Billing/SLEOB.php:L269-L270` as making the invoice re-billable.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The advance condition is a mixture of comparisons that lets the level reach three but never four, so when a secondary payer has been billed and closed there is a next level and when a tertiary payer has been billed there is not. The claim is then reopened rather than queued, at `src/Billing/SLEOB.php:L297-L302`, and reopening is silent: nothing is written that says a tertiary payer exists and was not billed. The consequences of that terminal state are carried in [defect-candidates.md](defect-candidates.md), and the state itself appears in the claim-status diagram in [claim-lifecycle.md](claim-lifecycle.md).

```php
$date_of_service = substr((string) $ferow['date'], 0, 10);
$new_payer_type = 0 + $ferow['last_level_billed'];
```

That is `src/Billing/SLEOB.php:L284-L285`. The watermark is coerced to a number by addition rather than by a cast, so a null watermark becomes zero and then advances to one.

### BR-E3 The institutional date converter hardcodes the century

- **Statement:** The institutional claim generator converts a six-character date in month, day, two-digit-year order into an eight-character date by moving the parts around and fixing the literal characters `20` to the front. The century is not derived from anything.
- **Evidence:** `src/Billing/X125010837I.php:L19-L22`, a single expression at `src/Billing/X125010837I.php:L21`. The behaviour is asserted by tests at `tests/Tests/Isolated/Billing/X125010837IDateTest.php:L25` and `tests/Tests/Isolated/Billing/X125010837IDateTest.php:L31`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the two-digit year arrives from a form field that cannot express a century, and the twenty-first century was assumed as the only one a claim would be filed in. Basis: the function is the only date converter in the institutional generator, its input is described by the tests as month, day and two-digit year, and no century parameter exists anywhere in the call chain.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Any institutional claim carrying a twentieth-century date, which a patient's birth date would be for most adults, is emitted with the wrong century if it passes through this converter. Changing the literal to a computed century would change every date this function produces, which is why the two tests that pin its output exist; they run under the isolated configuration described in [upgrade-risk-map.md](upgrade-risk-map.md).

```php
return ('20' . substr((string) $frmdate, 4, 2) . substr((string) $frmdate, 0, 2) . substr((string) $frmdate, 2, 2));
```

That is `src/Billing/X125010837I.php:L21`.

### BR-E4 An empty institutional date becomes the two-character string 20

- **Statement:** Given an empty input, the institutional date converter returns the two characters of the hardcoded century and nothing else. It does not return an empty string and does not raise anything.
- **Evidence:** The behaviour follows from `src/Billing/X125010837I.php:L21` and is asserted directly at `tests/Tests/Isolated/Billing/X125010837IDateTest.php:L34-L39`, with the assertion at `tests/Tests/Isolated/Billing/X125010837IDateTest.php:L38`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): this is not an intended behaviour but a consequence of concatenating three substring results that are each empty, and the test exists to pin the consequence rather than to endorse it. Basis: the test is named for a partial output rather than for a valid case, and its comment at `tests/Tests/Isolated/Billing/X125010837IDateTest.php:L36` explains the mechanism rather than the requirement.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** A two-character value emitted into a date element produces a claim a payer will reject, and the rejection arrives as an acknowledgement rather than at generation time, so the operator learns about it from the clearinghouse rather than from the screen. Because a test pins this output, any change that makes the converter reject an empty input will fail that test, which is the correct place for the decision to be made.

### BR-E5 Every envelope and transaction date is server-local

- **Statement:** Every date and time written into an outbound claim envelope is taken from the server clock in the server's local timezone, with no conversion and no zone marker. This applies to the functional group headers of both generators, the transaction headers of both, and the interchange header of the institutional generator.
- **Evidence:** `src/Billing/X125010837P.php:L112-L113` for the professional transaction header and `src/Billing/X125010837P.php:L83-L84` for its group header; `src/Billing/X125010837I.php:L87-L88` and `src/Billing/X125010837I.php:L68-L69` for the institutional equivalents; `src/Billing/X125010837I.php:L54-L56` for the institutional interchange header. The batch substitutes its own values from the same clock, at `src/Billing/BillingProcessor/BillingClaimBatch.php:L216`.
- **Status:** VERIFIED
- **Intent:** The format's date and time elements carry no timezone, so a local time is what the format expects; sending the server's own local time is the only interpretation available without a configured practice timezone.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** A payer or clearinghouse in a different timezone reads these as its own local time, so a batch transmitted late in the evening can be dated a day earlier or later than the payer's own record of receiving it, which matters wherever a filing deadline is counted in days. Because the values are produced from one timestamp captured once per file, every claim in a batch shares them, so the discrepancy is uniform rather than per claim.

### BR-E6 The professional interchange date is a placeholder the batch replaces

- **Statement:** The professional generator writes two fixed literal values where the interchange date and time belong, and relies on the batch writer to substitute the real ones. The institutional generator writes the real values itself. Both generators are only ever called through the batch, so both interchange headers reach a payer carrying the batch's timestamp.
- **Evidence:** The placeholders are at `src/Billing/X125010837P.php:L69-L70`, each carrying a comment naming the file that was expected to replace them. The institutional generator emits real values at the same positions, at `src/Billing/X125010837I.php:L54-L56`. The substitution is at `src/Billing/BillingProcessor/BillingClaimBatch.php:L216-L217`, which preserves the first seventy characters of the interchange header and writes the batch date and time after them. The only two callers of the professional generator are `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L70` and `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php:L241`, both of which append through the batch.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the placeholders date from a time when the generator's output was written directly and a separate script post-processed it, and the comments beside them name that script. Basis: the comments name a file that is no longer the writer, and the batch that now performs the substitution reproduces exactly the two elements the comments say would be replaced.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The substitution depends on the interchange header being exactly the expected width, because it copies a fixed seventy characters and then writes its own elements. That width dependency is enforced by the length check registered as [BR-F4](#br-f4-a-malformed-interchange-header-terminates-the-whole-batch-run), and the two rules must be changed together: widening any element before the date silently shifts what is preserved, and the length check is the only thing that catches it.

### BR-E7 The operator's pay date overrides the payer's own dates

- **Statement:** On the posting screen, if the operator entered a pay date, that date replaces both the payer's cheque date and the payer's production date for every claim in the file. Only when the operator left it blank are the payer's own dates used.
- **Evidence:** `interface/billing/sl_eob_process.php:L425-L426`, both expressions using the operator's value in preference to the parsed one.
- **Status:** VERIFIED
- **Intent:** The operator help explains why the field exists at all, at `Documentation/help_files/sl_eob_help.php:L100`: the source and pay date are entered once so they need not be re-entered per claim. Read as evidence of intent, the field was designed for manual posting, where there is no payer-supplied date to override.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The help file documents the field; it does not state that the field overrides a remittance file's own dates.
- **Blast radius:** The override is applied with a falsy test, so an empty field falls through correctly but a field holding a zero would also fall through. When a remittance file is posted with the field filled, the deposit and every ledger line carry the operator's date and the payer's dates are not retained anywhere, so the practice loses the ability to reconcile against the payer's own reporting date. This interacts with the backfill in [BR-B5](#br-b5-the-production-date-is-backfilled-from-the-check-date): if both dates were already equal because of the backfill, the override makes them equal to a third value.

### BR-E8 A void matches ledger lines by exact posting timestamp

- **Statement:** Voiding a checkout finds the ledger lines to reverse by matching the posting timestamp exactly, and reverses only lines that have not already been soft-deleted. The same exact-timestamp match is used to reopen the charges that were billed at that moment.
- **Evidence:** The read is at `src/Billing/BillingUtilities.php:L1873-L1882`, with the predicate at `src/Billing/BillingUtilities.php:L1877`. The soft delete is at `src/Billing/BillingUtilities.php:L1928-L1933` and the charge reopen at `src/Billing/BillingUtilities.php:L1934-L1939`.
- **Status:** VERIFIED
- **Intent:** To group the lines written by one checkout without storing a checkout identifier, using the shared timestamp as an implicit batch key.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The timestamp is the only grouping key, so two checkouts recorded in the same second for the same patient and encounter are indistinguishable and are voided together. Conversely, any line whose timestamp differs by a second from the rest of its checkout is left behind by the void, and the deposit-balancing check registered as [BR-B7](#br-b7-a-deposit-is-balanced-against-live-ledger-lines-only-and-only-in-a-browser-alert) will then report that deposit as short. The charge reopen additionally requires the billed date to be non-null and equal, so a charge billed at a different moment in the same checkout is not reopened.

### BR-E9 The copay in force is the one on the latest starting primary coverage row

- **Statement:** The copay effective on a given date is read from the primary coverage row whose date window contains that date, choosing the latest starting row when more than one matches, and only when that row names a provider. Absent such a row the answer is zero.
- **Evidence:** `src/Billing/BillingUtilities.php:L1769-L1778`, with the window predicate at `src/Billing/BillingUtilities.php:L1773`, the provider requirement at `src/Billing/BillingUtilities.php:L1774` and the fallback at `src/Billing/BillingUtilities.php:L1778`.
- **Status:** VERIFIED
- **Intent:** The comment above the method, at `src/Billing/BillingUtilities.php:L1766-L1767`, states the intent as returning the copay effective on the given date, or a negative sentinel when there is no insurance on that date. VERIFIED: the code returns zero rather than the negative sentinel, so the comment describes an intent the code does not implement.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Zero and minus one mean different things to a caller: zero is a valid copay and minus one is an absence. Any caller that distinguishes them cannot, and a patient with no coverage on the service date is treated as having a copay of nothing rather than as having no coverage. The predicate is the same null-tolerant window as [BR-E1](#br-e1-a-coverage-row-with-no-start-date-is-effective-for-every-service-date), so the same shadowing by undated rows applies, and the stored value is free text per [BR-A3](#br-a3-two-coverage-money-fields-escape-the-fixed-point-convention).

## Group F Behaviour That Would Silently Change Amounts

Eleven rules. This is the group the documentation requirements placed last in the priority list and it is the one with the most entries, because a decision that changes a dollar amount without telling anyone is the hardest kind of rule to recover: there is no message to search for and no screen that shows it. Every entry here names what an operator would see, and in several cases the answer is nothing at all.

### BR-F1 Non-primary insurance adjustments are recorded as notes worth zero

- **Statement:** When a payer other than the primary reports an adjustment, the amount is not posted. A ledger line is still written so that the note survives, and the amount on that line is the literal zero. The payer's amount appears inside the note text as a formatted number and nowhere else.
- **Evidence:** `interface/billing/sl_eob_process.php:L612-L631`. The note is built at `interface/billing/sl_eob_process.php:L616`, the payer's amount is appended to the note text at `interface/billing/sl_eob_process.php:L619`, and the ledger line is written by the call at `interface/billing/sl_eob_process.php:L622-L631` whose amount argument is the literal zero at `interface/billing/sl_eob_process.php:L626`.
- **Status:** VERIFIED
- **Intent:** The comment states it in unusually blunt terms at `interface/billing/sl_eob_process.php:L613-L615`: non-primary adjustments are held to be worthless, either repeating the primary payer's adjustments or not being adjustments at all, so they are reported as notes without posting any amount. The follow-up comment at `interface/billing/sl_eob_process.php:L620` states that the zero-dollar line exists only to preserve the comment.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. This is the rule whose omission would most clearly fail the requirement that this register contain what no existing document contains, and it is the rule that would most surprise an engineer reading only the schema.
- **Blast radius:** A secondary or tertiary payer's contractual write-off never reduces the receivable, so a claim adjudicated by two payers keeps a balance the second payer considers written off, and the operator sees a note rather than an adjustment. The condition that routes into this branch is the insurance-level derivation registered as [BR-C2](#br-c2-the-payers-own-status-code-decides-which-insurance-level-is-credited), so a primary remittance mislabelled as secondary by the payer's own status code lands here too. Reversing the rule would post amounts that the code's author believed were duplicates, which would double-count the primary payer's adjustments.

```php
amount: 0,
code: $codekey,
```

That is `interface/billing/sl_eob_process.php:L626-L627`, inside the posting call. The zero is a literal, not a computed value.

### BR-F2 A code whose colon is its first character keeps its modifier

- **Statement:** All three posting helpers split a procedure key into a code and a modifier at the first colon, and each of them tests the found position for truth rather than for presence. A key whose colon is at the very start is therefore treated as having no modifier, and the whole string including the colon is stored as the code.
- **Evidence:** Three identical implementations: `src/Billing/SLEOB.php:L136-L140`, `src/Billing/SLEOB.php:L188-L192` and `src/Billing/SLEOB.php:L227-L231`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the truth test is a shorthand for a presence test that happens to be correct for every real procedure code, because no procedure code is empty. Basis: the same three-line idiom appears three times unchanged, which is characteristic of a copied fragment rather than of a deliberate boundary decision, and no comment mentions the boundary.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The key is built by concatenating the payer's code with the payer's modifier, at `interface/billing/sl_eob_process.php:L453-L456`, so a remittance that reports an empty code with a modifier produces a key beginning with a colon. That key is then stored whole in the code column, and the modifier is lost, so a later match against the same charge fails and the payment is posted against a code that matches nothing. The condition is registered here as a rule because the split is the rule; the boundary is carried as a defect entry in [defect-candidates.md](defect-candidates.md).

```php
$tmp = strpos((string) $code, ':');
if ($tmp) {
```

That is `src/Billing/SLEOB.php:L136-L137`, and the same two lines appear at `src/Billing/SLEOB.php:L188-L189` and `src/Billing/SLEOB.php:L227-L228`.

### BR-F3 A failed transmission is recorded as a success

- **Statement:** The transport that uploads a batch file to a trading partner records a status for every outcome. Every failure before the upload records its status and moves to the next partner. The upload failure itself records its status and then falls through to the line that records success, so the final stored status of a failed upload is success.
- **Evidence:** The upload-failure branch is `src/Billing/BillingProcessor/X12RemoteTracker.php:L112-L117`, and it is the only error branch with no continuation. The unconditional success write is at `src/Billing/BillingProcessor/X12RemoteTracker.php:L119-L121`. The four branches that do continue are at `src/Billing/BillingProcessor/X12RemoteTracker.php:L70`, `src/Billing/BillingProcessor/X12RemoteTracker.php:L85`, `src/Billing/BillingProcessor/X12RemoteTracker.php:L96` and `src/Billing/BillingProcessor/X12RemoteTracker.php:L104`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the omission is an oversight rather than a policy, and the comment above the success write is itself stale, describing a transition to a different status than the one written. Basis: the four sibling error branches all continue, the status constant reserved for an upload error exists and is written immediately before being overwritten, and that constant's name is misspelled at `src/Billing/BillingProcessor/X12RemoteTracker.php:L30`, which is evidence that it is little exercised.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** An undelivered claim batch is displayed as delivered, so nobody resends it and the claims age silently until the payer's filing deadline passes. The failure messages are preserved on the record, so the evidence exists but the status that anyone would filter on does not reflect it. This entry is a rule about how transport outcome is recorded; the defect is carried in [defect-candidates.md](defect-candidates.md), and the transport stage is described in [claim-lifecycle.md](claim-lifecycle.md).

VERIFIED: the transport also falls back to a second location when the configured local file is absent, at `src/Billing/BillingProcessor/X12RemoteTracker.php:L75-L78`, and then reads the file without rechecking existence, at `src/Billing/BillingProcessor/X12RemoteTracker.php:L80`.

### BR-F4 A malformed interchange header terminates the whole batch run

- **Statement:** While assembling a batch, the writer rebuilds the interchange header and then checks that the result is exactly 105 characters. If it is not, the request is terminated immediately. The same happens if the first segment of a claim is not an interchange header. Neither case is an exception that a caller could handle.
- **Evidence:** The length check and termination are at `src/Billing/BillingProcessor/BillingClaimBatch.php:L219-L222`; the leading-segment check and termination are at `src/Billing/BillingProcessor/BillingClaimBatch.php:L225-L227`. The batch file is opened in append mode and written at `src/Billing/BillingProcessor/BillingClaimBatch.php:L159-L162`.
- **Status:** VERIFIED
- **Intent:** To refuse to transmit an envelope that a payer would reject, since the interchange header is fixed-width in this version of the format and a wrong width means an element is malformed.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The termination happens part-way through a run over many claims, so the claims already appended are in memory and the claims not yet reached are untouched, while the operator sees a truncated page ending in the message. Because the batch file is opened in append mode, a partially written file from an earlier successful write remains on disk and the next run appends to it. The width the check enforces is what the header rebuild in [BR-E6](#br-e6-the-professional-interchange-date-is-a-placeholder-the-batch-replaces) depends on, so the two are one mechanism.

### BR-F5 One batch file is queued once for every partner in the batch

- **Statement:** When automatic transmission is enabled, the single batch file just written is queued for upload once for each distinct trading partner among the claims it contains. The queued record names the same filename every time and carries a serialisation of every claim in the batch, not just that partner's claims.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L170-L186`, with the distinct-partner extraction at `src/Billing/BillingProcessor/BillingClaimBatch.php:L174` and the per-partner queue write at `src/Billing/BillingProcessor/BillingClaimBatch.php:L177-L184`. The claims payload is serialised at `src/Billing/BillingProcessor/BillingClaimBatch.php:L182`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the automatic upload was written for the common case of one partner per batch, and the loop over distinct partners was added for correctness of the queue rather than for correctness of the content. Basis: the comment at `src/Billing/BillingProcessor/BillingClaimBatch.php:L176` says the file is queued to all partners, which describes the loop faithfully and shows no awareness that the file contains other partners' claims.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** In a batch spanning two partners, each partner receives a file containing the other's claims, which is a disclosure of claim data to a party that has no relationship with those patients, and it happens with no message to the operator. Whether it happens at all is decided by the site global registered as [BR-I4](#br-i4-one-switch-decides-whether-anything-is-transmitted-at-all). The queue write is skipped entirely when the local file write failed, because the guard at `src/Billing/BillingProcessor/BillingClaimBatch.php:L171` requires success strictly.

### BR-F6 The attachment segment is emitted without being counted

- **Statement:** A professional claim that the practice marked as employment-related emits a paperwork segment announcing an electronic attachment, and that segment consumes a value from the control-number sequence. It does not increment the segment counter, while the transaction trailer reports that counter as the number of segments in the transaction.
- **Evidence:** The segment is emitted at `src/Billing/X125010837P.php:L785-L793`, drawing a sequence value at `src/Billing/X125010837P.php:L791`. Every other segment in the generator increments the counter immediately before or after being appended; this one does not. The trailer writes the counter at `src/Billing/X125010837P.php:L1616-L1621`.
- **Status:** VERIFIED
- **Intent:** The comment block above the segment, at `src/Billing/X125010837P.php:L778-L784`, records that medical attachments are not implemented and that the three code values are hardcoded. Read as evidence of intent, the segment was added as a placeholder for a feature that was never built, which is consistent with the four configuration columns for attachment transport that no code reads, catalogued in [transactions.md](transactions.md).
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The transaction reports one fewer segment than it contains for every employment-related claim, and the batch copies that count through unchanged per [BR-H7](#br-h7-the-segment-count-is-copied-from-the-generator-into-the-batch-unchanged), so the discrepancy reaches the payer. A payer that validates the count rejects the transaction, and the rejection arrives as an acknowledgement days later rather than at generation time. Separately, the claim promises an attachment that no transport exists to send, so the payer waits for a document that will never arrive. Both consequences are registered in [defect-candidates.md](defect-candidates.md).

### BR-F7 A service line carries at most four diagnosis pointers

- **Statement:** A service line emits its diagnosis pointers separated by colons and stops after the fourth. Any further diagnoses linked to that line are dropped, with no warning and no log entry.
- **Evidence:** `src/Billing/X125010837P.php:L1346-L1357`, with the limit enforced at `src/Billing/X125010837P.php:L1354-L1356`.
- **Status:** VERIFIED
- **Intent:** The implementation guide permits at most four diagnosis code pointers on a professional service line, so the limit is the format's rather than the application's. INFERRED (confidence: High): the truncation was chosen over an error because a claim with five linked diagnoses is still payable on the first four. Basis: the loop breaks silently rather than appending to the log that the same function writes to for other conditions, such as the zero-charge claim note at `src/Billing/X125010837P.php:L669-L671`.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The dropped pointers are chosen by the order the diagnosis index array happens to be in, so which diagnoses reach the payer is determined by that ordering and not by clinical priority. A payer that denies for medical necessity may be denying because the justifying diagnosis was the fifth. Nothing on the screen indicates that a pointer was dropped.

### BR-F8 A charge created from a remittance ignores the dry-run flag

- **Statement:** The helper that creates a new charge from a remittance accepts a dry-run flag, a deposit identifier and a date, and uses none of the three. A dry run therefore inserts a real charge row.
- **Evidence:** The signature is at `src/Billing/SLEOB.php:L165` and the whole body is `src/Billing/SLEOB.php:L166-L208`; the insert is the call at `src/Billing/SLEOB.php:L194-L207`, whose arguments include none of the three unused parameters. The comment naming the single caller is at `src/Billing/SLEOB.php:L161-L163`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the parameters were kept for signature compatibility with the sibling posting helpers, which do use their dry-run flag. Basis: the two sibling helpers accept the same trio and honour it, and this function retains a large commented-out block at `src/Billing/SLEOB.php:L167-L179` showing that it once did more work, so the signature predates the current body.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Previewing a remittance file is the operation an operator performs precisely in order to change nothing, and for this one path it creates charges. The charge is inserted as unauthorised, which the comment at `src/Billing/SLEOB.php:L163` states, so it does not immediately appear as billable, but it is present and it is not removed when the preview ends. Whether this path is reached at all is decided by the site global registered as [BR-I3](#br-i3-one-switch-turns-a-payer-reported-unknown-code-into-a-charge).

### BR-F9 The institutional generator always declares the claim chargeable

- **Statement:** The institutional generator's transaction header chooses between declaring the claim a report and declaring it chargeable, based on a variable that its own function does not declare as a parameter and that nothing assigns. The branch is therefore always taken the same way and every institutional claim is declared chargeable.
- **Evidence:** The branch is at `src/Billing/X125010837I.php:L89`; the function signature that omits the parameter is `src/Billing/X125010837I.php:L26`; the only caller passes five arguments, at `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L46-L52`. The same undeclared variable is tested again at `src/Billing/X125010837I.php:L283`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the branch was copied from the professional generator, where the equivalent variable is a real parameter, and the parameter was not carried across. Basis: the professional generator's equivalent line at `src/Billing/X125010837P.php:L114` is identical in structure and comment but reads a declared parameter, and the institutional version guards the read with a null coalescence, which is what a static analyser requires of an undefined variable.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The encounter-claims feature described in [BR-G7](#br-g7-an-encounter-counts-as-billed-only-when-every-fee-bearing-charge-is-billed) and in the task documentation at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L44-L46` is silently unavailable for institutional claims: a practice that enables it gets reporting claims for professional encounters and chargeable claims for institutional ones, from the same setting. The second occurrence has the same effect on payer identifier selection, so an institutional claim also always uses the primary payer identifier rather than the alternate.

### BR-F10 A voided receipt is journalled only under one configuration

- **Statement:** Voiding a checkout writes a row to the reversal journal only when the void is a purge or when the practice uses invoice reference number pools. Under any other combination nothing is journalled, and the reversal is visible only as the soft-deleted ledger lines.
- **Evidence:** The condition is `src/Billing/BillingUtilities.php:L1894`, with the comment stating it at `src/Billing/BillingUtilities.php:L1893` and the journal insert at `src/Billing/BillingUtilities.php:L1895-L1922`. The soft delete of the ledger lines is at `src/Billing/BillingUtilities.php:L1928-L1933`.
- **Status:** VERIFIED
- **Intent:** The comment states it directly: if the operation is neither undoing a checkout nor using reference number pools, nothing is done. Read as evidence of intent, the journal exists to preserve a reference number that is about to be reused, and the purge case was added to it because a purge also destroys evidence.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The journal is append-only and carries the payment and adjustment totals it reversed, at `src/Billing/BillingUtilities.php:L1901-L1902`, so under the configuration where it is not written there is no record of what a void reversed beyond the timestamps on the deleted rows. A deposit whose lines were voided then reports as short against the check registered as [BR-B7](#br-b7-a-deposit-is-balanced-against-live-ledger-lines-only-and-only-in-a-browser-alert) with nothing to explain why.

### BR-F11 A remittance carrying a Medicare inpatient adjudication segment posts nothing

- **Statement:** The modern remittance parser recognises a fixed vocabulary of segments and returns an error naming any segment outside it, which abandons the whole file. The Medicare inpatient adjudication segment is outside that vocabulary. The legacy remittance renderer does understand that segment and displays its contents. A Medicare institutional remittance can therefore be read correctly in the history browser and can never be posted to accounts receivable.
- **Evidence:** The rejection is the final branch of the parser's segment dispatch, at `src/Billing/ParseERA.php:L467-L468`. The legacy renderer recognises the segment at `library/edihistory/edih_835_html.php:L531` and renders eight of its elements at `library/edihistory/edih_835_html.php:L532-L545`. A search of `src/Billing/ParseERA.php` for that segment identifier returns nothing.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the parser was written against the professional remittances the practice actually received, and the institutional adjudication segment was never added because no institutional remittance was posted through it. Basis: the parser handles the professional counterpart of the same information as an explicitly ignored segment at `src/Billing/ParseERA.php:L318-L319`, which shows the author chose to name and ignore segments he knew about, and this one is absent from that list entirely.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. This is one of the three headline findings this register was written to record, and it inverts the usual assumption that newer code supersedes older: for this one segment the oldest generation is the more capable one. The generations themselves are described in [architecture.md](architecture.md), and the two components' differing segment vocabularies are compared in [transactions.md](transactions.md).
- **Blast radius:** The operator sees the file listed and readable in the history browser and sees the posting screen refuse it, with a message naming a segment identifier and no indication that the segment is unsupported rather than malformed. Adding the segment to the parser's vocabulary as an ignored segment would make these remittances postable in one line, which is why the asymmetry is worth recording precisely; the defect entry is in [defect-candidates.md](defect-candidates.md).

```php
} else {
    return "Unknown or unexpected segment ID $segid";
}
```

That is `src/Billing/ParseERA.php:L467-L469`. Because the return value is the error string the caller displays, one unrecognised segment ends the parse wherever it appears in the file.

## Group G Identity and Naming Conventions

Eight rules. A convention is a rule that nothing enforces, which makes this group the one where the schema's own comments carry the most weight: the database declares no check constraints anywhere, so several invariants below exist only as prose beside a column definition.

### BR-G1 A claim is identified by patient then encounter

- **Statement:** A claim is identified throughout this subsystem by the patient number and the encounter number joined with a hyphen, patient first. The identifier is not a database key; the claim record itself is keyed by the two numbers separately.
- **Evidence:** The convention is documented at `src/Billing/BillingProcessor/BillingClaim.php:L31-L38`, parsed at `src/Billing/BillingProcessor/BillingClaim.php:L114-L117` and emitted onto the claim at `src/Billing/X125010837P.php:L675`.
- **Status:** VERIFIED
- **Intent:** To give a claim a single stable string that survives a round trip through a payer, since the payer echoes it back on the remittance. That round trip is what makes the recovery rule in [BR-D5](#br-d5-the-claim-identifier-is-recovered-from-the-remittance-by-counting-its-parts) necessary.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The order is easy to state backwards, and it is stated backwards in comments elsewhere in this subsystem, which the contradiction census in [README.md](README.md) records. Code that reverses the two numbers posts a payer's money to the patient whose number equals an encounter number, and because both are integers there is nothing about the resulting identifier that looks wrong. The parse takes the two parts positionally with no validation, so an identifier with one part assigns nothing to the encounter.

```php
$ta = explode("-", (string) $claimId);
$this->pid = $ta[0];
$this->encounter = $ta[1];
```

That is `src/Billing/BillingProcessor/BillingClaim.php:L114-L117`, with the identifier assignment omitted from the excerpt.

### BR-G2 The ledger payer type is cut out of a user-interface label

- **Statement:** The payer type written onto every ledger line during remittance posting is produced by taking the fourth character onward of the display label chosen for the insurance level. The label is a screen artefact and the payer type is a persisted numeric field, and the second is a substring of the first.
- **Evidence:** Three call sites, all identical: `interface/billing/sl_eob_process.php:L566` on the payment, `interface/billing/sl_eob_process.php:L628` on the zero-dollar adjustment note and `interface/billing/sl_eob_process.php:L649` on the real adjustment. The label is constructed at `interface/billing/sl_eob_process.php:L348-L352` and is also used for the payer lookup at `interface/billing/sl_eob_process.php:L428` and the level watermark at `interface/billing/sl_eob_process.php:L698`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the label was needed for display first, and the numeric level was derived from it rather than the other way round, because the label already encodes the level. Basis: the label is built at the top of the claim handler before any posting occurs, and all five consumers derive from it rather than from a separate numeric variable.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The offset is a literal three, so renaming the label from a three-character prefix to anything else silently changes every payer type written to the ledger, and the change would be made by someone editing a display string. The result is also a string rather than an integer, which the column is, so the coercion is left to the database. The two meanings that the resulting value can carry are registered as [BR-G3](#br-g3-payer-type-zero-means-two-incompatible-things).

### BR-G3 Payer type zero means two incompatible things

- **Statement:** In the accounts-receivable ledger, a payer type of zero means the patient. In the claim model that feeds the batch pipeline, zero means the payer level could not be determined. One value carries two incompatible meanings inside one subsystem.
- **Evidence:** The ledger meaning is declared in the column comment at `sql/database.sql:L10195`, which enumerates zero as the patient and one upwards as insurance levels. The claim-model meaning is the constant at `src/Billing/BillingProcessor/BillingClaim.php:L79`, assigned by the fallback at `src/Billing/BillingProcessor/BillingClaim.php:L132-L134`. The deposit header carries the same patient convention in its own comment at `sql/database.sql:L10160`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the two were designed independently, the ledger convention first, and the claim model chose zero for absent because one, two and three were taken. Basis: the ledger comment is a schema comment of long standing and the claim constants are declared together under a comment naming them as options for the payer type, with no reference to the ledger.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The schema comment states the ledger meaning; nothing anywhere states that the other meaning also exists.
- **Blast radius:** A claim whose payer level was not recognised carries a value that the ledger would read as a patient responsibility. The two values never meet directly today, because the posting path derives its payer type from the display label per [BR-G2](#br-g2-the-ledger-payer-type-is-cut-out-of-a-user-interface-label) rather than from the claim model, but any future code that carried a claim-model payer type into the ledger would silently reassign a payer's money to the patient. Because there is no check constraint on the column, the database would accept it.

### BR-G4 A ledger line is either a payment or an adjustment, by comment alone

- **Statement:** A ledger line holds a payment amount and an adjustment amount, and exactly one of them is always zero. This invariant is recorded only in a column comment. Nothing in the schema enforces it, and the two posting helpers each hardcode the other field to zero to maintain it.
- **Evidence:** The comment is at `sql/database.sql:L10200`. The payment helper writes the literal zero adjustment at `src/Billing/SLEOB.php:L156` and the adjustment helper writes the literal zero payment at `src/Billing/SLEOB.php:L244`.
- **Status:** VERIFIED
- **Intent:** To let one table serve as both a cash ledger and an adjustment ledger without a discriminator column, since the zero amount is itself the discriminator.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. Column comments are not reproduced by the generated schema documentation in a way that presents them as invariants.
- **Blast radius:** The invariant is maintained by two call sites and asserted by none, so any third writer that set both fields would produce a row that every reader misinterprets. The database declares no check constraints anywhere, verified by searching `sql/database.sql` across all 282 table definitions, so this is representative rather than exceptional: the invariants in this schema live in comments. Both helpers pass the zero as a string rather than a number, which the fixed-point column coerces.

```php
'payAmount' => '0.0',
'adjustmentAmount' => $amount,
```

That is `src/Billing/SLEOB.php:L244-L245`, the adjustment helper. The payment helper is the mirror image at `src/Billing/SLEOB.php:L155-L156`.

### BR-G5 The claim-level ledger convention is an empty code, and the remittance path writes a word

- **Statement:** The ledger's code column declares by comment that an empty value means the amount is claim level rather than service level. The remittance path does not write an empty value for claim-level money; it writes the literal word `Claim` as the procedure code.
- **Evidence:** The convention is the column comment at `sql/database.sql:L10193`. The remittance path assigns the literal at `src/Billing/ParseERA.php:L278` for genuine claim-level amounts and at `src/Billing/ParseERA.php:L56` for the synthetic line registered as [BR-B4](#br-b4-an-artificial-service-line-named-claim-absorbs-the-residue). The value reaches the ledger through the key built at `interface/billing/sl_eob_process.php:L453-L456`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the empty-code convention belongs to a manual posting path and the literal word belongs to the remittance path, and they were never reconciled because both satisfy their own readers. Basis: the comment is a schema comment and the literal is a parser constant; the parser's literal is additionally load-bearing for its own recognition test, so it cannot simply be emptied without changing that test.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** A report that identifies claim-level ledger lines by an empty code misses every line the remittance path created, and a report that identifies them by the literal word misses every line a manual posting created. The column is twenty characters, so the literal fits, and there is nothing to distinguish it from a genuine procedure code of the same spelling.

### BR-G6 Values passed through the cleaner are uppercased and filtered to a fixed character set

- **Statement:** Values destined for an outbound claim are passed through a single cleaner that uppercases the whole string and then removes every character outside a fixed permitted set. The permitted set is letters, digits, space and fourteen punctuation characters. Anything else, including any accented letter, is deleted rather than replaced.
- **Evidence:** `src/Billing/Claim.php:L217-L220`, a single expression at `src/Billing/Claim.php:L219`. The docblock at `src/Billing/Claim.php:L212` names the character set the filter enforces and the page of the implementation guide that defines it.
- **Status:** VERIFIED
- **Intent:** The docblock states it: the function enforces the format's basic character set. Read as evidence of intent, that makes the cleaner a format-conformance boundary rather than a sanitiser, which matters because it is not a defence against anything.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Deleting rather than replacing means a name written with an accented letter loses that letter entirely rather than gaining a substitute, so a patient's name can reach a payer misspelled in a way that fails an eligibility match. Because the filter runs after the uppercase, adding a lowercase character to the permitted set would have no effect. The cleaner is also applied to monetary values on the way into the charge accumulation registered as [BR-A4](#br-a4-charge-totals-are-accumulated-as-floats-and-printed-at-two-decimals), where the uppercase is inert but the character filter is not.

```php
return preg_replace('/[^A-Z0-9!"\\&\'()+,\\-.\\/;?=@ ]/', '', strtoupper((string) $str));
```

That is `src/Billing/Claim.php:L219`.

### BR-G7 An encounter counts as billed only when every fee-bearing charge is billed

- **Statement:** An encounter is considered billed when it has at least one chargeable item and every one of them is billed. A charge counts as chargeable only if its code type is registered as fee-bearing, and product sales are always counted. An encounter with no chargeable item at all is not billed rather than being trivially billed.
- **Evidence:** `src/Billing/BillingUtilities.php:L1734-L1764`, with the fee-bearing requirement expressed as a join condition at `src/Billing/BillingUtilities.php:L1744-L1745`, the product-sales union at `src/Billing/BillingUtilities.php:L1746-L1749`, the three-state accumulator initialised at `src/Billing/BillingUtilities.php:L1736` and the final test at `src/Billing/BillingUtilities.php:L1763`.
- **Status:** VERIFIED
- **Intent:** The comment above the function states the rule in one sentence at `src/Billing/BillingUtilities.php:L1731-L1732`, and the initial value carries its own comment naming the no-chargeable-services state at `src/Billing/BillingUtilities.php:L1736`. Both are consistent with the code, which is worth noting in a subsystem where that is not reliable.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The fee-bearing flag lives on the code-type registry, so reclassifying a code type changes which encounters are considered billed without any change to the encounters themselves. The join is written as an implicit cross join with the condition in the predicate, and it matches the code type by key equality, so a charge carrying a code type that is not registered at all disappears from the calculation rather than being counted as unbilled. The related encounter-claims feature, in which claims are marked as reports rather than as chargeable, is documented at `src/Billing/BillingProcessor/Tasks/GeneratorX12.php:L44-L46` and is unavailable for institutional claims per [BR-F9](#br-f9-the-institutional-generator-always-declares-the-claim-chargeable).

### BR-G8 A deposit is found by a reference that is neither unique nor nullable

- **Statement:** The modern payment recorder finds an existing deposit by matching the operator's reference, which is a cheque or explanation-of-benefits number. The column is neither unique nor nullable, and the query has no ordering and no limit, so the match is best effort. The code says so about itself.
- **Evidence:** `src/PaymentProcessing/Recorder.php:L43-L54`, with the query at `src/PaymentProcessing/Recorder.php:L45-L49`. The column is declared not null with an empty-string default at `sql/database.sql:L10163`. The self-description is the docblock at `src/PaymentProcessing/Recorder.php:L39-L41`.
- **Status:** VERIFIED
- **Intent:** The docblock states it plainly: the check is best effort, and it would need the column to be unique and nullable to be fully correct, which it is not. Read as evidence of intent, that is an author documenting a known limitation rather than describing a design.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Because the column defaults to the empty string and is not nullable, every deposit created without a reference shares the same reference value, so a lookup for an empty reference matches an arbitrary one of them. A payment recorded against the wrong deposit changes which deposit reports as balanced under [BR-B7](#br-b7-a-deposit-is-balanced-against-live-ledger-lines-only-and-only-in-a-browser-alert). This class is the fourth generation of accounts-receivable code in this subsystem and is the destination named by the deprecation notice at `src/Billing/SLEOB.php:L221`; the generations are described in [architecture.md](architecture.md).

## Group H Control Number and Sequence Allocation

Eight rules. Control numbers are the identifiers a payer uses to acknowledge or reject a transmission, and sequence numbers are what make a ledger line unique. Both are allocated by the application rather than by the database, and in two cases the schema's own comment says so.

### BR-H1 The claim version is allocated by an unlocked aggregate inside a transaction

- **Statement:** Each claim record carries a version, and the version is part of the record's primary key together with the patient and encounter numbers. The next version is computed by selecting the current maximum for that patient and encounter and adding one. The computation and the insert are wrapped in a transaction, but the select takes no lock, there is no unique-key retry, and the schema's own comment concedes that the increment happens in application code.
- **Evidence:** The column and its comment are at `sql/database.sql:L381`; the composite primary key that includes it is at `sql/database.sql:L392`. The transaction wrapper is `src/Billing/BillingUtilities.php:L1677` and the allocation is the query at `src/Billing/BillingUtilities.php:L1678-L1681`, whose result is bound into the insert at `src/Billing/BillingUtilities.php:L1695` and `src/Billing/BillingUtilities.php:L1704`.
- **Status:** VERIFIED
- **Intent:** To version a claim record without an auto-increment surrogate key, so that the history of submissions for one encounter is addressable by patient, encounter and attempt.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The schema comment names the mechanism; nothing states the concurrency consequence.
- **Blast radius:** Two concurrent submissions for the same patient and encounter can read the same maximum and compute the same next version, and because the version participates in the primary key the second insert fails on a duplicate key rather than silently overwriting. That is the safe failure mode of the two available, and it is worth being precise about: the transaction wrapper does exist, so this is not an unguarded allocation, but a transaction alone does not serialise a plain aggregate read at the default isolation level. Adding a locking read or moving the aggregate into the insert as a subquery would close it. The claim data itself is stored in a text column at `sql/database.sql:L391`, which caps a submitted claim at 65,535 bytes.

```php
'SELECT IFNULL(MAX(version), 0) + 1 AS increment FROM claims WHERE patient_id = ? AND encounter_id = ?',
```

That is `src/Billing/BillingUtilities.php:L1679`. The transaction that encloses it opens at `src/Billing/BillingUtilities.php:L1677`, and the failure mode is registered in [defect-candidates.md](defect-candidates.md).

### BR-H2 The ledger sequence number is allocated the same way, and the code says so

- **Statement:** A ledger line's sequence number is allocated by the same pattern: an unlocked aggregate inside a transaction, with the sequence number participating in the ledger's primary key. Here the code documents the race itself and names two ways of closing it.
- **Evidence:** The transaction wrapper is `src/PaymentProcessing/Recorder.php:L169`, the allocation is `src/PaymentProcessing/Recorder.php:L207-L215` and the self-description is the comment at `src/PaymentProcessing/Recorder.php:L202-L206`. The column comment is at `sql/database.sql:L10191` and the composite primary key at `sql/database.sql:L10210`.
- **Status:** VERIFIED
- **Intent:** The comment states it: even in a default-configured transaction there is a potential race, and it should be done as a subquery in the insert or with a locking read. Read as evidence of intent, that is a known and documented limitation in the newest generation of this code rather than an inherited one.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Two payments recorded for the same patient and encounter at the same moment can collide on the primary key, which fails the second insert. Because this recorder is the destination of the deprecation notice on the legacy poster, the same allocation pattern exists in the newest and the second-newest generation of accounts-receivable code, which is the shape of an inherited design rather than a bug introduced once. The allocation returns a string, and the column is an unsigned integer, so the coercion is left to the database.

### BR-H3 A crossover claim row records five columns and discards the rest

- **Statement:** When a payer forwarded a claim onwards of its own accord, the new claim record is written with only the patient, the encounter, the time, the status and the version. Every other attribute that the caller assembled, including the payer identifier, the payer level, the target and the trading partner, is discarded for that row.
- **Evidence:** The crossover branch is `src/Billing/BillingUtilities.php:L1696-L1704`; the non-crossover branch that does include the assembled fragment is `src/Billing/BillingUtilities.php:L1686-L1695`. The difference is that the crossover statement is a static string while the other interpolates the caller's fragment.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the crossover row exists to record that a submission happened rather than to describe it, because the practice did not choose the payer it went to. Basis: the branch is introduced by a comment naming the automatic forward case, and the caller that reaches it passes a status of six, which the forwarding path sets at `src/Billing/SLEOB.php:L273-L275`.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Every column omitted from the insert takes its schema default, so the payer identifier and the payer level on a crossover claim record are zero, which the schema declares as the defaults at `sql/database.sql:L382` and `sql/database.sql:L384`. A report that reads a claim's payer from this table sees no payer for exactly the claims a payer forwarded, and zero is the value that carries two meanings per [BR-G3](#br-g3-payer-type-zero-means-two-incompatible-things). The row still consumes a version, so it participates in the allocation in [BR-H1](#br-h1-the-claim-version-is-allocated-by-an-unlocked-aggregate-inside-a-transaction).

### BR-H4 Interchange and group control numbers are two separate draws on one sequence

- **Statement:** A batch's interchange control number and its functional group control number are drawn separately from the same site-wide counter. The interchange number is zero-padded to nine characters and the group number is not padded at all, so within one envelope the two numbers are consecutive values of the same counter rendered two different ways.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatchControlNumber.php:L22-L25` for the padded interchange number and `src/Billing/BillingProcessor/BillingClaimBatchControlNumber.php:L27-L30` for the unpadded group number; both call the same generator. That generator is `src/Common/Database/QueryUtils.php:L360-L363`, which allocates from the table declared at `sql/database.sql:L1635-L1637` and seeded at `sql/database.sql:L1639`. The batch draws each once, at `src/Billing/BillingProcessor/BillingClaimBatch.php:L63` and `src/Billing/BillingProcessor/BillingClaimBatch.php:L66`.
- **Status:** VERIFIED
- **Intent:** The class docblock at `src/Billing/BillingProcessor/BillingClaimBatchControlNumber.php:L4-L7` states the padding rule and where each number goes. Read as evidence of intent, the two accessors exist precisely to encode the different width requirements of the two positions.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The trading-partner and envelope reference in [transactions.md](transactions.md) documents where each number lands; this rule concerns where the values come from.
- **Blast radius:** The counter column is a nine-digit unsigned integer with no key, so it can exceed nine digits, and when it does the pad becomes a no-op and the interchange control number is emitted ten characters wide. That widens the interchange header past the fixed length that the batch rebuild depends on, and it is the length check registered as [BR-F4](#br-f4-a-malformed-interchange-header-terminates-the-whole-batch-run) that would catch it, by terminating the run. Separately, every draw consumes a value even when the resulting file is never transmitted, and the attachment segment in [BR-F6](#br-f6-the-attachment-segment-is-emitted-without-being-counted) draws one per employment-related claim.

### BR-H5 A validation run substitutes literal control numbers

- **Statement:** When the batch is being built for validation rather than for transmission, the interchange control number is the literal nine-character string of eight zeroes and a one, and the group control number is the literal `2`. No sequence value is consumed. The mode is detected by looking for the word `validate` inside the first claim's action string.
- **Evidence:** `src/Billing/BillingProcessor/BillingClaimBatch.php:L63` and `src/Billing/BillingProcessor/BillingClaimBatch.php:L66`, both conditional on the same substring test.
- **Status:** VERIFIED
- **Intent:** To avoid burning control numbers on a run that produces nothing a payer will ever see, since a gap in the interchange control number sequence is something a clearinghouse can notice.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The detection reads the action of the first claim only and matches a substring, so a run whose first claim carries a different action produces a real control number for a validation batch, and a run whose action merely contains the word produces literals for a real batch. Two batches built in validation mode are byte-identical in their control numbers, which is what makes validation output comparable, so changing this would change the only reproducible output the subsystem produces.

### BR-H6 The batch renumbers the transaction set and rewrites the reference it planted

- **Statement:** As each claim is appended to a batch, the transaction set header is renumbered to a four-digit counter of transaction sets in that batch, and the transaction's reference identification is rewritten by locating a literal marker that the generator planted and replacing it with the number one. The functional group and interchange trailers emitted by the generator are dropped entirely and re-emitted once by the batch.
- **Evidence:** The transaction renumbering is `src/Billing/BillingProcessor/BillingClaimBatch.php:L240-L249`; the reference rewrite is `src/Billing/BillingProcessor/BillingClaimBatch.php:L252-L257`, which searches for the marker at `src/Billing/BillingProcessor/BillingClaimBatch.php:L254`; the trailers are dropped at `src/Billing/BillingProcessor/BillingClaimBatch.php:L264-L266` and re-emitted at `src/Billing/BillingProcessor/BillingClaimBatch.php:L274-L278`. The marker itself is written by the professional generator at `src/Billing/X125010837P.php:L111` and the institutional generator at `src/Billing/X125010837I.php:L86`.
- **Status:** VERIFIED
- **Intent:** The comment at `src/Billing/BillingProcessor/BillingClaimBatch.php:L253` states the coupling: the marker is set in the professional generator. Read as evidence of intent, the batch and the generator are two halves of one mechanism that communicate through a literal string in the output rather than through a parameter.
- **Novel:** no - checked against Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php, and the first of those does record it. `Documentation/Readme_edihistory.html:L95` states that the claim generation file does not produce a usable value in this position and that tracing a payment back to an exact claim submission is therefore unreliable, and `Documentation/Readme_edihistory.html:L220-L235` proposes a patch that would make the value unique per claim by combining the interchange control number with the transaction counter. The patch was never applied: the rewrite still substitutes the constant one at `src/Billing/BillingProcessor/BillingClaimBatch.php:L254`, and the literal marker is still planted at `src/Billing/X125010837P.php:L111` and `src/Billing/X125010837I.php:L86`. This entry agrees with that decade-old observation and adds what it does not say, which is that the literal is now load-bearing because the batch locates it by searching for it.
- **Blast radius:** The rewrite finds the marker by substring search and replaces six characters at that offset, so if a claim ever contains the same six characters earlier in its transaction header the wrong span is replaced. Changing the literal in either generator without changing the batch's search string leaves the reference unrewritten, and the two generators plant the same literal independently. This is the rule where an existing document and the current code agree most directly, and the defect entry is carried in [defect-candidates.md](defect-candidates.md).

### BR-H7 The segment count is copied from the generator into the batch unchanged

- **Statement:** The transaction trailer reports the number of segments in the transaction. The generator computes that number as it emits, and the batch copies the generator's number into its own trailer verbatim while renumbering the trailer's control number. The batch does not recount.
- **Evidence:** The generator writes its counter at `src/Billing/X125010837P.php:L1616-L1621`. The batch copies it and renumbers only the second element at `src/Billing/BillingProcessor/BillingClaimBatch.php:L259-L262`, in the format expression at `src/Billing/BillingProcessor/BillingClaimBatch.php:L260`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the batch trusts the generator's count because it does not itself know how many segments it wrote for that transaction, having filtered some segments out. Basis: the batch drops the two trailers and rewrites two headers as it appends, so its own segment count would differ from the generator's, and it renumbers the control number rather than the count, which is the element it does have authority over.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Any miscount in the generator reaches the payer unaltered, which is what makes the uncounted attachment segment in [BR-F6](#br-f6-the-attachment-segment-is-emitted-without-being-counted) a payer-visible condition rather than an internal one. A payer that validates the count rejects the transaction, and the rejection arrives as an acknowledgement rather than at generation time; the acknowledgement path is described in [claim-lifecycle.md](claim-lifecycle.md).

### BR-H8 Envelope validity is encoded in the length of a string

- **Statement:** The modern file scanner decides whether a file is a usable interchange by building a string that grows one character for each required envelope segment it finds, in order. The string begins as two characters meaning valid, gains a third when an interchange header is present, a fourth when a functional group header follows it, and a fifth when a transaction set header follows that. Every later validity question is answered by searching that string for a character.
- **Evidence:** The accretion is `src/Billing/EdiHistory/X12File.php:L324-L336`, with the four states at `src/Billing/EdiHistory/X12File.php:L324`, `src/Billing/EdiHistory/X12File.php:L327`, `src/Billing/EdiHistory/X12File.php:L331` and `src/Billing/EdiHistory/X12File.php:L335`. The four questions are answered at `src/Billing/EdiHistory/X12File.php:L114-L117`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): the sentinel is a compact way to return four booleans from one procedural function, carried forward unchanged when the function was lifted into a class. Basis: the four flags are unpacked immediately at the single call site, which is what a caller does when it would have preferred four return values, and the class this now lives in is one of the four strict-typed extraction targets described in [architecture.md](architecture.md).
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The four questions are answered by taking a string position and casting it to a boolean, which is false for position zero. That is correct only because the sentinel always begins with a character that none of the four questions asks about, so no searched character can ever be at position zero. Any change to the leading character, or to the order in which the characters accrete, silently inverts one of the four answers. The scanner also refuses a file outright when it contains a character outside the printable range or matches one of several script-like patterns, at `src/Billing/EdiHistory/X12File.php:L318-L322`, and in that case returns the sentinel unchanged from its initial empty value.

```php
$hasval = 'ov'; // valid
if (str_starts_with($ftxt, 'ISA')) {
    $hasval = 'ovi';
```

That is `src/Billing/EdiHistory/X12File.php:L324-L327`, with the comment on the first line preserved because it is the only statement of what the string means.

## Group I Site Global Gated Rules

Five rules. A site global is an administrator-facing setting stored per site. Four of the five entries below are switches whose position changes a dollar amount, a payer routing decision, or whether anything is transmitted at all, and the fifth is a schema default that behaves like one. Each entry names the declaration, its shipped default and its consumers, because a rule that only fires under a setting is not documented until the setting is named.

### BR-I1 Claim balancing is on by default

- **Statement:** The switch that enables the balancing rewrite is declared as a boolean and ships enabled. Every rule in [Group B](#group-b-remittance-balancing) that alters a payer's amounts is therefore active on a site that has never touched its configuration.
- **Evidence:** The switch is `force_claim_balancing`, declared at `library/globals.inc.php:L1297-L1302`, with the default at `library/globals.inc.php:L1300`. The single consumer is `src/Billing/ParseERA.php:L37`, which gates `src/Billing/ParseERA.php:L38-L79`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: High): the default is enabled because an unbalanced remittance is otherwise unpostable, and the rewrite makes it postable. Basis: the comment block the switch gates, at `src/Billing/ParseERA.php:L29-L35`, describes the rewrite as forcing the sums to agree and names poorly reported reversals as the cause, which is a condition a practice cannot fix at its end.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. The operator help acknowledges the balancing difficulty at `Documentation/help_files/sl_eob_help.php:L100` without naming the switch or its default.
- **Blast radius:** Turning the switch off does not make posting stricter in any visible way; it makes the residue disappear from the ledger, because the rewrite was the only thing putting it there. A practice that turns it off after months of use will find that new remittances post differently from old ones with no message marking the change, and the amounts differ by exactly the residue.

### BR-I2 One switch changes the output channel and the submitter identity

- **Statement:** One switch does two unrelated things. It selects a different generator task, which writes one batch per trading partner instead of one batch overall, and it unlocks the submitter-name accessor, which returns nothing at all when the switch is off. It also changes how the professional generator counts hierarchical levels and when it emits the transaction trailer.
- **Evidence:** The switch is `gen_x12_based_on_ins_co`, declared at `library/globals.inc.php:L1543-L1548`, with the default at `library/globals.inc.php:L1546`. The task selection is `src/Billing/BillingProcessor/BillingProcessor.php:L165` and `src/Billing/BillingProcessor/BillingProcessor.php:L168`. The submitter-name gate is `src/Billing/Claim.php:L656-L658`. The generator consumers are `src/Billing/X125010837P.php:L94`, `src/Billing/X125010837P.php:L97`, `src/Billing/X125010837P.php:L377` and `src/Billing/X125010837P.php:L1614`.
- **Status:** VERIFIED
- **Intent:** The declaration's own description names the purpose as sending claims directly to an insurance company based on the trading-partner settings. INFERRED (confidence: Medium): the submitter-name gate was added later and coupled to the same switch because a direct sender needs to identify itself differently from a clearinghouse submitter. Basis: the accessor carries an inline authorship marker and a comment about being a third-party administrator, at `src/Billing/Claim.php:L653`, which is characteristic of a separate later change rather than of the original design.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** Because one switch governs both the channel and the identity, a site cannot have one without the other. VERIFIED: the switch does not change where the envelope's receiver identity comes from; the receiver is read from the trading-partner row in both positions regardless, which [transactions.md](transactions.md) sets out in detail. That is worth stating explicitly because the natural reading of the switch's name suggests otherwise, and the four generator consumers make its effect on the transaction structure genuinely broad: it controls whether the transaction set header and trailer are emitted per claim or once per batch, at `src/Billing/X125010837P.php:L91-L99` and `src/Billing/X125010837P.php:L1612-L1622`.

### BR-I3 One switch turns a payer-reported unknown code into a charge

- **Statement:** When a remittance reports a service the practice has no charge for, the default behaviour is to treat it as an error and post nothing for the claim. One switch changes that to inserting a new charge into the charge queue, built from the payer's own code and amount, with a description naming the insurance level and the production date.
- **Evidence:** The switch is `add_unmatched_code_from_ins_co_era_to_billing`, declared at `library/globals.inc.php:L1564-L1569`, with the default at `library/globals.inc.php:L1567`. The consumer is `interface/billing/sl_eob_process.php:L501`, whose two arms are `interface/billing/sl_eob_process.php:L502` and `interface/billing/sl_eob_process.php:L504-L505`, and the insert follows at `interface/billing/sl_eob_process.php:L507-L521`.
- **Status:** VERIFIED
- **Intent:** The comment above the branch, at `interface/billing/sl_eob_process.php:L496-L500`, states that an unmatched service item is not inherently an error and that the switch exists for practices that prefer it to be one. Read as evidence of intent, the default is the strict setting and the switch relaxes it.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** With the switch on, a payer can add a charge to a practice's charge queue by reporting a service the practice never billed, and the amount on that charge is the payer's reported charge. The insert runs through the helper registered as [BR-F8](#br-f8-a-charge-created-from-a-remittance-ignores-the-dry-run-flag), so it happens even in a dry run, and it passes a deposit identifier of zero at `interface/billing/sl_eob_process.php:L511` that the helper discards. The charge is inserted as unauthorised, so it does not immediately become billable, but it is present and the invoice total is advanced by it at `interface/billing/sl_eob_process.php:L520`.

### BR-I4 One switch decides whether anything is transmitted at all

- **Statement:** Writing a batch file and transmitting it are two separate steps, and the second happens only when a switch is on. With the switch off the file is written to the outbound directory and nothing queues it, so transmission is whatever the practice arranges outside the application.
- **Evidence:** The switch is `auto_sftp_claims_to_x12_partner`, declared at `library/globals.inc.php:L1550-L1555`, with the default at `library/globals.inc.php:L1553`. The consumer is `src/Billing/BillingProcessor/BillingClaimBatch.php:L172`, inside the guard at `src/Billing/BillingProcessor/BillingClaimBatch.php:L170-L173` that also requires the local write to have succeeded.
- **Status:** VERIFIED
- **Intent:** The declaration's description names the purpose as automatically sending claims from the outbound directory to the trading partner using its stored credentials. The comment at `src/Billing/BillingProcessor/BillingClaimBatch.php:L168-L169` places the queueing immediately after the file write and calls the file the official one, which reads as an assertion that the file is the record and the transmission is an addition to it.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php.
- **Blast radius:** The default is off, so a practice that has configured trading-partner credentials but not this switch produces batch files that are never sent, and the only sign is the absence of records in the transport queue. With the switch on, the per-partner queueing registered as [BR-F5](#br-f5-one-batch-file-is-queued-once-for-every-partner-in-the-batch) applies, and the transport outcome recording registered as [BR-F3](#br-f3-a-failed-transmission-is-recorded-as-a-success) applies to every queued record.

### BR-I5 A newly inserted trading partner transmits live claims

- **Statement:** The envelope carries a usage indicator that tells the receiver whether the interchange is a test or production traffic. The column that supplies it defaults to production. Any insert that omits the column therefore creates a trading partner that transmits live claims, and an operator must explicitly choose test mode to get it.
- **Evidence:** The column is `x12_isa15` and its default is declared at `sql/database.sql:L10039`. The two-value vocabulary is recorded inline on the model that carries it, at `library/classes/X12Partner.class.php:L33`. The value is returned to the generators at `src/Billing/Claim.php:L708-L711` and emitted at `src/Billing/X125010837P.php:L75` and `src/Billing/X125010837I.php:L61`, and preserved across the batch rebuild at `src/Billing/BillingProcessor/BillingClaimBatch.php:L216-L217`.
- **Status:** VERIFIED
- **Intent:** INFERRED (confidence: Medium): production was chosen as the default because the overwhelming majority of partners are production partners, and a test default would have meant every real partner needed an edit. Basis: the neighbouring envelope columns are defaulted the same way, to the value a working partner needs, at `sql/database.sql:L10037` and `sql/database.sql:L10038`; no comment states a reason.
- **Novel:** yes - absent from Readme_edihistory.html, DEVELOPER_GUIDE.md, sl_eob_help.php and cms_1500_help.php. This is one of the three headline findings this register was written to record.
- **Blast radius:** The default applies to an insert that omits the column, which is the shape of a programmatic or migrated partner creation rather than of the administrative screen; the scoping of that distinction is set out in the trading-partner reference in [transactions.md](transactions.md). Where it does apply, the consequence is a live transmission that the practice believes is a test, and the receiving payer adjudicates it. Nothing in the generation path warns when the indicator says production, because production is the normal case.

## Confidence Summary

The register contains **71 rules**. Of those, **70 are VERIFIED** and **1 is INFERRED**.

| Status | Count | Detail |
|--------|------:|--------|
| VERIFIED | 70 | Traced in code, schema DDL or a test, each with a citation |
| INFERRED - High confidence | 0 | none at entry level |
| INFERRED - Medium confidence | 1 | [BR-D3](#br-d3-the-processing-format-is-read-per-claim-stored-and-branches-on-nothing) |
| INFERRED - Low confidence | 0 | none |
| **Total** | **71** | |

The distribution by group:

| Group | Subject | Rules |
|-------|---------|------:|
| A | Monetary calculation and rounding | 9 |
| B | Remittance balancing | 7 |
| C | Claim and status-code mappings | 6 |
| D | Payer- and partner-specific branches | 8 |
| E | Date-boundary logic | 9 |
| F | Behaviour that would silently change amounts | 11 |
| G | Identity and naming conventions | 8 |
| H | Control-number and sequence allocation | 8 |
| I | Site-global-gated rules | 5 |
| **Total** | | **71** |

Two things about that distribution are worth stating rather than leaving to be noticed.

The first is that one entry-level INFERRED status does not mean the register contains one inference. Thirty-one of the seventy verified entries carry a labelled inference inside them, almost always in the `Intent:` field and occasionally about a consequence, and each of those carries its own confidence and its own one-line basis. The distinction the notation draws, and which [README.md](README.md) defines, is between a rule whose *existence and effect* are observed and a rule whose existence rests on reading rather than on tracing. Only one rule in this register is of the second kind, and it is the one where the sole direct statement of purpose is a docblock.

The second is that the confidence in a rule is not the confidence in its safety. Every entry in [Group F](#group-f-behaviour-that-would-silently-change-amounts) is verified, which means the register is certain about what those eleven decisions do; it says nothing about whether they are correct. Fourteen entries cross-reference [defect-candidates.md](defect-candidates.md) precisely because a rule can be both faithfully recovered and wrong, and the two registers are meant to be read together at those points.

Novelty, which is the measure this register was written to satisfy, is recorded per entry rather than in aggregate. Seventy of the seventy-one entries are absent from all four existing documentation sources. The single exception is [BR-H6](#br-h6-the-batch-renumbers-the-transaction-set-and-rewrites-the-reference-it-planted), where a 2016 document records the same condition, proposes a patch that was never applied, and is cited as an agreement rather than restated.

## Related Documents

- [README.md](README.md) - index, citation format, the VERIFIED and INFERRED notation, the source-of-truth ordering, and the scope statement this register inherits
- [architecture.md](architecture.md) - the four coexisting generations, how they call one another, and the storage topology
- [claim-lifecycle.md](claim-lifecycle.md) - the stage a rule fires in, with tables read and written and the operator-visible failure symptom per stage
- [transactions.md](transactions.md) - per-transaction reference and the trading-partner configuration surface these rules are configured through
- business-rules.md - this document
- [upgrade-risk-map.md](upgrade-risk-map.md) - per-file refactor risk, test coverage and inbound coupling for the files these rules live in
- [defect-candidates.md](defect-candidates.md) - the suspected defects, including every case where a rule recorded here has an implementation that looks wrong
- [extraction-roadmap.md](extraction-roadmap.md) - the ordered plan for lifting the remaining legacy responsibilities into the modern namespace

---

## Documentation Attribution

### Authorship

This document was written by reading the OpenEMR source at branch `master`, head commit `b7a7e690e419de3451740f995b768a8e8e5fba87`, dated 2026-07-26, on project version 8.3.0-dev with database schema version 541.

### Method

Rules were extracted from the conditional and arithmetic expressions themselves, because in this subsystem a rule is frequently expressed only as a comparison operator, a rounding choice or a whitelist membership test. Schema column comments were treated as first-class rule sources, because the database declares no check constraints anywhere across its 282 table definitions and several invariants are recorded only in comments. Comments and docblocks in code were read as evidence of intent and never as evidence of behaviour, in accordance with the source-of-truth ordering defined in [README.md](README.md). Every behavioural claim carries a citation to the file and line range that supports it, and every citation was re-read against its target before publication. Nothing was executed: PHP and Composer are not installed in the authoring environment, so no claim here reports an observed run, and where a rule's consequence would need a run to confirm it, the consequence is labelled as an inference with its basis.

### Contributing

- Report an incorrect citation or a rule that no longer holds through GitHub Issues
- Discuss the interpretation of a rule on the OpenEMR Community Forum
- Submit a corrected or additional rule as a Pull Request, keeping the six-field entry template and the citation format unchanged

**Last Updated:** August 2026

**License:** GPL v3

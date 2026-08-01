# OpenEMR Revenue Cycle and X12 EDI Documentation

Recovered business logic for OpenEMR's revenue cycle and its handling of X12, the electronic data interchange (EDI) standard that United States healthcare payers, providers and clearinghouses use to exchange claims, eligibility questions and payments; written down so that a claim can be traced from charge capture to cash posting and the code can be changed without silently altering what a patient or a payer is billed.

**Scope and sources.** This index defines the conventions the seven sibling documents share, routes a reader to whichever of them answers the question at hand, and states what the set covers and what it deliberately leaves alone. It rests on the same first-hand reading as its siblings: 46 files in `src/Billing/`, 17 in `library/edihistory/`, three reachable models in `library/classes/`, one class in `src/PaymentProcessing/` documented at boundary level, the schema in `sql/database.sql`, the autoload configuration in `composer.json`, and the billing test tree. The full enumeration, with line counts, is in [Scope](#scope).

## Audience

These documents are written for engineers who are:

- **Reading** this subsystem for the first time and need to understand it before touching it
- **Refactoring** legacy billing code and need to know which behaviour must survive the change
- **Extending** claim generation, remittance parsing or eligibility checking
- **Diagnosing** a claim that was rejected, a remittance that never posted, or a balance that will not close

**Knowledge assumed.** Fluency in PHP and SQL. Language constructs, prepared-statement binding, class inheritance and join syntax are used without explanation.

**Knowledge not assumed.** Nothing about X12, which the purpose statement above expands. Every X12 term, segment identifier, loop identifier and code list is expanded on first use in each document rather than once across the set, because readers arrive at individual documents through search rather than by reading the set in order.

**Also not assumed.** Anything about the history of this codebase. Four architectural generations of the same subsystem coexist in the tree, and explaining why is treated as required content rather than as background material. The marker that separates them is mechanical rather than a matter of taste: `declare(strict_types=1)` appears in exactly 8 of the 46 files under `src/Billing/` - `src/Billing/DaySheet/BillRow.php:L13`, `src/Billing/DaySheet/DaySheetTotals.php:L13`, `src/Billing/DaySheet/SlotTotals.php:L13`, `src/Billing/DaySheet/DaySheetAggregator.php:L19`, `src/Billing/EdiHistory/RemitAccounting.php:L13`, `src/Billing/EdiHistory/X12File.php:L20`, `src/Billing/EdiHistory/EdiFormat.php:L22` and `src/Billing/EdiHistory/Claim277Renderer.php:L24` - and in 10 of the 67 documented files once the compatibility shim in `library/edihistory/` (`library/edihistory/edih_x12file_class.php:L19`) and the payment recorder in `src/PaymentProcessing/` (`src/PaymentProcessing/Recorder.php:L11`) are counted. [architecture.md](architecture.md) uses that marker to assign every in-scope file to a generation.

## How to read this

Two entry paths cover the two questions this set exists to answer.

### I need to trace a claim end-to-end

→ Start with [claim-lifecycle.md](claim-lifecycle.md)

It walks the revenue cycle as fourteen numbered stages and, for each one, records the code entry point, the tables read, the tables written, the files produced or consumed, the state transitions, and the failure mode together with the symptom an operator actually sees. From there, [transactions.md](transactions.md) supplies segment-level detail for any stage that needs it, and [business-rules.md](business-rules.md) supplies the decision that a stage embodies.

### I need to know whether a change I am about to make is safe

→ Start with [upgrade-risk-map.md](upgrade-risk-map.md)

It carries one row per in-scope file: size, age of the last substantive change, change frequency, the tests that cover it or the literal word `none`, inbound coupling, and a resulting risk classification. Read its method section before its table: it explains why an unfiltered commit history is not a usable age signal for this subsystem, and publishes the mechanical-versus-substantive classifier it uses in place of one. Where a file is classified high-risk, the reason is frequently set out in full in [defect-candidates.md](defect-candidates.md).

## Document index

| Document | Purpose | Read this if... |
|----------|---------|-----------------|
| [architecture.md](architecture.md) | Generation map covering every in-scope file, the mechanisms by which the generations call one another, the storage topology, and the extraction ledger | You want to know why several generations of the same subsystem coexist and which one you are looking at |
| [claim-lifecycle.md](claim-lifecycle.md) | The revenue cycle as fourteen stages, each with entry point, tables read and written, files, state transitions and failure modes | You want to know what happens in order, and what changes when each step runs or fails |
| [transactions.md](transactions.md) | One reference section per X12 transaction type handled, plus the trading-partner and payer-identity configuration references | You want to know, for a given transaction type, where it is built or read and what is non-obvious about it |
| [business-rules.md](business-rules.md) | The business-rule register: one entry per non-obvious rule about money, eligibility, claim identity or payer routing | You want to know what decision the code makes about money or eligibility, and how much to trust that reading |
| [upgrade-risk-map.md](upgrade-risk-map.md) | Per-file refactor risk sorted by risk, published together with the method that produced it | You want to know how likely a change to a given file is to break something silently |
| [defect-candidates.md](defect-candidates.md) | Suspected defects with observable symptoms and concrete verifications, plus a short security appendix | You want to know what already looks wrong, and how you would prove it |
| [extraction-roadmap.md](extraction-roadmap.md) | An ordered plan for completing the legacy-to-modern extraction, including a golden-file X12 corpus strategy | You want to know what to modernize next, in what order, and how behaviour preservation would be shown |

The set is eight documents rather than one because the eight answer different kinds of question. A register has to be individually addressable, citable and diffable; a narrative has to be readable in order; a risk table has to be sortable at a glance. Interleaving them produces a file that serves none of the three. Every sibling links back here for the conventions defined below, and no document in this directory is reachable only from outside it.

## Conventions

Three conventions are defined here, once, for all eight documents. Siblings apply them without variation and none of them redefines a convention locally.

### Citation format

| Kind | Form | Live example |
|------|------|--------------|
| Line range | `path/to/file.php:L120-L145` | `src/Billing/EdiHistory/X12File.php:L101-L102` |
| Single line | `path/to/file.php:L120` | `version.php:L33` |
| Schema | `sql/database.sql:LNNNNN` | `sql/database.sql:L10039` |
| Configuration | `composer.json:LNNN` | `composer.json:L177-L179` |

The rule that governs the whole set: **every claim about behaviour carries a citation, and a sentence describing what the code does without a citation is not acceptable output.**

Three mechanical details follow from that rule. Citations are plain text placed immediately next to the claim they support - never markdown links, never footnotes - so that a claim and its evidence cannot drift apart during editing and so that an anchor stays greppable after the surrounding prose is rewritten. Every anchor is relative to the head commit recorded in [Keeping this current](#keeping-this-current); anchors drift as code changes, which is a property of the format rather than a defect in it. And a link to a sibling document is navigation, not evidence: pointing at the document that will elaborate on a claim does not source it, so a behavioural statement accompanied only by a cross-reference is an uncited statement. Where a count or a census is the claim, the same rule applies to it - either the enumeration is anchored, or the method that produced it is stated next to the number so that a reader can repeat it.

### Claim classes: VERIFIED and INFERRED

Exactly two classes of claim exist, notated identically in all eight documents.

**VERIFIED** - behaviour traced in the code as it stands at the recorded commit. Always carries a citation.

> VERIFIED: the trading-partner usage indicator column defaults to `P` for production (`sql/database.sql:L10039`), that column is returned to the claim as `x12gsisa15()` (`src/Billing/Claim.php:L708-L710`), and both generators emit its value into the interchange envelope unaltered (`src/Billing/X125010837P.php:L75`, `src/Billing/X125010837I.php:L61`), so a newly created partner row transmits live unless an operator explicitly switches it to test.

**INFERRED** - probable intent rather than observed behaviour. Always carries a confidence of **High**, **Medium** or **Low** and a one-line statement of what that confidence rests on.

> INFERRED (confidence: Medium): the comment above the success assignment in the transport tracker is a copy-paste artifact rather than a deliberate note (`src/Billing/BillingProcessor/X12RemoteTracker.php:L119-L120`). Basis: the identical comment sits correctly above the in-progress assignment twelve lines earlier at `src/Billing/BillingProcessor/X12RemoteTracker.php:L107-L108`.

Three rules govern the notation, and they exist because inference presented as observation is worse than no documentation at all.

1. No sentence mixes the two classes. A claim is wholly one or wholly the other.
2. Where a verified observation and an inference about the same code point conflict, the verified statement leads and the inference follows on a separate, labelled line.
3. The notation is defined here and only here. No sibling introduces a third class, extends the confidence vocabulary, or relabels an inference as a finding.

### Source of truth

When two sources disagree about this subsystem, the higher entry wins:

1. **Executable code** - what the interpreter will actually do.
2. **Schema DDL** (data definition language) in `sql/database.sql` - column types, defaults, keys and comments.
3. **Tests** - what someone once asserted the behaviour to be, and what continuous integration still enforces.
4. **Comments, docblocks and existing prose** - last, and only ever as evidence of *intent*, never as evidence of behaviour.

That ordering is forced by evidence rather than chosen as a matter of style. Three worked examples, all verified first-hand at the recorded commit, show why:

- The operator help page for the EOB posting screen - EOB being the explanation of benefits, a payer's statement of how it adjudicated a claim - opens with a docblock announcing itself as access-control help (`Documentation/help_files/sl_eob_help.php:L4`), while the title it actually renders is EOB posting instructions (`Documentation/help_files/sl_eob_help.php:L24`). The file's own description names a different subject than the page it produces.
- In the outbound transport tracker, a comment about changing status from waiting to in-progress sits directly above a line that sets the success status instead (`src/Billing/BillingProcessor/X12RemoteTracker.php:L119-L120`). The same file declares its upload-error status through a misspelled constant name (`src/Billing/BillingProcessor/X12RemoteTracker.php:L30`) that it then uses on the failure path (`src/Billing/BillingProcessor/X12RemoteTracker.php:L113`).
- The claim identifier is documented as patient-then-encounter (`src/Billing/BillingProcessor/BillingClaim.php:L31-L38`) and generated in that order (`src/Billing/X125010837P.php:L675`), yet a comment describing the same key writes it the other way round (`src/Billing/BillingProcessor/BillingProcessor.php:L109`).

Those three bullets account for four of the thirteen comment-versus-code contradictions found in this subsystem at the recorded commit - the transport-tracker bullet carries two. The remaining nine are listed here with their anchors, grouped into five bullets so that the shared kinds stay together, so that the total is a count a reader can check rather than a figure to be taken on trust. Each is elaborated where it belongs, in [architecture.md](architecture.md) and [defect-candidates.md](defect-candidates.md); the list below exists to make the arithmetic verifiable, not to duplicate that analysis.

- The transport tracker's class docblock states that each run is saved as an entry in a `billing_tracker_batch` table (`src/Billing/BillingProcessor/X12RemoteTracker.php:L5`). The schema contains no such table, and the class's own table constant names `x12_remote_tracker` instead (`src/Billing/BillingProcessor/X12RemoteTracker.php:L33`), whose DDL is at `sql/database.sql:L14149`.
- Five legacy files open by naming a file other than themselves: one repeating its own extension (`library/edihistory/edih_277_html.php:L4`), and four naming files that do not exist in the repository (`library/edihistory/edih_278_html.php:L4`, `library/edihistory/edih_835_html.php:L4`, `library/edihistory/codes/edih_271_code_class.php:L4`, `library/edihistory/codes/edih_997_codes.php:L4`).
- The description above the 277 claim-status renderer discusses a 271 eligibility report instead (`library/edihistory/edih_277_html.php:L27-L28`), while the function it introduces renders a parsed 277 (`library/edihistory/edih_277_html.php:L40`).
- The archive script announces a security measure preventing direct web access and names the script it must be reached through (`library/edihistory/edih_archive.php:L15-L17`), but the guard immediately below it is commented out (`library/edihistory/edih_archive.php:L18`), no file of that name exists anywhere in the repository, and the constant that guard would have tested is defined in the screen that actually calls this tree (`interface/billing/edih_main.php:L34`). The same non-existent script is named again in the CSV include (`library/edihistory/edih_csv_inc.php:L52`).
- A docblock states that its function orders CSV output using the table header row, and declares two header helpers as used (`library/edihistory/edih_csv_data.php:L31-L46`), yet that function reads its fields by name and calls neither helper (`library/edihistory/edih_csv_data.php:L47-L197`). The header-ordered rendering it describes is performed by a different function in the same file (`library/edihistory/edih_csv_data.php:L607`).

The ordering is reinforced structurally by what the schema does not contain. `sql/database.sql` runs to 15,395 lines, ending at `sql/database.sql:L15395`, and declares 282 tables; that figure is a case-insensitive enumeration of `CREATE TABLE` at the recorded commit, and a count anchored to the start of a line returns 281 instead because one statement is indented (`sql/database.sql:L1180`). Across all 282 there are no `CHECK` constraints, no declared foreign keys, no triggers and no views. The two tables at the centre of this subsystem show the shape - each closes on a primary key and, at most, one secondary index, with nothing else between the last column and the storage-engine clause (`sql/database.sql:L245-L278` for `billing`, `sql/database.sql:L378-L393` for `claims`). Several invariants of this subsystem are therefore recorded only as column `COMMENT` text, as the claim version is when it states that it is incremented in code (`sql/database.sql:L381`), which makes those comments first-class evidence of a rule. That is not a licence to trust a comment about behaviour: a comment is admissible for what an invariant was meant to be and never for what the code does. The schema shows the difference itself. The phrase "foreign key" appears in it four times, and every occurrence is inside a column comment describing a relationship the database does not enforce (`sql/database.sql:L15138-L15139` and `sql/database.sql:L15155-L15156`).

The closing reason is the one that matters most for a reader deciding how much to trust these documents. In a subsystem where descriptive text is demonstrably unreliable, documentation that trusted comments would propagate their errors with a citation attached, lending false authority to a wrong statement. That is worse than no documentation, so every behavioural claim in this set is anchored to code or to DDL, and comments appear only where the question being answered is what somebody meant.

## Scope

### In scope

Sixty-seven files and 32,621 lines are documented. The enumeration below is this run's own count at the recorded commit, not a figure carried over from any existing document: each row was produced by listing the PHP files beneath the named path and summing their lines, so any row can be re-derived in a single command and checked against what is printed here.

| Surface | Files | Lines |
|---------|------:|------:|
| `src/Billing/**` - 14 top-level files (10,945 lines), `BillingProcessor/` 10, `BillingProcessor/Tasks/` 13 files yielding 11 concrete task classes plus 2 abstract bases, `BillingProcessor/Traits/` 1, `DaySheet/` 4, `EdiHistory/` 4 | 46 | 16,186 |
| `library/edihistory/**` - 13 top-level procedural scripts, 3 PHP code tables under `codes/`, plus the spreadsheet `library/edihistory/codes/code_formatter.ods`, which sits in the same directory, is not PHP, and - by a repository-wide search for its name at the recorded commit - is referenced by no code; what it is for is registered as a labelled inference in [architecture.md](architecture.md) | 17 | 14,979 |
| `library/classes/X12Partner.class.php` (496 lines, `class X12Partner` at `library/classes/X12Partner.class.php:L17`), `library/classes/InsuranceCompany.class.php` (416, `library/classes/InsuranceCompany.class.php:L32`), `library/classes/Controller.class.php` (316, `library/classes/Controller.class.php:L27`) | 3 | 1,228 |
| `src/PaymentProcessing/Recorder.php` - a concrete implementation documented here at boundary level, that is, for the surface its methods present rather than their internals (`class Recorder` at `src/PaymentProcessing/Recorder.php:L22`); in scope because it is the destination named by a deprecation notice inside the legacy accounts-receivable poster (`src/Billing/SLEOB.php:L221`) | 1 | 228 |
| **Total** | **67** | **32,621** |

Line figures count PHP source lines; `code_formatter.ods` is a binary and contributes none, which is why the second row counts 17 files but only 16 files' worth of lines.

Source for the table: PHP file listings and line sums taken beneath `src/Billing/`, `library/edihistory/`, `library/classes/` and `src/PaymentProcessing/` at the recorded commit. The two directory rows carry no line anchor because a directory census is not a claim about any single line; where a row names individual files, each is anchored above. Every one of the 67 appears as its own row, with its size, in [upgrade-risk-map.md](upgrade-risk-map.md).

Five supporting counts define the rest of the documented surface. Each was derived by enumeration at the recorded commit, and each is given here with either an anchor or the method that produced it:

- **21** database tables read or written by in-scope code. Fifteen of them are named in no existing document, and two of those fifteen are referenced by this code more often than either `claims` (`sql/database.sql:L378`) or `ar_activity` (`sql/database.sql:L10188`): `users`, whose DDL is at `sql/database.sql:L9786` and which is read at `src/Billing/Claim.php:L132` and `src/Billing/MiscBillingOptions.php:L83` among others, and `insurance_data`, whose DDL is at `sql/database.sql:L3306` and which is read at `src/Billing/BillingUtilities.php:L1771` and `src/Billing/SLEOB.php:L258`. Every one of the 21 is anchored to its DDL, with its reference count, in [claim-lifecycle.md](claim-lifecycle.md).
- **9** X12 transaction types: 837P and 837I (professional and institutional claims), 835 (remittance advice), 270 and 271 (eligibility request and response), 276 and 277 (claim-status inquiry and response), 278 (services review, that is, authorisation) and the 997/999 acknowledgement family. The handler is selected from a single dispatch map of eight functional-group codes at `src/Billing/EdiHistory/X12File.php:L101-L102`; the count reaches nine because the one 837 entry in that map covers both the professional and the institutional flavour, and those are built by two separate generators with distinct entry points (`src/Billing/X125010837P.php:L40`, `src/Billing/X125010837I.php:L26`).
- **14** lifecycle stages: one boundary stage, S0 (encounter and fee-sheet entry), plus S1 through S13 from charge capture to history indexing. That decomposition is this documentation set's own. The code declares no stage numbering, so the numbers exist only to let a stage be referred to unambiguously across the eight documents.
- **32** trading-partner configuration columns, at `sql/database.sql:L10026-L10057` inside a `CREATE TABLE` block spanning `sql/database.sql:L10025-L10059`. A count of 33 for this table includes the `PRIMARY KEY` clause at `sql/database.sql:L10058`, which is not a column. [transactions.md](transactions.md) documents all 32, each with a census of what actually reads it, which is where the columns that nothing consumes are identified.
- **16** billing test files totalling 2,994 lines, counted the same way as the table above and read as coverage evidence only. Thirteen live under `tests/Tests/Isolated/Billing/` and execute only under the secondary configuration, which is the only one declaring that directory as a suite (`phpunit-isolated.xml:L65-L67`) and which continuous integration invokes by name (`.github/workflows/isolated-tests.yml:L50`); the primary configuration's suite list does not reference that directory at all (`phpunit.xml:L43-L93`). The other three live under `tests/Tests/Services/Billing/`, which the primary configuration does cover (`phpunit.xml:L67-L69`). Which classes those tests actually exercise, and which of the largest in-scope files they leave with the literal coverage value `none`, is set out file by file in [upgrade-risk-map.md](upgrade-risk-map.md).

### Boundary only

Five systems are named as entry or exit points so that a reader can find the seam. Their internals are not documented.

| Boundary system | Resolved paths | Treated as |
|-----------------|----------------|------------|
| Fee sheet user interface | `interface/forms/fee_sheet/**`, `src/Forms/FeeSheet/**` | Entry point of stage S0, and home of the only optimistic-concurrency control in the charge-capture path: a visit checksum computed as the form is built (`interface/forms/fee_sheet/new.php:L487`), posted back in a hidden field (`interface/forms/fee_sheet/new.php:L1768`), compared on submission (`interface/forms/fee_sheet/new.php:L510-L511`), and guarding the charge-saving branch, which runs only if that comparison raised no message (`interface/forms/fee_sheet/new.php:L534`). The guard covers that branch rather than the whole request: a diagnosis-update save earlier in the same handler runs before the comparison (`interface/forms/fee_sheet/new.php:L491-L501`). Screen internals are not documented. |
| EOB posting screens | `interface/billing/sl_eob_process.php`, `sl_eob_search.php`, `sl_eob_invoice.php`, `sl_eob_patient_note.php`, with remittance intake at `interface/billing/era_payments.php`; 31 PHP files at the top level of `interface/billing/`, counted by listing that directory at the recorded commit | Entry points of stages S9 and S11, subject to the interpretation recorded below. |
| Clearinghouse module integrations | `interface/modules/custom_modules/**`, plus the Composer-declared `claimrevolution/oe-module-claimrev-connect` at `composer.json:L52` | Exit point of stage S6. The declared module is absent from this checkout: listing `interface/modules/custom_modules/` at the recorded commit returns a readme and seven module directories - `oe-module-comlink-telehealth`, `oe-module-dashboard-context`, `oe-module-dorn`, `oe-module-ehi-exporter`, `oe-module-faxsms`, `oe-module-prior-authorizations` and `oe-module-weno` - and no `oe-module-claimrev-connect`, so its behaviour cannot be read from this repository at all. The two built-in transports are documented instead: the SFTP path, which uploads the batch file at `src/Billing/BillingProcessor/X12RemoteTracker.php:L112`, and the real-time HTTP eligibility path, which posts to the endpoint on the partner row at `src/Billing/EDI270.php:L852`. |
| Patient statements | `sites/*/statement.inc.php`, required at `interface/billing/era_payments.php:L34`, and `interface/patient_file/front_payment.php` | Downstream consumer of accounts-receivable data. The statement include is resolved per site through the site directory rather than from a fixed path (`interface/billing/era_payments.php:L34`), and the copy shipped as the default instructs the practice to customise it (`sites/default/statement.inc.php:L4`), so its content is not a property of this repository. |
| FHIR and REST billing endpoints | `src/RestControllers/FHIR/FhirCoverageRestController.php`, `src/Services/FHIR/FhirCoverageService.php`, `src/FHIR/R4/**` | Exit point. This surface is Coverage-only: a controller and a service exist for Coverage (`src/RestControllers/FHIR/FhirCoverageRestController.php:L22`, `src/Services/FHIR/FhirCoverageService.php:L49`), while listing `src/Services/FHIR/` and `src/RestControllers/FHIR/` at the recorded commit returns neither a service nor a controller for Claim, ClaimResponse or ExplanationOfBenefit - although all three data-transfer objects do exist (`src/FHIR/R4/FHIRDomainResource/FHIRClaim.php:L71`, `src/FHIR/R4/FHIRDomainResource/FHIRClaimResponse.php:L71`, `src/FHIR/R4/FHIRDomainResource/FHIRExplanationOfBenefit.php:L71`). No external system can retrieve a claim or an explanation of benefit over FHIR today. |

### Out of scope

- **All source-code modification.** No PHP file is edited, no refactor is performed, and no docblock or inline comment is added to any source file. That last exclusion is worth stating explicitly, because correcting a misleading comment is a tempting adjacent improvement in a subsystem with thirteen of them, each anchored under [Source of truth](#source-of-truth) - and it would still be a code change.
- **All test code.** Nothing under `tests/` is created or modified. Where [extraction-roadmap.md](extraction-roadmap.md) and [defect-candidates.md](defect-candidates.md) propose a test, they describe it in prose; they do not write it.
- **The root `README.md`.** It links outward to the project website and does not link the in-repository `docs/` tree at all (`README.md:L33`), so leaving it untouched creates no stale or contradictory link.
- **The `Documentation/` tree.** Cited extensively for agreements and contradictions with observed code and used as the style model for this set, but never edited. The 2016 legacy readme in particular is neither corrected nor superseded: it is the only contemporaneous record of the original design intent, and several inferences in [architecture.md](architecture.md) rest on it as evidence.
- **Documentation infrastructure.** No site generator, navigation manifest, build script or dependency is introduced. Diagrams are fenced `mermaid` blocks, which the hosting platform renders natively.
- **Security remediation.** Security-sensitive observations are flagged with a citation and a severity in a short appendix to [defect-candidates.md](defect-candidates.md) and are deliberately neither analysed at exploit depth nor fixed.
- **Defect remediation.** The registers document; they do not repair.
- **Clinical subsystems, scheduling, the patient portal, access-control internals and the forms engine.** The clinical-to-revenue seam by which a clinical form can post charges is identified in stage S0, because a reader tracing a claim needs to know charges can originate there, but the forms engine itself is not documented.

### One documented interpretation

One pair of requirements pulls in opposite directions, and the resolution is recorded here rather than absorbed silently, because it changes what a reader should expect to find in the rule register.

**The tension.** The EOB posting screens are boundary-only, yet [business-rules.md](business-rules.md) must capture every non-obvious rule and gives first priority to behaviour that would silently change patient- or payer-facing amounts. Four such rules live inside a screen file:

- Non-primary insurance adjustments are reported as notes and posted with a zero amount (`interface/billing/sl_eob_process.php:L612-L631`).
- The accounts-receivable payer type is derived from a user-interface label by string offset, at three separate call sites (`interface/billing/sl_eob_process.php:L566`, `interface/billing/sl_eob_process.php:L628`, `interface/billing/sl_eob_process.php:L649`).
- A site-level global flag adds codes from the payer's remittance that match nothing on the claim into the charge table (`interface/billing/sl_eob_process.php:L501`).
- A two-pass parse commits accounts-receivable session rows before the file has been validated. The check pass runs first (`interface/billing/sl_eob_process.php:L850`) and calls back into the screen at the end of its loop (`src/Billing/ParseERA.php:L553`); that callback inserts the session row (`interface/billing/sl_eob_process.php:L277-L285`); and the interchange-trailer check that decides whether the file ended prematurely happens only afterwards (`src/Billing/ParseERA.php:L555-L556`), with the posting pass following on the next line (`interface/billing/sl_eob_process.php:L851`).

**Resolution.** `interface/billing/**` is boundary-only for its user-interface, presentation and screen-flow internals, which is what "internals" means for a screen. The specific lines at which it makes accounts-receivable posting decisions are cited, because those lines are revenue-cycle business logic that happens to live in a screen file.

**Justification.** The alternative produces a register that knowably omits the rule that secondary-payer adjustments are never posted as amounts. Naming these files as the entry points of stages S9 and S11 is in any case exactly what identifying entry and exit points permits; the resolution extends that permission by the minimum needed, which is citing decision lines rather than documenting screens.

## Keeping this current

- **Every anchor is relative to one commit.** Branch `master`, head `b7a7e690e419de3451740f995b768a8e8e5fba87`, on project version 8.3.0-dev (`version.php:L17-L20`) with database schema version 541 (`version.php:L33`). The subsystem's own runtime floor is PHP 8.2.0 (`composer.json:L14`), with continuous integration exercising 8.2 through 8.6: the syntax check runs the whole matrix (`.github/workflows/syntax.yml:L28`) and the isolated test suite runs the same span (`.github/workflows/isolated-tests.yml:L30-L35`), while single-version jobs pin 8.5 (`.github/workflows/build-release.yml:L154`, `.github/workflows/database.yml:L34`, `.github/workflows/acceptance-docker.yml:L341`). Line numbers drift; a citation that no longer lands where it says it does means the anchor moved, not that the claim was wrong when it was written.
- **The risk table is regenerable.** [upgrade-risk-map.md](upgrade-risk-map.md) publishes the commands and the mechanical-versus-substantive commit classifier that produced it, so it can be re-derived rather than hand-maintained. That method is not duplicated here.
- **No code was executed to produce this set.** PHP and Composer are not installed in the authoring environment, so no part of the subsystem was run and no test suite was executed. Every claim rests on static reading of file contents at the commit above. This is exactly why [defect-candidates.md](defect-candidates.md) proposes a verification for each entry - a named test with the configuration it runs under, or a reproduction with screen, input and expected outcome - instead of asserting a reproduction it has not performed.
- **No user-specified rules govern this documentation.** The project's rules facility was consulted and reports that none were provided, so nothing here was written to satisfy a rule and no rule is invented to justify a decision. In their place these documents bind themselves to eleven conventions - but those conventions come from two different sources, and which source governs which convention matters when one of them has to be changed. Six are directives of the documentation requirements this set was written to satisfy: they are not repository practice, the repository would not have suggested them, and they carry no repository anchor because the requirements are not a file in this tree. The other five are taken from the repository's own observed practice, and each of those is anchored to it.

  Governed by the documentation requirements - six:

  1. **The citation format**, defined under [Citation format](#citation-format). The requirements specify this notation verbatim. Nothing in the existing documentation estate cites source this way, so there was no repository practice to follow.
  2. **The two claim classes and their confidence vocabulary**, defined under [Claim classes: VERIFIED and INFERRED](#claim-classes-verified-and-inferred). The requirements direct that inferred intent never be presented as verified behaviour and that every inference be labelled; collapsing that into one notation used identically across eight documents is how the direction is met.
  3. **The source-of-truth ordering**, defined under [Source of truth](#source-of-truth). The ordering is required. The repository supplies only the evidence for why it is the right ordering, which is the thirteen contradictions enumerated there.
  4. **The audience register** declared at the top of this file - PHP and SQL fluency assumed, X12 and the history of this codebase not. That sentence is a requirement, and it is what fixes the register of all eight documents.
  5. **The separation of document purposes** that produced eight files instead of one. The requirements enumerate seven deliverables by filename plus this index, so the split is specified rather than chosen; the reasoning restated under [Document index](#document-index) explains it but did not decide it.
  6. **The rule that no document in this directory is orphaned.** Every sibling is linked from the index above and links back to this file for the conventions.

  Taken from the repository's own observed practice - five:

  1. **The markdown house style**, modelled on `Documentation/api/DEVELOPER_GUIDE.md:L1-L3` for the opening and on `Documentation/api/README.md:L7-L14` for the index table.
  2. **The quality gates** listed in the next bullet, which are the gates this repository already runs against every file.
  3. **The link convention used inside `docs/`:** sibling documents by bare filename, intra-document targets by anchor, and repository-root files through `../../`. The last of those is arithmetic rather than taste - these documents sit two directories below the repository root, so a single `../` resolves to `docs/` and not to the root, which makes `../../composer.json` correct and `../composer.json` a broken link. No document in this set links to a repository-root file today: where one is needed as evidence rather than as navigation it appears as a plain-text citation, which is what the citation rule requires in any case.
  4. **Plain ASCII punctuation**, the one exception being the arrow of the reader-routing idiom this repository already uses (`Documentation/api/README.md:L29`).
  5. **GPL v3 attribution**, matching the surrounding estate.
- **Five gates run against these files and they must pass unaided.** Trailing whitespace (`.pre-commit-config.yaml:L68`), end-of-file newline (`.pre-commit-config.yaml:L69`), uniform line endings (`.pre-commit-config.yaml:L138`), file size - configured with no arguments and therefore enforcing the hook's 500 KB default (`.pre-commit-config.yaml:L84`) - and spell checking with codespell v2.4.3 (`.pre-commit-config.yaml:L151-L155`). The spell-check skip list excludes neither `docs/` nor markdown generally (`.codespellrc:L21`), so these files are checked by the hook and again in continuous integration (`.github/workflows/spellcheck.yml:L35`); no dictionary entry was added for them. No markdown linter exists in that hook set, so heading levels, fencing and table consistency rest on authoring discipline alone.
- **A citation that does not support its claim is a defect in the document.** It is not a triviality and it is not a rounding error: the whole value of this set is that any statement in it can be checked in under a minute. Spot-verify by re-reading the cited range. When a claim and its citation disagree, fix the document.

---
## Documentation Attribution

### Authorship

This documentation was produced by reading OpenEMR's revenue-cycle and X12 EDI source code, its schema DDL and its test tree at the commit recorded above. It builds on the collective work of the OpenEMR community, and in particular on the one existing narrative account of the legacy EDI history tree, written in 2016 (`Documentation/Readme_edihistory.html:L4`), which is cited for its agreements and contradictions with observed code rather than repeated. Its author twice labelled his own reasoning as assumption rather than fact (`Documentation/Readme_edihistory.html:L112` and `Documentation/Readme_edihistory.html:L115`) - the same verified-versus-inferred discipline this set adopts, practised in this very subsystem a decade earlier.

### Method

Facts were established from executable code first, schema DDL second, tests third, and comments last and only as evidence of intent, for the reasons set out in [Source of truth](#source-of-truth). Counts were derived by enumeration at the recorded commit rather than taken from existing documents, and where an existing figure and a first-hand count disagreed, the first-hand count is published with the anchor that supports it. No code was executed and no tests were run.

### Contributing

OpenEMR is an open-source project. To improve these documents:

- **Report Issues:** [GitHub Issues](https://github.com/openemr/openemr/issues)
- **Discuss:** [Community Forum](https://community.open-emr.org/)
- **Submit Changes:** [Pull Requests](https://github.com/openemr/openemr/pulls)

**Last Updated:** August 2026
**License:** GPL v3

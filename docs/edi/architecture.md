# OpenEMR Revenue Cycle and X12 EDI Architecture

Why several versions of the same subsystem coexist in this tree, how to tell which one you are looking at, and how they call one another.

**Scope and sources.** This document assigns all 67 in-scope files of the revenue-cycle and X12 EDI subsystem to an architectural generation, documents the five mechanisms by which those generations invoke each other, records the on-disk storage topology, and keeps the ledger of what has been extracted from the legacy tree so far. It was traced from `src/Billing/` (46 files), `library/edihistory/` (17 files), three reachable models in `library/classes/`, one contract in `src/PaymentProcessing/`, and the autoload configuration in `composer.json`. Conventions, claim classes and the source-of-truth ordering are defined once in [README.md](README.md) and are used here without variation.

**Provenance.** Every line anchor below is relative to branch `master` at head commit `b7a7e690e419de3451740f995b768a8e8e5fba87`, on project version 8.3.0-dev (`version.php:L17-L20`). No code was executed to produce this document: PHP and Composer are not installed in the authoring environment, so every claim rests on static reading of file contents at that commit. Counts were derived by enumeration rather than carried over from any existing document.

## Table of Contents

- [Why Four Generations Coexist](#why-four-generations-coexist)
    - [The X12 vocabulary used in this document](#the-x12-vocabulary-used-in-this-document)
    - [How to tell which generation you are in](#how-to-tell-which-generation-you-are-in)
- [The Four Generations](#the-four-generations)
    - [The objective marker](#the-objective-marker)
    - [How the project itself describes the split](#how-the-project-itself-describes-the-split)
    - [What final means and does not mean here](#what-final-means-and-does-not-mean-here)
    - [Why generation four matters concretely](#why-generation-four-matters-concretely)
- [Component to Generation Map](#component-to-generation-map)
    - [Generation 2 top level classes in src/Billing](#generation-2-top-level-classes-in-srcbilling)
    - [Generation 2 the batch pipeline](#generation-2-the-batch-pipeline)
    - [Generation 2 the task family](#generation-2-the-task-family)
    - [Generation 3 the strict typed classes](#generation-3-the-strict-typed-classes)
    - [Generation 1 the legacy edihistory tree](#generation-1-the-legacy-edihistory-tree)
    - [Generation 1 the reachable library classes](#generation-1-the-reachable-library-classes)
    - [Generation 4 the payment recording contract](#generation-4-the-payment-recording-contract)
    - [Files inside src/Billing that are not on the X12 path](#files-inside-srcbilling-that-are-not-on-the-x12-path)
    - [Coverage arithmetic](#coverage-arithmetic)
- [The Five Interoperation Mechanisms](#the-five-interoperation-mechanisms)
    - [Mechanism 1 the class_alias shim](#mechanism-1-the-class_alias-shim)
    - [Mechanism 2 the global function backfill](#mechanism-2-the-global-function-backfill)
    - [Mechanism 3 modern signatures type hinted on legacy classes](#mechanism-3-modern-signatures-type-hinted-on-legacy-classes)
    - [Mechanism 4 implicit globals invisible to static analysis](#mechanism-4-implicit-globals-invisible-to-static-analysis)
    - [Mechanism 5 the Composer autoload configuration](#mechanism-5-the-composer-autoload-configuration)
    - [How little of this is an explicit include](#how-little-of-this-is-an-explicit-include)
- [Extraction Status Ledger](#extraction-status-ledger)
- [The Dependency Cycle the Extraction Created](#the-dependency-cycle-the-extraction-created)
- [Storage Topology](#storage-topology)
- [Three Parallel Trading Partner Loaders](#three-parallel-trading-partner-loaders)
- [The Paper CMS 1500 Channel](#the-paper-cms-1500-channel)
- [Agreements and Contradictions with Existing Documentation](#agreements-and-contradictions-with-existing-documentation)
    - [The 2016 legacy readme](#the-2016-legacy-readme)
    - [The architecture stub](#the-architecture-stub)
    - [The developer guide](#the-developer-guide)
    - [Comment versus code contradictions inside the subsystem](#comment-versus-code-contradictions-inside-the-subsystem)
- [Inference Register](#inference-register)
- [Documentation Attribution](#documentation-attribution)

## Why Four Generations Coexist

Nothing in this subsystem was ever replaced. It was added to. Four bodies of code that do overlapping jobs are all present and all reachable in a single request, because each new effort wrapped or partially lifted the previous one instead of retiring it. Reachability is not a figure of speech here: the modern trees are autoloaded by the mapping at `composer.json:L173-L176`, the legacy classes by the classmap at `composer.json:L177-L179`, eight legacy global-function files on every single request by the eager list at `composer.json:L180-L189`, and the whole legacy procedural tree by the sixteen `require_once` statements the operator screen issues in one block at `interface/billing/edih_main.php:L71-L86`. The reason the tree looks the way it does is that every migration so far has been designed to keep the old call sites working, and every one of those compatibility measures is still in place.

That history is not recorded anywhere in the repository, so it has to be reconstructed from the artifacts the migrations left behind. Those artifacts are unusually legible. A 21-line file exists whose entire body is one `class_alias` call, and whose own docblock says the class was lifted out and that the file remains so that procedural callers keep resolving the old name (`library/edihistory/edih_x12file_class.php:L3-L17`, `library/edihistory/edih_x12file_class.php:L21`). Three global formatting functions still exist but now do nothing except forward to a class, with a comment on each saying so (`library/edihistory/edih_csv_inc.php:L1070-L1075`, `library/edihistory/edih_csv_inc.php:L1084-L1089`, `library/edihistory/edih_csv_inc.php:L1098-L1103`). A method on a deprecated posting helper still exists but constructs a class from a different namespace and hands the work over (`src/Billing/SLEOB.php:L221`, `src/Billing/SLEOB.php:L233-L235`). Each of these is a migration frozen at the point where backward compatibility was preserved and the cleanup was left for later.

The practical consequence for a reader is that the question "where does this behaviour live?" usually has more than one answer, and the answers are not equivalent. Two components can hold two versions of the same arithmetic, and only one of them is on the path that touches money. VERIFIED: the generation-3 balance test adds provider-level adjustments into the accounted total at `src/Billing/EdiHistory/RemitAccounting.php:L29`, while the generation-2 remittance parser deliberately excludes them from accounts receivable at `src/Billing/ParseERA.php:L429-L431`; the only caller of that balance test is the legacy history viewer at `library/edihistory/edih_835_html.php:L1291`, so the copy a reader is most likely to find is the copy that never posts a dollar. The rule and the discrepancy themselves belong to [business-rules.md](business-rules.md) and [defect-candidates.md](defect-candidates.md). That is why this document exists before the lifecycle document: you cannot follow a claim through the system until you can tell which copy of a component the request actually reached.

### The X12 vocabulary used in this document

X12 is the electronic data interchange (EDI) standard that United States healthcare payers, providers and clearinghouses use to exchange claims, eligibility questions and payments. An X12 file is a flat text file of segments; a transaction set inside it is identified by a number. The terms below appear throughout this document and are expanded here on first use, because the audience for this set is assumed to know PHP and SQL and not to know X12.

| Term | Expansion | Relevance here |
|------|-----------|----------------|
| 837 | Health care claim | The outbound claim transaction set; this subsystem builds two flavours of it |
| 837P | 837 professional | Professional claims, built by `src/Billing/X125010837P.php` |
| 837I | 837 institutional | Institutional claims, built by `src/Billing/X125010837I.php` |
| 835 | Health care claim payment and remittance advice | The inbound payment transaction set, parsed by `src/Billing/ParseERA.php` |
| 270 | Eligibility, coverage or benefit inquiry | The outbound eligibility question |
| 271 | Eligibility, coverage or benefit information | The inbound eligibility answer |
| 276 | Health care claim status request | The outbound status question |
| 277 | Health care claim status notification | The inbound status answer, including the 277CA claim acknowledgement variant |
| 278 | Health care services review information | Authorisation and referral, parsed and displayed but never generated here |
| 997 | Functional acknowledgement | The older syntactic acknowledgement of a transmitted file |
| 999 | Implementation acknowledgement | The current syntactic acknowledgement, handled together with the 997 |
| ISA and IEA | Interchange control header and trailer | The outermost envelope of an X12 file |
| GS and GE | Functional group header and trailer | The group envelope; its GS01 code is what this subsystem dispatches on |
| ST and SE | Transaction set header and trailer | The envelope around one transaction |
| BHT | Beginning of hierarchical transaction | Carries the submitter reference used to trace a claim back to its batch |

Two of these are worth flagging now because they shape the architecture rather than just the file format. The functional group code in GS01 is the value this subsystem uses to decide what kind of file it is holding, from a map of eight codes at `src/Billing/EdiHistory/X12File.php:L101-L102`; that single map is why [transactions.md](transactions.md) is organised by transaction type. And the 837 has no legacy renderer at all, which is why the outbound and inbound halves of the subsystem sit in different generations.

### How to tell which generation you are in

Three checks, in order, answer the question for any file in the subsystem.

1. **Is it under `library/`?** Then it is generation 1. Under `library/edihistory/` it is procedural and not autoloaded at all, so it is reachable only once something has explicitly required it: usually a screen, as the sixteen `require_once` statements at `interface/billing/edih_main.php:L71-L86` do, and in one case a generation-2 class, at `src/Billing/EDI270.php:L35`. Under `library/classes/` it is a root-namespace class that is autoloaded, because the classmap at `composer.json:L177-L179` names that directory and no other.
2. **Is it under `src/` without `declare(strict_types=1)`?** Then it is generation 2: namespaced and autoloaded, but written in the same coding idiom as the procedural code it replaced.
3. **Is it under `src/` with `declare(strict_types=1)`?** Then it is generation 3 or 4, distinguished only by namespace: `OpenEMR\Billing\EdiHistory` and `OpenEMR\Billing\DaySheet` are generation 3, `OpenEMR\PaymentProcessing` is generation 4.

The following block map answers one question: which trees hold which generation, and how big is each. It carries no dependency edges, because the ways the generations reach one another are the subject of [The Five Interoperation Mechanisms](#the-five-interoperation-mechanisms) and of the diagram there. The three links between the blocks are layout-only invisible links whose sole effect is to stack the four blocks vertically so the labels stay readable; they assert nothing.

```mermaid
flowchart TB
    subgraph GEN1["Generation 1 legacy procedural 17 plus 3 files 16207 lines"]
        direction TB
        G1A["library/edihistory 13 top level scripts"]
        G1B["library/edihistory/codes 3 code tables plus 1 spreadsheet"]
        G1C["library/classes 3 reachable billing models"]
    end
    subgraph GEN2["Generation 2 namespaced non strict 38 files 13906 lines"]
        direction TB
        G2A["src/Billing 14 top level classes"]
        G2B["src/Billing/BillingProcessor 10 pipeline files"]
        G2C["src/Billing/BillingProcessor/Tasks 13 task files"]
        G2D["src/Billing/BillingProcessor/Traits 1 logging trait"]
    end
    subgraph GEN3["Generation 3 strict typed extraction target 8 files 2280 lines"]
        direction TB
        G3A["src/Billing/EdiHistory 4 classes"]
        G3B["src/Billing/DaySheet 4 classes"]
    end
    subgraph GEN4["Generation 4 strict typed payment recording 1 documented contract"]
        direction TB
        G4A["src/PaymentProcessing/Recorder.php"]
    end
    GEN1 ~~~ GEN2
    GEN2 ~~~ GEN3
    GEN3 ~~~ GEN4
```

The figures in that diagram are the in-scope figures, and they are the sum of the per-file counts published in [Component to Generation Map](#component-to-generation-map): generation 1 is the 17 files of `library/edihistory/` at 14,979 PHP lines plus the three `library/classes/` models at 1,228 lines; generation 2 is the 14 top-level `src/Billing/` files at 10,945 lines plus 10 pipeline files at 1,225, 13 task files at 1,693 and one trait at 43; generation 3 is `src/Billing/EdiHistory/` at 2,054 lines plus `src/Billing/DaySheet/` at 226. Generation 4 is documented at contract level only, as the single file `src/PaymentProcessing/Recorder.php` at 228 lines; the namespace it belongs to holds 14 files and 1,846 lines in total, of which 10 declare strict types.

## The Four Generations

| Generation | Location | Idiom | Objective marker | Files | PHP lines | Strict-typed |
|------------|----------|-------|------------------|------:|----------:|-------------:|
| 1 | `library/edihistory/`, `library/edihistory/codes/`, `library/classes/` | Procedural functions and root-namespace classes, loaded by explicit `require_once` | Not under `src/`; no namespace declaration | 20 | 16,207 | 0 |
| 2 | `src/Billing/` top level, `BillingProcessor/`, `BillingProcessor/Tasks/`, `BillingProcessor/Traits/` | Namespaced and autoloaded, but untyped signatures, static entry points and reference parameters | Under `src/`, no `declare(strict_types=1)` | 38 | 13,906 | 0 |
| 3 | `src/Billing/EdiHistory/`, `src/Billing/DaySheet/` | Strict-typed classes with static methods and value objects | `declare(strict_types=1)` | 8 | 2,280 | 8 |
| 4 | `src/PaymentProcessing/` | Strict-typed service classes | `declare(strict_types=1)` plus the `OpenEMR\PaymentProcessing` namespace | 1 documented of 14 | 228 documented of 1,846 | 10 of 14 in the namespace |

Sources for the table: the file and line counts are enumerated per file in [Component to Generation Map](#component-to-generation-map); the strict-types counts are the presence of `declare(strict_types=1)` in each file, for example `src/Billing/EdiHistory/X12File.php:L20` and `src/Billing/EdiHistory/EdiFormat.php:L22`; the generation-1 loading idiom is visible in one place as a block of sixteen `require_once` calls at `interface/billing/edih_main.php:L71-L86`.

That last citation is worth pausing on, because it is the whole of generation 1's loading strategy. All sixteen PHP files of `library/edihistory/` are required, in one block, by a single operator screen (`interface/billing/edih_main.php:L71-L86`). There is no autoloading for that tree: nothing in it is reachable unless something ran those requires. This is the single most important structural fact about generation 1 and it explains behaviour that otherwise looks arbitrary, including why the compatibility alias described in [Mechanism 1](#mechanism-1-the-class_alias-shim) exists only after that screen has been entered.

### The objective marker

Generations are assigned in this document by the presence or absence of `declare(strict_types=1)`, not by reading style. The marker is reliable here because it partitions the modern tree exactly: it appears in 8 of the 46 files under `src/Billing/`, and those eight are precisely the four classes in `src/Billing/EdiHistory/` and the four in `src/Billing/DaySheet/`, with no exceptions in either direction. A reader can re-derive the entire generation-2 versus generation-3 split with one search for that declaration, which is why it is used here in preference to any judgement about code quality or age.

INFERRED (confidence: High): the declaration marks authorship era rather than a per-file stylistic choice. Basis: it partitions 46 files exactly along directory boundaries with no mixed directory, which a per-file preference would not produce.

### How the project itself describes the split

The repository states the `src/` versus `library/` versus `interface/` division in its own contributor guidance, inside a fenced block under `## Project Structure` at `CLAUDE.md:L3`. Quoted verbatim from `CLAUDE.md:L6-L8`:

```text
/src/              - Modern PSR-4 code (OpenEMR\ namespace)
/library/          - Legacy procedural PHP code
/interface/        - Web UI controllers and templates
```

The same document restates the rule as a directive rather than a description at `CLAUDE.md:L346-L347`, where it requires PSR-4 namespacing under `OpenEMR\` for `/src/` and says new code goes in `/src/` while legacy helpers go in `/library/`. The first anchor is quoted verbatim above and the second is cited and summarised, because together they are the only in-repository statement of the boundary that this document's generation model rests on. Neither file is modified by this documentation run.

### What final means and does not mean here

Generation 3 is often described as a set of strict-typed final classes. That is half right, and the half that is wrong carries an implication about extension that does not hold.

Two of the four are final: `final class Claim277Renderer` at `src/Billing/EdiHistory/Claim277Renderer.php:L30` and `final class EdiFormat` at `src/Billing/EdiHistory/EdiFormat.php:L26`. Both additionally declare a private constructor, at `src/Billing/EdiHistory/Claim277Renderer.php:L32-L34` and `src/Billing/EdiHistory/EdiFormat.php:L28-L30`, so neither can be extended nor instantiated.

Two are not final: `class X12File` at `src/Billing/EdiHistory/X12File.php:L82` and `class RemitAccounting` at `src/Billing/EdiHistory/RemitAccounting.php:L17`. `X12File` is instantiable by design, since legacy code constructs it through the compatibility alias, for instance at `library/edihistory/edih_csv_inc.php:L590`. `RemitAccounting` exposes a single static method, `isBalanced()` at `src/Billing/EdiHistory/RemitAccounting.php:L27`, and declares no constructor at all.

INFERRED (confidence: Medium): the non-finality of `RemitAccounting` is an omission rather than a decision. Basis: the two final classes both also declare private constructors as a deliberate no-extension stance, while `RemitAccounting` has only static members and would lose nothing by matching them.

The other four generation-3 files, in `src/Billing/DaySheet/`, are all strict-typed and all final, and two are additionally readonly: `final class DaySheetAggregator` at `src/Billing/DaySheet/DaySheetAggregator.php:L23`, `final class SlotTotals` at `src/Billing/DaySheet/SlotTotals.php:L17`, `final readonly class BillRow` at `src/Billing/DaySheet/BillRow.php:L17` and `final readonly class DaySheetTotals` at `src/Billing/DaySheet/DaySheetTotals.php:L17`. So the "strict-typed final classes" description holds for the day-sheet half of generation 3 without qualification and for the EDI half only in part.

### Why generation four matters concretely

The subsystem is commonly described as having three generations. It has four, and the fourth is not a curiosity: it is the current destination of an extraction that is already under way.

VERIFIED: the legacy accounts-receivable adjustment poster is marked deprecated in favour of a class in a different namespace, at `src/Billing/SLEOB.php:L221`, which reads `@deprecated Use \OpenEMR\PaymentProcessing\Recorder::recordActivity directly`. The same method already does exactly that, constructing the recorder and delegating to it at `src/Billing/SLEOB.php:L233-L235`, having imported it at `src/Billing/SLEOB.php:L20`. The destination namespace holds 14 files and 1,846 lines, of which 10 declare strict types.

The consequence is practical rather than taxonomic. An extraction plan built on a three-generation model would propose moving accounts-receivable posting into `src/Billing/EdiHistory/`, because that is where the visible extraction work has been happening. The deprecation notice says the project has already chosen a different target for this particular responsibility. Recording generation 4 is what stops [extraction-roadmap.md](extraction-roadmap.md) from proposing the wrong destination.

INFERRED (confidence: Medium): `Recorder` is intended as the destination for accounts-receivable recording generally, not only for adjustments. Basis: the deprecation notice names `recordActivity` without qualification, and the method it deprecates is one of several posting helpers in the same class.

## Component to Generation Map

Every one of the 67 in-scope files appears below with its generation, its line count and one line about what it is for. Files with nothing notable about them are still listed with that one line, because an omission is indistinguishable from an oversight. Line counts were taken at the recorded commit; they are given so that the size distribution, which is the single best predictor of refactor risk in this subsystem, is visible in the same table as the generation.

### Generation 2 top level classes in src/Billing

Fourteen files, 10,945 lines. This is where most of the subsystem's behaviour lives, and none of it declares strict types.

| File | Lines | Responsibility |
|------|------:|----------------|
| `src/Billing/Claim.php` | 2,287 | The claim data model both 837 generators read; loads the trading partner at `src/Billing/Claim.php:L162` and instantiates a generation-1 payer class at `src/Billing/Claim.php:L289` |
| `src/Billing/BillingUtilities.php` | 1,996 | The claim write path plus large hardcoded X12 code tables; the claim-status and adjustment-reason text used across the subsystem |
| `src/Billing/X125010837P.php` | 1,640 | Builds the 837P professional claim; entry point `genX12837P()` at `src/Billing/X125010837P.php:L40` |
| `src/Billing/X125010837I.php` | 1,225 | Builds the 837I institutional claim over the UB-04 array; entry point `generateX12837I()` at `src/Billing/X125010837I.php:L26` |
| `src/Billing/EDI270.php` | 1,162 | Builds the 270 eligibility inquiry and parses the 271 response, including the real-time HTTP path |
| `src/Billing/Hcfa1500.php` | 762 | Builds the paper CMS-1500 claim form; not an X12 path, see [The Paper CMS 1500 Channel](#the-paper-cms-1500-channel) |
| `src/Billing/ParseERA.php` | 561 | Parses the inbound 835 remittance advice and drives posting through a callback |
| `src/Billing/BillingReport.php` | 308 | Helper functions for the billing report screen; not an X12 path |
| `src/Billing/SLEOB.php` | 304 | Accounts-receivable posting helpers; carries the generation-4 deprecation at `src/Billing/SLEOB.php:L221` and one of the subsystem's two `library/` includes at `src/Billing/SLEOB.php:L17` |
| `src/Billing/InvoiceSummary.php` | 255 | Per-invoice charge and payment summary, class declared at `src/Billing/InvoiceSummary.php:L43`; consumed by the EOB posting screen at `interface/billing/sl_eob_process.php:L25` |
| `src/Billing/MiscBillingOptions.php` | 161 | Date-qualifier entries for the CMS-1500 boxes 14 and 15; supports both the paper form and claim-level billing attributes |
| `src/Billing/PaymentGateway.php` | 152 | Credit-card gateway support; not an X12 path and not part of the claim lifecycle |
| `src/Billing/HCFAInfo.php` | 78 | Row and column bookkeeping for the CMS-1500 form, allowing out-of-order composition; not an X12 path |
| `src/Billing/InsurancePolicyTypes.php` | 54 | Hardcoded policy-type codes; see the note below, because its consumers are not what its docblock suggests |

`src/Billing/InsurancePolicyTypes.php` needs one extra line, because it is easy to misfile. Its docblock states the values are fixed by the 837P standard (`src/Billing/InsurancePolicyTypes.php:L3-L5`) and an inline comment repeats the point (`src/Billing/InsurancePolicyTypes.php:L19-L20`), which suggests a claim-generation dependency. VERIFIED: no in-scope X12 generator reads it. Its only consumers are an insurance display card at `src/Patient/Cards/InsuranceViewCard.php:L37`, a coverage validator at `src/Validators/CoverageValidator.php:L156`, and a legacy patient include at `library/patient.inc.php:L54`. It declares codes defined by the 837P standard without being on the 837P write path.

### Generation 2 the batch pipeline

Ten files, 1,225 lines, in `src/Billing/BillingProcessor/`. Four of the ten are interfaces, which makes this the most explicitly designed part of the subsystem.

| File | Lines | Declaration | Responsibility |
|------|------:|-------------|----------------|
| `src/Billing/BillingProcessor/BillingClaimBatch.php` | 280 | `class` at `:L25` | The batch file: accumulates claims, rewrites the envelope and writes the file to disk |
| `src/Billing/BillingProcessor/BillingClaim.php` | 239 | `class` implementing `\JsonSerializable` at `:L20` | One claim as submitted from the billing manager screen, including its per-partner routing data |
| `src/Billing/BillingProcessor/BillingProcessor.php` | 223 | `class` at `:L49` | Takes the billing manager's input, selects the task and runs it over the claim set |
| `src/Billing/BillingProcessor/X12RemoteTracker.php` | 216 | `class` extending `BaseService` at `:L22` | The transport outbox and its status vocabulary for remote submission |
| `src/Billing/BillingProcessor/BillingLogger.php` | 140 | `class` at `:L42` | Logging extracted from the original procedural batch script |
| `src/Billing/BillingProcessor/BillingClaimBatchControlNumber.php` | 31 | `class` at `:L20` | Allocates the ISA and ST control numbers for a batch; its own docblock describes the ISA13 zero padding |
| `src/Billing/BillingProcessor/LoggerInterface.php` | 26 | `interface` at `:L17` | Lets a processing task write to the billing log; default implementation is the trait below |
| `src/Billing/BillingProcessor/GeneratorInterface.php` | 25 | `interface` extending `ProcessingTaskInterface` at `:L18` | The contract for tasks that produce an output file |
| `src/Billing/BillingProcessor/GeneratorCanValidateInterface.php` | 23 | `interface` at `:L16` | Optional contract for a generator that can validate before generating |
| `src/Billing/BillingProcessor/ProcessingTaskInterface.php` | 22 | `interface` at `:L15` | The contract every processing task implements |

### Generation 2 the task family

Thirteen files, 1,693 lines, in `src/Billing/BillingProcessor/Tasks/`: eleven concrete tasks and two abstract bases. The concrete tasks are where the choice of output format is made, so this table doubles as the map from a billing-manager action to the code that produces the artifact.

| File | Lines | Kind | What it produces or does |
|------|------:|------|--------------------------|
| `src/Billing/BillingProcessor/Tasks/GeneratorX12Direct.php` | 370 | Concrete, `class` at `:L34` | An 837P for direct submission, calling `X125010837P::genX12837P()` at `:L241` |
| `src/Billing/BillingProcessor/Tasks/GeneratorHCFA_PDF.php` | 234 | Concrete, `class` at `:L30` | A CMS-1500 PDF, calling `Hcfa1500::genHcfa1500()` at `:L95-L96` |
| `src/Billing/BillingProcessor/Tasks/GeneratorX12.php` | 219 | Concrete, `class` at `:L35` | An 837P batch file, calling `X125010837P::genX12837P()` at `:L70` |
| `src/Billing/BillingProcessor/Tasks/GeneratorHCFA.php` | 179 | Concrete, `class` at `:L29` | CMS-1500 text output, calling `Hcfa1500::genHcfa1500()` at `:L65-L66` |
| `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php` | 160 | Concrete, `class` at `:L30` | An 837I batch file, calling `X125010837I::generateX12837I()` at `:L46` |
| `src/Billing/BillingProcessor/Tasks/AbstractGenerator.php` | 115 | Abstract, `abstract class` at `:L27` | The generator template; drives each claim through `$this->generate()` at `:L58` |
| `src/Billing/BillingProcessor/Tasks/GeneratorUB04Form_PDF.php` | 88 | Concrete, `class` at `:L28` | A UB-04 form PDF, using the procedural helpers `ub04_dispose()` at `:L42` and `get_ub04_array()` at `:L52` |
| `src/Billing/BillingProcessor/Tasks/GeneratorHCFA_PDF_IMG.php` | 67 | Concrete, `class` at `:L30` extending `GeneratorHCFA_PDF` | The same CMS-1500 PDF with the form image behind it, at `:L48-L49` |
| `src/Billing/BillingProcessor/Tasks/GeneratorUB04NoForm.php` | 64 | Concrete, `class` at `:L28` | UB-04 output without the pre-printed form, at `:L44` and `:L49` |
| `src/Billing/BillingProcessor/Tasks/AbstractProcessingTask.php` | 59 | Abstract, `abstract class` at `:L18` | The base every task extends |
| `src/Billing/BillingProcessor/Tasks/GeneratorExternal.php` | 50 | Concrete, `class` at `:L21` | Delegates to a site-supplied exporter, included dynamically at `:L30-L32` and instantiated at `:L35` |
| `src/Billing/BillingProcessor/Tasks/TaskReopen.php` | 49 | Concrete, `class` at `:L21` | Reopens claims rather than generating anything |
| `src/Billing/BillingProcessor/Tasks/TaskMarkAsClear.php` | 39 | Concrete, `class` at `:L20` | Marks claims as cleared rather than generating anything |

`src/Billing/BillingProcessor/Traits/WritesToBillingLog.php`, 43 lines, is the single file in `src/Billing/BillingProcessor/Traits/`: the `trait` at `:L20` that supplies the default `LoggerInterface` implementation to processing tasks.

### Generation 3 the strict typed classes

Eight files, 2,280 lines, split across two namespaces with unrelated subjects. This is the only part of the subsystem where a class was deliberately carved out of older code and left with a compatibility path back to it.

| File | Lines | Declaration | Responsibility |
|------|------:|-------------|----------------|
| `src/Billing/EdiHistory/X12File.php` | 1,566 | `class` at `:L82`, strict types at `:L20` | Reads and validates an X12 file; holds the functional-group dispatch map at `:L101-L102`. Lifted from the legacy tree, per its own docblock at `:L6-L9` |
| `src/Billing/EdiHistory/Claim277Renderer.php` | 373 | `final class` at `:L30` | Renders 277 claim-status segments as HTML; every helper is static and eight take a generation-1 code table |
| `src/Billing/EdiHistory/EdiFormat.php` | 83 | `final class` at `:L26` | The canonical date, money and percentage formatters for EDI display; the legacy globals now forward here |
| `src/Billing/EdiHistory/RemitAccounting.php` | 32 | `class` at `:L17` | `isBalanced()` at `:L27`: whether an 835 remittance balances, compared in integer cents at `:L30` |
| `src/Billing/DaySheet/BillRow.php` | 74 | `final readonly class` at `:L17` | One input row to the day-sheet aggregator; a constructor-promoted value object |
| `src/Billing/DaySheet/SlotTotals.php` | 69 | `final class` at `:L17` | Per-slot accumulator for the day-sheet report |
| `src/Billing/DaySheet/DaySheetAggregator.php` | 54 | `final class` at `:L23` | Aggregates day-sheet rows; not an X12 path |
| `src/Billing/DaySheet/DaySheetTotals.php` | 29 | `final readonly class` at `:L17` | The aggregated per-user, per-provider and grand totals |

The balance test in `RemitAccounting` is the one component in this table that a reader is most likely to be looking for, because it is where a remittance is judged to reconcile or not. Its arithmetic and its disagreement with the parser that feeds accounts receivable are registered as a rule in [business-rules.md](business-rules.md) and as a defect candidate in [defect-candidates.md](defect-candidates.md); this document records only that the class exists, is not final, and sits in generation 3.

The four `src/Billing/DaySheet/` classes are in scope because the documented surface is all of `src/Billing/`, and they are not on the X12 path. VERIFIED: their only consumer is a report screen, which imports all three of the public ones at `interface/billing/print_daysheet_report_num1.php:L20-L22` and runs the aggregator at `interface/billing/print_daysheet_report_num1.php:L140`. The aggregator's docblock at `src/Billing/DaySheet/DaySheetAggregator.php:L5-L11` states that it replaces a fixed set of twenty per-slot accumulator variables in that same screen and that the previous code silently dropped rows past the twentieth distinct user or provider; under the source-of-truth ordering in [README.md](README.md) that docblock is evidence of intent rather than of behaviour, and the behavioural claim is not repeated here.

### Generation 1 the legacy edihistory tree

Seventeen files, of which sixteen are PHP totalling 14,979 lines. The thirteen top-level scripts are the whole of the EDI history feature: parsing, indexing, rendering, uploading and archiving.

| File | Lines | Responsibility |
|------|------:|----------------|
| `library/edihistory/edih_csv_inc.php` | 1,892 | The general-utility include: path helpers, the per-type parameter table at `:L718-L757`, the setup routine at `:L381`, and the formatting globals at `:L1047-L1103` |
| `library/edihistory/edih_csv_parse.php` | 1,599 | Parses X12 files into CSV index rows, treating each ISA envelope as a separate file per its docblock at `:L29-L30` |
| `library/edihistory/edih_835_html.php` | 1,589 | Renders an 835 remittance for display, including the claim payment summary at `:L25` |
| `library/edihistory/edih_archive.php` | 1,305 | The only archival routine in the subsystem: ages out files and CSV rows into a zip archive, per its docblock at `:L5` |
| `library/edihistory/edih_segments.php` | 1,238 | Formats raw X12 segments for display, per its docblock at `:L16` |
| `library/edihistory/edih_csv_data.php` | 949 | Renders CSV index data as HTML tables, ordered by the CSV header row per its docblock at `:L32-L34` |
| `library/edihistory/edih_278_html.php` | 916 | Renders a 278 services-review file; entry point `edih_278_transaction_html()` at `:L39` |
| `library/edihistory/edih_io.php` | 753 | Input and output helpers, and the only database query in the entire legacy tree, at `:L737` |
| `library/edihistory/edih_271_html.php` | 628 | Renders a 271 eligibility response, and by extension the 270 request that shares it |
| `library/edihistory/edih_uploads.php` | 576 | Upload handling, starting with the multi-file array rearrangement at `:L22` |
| `library/edihistory/edih_997_error.php` | 335 | Extracts rejection information from a 997 or 999 acknowledgement, at `:L41` |
| `library/edihistory/edih_277_html.php` | 307 | What remains of the 277 renderer after the case bodies were lifted out; imports the modern renderer at `:L24` |
| `library/edihistory/edih_x12file_class.php` | 21 | The compatibility alias described in [Mechanism 1](#mechanism-1-the-class_alias-shim); no class body at all |
| `library/edihistory/codes/edih_271_code_class.php` | 2,432 | `class edih_271_codes` at `:L26`: the largest single file in the subsystem, and a dependency of two generations |
| `library/edihistory/codes/edih_835_code_class.php` | 264 | The code table for values unique to the 835, per its docblock at `:L27` |
| `library/edihistory/codes/edih_997_codes.php` | 175 | Acknowledgement error-code text, at `:L37` |
| `library/edihistory/codes/code_formatter.ods` | not PHP | An OpenDocument spreadsheet sitting beside the three code tables; it contributes no lines to the 14,979 total |

INFERRED (confidence: Medium): `code_formatter.ods` is the source from which the three PHP code tables were generated. Basis: its name and its placement in the same directory as exactly those three files, with no code in the repository referencing it.

The rendering coverage of this tree is as informative for what it lacks as for what it holds, and one branch of the input and output script decides all of it. VERIFIED: HTML rendering exists for the 271 at `library/edihistory/edih_io.php:L440`, for the 277 at `library/edihistory/edih_io.php:L438`, for the 278 at `library/edihistory/edih_io.php:L442`, for the 835 at `library/edihistory/edih_io.php:L408` and `library/edihistory/edih_io.php:L411`, and for the acknowledgement family through `library/edihistory/edih_csv_data.php:L221`, which calls the report builder at `library/edihistory/edih_997_error.php:L320`. There is no HTML renderer for the 837, which is routed to the raw segment display instead at `library/edihistory/edih_io.php:L401`. There is none for the 270 and none for the 276 either: both reach the same branch as the 271, the 277 and the 278 but match none of its cases, so they fall to the segment display at `library/edihistory/edih_io.php:L443-L445` under a comment stating that HTML display is not available for them. Outbound claim generation was never a generation-1 responsibility at all, so a reader who cannot find a legacy 837 renderer is not looking at a gap in this document. The per-transaction consequences of those asymmetries are set out in [transactions.md](transactions.md).

### Generation 1 the reachable library classes

Three files, 1,228 lines, in `library/classes/`. These are root-namespace classes rather than procedural scripts, and they are reachable from modern code without any include at all, because the classmap at `composer.json:L177-L179` covers this directory, for the reason set out in [Mechanism 5](#mechanism-5-the-composer-autoload-configuration).

| File | Lines | Declaration | Responsibility |
|------|------:|-------------|----------------|
| `library/classes/X12Partner.class.php` | 496 | `class X12Partner extends ORDataObject` at `:L17` | The trading-partner model, documenting the ISA and GS element positions inline at `:L28-L36` and `:L40`; never instantiated from `src/` |
| `library/classes/InsuranceCompany.class.php` | 416 | `class InsuranceCompany extends ORDataObject` at `:L32` | The payer model instantiated by the claim model; its constructor at `:L84` reaches forward into a modern service at `:L90` and into `X12Partner` at `:L100` |
| `library/classes/Controller.class.php` | 316 | `class Controller extends Smarty implements ControllerInterface` at `:L27` | The legacy controller base that the trading-partner and payer screens are built on; a template-engine subclass, which is why it cannot move without moving its callers |

### Generation 4 the payment recording contract

One file is documented, at contract level only, because it is the named destination of an in-progress extraction rather than part of the subsystem's own code.

| File | Lines | Declaration | Why it is documented |
|------|------:|-------------|----------------------|
| `src/PaymentProcessing/Recorder.php` | 228 | `class Recorder` at `:L22`, strict types declared above it | The class named by the deprecation notice at `src/Billing/SLEOB.php:L221` and already called at `src/Billing/SLEOB.php:L233-L235` |

### Files inside src/Billing that are not on the X12 path

Nine of the 46 files under `src/Billing/` do not participate in X12 generation or parsing. They are in scope because the documented surface is all of `src/Billing/`, and they are documented at responsibility level with this note attached, so that a reader can tell they were examined and found to be off the X12 path rather than missed.

| File | Lines | Why it is off the X12 path |
|------|------:|----------------------------|
| `src/Billing/Hcfa1500.php` | 762 | Produces the paper CMS-1500 form, not an X12 transaction |
| `src/Billing/BillingReport.php` | 308 | Screen helpers for the billing report |
| `src/Billing/PaymentGateway.php` | 152 | Card-payment gateway configuration and calls |
| `src/Billing/HCFAInfo.php` | 78 | Layout bookkeeping for the paper form |
| `src/Billing/InsurancePolicyTypes.php` | 54 | Declares 837P-defined codes but is read only by a display card, a validator and a legacy include |
| `src/Billing/DaySheet/BillRow.php` | 74 | Day-sheet report input row |
| `src/Billing/DaySheet/SlotTotals.php` | 69 | Day-sheet report accumulator |
| `src/Billing/DaySheet/DaySheetAggregator.php` | 54 | Day-sheet report aggregation |
| `src/Billing/DaySheet/DaySheetTotals.php` | 29 | Day-sheet report result object |

`src/Billing/MiscBillingOptions.php` is deliberately not in that list. Its docblock describes it as supplying the date qualifiers for the CMS-1500 boxes 14 and 15 (`src/Billing/MiscBillingOptions.php:L3-L4`), which is paper-form vocabulary. VERIFIED: the same claim-level attributes reach the X12 output too. The claim model loads that form's row into `$claim->billing_options` at `src/Billing/Claim.php:L82` through the query at `src/Billing/Claim.php:L176-L181`, and the 837P generator reads it when deciding whether to emit a resubmission reference at `src/Billing/X125010837P.php:L818`. The file is therefore on the boundary rather than off the path.

### Coverage arithmetic

The counts in this document are checkable rather than asserted. Files: 46 under `src/Billing/`, 17 under `library/edihistory/`, 3 in `library/classes/`, and 1 in `src/PaymentProcessing/`, which is 46 + 17 + 3 + 1 = 67. Lines: 16,186 + 14,979 + 1,228 + 228 = 32,621. The `src/Billing/` figure decomposes as 10,945 top level + 1,225 pipeline + 1,693 tasks + 43 trait + 226 day sheet + 2,054 EDI history = 16,186. The `library/edihistory/` figure counts PHP only, which is why that tree contributes 17 files but 16 files' worth of lines.

Regrouped by generation, the same 67 files come out as 20 files and 16,207 lines in generation 1, 38 files and 13,906 lines in generation 2, 8 files and 2,280 lines in generation 3, and 1 documented file and 228 lines in generation 4. The largest single stratum is generation 1 by lines and generation 2 by behaviour, and the entire strict-typed surface of the subsystem is 2,280 lines, or seven per cent of it.

## The Five Interoperation Mechanisms

Four generations in one request only works because five distinct compatibility devices are in place. None of them is documented in the repository, and three of the five are invisible in the header of the file that depends on them, which is why reading a single file is not enough to know what it needs at runtime.

The following diagram answers one question: by what route does each generation reach the others. Edges whose label begins with a mechanism number correspond to the numbered subsections below and are substantiated by the citations there rather than by the diagram; the unnumbered edges are the ordinary loading and delegation paths that make the numbered ones reachable at all.

```mermaid
flowchart TB
    UI["interface/billing screens"]
    G1["generation 1 library/edihistory procedural scripts"]
    G1CODES["generation 1 library/edihistory/codes code tables"]
    G1MODELS["generation 1 library/classes root namespace models"]
    G2["generation 2 src/Billing"]
    G3["generation 3 src/Billing/EdiHistory"]
    G4["generation 4 src/PaymentProcessing"]
    SVC["modern services in src/Services and src/Common"]
    UI -- "bootstrap defines DS then requires all 16 legacy files" --> G1
    G1 -- "M1 class_alias keeps edih_x12_file resolving" --> G3
    G1 -- "M2 legacy format globals delegate to EdiFormat" --> G3
    G3 -- "M3 eight signatures type hinted on edih_271_codes" --> G1CODES
    G2 -- "M4 bare call to a global defined in a screen" --> UI
    G1CODES -- "not autoloaded so available only after an include" --> G1
    G2 -- "M5 root namespace import resolved by classmap" --> G1MODELS
    G1MODELS -- "M5 legacy model reaches forward" --> SVC
    G2 -- "deprecation delegates posting" --> G4
```

### Mechanism 1 the class_alias shim

`library/edihistory/edih_x12file_class.php` is 21 lines long and contains no class body. Its docblock at `library/edihistory/edih_x12file_class.php:L3-L17` states that the class body was lifted to `OpenEMR\Billing\EdiHistory\X12File` and that the file remains so that existing procedural callers keep resolving the unqualified `edih_x12_file` symbol. It declares strict types at `library/edihistory/edih_x12file_class.php:L19` and its entire executable content is one statement at `library/edihistory/edih_x12file_class.php:L21`:

```php
class_alias(\OpenEMR\Billing\EdiHistory\X12File::class, 'edih_x12_file');
```

The receiving class corroborates the arrangement from its own side at `src/Billing/EdiHistory/X12File.php:L6-L9`, which records that it was lifted out of the legacy file and that the old name is preserved as an alias so existing callers keep working unchanged.

Two facts make this more than a formality. First, the alias is load-bearing: the old name is still used across the legacy tree, including as a return type declaration at `library/edihistory/edih_csv_inc.php:L582` and as a constructor call at `library/edihistory/edih_csv_inc.php:L590`. Second, the alias exists only if that shim file was required, and it is required in exactly one place in the repository, at `interface/billing/edih_main.php:L73`. A `class_alias` call is a runtime statement rather than a declaration, so no autoloader can produce it: any code path that reaches legacy EDI history code without passing through that screen will not have the alias.

INFERRED (confidence: High): the alias was retained specifically to avoid editing the legacy call sites, not because the old name is preferred. Basis: the shim's own docblock states the file remains so procedural callers keep resolving the symbol, and the lifted class carries the reciprocal note.

### Mechanism 2 the global function backfill

`src/Billing/EdiHistory/EdiFormat.php` holds the canonical date, money and percentage formatters for EDI display. Its docblock at `src/Billing/EdiHistory/EdiFormat.php:L3-L20` states that the class exists so that namespaced code such as `Claim277Renderer` can call these routines without depending on the non-autoloaded procedural include `library/edihistory/edih_csv_inc.php`, and that the global `edih_format_date()`, `edih_format_money()` and `edih_format_percent()` functions delegate to it as a backfill until every legacy call site is migrated. The class is declared `final` at `src/Billing/EdiHistory/EdiFormat.php:L26` with a private constructor at `src/Billing/EdiHistory/EdiFormat.php:L28-L30`.

The delegation is real and can be read at the three legacy definitions. `edih_format_date()` at `library/edihistory/edih_csv_inc.php:L1070-L1075` returns `EdiFormat::date()`, `edih_format_money()` at `library/edihistory/edih_csv_inc.php:L1084-L1089` returns `EdiFormat::money()`, and `edih_format_percent()` at `library/edihistory/edih_csv_inc.php:L1098-L1103` returns `EdiFormat::percent()`. Each carries a comment saying the canonical implementation now lives in the autoloadable class.

The direction of this edge is the interesting part. It is the only one of the five mechanisms in which generation 1 calls generation 3 by name and gets a real answer: the legacy functions have been hollowed out, so a caller that still uses the procedural name is executing strict-typed code. One legacy formatter was not lifted, `edih_format_telephone()` at `library/edihistory/edih_csv_inc.php:L1047-L1059`, which is why the include is still required for its own sake.

### Mechanism 3 modern signatures type hinted on legacy classes

The 277 claim-status renderer is a partial lift, and it advertises that in its signatures. `src/Billing/EdiHistory/Claim277Renderer.php:L28` is `use edih_271_codes;`, a root-namespace generation-1 class imported into a `final` generation-3 class declared at `src/Billing/EdiHistory/Claim277Renderer.php:L30`. Eight of its methods take that legacy class as a parameter, at `src/Billing/EdiHistory/Claim277Renderer.php:L42`, `:L76`, `:L111`, `:L138`, `:L182`, `:L296`, `:L313` and `:L336`. The narrowest example is the private helper at `src/Billing/EdiHistory/Claim277Renderer.php:L42`:

```php
private static function code(edih_271_codes $cd, string $set, string $value): string
```

The consequence is that the modern class cannot be loaded and exercised on its own. A strict-typed final class in `src/` has a hard type dependency on a 2,432-line procedural code table in `library/edihistory/codes/`, and PHP will not resolve that type unless the legacy file has been included. The clearest demonstration is in the test tree: the renderer's own test has to require the legacy code table by hand in `setUpBeforeClass()` at `tests/Tests/Isolated/Billing/EdiHistory/Claim277RendererTest.php:L35-L38`, reaching five directory levels up to do it, and then construct one at `tests/Tests/Isolated/Billing/EdiHistory/Claim277RendererTest.php:L43`. That single line is both a coverage fact for [upgrade-risk-map.md](upgrade-risk-map.md) and the justification for the first item in [extraction-roadmap.md](extraction-roadmap.md).

### Mechanism 4 implicit globals invisible to static analysis

Some cross-generation calls are not declared at all. They work because a function or a constant happens to be in scope by the time the code runs.

The clearest case sits in the remittance parser. `src/Billing/ParseERA.php:L553` is a bare call:

```php
eob_process_era_callback_check($out);
```

That function is defined in neither the file nor any of its imports; its only definition in the repository is at `interface/billing/sl_eob_process.php:L220`, inside the EOB posting screen. A namespaced class in `src/Billing/` therefore calls a global function that exists only because a particular screen defined it, and PHP will fall back to the global namespace for an unqualified function call, so nothing in the class signals the dependency. The parser's imports are limited to a single modern helper at `src/Billing/ParseERA.php:L17`, which makes the omission plain on inspection of the header.

Constants work the same way. `library/edihistory/edih_csv_inc.php:L77-L79` defines `DS` as the directory separator at include time, and the same file defines the direct-access guard `_EDIH` at `library/edihistory/edih_csv_inc.php:L75`. Legacy path expressions are built from `DS` throughout, including the base-path expression at `library/edihistory/edih_csv_inc.php:L335` and the whole per-type parameter table at `library/edihistory/edih_csv_inc.php:L738-L757`, so no file in that tree can compose a path until this include has run. The operator screen defends against the ordering problem by defining `DS` itself, guarded, before the requires, at `interface/billing/edih_main.php:L64-L66`.

VERIFIED: this particular implicit dependency does not cross into the modern namespace. `DS` appears 177 times in `library/edihistory/` and nowhere in `src/Billing/` except inside an unrelated comment at `src/Billing/Claim.php:L1101`; generation 2 uses the PHP built-in `DIRECTORY_SEPARATOR` instead, for instance when composing the batch directory at `src/Billing/BillingProcessor/BillingClaimBatch.php:L65`. The code tables under `library/edihistory/codes/` are subject to the same loading rule as the rest of the tree for a different reason: they are not covered by the classmap, so they exist only if something required them.

INFERRED (confidence: High): the defensive redefinition of `DS` in the screen exists because the include order could not be guaranteed. Basis: the screen guards the definition with `!defined("DS")` and the include it later loads guards it identically, which is the shape of two authors independently protecting against the other running first.

### Mechanism 5 the Composer autoload configuration

The mechanism that makes generation 1 reachable from generation 2 without any include at all is in the manifest rather than in any PHP file. `composer.json:L173-L176` maps the `OpenEMR\` prefix to `src` under PSR-4. `composer.json:L177-L179` then adds a classmap over `library/classes`, which makes every root-namespace class in that directory autoloadable with no `require` and no path anywhere in the consuming file. `composer.json:L180-L189` declares eight files that are loaded eagerly on every request: `library/global_functions.inc.php`, `library/htmlspecialchars.inc.php`, `library/formdata.inc.php`, `library/sanitize.inc.php`, `library/formatting.inc.php`, `library/date_functions.php`, `library/validation/validate_core.php` and `library/translation.inc.php`. A short `exclude-from-classmap` list at `composer.json:L190-L195` removes four paths from the classmap, none of them in this subsystem.

VERIFIED: exactly three root-namespace imports exist across all 46 files of `src/Billing/`. They are `use InsuranceCompany;` at `src/Billing/Claim.php:L17`, `use edih_271_codes;` at `src/Billing/EDI270.php:L27`, and `use edih_271_codes;` at `src/Billing/EdiHistory/Claim277Renderer.php:L28`. Only the first is satisfied by the classmap, because the classmap covers `library/classes` and not `library/edihistory/codes`. `src/Billing/EDI270.php` solves its own problem with a `require_once` at `src/Billing/EDI270.php:L35`; `src/Billing/EdiHistory/Claim277Renderer.php` does not, which is the same gap that [Mechanism 3](#mechanism-3-modern-signatures-type-hinted-on-legacy-classes) shows up in the test tree.

The classmap import is what makes the dependency easy to miss, because the import statement names a class with no namespace and no path, and resolution depends entirely on a manifest entry two directories away. The generation-2 consumer is the claim model, which constructs the legacy payer class at `src/Billing/Claim.php:L289`:

```php
$orow = new InsuranceCompany($drow['provider']);
```

The sandwich closes on the other side. That generation-1 class immediately reaches forward into modern code: its constructor instantiates a modern service at `library/classes/InsuranceCompany.class.php:L90` and a second legacy model at `library/classes/InsuranceCompany.class.php:L100`, and both it and that second model extend a modern base class, at `library/classes/InsuranceCompany.class.php:L32` and `library/classes/X12Partner.class.php:L17`. So a single property access on a claim can run generation-2 code, then generation-1 code, then modern service code, with no include statement anywhere on the path.

### How little of this is an explicit include

A reader might reasonably expect a family of `require` statements linking the modern billing namespace to the legacy tree. There is almost none, and the exact census matters because it establishes that the coupling is configuration-driven rather than visible in code.

VERIFIED: seven `include` or `require` statements exist across all 46 files of `src/Billing/`, and only two of them reach into `library/`.

| Statement | Target | Note |
|-----------|--------|------|
| `src/Billing/SLEOB.php:L17` | `library/patient.inc.php` | One of the two `library/` includes; executed at file scope, so importing the class loads the include |
| `src/Billing/EDI270.php:L35` | `library/edihistory/codes/edih_271_code_class.php` | The other `library/` include; the only place a modern file makes its own `edih_271_codes` import resolvable |
| `src/Billing/InvoiceSummary.php:L37` | `custom/code_types.inc.php` | Reaches into the site-customisable tree rather than `library/` |
| `src/Billing/BillingProcessor/Tasks/GeneratorUB04Form_PDF.php:L26` | `interface/billing/ub04_dispose.php` | A generation-2 class requiring a screen-tree file |
| `src/Billing/BillingProcessor/Tasks/GeneratorUB04NoForm.php:L26` | `interface/billing/ub04_dispose.php` | The same file, required again |
| `src/Billing/BillingProcessor/Tasks/GeneratorUB04X12.php:L28` | `interface/billing/ub04_dispose.php` | The same file, required a third time |
| `src/Billing/BillingProcessor/Tasks/GeneratorExternal.php:L32` | A path built at `:L30` from a global | A dynamic `include_once` of a site-supplied exporter, guarded by `file_exists()` at `:L31` |

Three of the seven point at `interface/billing/ub04_dispose.php`, which means the dependency direction between the modern task classes and the screen tree runs the opposite way to the layering the project describes at `CLAUDE.md:L6-L8`: `src/` requires from `interface/`.

One further fact corrects an expectation in the other direction. An include that older material names as part of this subsystem no longer exists as a file at all: the invoice-summary logic is now the namespaced class `src/Billing/InvoiceSummary.php`, declared at `src/Billing/InvoiceSummary.php:L43` and consumed as an import at `interface/billing/sl_eob_process.php:L25`. The migration this documentation set maps has already absorbed part of its own subject, so a reader searching for the legacy include will not find it and should not conclude it was missed.

## Extraction Status Ledger

The extraction out of `library/edihistory/` is partial and still moving. The ledger below records, per responsibility, what remains in generation 1, what has been lifted into generation 3, what has been lifted somewhere else, and the anchor that settles it. A blank cell means nothing has moved.

| Responsibility | Still in generation 1 | Lifted to generation 3 | Lifted elsewhere | Citation |
|----------------|----------------------|------------------------|------------------|----------|
| X12 file reading and validation | The alias only, 21 lines with no class body | `src/Billing/EdiHistory/X12File.php`, 1,566 lines | | `library/edihistory/edih_x12file_class.php:L21` and `src/Billing/EdiHistory/X12File.php:L82` |
| Date, money and percentage formatting for display | The three global functions, now hollow, plus `edih_format_telephone()` which did not move | `src/Billing/EdiHistory/EdiFormat.php`, 83 lines | | `library/edihistory/edih_csv_inc.php:L1070-L1103` and `library/edihistory/edih_csv_inc.php:L1047-L1059` |
| 277 claim-status rendering | `library/edihistory/edih_277_html.php`, 307 lines of loop and state glue | `src/Billing/EdiHistory/Claim277Renderer.php`, 373 lines of segment rendering | | `library/edihistory/edih_277_html.php:L24` and `src/Billing/EdiHistory/Claim277Renderer.php:L30` |
| Remittance balance arithmetic | | `src/Billing/EdiHistory/RemitAccounting.php`, 32 lines | | `src/Billing/EdiHistory/RemitAccounting.php:L27-L30` |
| The 271 and 277 code table | `library/edihistory/codes/edih_271_code_class.php`, 2,432 lines, and two generations depend on it | | | `library/edihistory/codes/edih_271_code_class.php:L26`, consumed at `src/Billing/EdiHistory/Claim277Renderer.php:L28` and `src/Billing/EDI270.php:L27` |
| The 835 code table | `library/edihistory/codes/edih_835_code_class.php`, 264 lines | | | `library/edihistory/edih_835_html.php:L25` takes it as a parameter |
| The acknowledgement code table | `library/edihistory/codes/edih_997_codes.php`, 175 lines | | | `library/edihistory/codes/edih_997_codes.php:L37` |
| 271, 278 and 835 rendering | `library/edihistory/edih_271_html.php` 628 lines, `edih_278_html.php` 916, `edih_835_html.php` 1,589 | | | `library/edihistory/edih_278_html.php:L39` and `library/edihistory/edih_835_html.php:L25` |
| Acknowledgement rejection extraction | `library/edihistory/edih_997_error.php`, 335 lines | | | `library/edihistory/edih_997_error.php:L41` |
| CSV indexing and the per-type parameter table | `library/edihistory/edih_csv_inc.php` 1,892 lines, `edih_csv_parse.php` 1,599, `edih_csv_data.php` 949 | | | `library/edihistory/edih_csv_inc.php:L718-L757` |
| Segment display formatting | `library/edihistory/edih_segments.php`, 1,238 lines | | | `library/edihistory/edih_segments.php:L16` |
| Archival and restore | `library/edihistory/edih_archive.php`, 1,305 lines | | | `library/edihistory/edih_archive.php:L5` |
| Upload handling | `library/edihistory/edih_uploads.php` 576 lines, `edih_io.php` 753 | | | `library/edihistory/edih_uploads.php:L22` |
| Accounts-receivable recording | | | `src/PaymentProcessing/Recorder.php` | `src/Billing/SLEOB.php:L221` and `src/Billing/SLEOB.php:L233-L235` |
| Invoice summary | | | `src/Billing/InvoiceSummary.php` in generation 2 | `interface/billing/sl_eob_process.php:L25` |
| Day-sheet aggregation | | `src/Billing/DaySheet/`, four classes | | `interface/billing/print_daysheet_report_num1.php:L20-L22` |

One measurement crystallises where the extraction has reached. `library/edihistory/edih_277_html.php` is 307 lines and the class extracted from it, `src/Billing/EdiHistory/Claim277Renderer.php`, is 373. The extracted renderer is larger than the legacy file that remains. That is concrete evidence that the per-segment case bodies moved out and only the loop and state-machine glue stayed behind, and it is the pattern a reader should expect from the next extraction rather than a one-off. The legacy file still drives the whole render: it constructs the code table at `library/edihistory/edih_277_html.php:L96` and then calls eleven static renderer methods between `library/edihistory/edih_277_html.php:L136` and `library/edihistory/edih_277_html.php:L229`.

## The Dependency Cycle the Extraction Created

Lifting the segment rendering out of the 277 renderer while leaving the code table behind produced a dependency edge in each direction between generation 1 and generation 3: outward at `library/edihistory/edih_277_html.php:L24` and back at `src/Billing/EdiHistory/Claim277Renderer.php:L28`. At the generation level that is a cycle, and it is the reason the extraction cannot continue in the obvious order.

The following diagram answers one question: what exactly makes this circular. Read the two solid edges as the cycle and the dashed edge as the ordinary same-generation dependency that makes it visible in a single request.

```mermaid
flowchart LR
    A["edih_277_html.php generation 1 loop and state glue"]
    B["Claim277Renderer generation 3 final strict typed"]
    C["edih_271_codes generation 1 code table 2432 lines"]
    A -- "imports and calls eleven distinct static methods" --> B
    B -- "eight signatures require this type" --> C
    A -. "constructs and passes the instance" .-> C
```

Both solid edges are verified. The generation-1 to generation-3 edge is the import at `library/edihistory/edih_277_html.php:L24`, which is `use OpenEMR\Billing\EdiHistory\Claim277Renderer;`, followed by the twelve call sites between `library/edihistory/edih_277_html.php:L136` and `library/edihistory/edih_277_html.php:L229`, which reach eleven distinct static methods because the row-class helper is called twice. The generation-3 back to generation-1 edge is the import at `src/Billing/EdiHistory/Claim277Renderer.php:L28` and the eight signatures listed in [Mechanism 3](#mechanism-3-modern-signatures-type-hinted-on-legacy-classes). The dashed edge is the instance construction at `library/edihistory/edih_277_html.php:L96`, where the legacy file builds the code table itself and hands it to the modern class as an argument.

The precise shape matters for anyone planning work here, so it is worth stating exactly. At file level this is not a cycle: it is a diamond, in which the legacy renderer depends on both the modern class at `library/edihistory/edih_277_html.php:L24` and the code table at `library/edihistory/edih_277_html.php:L96`, and the modern class depends on the code table at `src/Billing/EdiHistory/Claim277Renderer.php:L28`. At generation level it is a cycle, because generation 3 contains a hard type dependency on generation 1 while generation 1 contains a call into generation 3. The consequence is that generation 3 cannot be loaded, tested or reasoned about in isolation, which is a property of the boundary rather than of any one file. Breaking it requires moving the code table, which is why that is the first item in [extraction-roadmap.md](extraction-roadmap.md) rather than a later cleanup.

## Storage Topology

Four directories are in play. Three of them belong to the EDI history feature and one belongs to the remittance posting path, and the relationship between the last two is the fact most likely to surprise a reader.

The following diagram answers one question: which component writes and which reads each directory. Paths are shown relative to the site directory, which is the per-site root that the application resolves at runtime.

```mermaid
flowchart LR
    subgraph OUT["documents/edi outbound batch directory"]
        OUTF["dated batch files named Y-m-d-His-batch"]
    end
    subgraph HIST["documents/edi/history the EDI history root"]
        CSVD["csv the index tables"]
        TYPED["f270 f271 f276 f277 f278 f835 f997 per type stores"]
        LOGD["log dated log files named edih_log_Y-m-d"]
        ARCD["archive zip archives and the notes file"]
        TMPD["tmp upload staging"]
    end
    subgraph ERA["documents/era inbound remittance staging"]
        ERAF["uploaded remittance files"]
    end
    BATCH["BillingClaimBatch generation 2"] -- writes --> OUTF
    IDXSCAN["csv_dirfile_list in edih_csv_inc generation 1"] -- "lists as type f837" --> OUTF
    CSVIO["edih_csv_write and csv_assoc_array in edih_csv_inc generation 1"] -- "writes and reads" --> CSVD
    UPLOAD["edih_uploads generation 1"] -- writes --> TMPD
    UPLOAD -- "files land in" --> TYPED
    LOGGER["csv_edihist_log and csv_log_manage generation 1"] -- "writes and reads" --> LOGD
    ARCH["edih_archive generation 1"] -- "reads and writes" --> ARCD
    EOB["sl_eob_process and era_payments screens"] -- "writes and reads" --> ERAF
    PARSE["ParseERA generation 2"] -- reads --> ERAF
```

The base path of the history tree is settled in code rather than in documentation. `library/edihistory/edih_csv_inc.php:L329-L340` defines `csv_edih_basedir()`, and its return expression at `library/edihistory/edih_csv_inc.php:L335` is:

```php
return OEGlobalsBag::getInstance()->get('OE_SITE_DIR') . DS . 'documents' . DS . 'edi' . DS . 'history';
```

That is an expression rather than a bare constant, and the distinction matters: the path is composed at call time from a globals bag lookup and the `DS` constant that [Mechanism 4](#mechanism-4-implicit-globals-invisible-to-static-analysis) describes, so it cannot be resolved by reading a configuration file. The upload staging path is derived from it one level down by `csv_edih_tmpdir()` at `library/edihistory/edih_csv_inc.php:L348-L360`.

The setup routine confirms the parentage explicitly. `csv_setup()` at `library/edihistory/edih_csv_inc.php:L381` builds the outbound directory first, at `library/edihistory/edih_csv_inc.php:L392`, and then derives the history root as its child at `library/edihistory/edih_csv_inc.php:L393`. It creates `csv`, `archive`, `log` and `tmp` between `library/edihistory/edih_csv_inc.php:L411` and `library/edihistory/edih_csv_inc.php:L441`, and the per-type directories between `library/edihistory/edih_csv_inc.php:L485` and `library/edihistory/edih_csv_inc.php:L508`. The per-type paths themselves come from the parameter table at `library/edihistory/edih_csv_inc.php:L718-L757`, where each of the seven inbound types is `history` plus a directory named for the transaction, at `library/edihistory/edih_csv_inc.php:L743-L757`.

The outbound side is written by generation 2. `src/Billing/BillingProcessor/BillingClaimBatch.php:L65` composes the batch directory as the site directory plus `documents` plus `edi`, and `src/Billing/BillingProcessor/BillingClaimBatch.php:L64` names the file from a timestamp in `Y-m-d-His` form with a `-batch` suffix. Generation 1 then treats that same directory as its `f837` store, at `library/edihistory/edih_csv_inc.php:L738`, which is how a batch file that generation 2 wrote appears in the history index without being copied.

The index tables themselves are written and read by the same include: the writer is `edih_csv_write()` at `library/edihistory/edih_csv_inc.php:L1348`, which creates the two csv files with their header rows at `library/edihistory/edih_csv_inc.php:L1377-L1393` and appends one row per record at `library/edihistory/edih_csv_inc.php:L1419-L1429`, and the reader is `csv_assoc_array()` at `library/edihistory/edih_csv_inc.php:L1284`, which parses them at `library/edihistory/edih_csv_inc.php:L1309`. The row content it writes is built by the per-transaction builders in `library/edihistory/edih_csv_parse.php`, for instance `edih_837_csv_data()` at `library/edihistory/edih_csv_parse.php:L245`, and rendered for the operator by `library/edihistory/edih_csv_data.php:L32-L34`.

The nesting of the history root inside the outbound directory is visible in the scanning code as a special case rather than as a comment. `csv_dirfile_list()` at `library/edihistory/edih_csv_inc.php:L861` lists the files of a type's directory using the parameter table, and when the type is `f837` it has to skip the entries named `history` and `README.txt` explicitly, at `library/edihistory/edih_csv_inc.php:L885-L890`, because its own index tree is a child of the directory it is listing. The `archive` subdirectory holds the zip archives written by the archival routine, for instance at `library/edihistory/edih_archive.php:L737` and `library/edihistory/edih_archive.php:L1104`, together with the operator notes file at `library/edihistory/edih_csv_inc.php:L271`, while the logs and their own rolled-up zip stay in `log`, at `library/edihistory/edih_csv_inc.php:L153` and `library/edihistory/edih_csv_inc.php:L198`. The log writer is the subsystem-wide logging helper at `library/edihistory/edih_csv_inc.php:L87`, which appends to a file named for the current date at `library/edihistory/edih_csv_inc.php:L93-L94` and `library/edihistory/edih_csv_inc.php:L99`, and the reader is the log viewer at `library/edihistory/edih_csv_inc.php:L120`.

Now the fact that no existing document records. The remittance staging directory is `documents/era`, a sibling of `documents/edi` rather than a child of it: the EOB posting screen composes it at `interface/billing/sl_eob_process.php:L749` and creates it if absent at `interface/billing/sl_eob_process.php:L750`, and the remittance intake screen composes the same path at `interface/billing/era_payments.php:L84`. Because the history index is rooted at `documents/edi/history`, it never indexes `documents/era`. A remittance can therefore be posted to accounts receivable and be entirely absent from the EDI history browser, and a remittance can be present in the history browser's `f835` store and never have been posted, because those are two different directories populated by two different code paths.

INFERRED (confidence: High): the split is deliberate rather than accidental. Basis: the comment immediately above the `f835` entry at `library/edihistory/edih_csv_inc.php:L755` says the project keeps its own directory for these files because the existing naming scheme was confusing, and a second comment at `library/edihistory/edih_csv_inc.php:L735` says this project never writes to the outbound directory.

There is one relocation between the two worlds, and it does not connect them. The upgrade branch inside `csv_setup()` moves files into the new `f835` directory from `history/era`, a history-local directory, at `library/edihistory/edih_csv_inc.php:L491-L498`. It does not read `documents/era`. A reader who assumes the upgrade path migrated the posting directory into the index will be wrong.

The legacy tree's relationship to the database is the corollary of all this, and it is stark. VERIFIED: the entire 14,979-line tree issues exactly one database query, at `library/edihistory/edih_io.php:L737`, a single `SELECT` of three columns from `ar_session` keyed on a payment reference. Generation 1 is a filesystem and CSV subsystem that deliberately does not use the database, and its one query exists only to answer whether a given check has already been posted. Which tables the rest of the subsystem touches, and at which lifecycle stage, is the subject of [claim-lifecycle.md](claim-lifecycle.md); the artifacts each stage produces in each of these four directories are named there too.

## Three Parallel Trading Partner Loaders

Loading a trading partner is implemented three times, in three generations, and the implementation that documents the format best is the one the modern code never calls.

The first is the model. `library/classes/X12Partner.class.php` is 496 lines and extends a modern base class at `library/classes/X12Partner.class.php:L17`. It is the only place in the repository where the meaning of the envelope fields is written down next to the fields themselves: the envelope properties are declared between `library/classes/X12Partner.class.php:L24` and `library/classes/X12Partner.class.php:L40`, and the inline comments at `library/classes/X12Partner.class.php:L28-L36` and `library/classes/X12Partner.class.php:L40` name their ISA and GS element positions, including the sender and receiver interchange identifier qualifiers, the sender and receiver identifiers as ISA06 and ISA08, the acknowledgement request flag as ISA14, the usage indicator as ISA15, and the application sender codes as GS02 and GS03. Its constructor sets the implementation-guide version and four envelope defaults at `library/classes/X12Partner.class.php:L68-L72`.

VERIFIED: it is never instantiated from the modern namespace. Every `new X12Partner` in the repository is in generation-1 or screen code, at `library/classes/X12Partner.class.php:L81` and `library/classes/X12Partner.class.php:L85`, `library/classes/InsuranceCompany.class.php:L100`, `controllers/C_X12Partner.class.php:L43`, `controllers/C_X12Partner.class.php:L45`, `controllers/C_X12Partner.class.php:L61`, `controllers/C_X12Partner.class.php:L74`, `controllers/C_InsuranceCompany.class.php:L36` and `interface/billing/billing_report.php:L116`. There is no such construction anywhere under `src/`.

The second is a method on the claim model. `src/Billing/Claim.php:L162` declares `getX12Partner($x12_partner_id)` and runs `SELECT * FROM x12_partners WHERE id = ?` at `src/Billing/Claim.php:L164-L165`, returning the raw row. It is called from the constructor at `src/Billing/Claim.php:L72`, so every claim built for generation carries a partner array rather than a partner object.

The third is a static method on the eligibility builder. `src/Billing/EDI270.php:L744` declares `getX12Partner($id = 0)` and issues the same query at `src/Billing/EDI270.php:L752`, additionally assigning the result to a global at `src/Billing/EDI270.php:L753` and returning all partners when no identifier is given. It carries its own comment acknowledging the duplication, immediately below the signature at `src/Billing/EDI270.php:L746`, which reads `// @TODO move to class`.

INFERRED (confidence: Low): the two modern implementations exist because a raw row was easier to thread through the untyped generator code than an object built on a template-engine-era base class. Basis: both modern versions return arrays and consume them positionally, and the one that annotates itself asks to be moved to a class rather than to be pointed at the existing one. The confidence is Low because that evidence establishes what the authors did and not why they did it, and no comment states the reason.

Consolidating these three is a roadmap item rather than a defect, because none of them is wrong on its own terms; the cost is that the envelope semantics are documented in the copy that is not used, so a change made in the model has no effect on generated claims. The per-column configuration reference lives in [transactions.md](transactions.md), and the consolidation is sequenced in [extraction-roadmap.md](extraction-roadmap.md).

## The Paper CMS 1500 Channel

The same charge data that becomes an 837 also becomes paper, and the paper path sits inside `src/Billing/` alongside the X12 path without sharing any of it. `src/Billing/Hcfa1500.php`, 762 lines with its class at `src/Billing/Hcfa1500.php:L22`, builds the CMS-1500 claim form, and `src/Billing/HCFAInfo.php`, 78 lines with its class at `src/Billing/HCFAInfo.php:L19`, holds the row and column bookkeeping that lets the form be composed out of order according to its docblock at `src/Billing/HCFAInfo.php:L3-L5`.

What makes this a distinct output target rather than a variant of claim generation is that the batch pipeline treats the two as interchangeable tasks. Three task classes call the paper builder, at `src/Billing/BillingProcessor/Tasks/GeneratorHCFA.php:L65-L66`, `src/Billing/BillingProcessor/Tasks/GeneratorHCFA_PDF.php:L95-L96` and `src/Billing/BillingProcessor/Tasks/GeneratorHCFA_PDF_IMG.php:L48-L49`, and two more produce UB-04 institutional paper output through the procedural helpers at `src/Billing/BillingProcessor/Tasks/GeneratorUB04Form_PDF.php:L42` and `src/Billing/BillingProcessor/Tasks/GeneratorUB04NoForm.php:L44-L49`. All five extend the same abstract generator as the X12 tasks, at `src/Billing/BillingProcessor/Tasks/AbstractGenerator.php:L27`, so the choice between an X12 file and a printed form is a choice of task and nothing else. Its user-facing counterpart is the in-application help page `Documentation/help_files/cms_1500_help.php`, 120 lines, which is the only operator documentation for the paper channel.

The reason to record the channel in an architecture document at all is that a reader tracing a claim through [claim-lifecycle.md](claim-lifecycle.md) will find stages that have no X12 artifact, and the explanation is that the same charge rows were routed to paper. Nothing in the paper path parses or generates X12, so it appears in this document only here and in the generation map.

## Agreements and Contradictions with Existing Documentation

Two documents in the repository bear on this subsystem, and one that ought to is silent. The instruction this documentation set follows is to note agreements and contradictions with observed code rather than to repeat existing material, so each is cited and none is restated, corrected or edited.

### The 2016 legacy readme

`Documentation/Readme_edihistory.html` is 246 lines, carries a GPL v2 header at `Documentation/Readme_edihistory.html:L1-L22` with its copyright line at `Documentation/Readme_edihistory.html:L4`, and is dated 2016. It is the only narrative account of any part of this subsystem, and it covers one of the four generations. It is a genuinely good document about generation 1 and it remains worth reading; the notes below record only where it agrees with the code and where it does not.

**Agreement on the handled transaction set.** `Documentation/Readme_edihistory.html:L46` enumerates the types the legacy tree deals with as the 837 claim, the 835 payment, the 270 benefit inquiry, the 271 benefit response, the 276 claim status inquiry, the 277 and 277CA claim status, the 278 authorisation and the 999 acknowledgement, and explicitly records that the 824 type is not dealt with. That list matches the dispatch map in the modern file reader at `src/Billing/EdiHistory/X12File.php:L101-L102`, which maps eight functional-group codes and includes no entry for the 824. A decade later the handled set is unchanged.

**Agreement on the storage tree and how the path is derived.** `Documentation/Readme_edihistory.html:L208-L211` states that the setup routine creates the tree at `documents/edi/history` under the site directory, with the subdirectories `archive csv f270 f271 f276 f277 f278 f835 f997 log tmp`, and `Documentation/Readme_edihistory.html:L217` credits `csv_edih_basedir()` with deriving the path. Both hold: the function is at `library/edihistory/edih_csv_inc.php:L329-L340`, the path is composed at `library/edihistory/edih_csv_inc.php:L335`, and the directory creation is at `library/edihistory/edih_csv_inc.php:L411-L441` and `library/edihistory/edih_csv_inc.php:L485-L508`. The document also describes the CSV file naming at `Documentation/Readme_edihistory.html:L213-L214`, which matches the parameter table at `library/edihistory/edih_csv_inc.php:L738-L757`.

**Internal self-contradiction, resolved by code.** The same document gives the storage location twice and differently. `Documentation/Readme_edihistory.html:L52` says the files and tables live under the site directory in `edi/history/`, while `Documentation/Readme_edihistory.html:L208-L211` says `documents/edi/history`. These cannot both be right. The code settles it at `library/edihistory/edih_csv_inc.php:L335`, which composes `documents` into the path: the later passage is correct and the earlier one omits a directory level. This is the clearest single illustration of why the source-of-truth ordering in [README.md](README.md) puts prose last.

**Agreement on the upgrade behaviour.** `Documentation/Readme_edihistory.html:L54-L55` describes what the upgrade does, in a passage this document paraphrases rather than quotes because the source text contains a typographical error. Its account is that existing index files are renamed with an `old_` prefix, that existing remittance files are moved into a newly created `f835` directory, that the setup function in the legacy include performs this, that the only other action is directory creation, and that eleven named directories must not be deleted. The code agrees on each point: the renaming loop is at `library/edihistory/edih_csv_inc.php:L465-L478`, the relocation into `f835` is at `library/edihistory/edih_csv_inc.php:L491-L498`, and the directory creation is at the anchors above. One clarification belongs with it, because the passage is easy to over-read: the files that are relocated come from a history-local `era` directory, at `library/edihistory/edih_csv_inc.php:L491`, and not from the `documents/era` staging directory the posting path uses, as [Storage Topology](#storage-topology) sets out.

**Contradiction with the repository, on the file inventory.** `Documentation/Readme_edihistory.html:L160-L181` inventories the legacy scripts. Its list of the thirteen top-level files at `Documentation/Readme_edihistory.html:L162-L174` matches the repository exactly. Its list of the `codes/` directory at `Documentation/Readme_edihistory.html:L178-L180` names three files; the directory holds four. The fourth is `library/edihistory/codes/code_formatter.ods`, which is not PHP and contributes no lines to the 14,979-line total, but which is present and is not mentioned.

**Agreement on the operator interface.** `Documentation/Readme_edihistory.html:L88-L103` describes each tab of the EDI history screen: uploading and processing new files at `Documentation/Readme_edihistory.html:L88`, the index tables and the per-encounter lookup at `Documentation/Readme_edihistory.html:L94`, the logs and notes at `Documentation/Readme_edihistory.html:L99`, and the ageing-period archive with its report, archive and restore actions at `Documentation/Readme_edihistory.html:L102`. All four tabs are still there, and all four are still dispatched through the display functions of `library/edihistory/edih_io.php`: uploading and processing at `library/edihistory/edih_io.php:L282` and `library/edihistory/edih_io.php:L177`, the index tables and the per-encounter lookup at `library/edihistory/edih_io.php:L659` and `library/edihistory/edih_io.php:L708`, the logs and notes at `library/edihistory/edih_io.php:L39-L50` and `library/edihistory/edih_io.php:L77`, and the archive report, archive and restore at `library/edihistory/edih_io.php:L130`, `library/edihistory/edih_io.php:L153` and `library/edihistory/edih_io.php:L110`. Those functions delegate to `library/edihistory/edih_uploads.php`, `library/edihistory/edih_csv_data.php`, `library/edihistory/edih_archive.php` and the log and notes helpers in `library/edihistory/edih_csv_inc.php` at `library/edihistory/edih_csv_inc.php:L116`, `library/edihistory/edih_csv_inc.php:L149` and `library/edihistory/edih_csv_inc.php:L266`, all of which are still required by the same screen at `interface/billing/edih_main.php:L71-L86`.

**A methodological agreement worth saying out loud.** Twice the author labels his own reasoning as an assumption rather than as a fact: at `Documentation/Readme_edihistory.html:L112`, about how remittance files are grouped internally, and at `Documentation/Readme_edihistory.html:L115`, about multiple interchange envelopes appended into one file being of the same type. That is precisely the verified-versus-inferred discipline this documentation set adopts, practised in this very subsystem a decade earlier. It is also the reason the document remains usable: its factual claims and its guesses can be told apart.

**One defect claim and one patch, both carried elsewhere.** The document records a single suspected defect about the value the claim generator writes into the BHT segment, at `Documentation/Readme_edihistory.html:L95`, and proposes a concrete patch for it at `Documentation/Readme_edihistory.html:L220-L235`. Both are paraphrased rather than quoted here, because the first of those passages also contains a typographical error in the product name. The claim and the fate of the patch belong to [defect-candidates.md](defect-candidates.md) and are not duplicated in this document.

### The architecture stub

`Documentation/SystemArchitecture.txt` is two lines long. `Documentation/SystemArchitecture.txt:L1-L2` says the system architecture description is on an external wiki and gives the address. It contains no billing, EDI, X12, claim or remittance content of any kind. That is the evidence that there is no in-repository architecture document to extend, and therefore the reason this file is a creation rather than an amendment to something existing.

### The developer guide

`Documentation/api/DEVELOPER_GUIDE.md` is the repository's engineer-facing guide and it is completely silent on this subsystem. VERIFIED: a word-boundary search of it for billing, x12, edi, 837, 835 and remittance returns nothing at all, and a case-insensitive search for claim returns only an unrelated JSON Web Token usage at `Documentation/api/DEVELOPER_GUIDE.md:L333-L334`, where the local variable holding decoded token claims is read. The silence is not an oversight to be noted in passing: it is the documentation gap this set exists to close, and the honest finding required by the instruction to reconcile against existing documentation is that there was nothing here to reconcile with.

Its value to this document is entirely as a style model, and that is how it was used. The opening convention of a single H1 followed by a one-line purpose statement is taken from `Documentation/api/DEVELOPER_GUIDE.md:L1-L3`, the table-of-contents form with sub-items indented by four spaces from `Documentation/api/DEVELOPER_GUIDE.md:L5-L48`, and the closing attribution block with its last-updated stamp and licence line from the same source and from `Documentation/api/README.md`.

### Comment versus code contradictions inside the subsystem

[README.md](README.md) records that this documentation run catalogued thirteen instances in the subsystem where descriptive text contradicts the code it describes, and defers the remainder to this document and to [defect-candidates.md](defect-candidates.md). Five of them are in the legacy tree, and they are all of one kind: a file whose own docblock names a different file than the one it is in. They are recorded here because they are the direct evidence for the source-of-truth ordering, not because any of them changes behaviour.

| Anchor | What the descriptive text says | What the file is |
|--------|-------------------------------|------------------|
| `library/edihistory/edih_277_html.php:L4` | Names the file with the extension repeated twice | `library/edihistory/edih_277_html.php` |
| `library/edihistory/edih_277_html.php:L27-L28` | Describes the function below it as producing a display of the 271 eligibility report for a patient | The function at `library/edihistory/edih_277_html.php:L40` is the 277 claim-status transaction renderer, and its own parameter is the parsed 277 file object at `library/edihistory/edih_277_html.php:L35` |
| `library/edihistory/edih_278_html.php:L4` | Names the file as a 278 parsing test script | The 278 renderer, whose entry point is at `library/edihistory/edih_278_html.php:L39` |
| `library/edihistory/edih_835_html.php:L4` | Names the file with a `new_` prefix it does not carry | The 835 renderer, whose summary builder is at `library/edihistory/edih_835_html.php:L25` |
| `library/edihistory/codes/edih_271_code_class.php:L4` | Names the file as a 271 codes test script | The 2,432-line production code table, class declared at `library/edihistory/codes/edih_271_code_class.php:L26` |
| `library/edihistory/codes/edih_997_codes.php:L4` | Names the file as a 997 codes test script | The acknowledgement code text used by the error extractor, at `library/edihistory/codes/edih_997_codes.php:L37` |

INFERRED (confidence: High): these five self-naming errors are copy-and-rename artifacts from the files' original development rather than references to files that once existed. Basis: no file bearing any of the five asserted names has ever existed at any path in the repository's history; four of the five asserted names carry a `test_` or `new_` prefix, which is the shape of a scratch file renamed into place; and the original contribution that created this tree demonstrably did ship scratch files of exactly that shape next to the production ones, because `library/edihistory/test_edih_835_accounting.php` and `library/edihistory/test_edih_sftp_files.php` were both added by that same commit and both removed later.

The second row deserves separate mention, because it is the one a reader can be actively misled by. A docblock describing a 271 eligibility report at `library/edihistory/edih_277_html.php:L27-L28` sits directly above the 277 claim-status renderer declared at `library/edihistory/edih_277_html.php:L40`, in a file that is one of the two halves of the dependency cycle documented above. Anyone reading that header to decide what the file does will get the wrong transaction type. Nothing about it is repaired by this documentation run: the constraint on this run is that no source file is edited at all, including its comments, and correcting a misleading docblock would be a code change.

## Inference Register

Every claim in this document is either traced in code and carries a citation, or is an inference about intent and is labelled as one. The inferences are collected here so that the whole inferential surface of the document can be read at a glance, and so that a future reader can see immediately which statements are provisional. Nothing in this table is a behavioural claim; each is a statement about why the code is the way it is.

| Inference | Confidence | Basis | Related citation |
|-----------|-----------:|-------|------------------|
| `declare(strict_types=1)` marks authorship era rather than a per-file stylistic choice | High | It partitions 46 files exactly along directory boundaries with no mixed directory, which a per-file preference would not produce | `src/Billing/EdiHistory/X12File.php:L20`, `src/Billing/EdiHistory/EdiFormat.php:L22` |
| The non-finality of `RemitAccounting` is an omission rather than a decision | Medium | The two final generation-3 EDI classes both also declare private constructors as a deliberate no-extension stance, while this class has only static members | `src/Billing/EdiHistory/RemitAccounting.php:L17`, `src/Billing/EdiHistory/Claim277Renderer.php:L32-L34`, `src/Billing/EdiHistory/EdiFormat.php:L28-L30` |
| `Recorder` is intended as the destination for accounts-receivable recording generally, not only for adjustments | Medium | The deprecation notice names `recordActivity` without qualification, and the method it deprecates is one of several posting helpers in the same class | `src/Billing/SLEOB.php:L221` |
| `code_formatter.ods` is the source from which the three PHP code tables were generated | Medium | Its name and its placement in the same directory as exactly those three files, with no code in the repository referencing it | `library/edihistory/codes/code_formatter.ods` |
| The `class_alias` shim was retained to avoid editing legacy call sites, not because the old name is preferred | High | The shim's own docblock states the file remains so procedural callers keep resolving the symbol, and the lifted class carries the reciprocal note | `library/edihistory/edih_x12file_class.php:L3-L17`, `src/Billing/EdiHistory/X12File.php:L6-L9` |
| The defensive redefinition of the `DS` constant in the operator screen exists because include order could not be guaranteed | High | Both the screen and the include it later loads guard the definition identically, which is the shape of two authors each protecting against the other running first | `interface/billing/edih_main.php:L64-L66`, `library/edihistory/edih_csv_inc.php:L77-L79` |
| The separation of the remittance staging directory from the history index is deliberate rather than accidental | High | A comment above the `f835` parameter entry says the project keeps its own directory because the existing naming scheme was confusing, and a second says the project never writes to the outbound directory | `library/edihistory/edih_csv_inc.php:L755`, `library/edihistory/edih_csv_inc.php:L735` |
| The two modern trading-partner loaders exist because a raw row was easier to thread through untyped generator code than an object built on a template-engine-era base class | Low | Both modern versions return arrays and consume them positionally, and the one that annotates itself asks to be moved to a class rather than pointed at the existing model | `src/Billing/Claim.php:L164-L165`, `src/Billing/EDI270.php:L746`, `library/classes/X12Partner.class.php:L17` |
| The five self-naming errors in legacy docblocks are copy-and-rename artifacts from original development rather than references to files that once existed | High | No file bearing any of the five asserted names has ever existed at any path in the repository's history, four of the five carry a `test_` or `new_` prefix, and the contribution that created this tree did ship scratch files of that shape beside the production ones | `library/edihistory/edih_278_html.php:L4`, `library/edihistory/edih_835_html.php:L4`, `library/edihistory/codes/edih_271_code_class.php:L4` |

Nine inferences, of which five are High confidence, three are Medium and one is Low. Every other statement in this document is verified against the cited range at the recorded commit.

---
## Documentation Attribution

### Authorship

This document was produced by reading the revenue-cycle and X12 EDI source of OpenEMR, its autoload configuration and its test tree at branch `master`, head commit `b7a7e690e419de3451740f995b768a8e8e5fba87`. It builds on the collective work of the OpenEMR community, and in particular on the 2016 account of the legacy EDI history tree at `Documentation/Readme_edihistory.html:L4`, which is cited here for its agreements and contradictions with observed code and is neither corrected nor superseded.

### Method

The generation model rests on one mechanical marker, `declare(strict_types=1)`, so that the classification can be re-derived with a single search rather than taken on trust. File and line counts were enumerated at the recorded commit. Facts were established from executable code first, schema definitions second, tests third, and comments last and only as evidence of intent, for the reasons set out in [README.md](README.md). No code was executed and no tests were run: PHP and Composer are not installed in the authoring environment, so every claim here rests on static reading. No source file, test, manifest or gate configuration was modified in producing it.

### Contributing

OpenEMR is an open-source project. To improve these documents:

- **Report Issues:** [GitHub Issues](https://github.com/openemr/openemr/issues)
- **Discuss:** [Community Forum](https://community.open-emr.org/)
- **Submit Changes:** [Pull Requests](https://github.com/openemr/openemr/pulls)

**Last Updated:** July 2026
**License:** GPL v3

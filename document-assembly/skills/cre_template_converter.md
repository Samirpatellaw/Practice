---
name: cre-template-converter
description: >
  Convert any commercial real estate loan document template into a maximum template
  using the [PLACEHOLDER] and [IF: LABEL] syntax. Use this skill whenever the user
  uploads or pastes a CRE loan document template — regardless of the firm that drafted
  it — and asks to convert it, standardize it, or prepare it for automated document
  generation. Triggers include: "convert this template", "prepare this for automation",
  "turn this into a maximum template", "this is a note/mortgage/guaranty template",
  or any upload of a .docx legal form with blanks, NTDs, or bracketed instructions.
  Covers promissory notes, mortgages, deeds of trust, guaranty agreements, commitment
  letters, assignment of leases, loan agreements, environmental indemnities, and all
  other standard CRE closing documents.
---

# CRE Template → Maximum Template Converter

## Overview

Commercial real estate loan documents share a deep common vocabulary regardless of
which firm drafted them. A promissory note from Greenberg Traurig and one from
Latham & Watkins will have different surface syntax — different ways of expressing
blanks, conditions, and alternatives — but they address the same underlying concepts:
borrower identity, loan amount, interest rate, maturity, prepayment, guaranty, collateral.

This skill reads any CRE loan template, recognizes every drafting convention used to
express variables and conditionals, and converts them into the standard maximum template
format: `[PLACEHOLDER]` for variables and `[IF: LABEL]`...`[END IF: LABEL]` for
conditional sections.

The output is a `.docx` file ready for use with `generate_documents.py`.

---

## INVARIANT RULES

**These rules are absolute. They override all other guidance.**

1. **Never rewrite substantive language.** The firm's template text is approved form
   language. Only convert the drafting apparatus — blanks, NTDs, conditions, alternatives.
   Every word of substantive legal text stays exactly as written.

2. **When a blank cannot be confidently mapped, coin a new placeholder.** Never force
   a bad match. A new placeholder named `[LEASE TERMINATION NOTICE PERIOD]` is better
   than a wrong mapping to `[TERM]`.

3. **When a condition cannot be confidently mapped, coin a new label.** Follow the
   labeling conventions in this skill. Report every new label coined.

4. **Never silently drop content.** If a section exists in the original template —
   even if you cannot determine what condition controls it — preserve it and flag it
   with a `[REVIEW:]` comment.

5. **Preserve all formatting.** The firm's indentation, numbering, spacing, fonts,
   and paragraph styles are part of the document. Do not reformat.

6. **Produce a conversion report.** Every mapping decision, every new placeholder or
   label coined, every item flagged for review must appear in the report.

7. **The output must be immediately usable.** After conversion, `generate_documents.py`
   should be able to process the template without manual XML fixes.

---

## Input Recognition — Drafting Convention Taxonomy

Law firms express the same underlying concepts in many different ways. The first task
is recognizing all of them.

### Category 1: Simple Blanks (Variable Placeholders)

These all mean "insert a deal-specific value here":

| Convention | Examples |
|---|---|
| Bracketed descriptor | `[Borrower's Name]`, `[Insert Name of Borrower]`, `[●]`, `[__]` |
| Underline blanks | `________________`, `______` |
| Highlighted/colored text | Yellow-highlighted `BORROWER NAME` |
| ALL CAPS with no brackets | `BORROWER NAME`, `LOAN AMOUNT` (in context clearly a blank) |
| NTD inline | `[NTD: Insert borrower legal name]`, `NTD: confirm rate` |
| Footnote NTD | Footnote saying "Insert closing date" anchored to a blank |
| Word comment NTD | Comment balloon: "Partner: confirm folio number" |
| Double brackets | `[[Borrower]]`, `((Amount))` |
| Curly braces | `{BorrowerName}`, `{LoanAmount}` |

### Category 2: Conditional Sections (If/Then Structures)

These all mean "include or exclude this section based on deal parameters":

| Convention | Examples |
|---|---|
| Bracketed instruction | `[Include if construction loan]`, `[Delete if not applicable]` |
| Option labels | `[OPTION A — Fixed Rate]` ... `[OPTION B — Floating Rate]` |
| Alternative labels | `[ALTERNATIVE 1]` ... `[ALTERNATIVE 2]` |
| NTD block instruction | `NTD: Delete this section if guarantor is an individual` |
| Word comment on section | Comment: "Include only if cross-default applies" |
| Partner note | `[Partner note: use if hotel property]` |
| Highlighted block | Entire section highlighted yellow with "delete if N/A" |
| Strikethrough alternatives | Two versions, one struck through as the "delete" signal |
| `[___] OR [___]` inline | `[does] OR [does not]` require UCC filing |
| `[his/her/its]` style | Article/pronoun alternatives within a sentence |

### Category 3: Repeating Blocks (For Each Structures)

These all mean "repeat this block once per item in a collection":

| Convention | Examples |
|---|---|
| Explicit repeat instruction | `[Repeat for each Guarantor]`, `[Add additional signature blocks as needed]` |
| Numbered alternatives | `[Guarantor 1]`...`[Guarantor 2]`...`[Guarantor 3]` |
| NTD with collection | `NTD: Add one block per property in the portfolio` |
| Schedule reference | "See Schedule A for all Folio Numbers" (signals a FOR EACH) |

### Category 4: Attorney-Fill Items (Double-Bracketed)

These are items not driven by the CAS — the attorney supplies them at closing:

| Convention | Examples |
|---|---|
| Double brackets | `[[Closing Date]]`, `[[Legal Description]]` |
| "To be provided" | `[to be provided]`, `[TBP]`, `[TBD]` |
| NTD "external input" | `NTD: Attorney to confirm signing authority` |
| Recording information | Book/page numbers, instrument numbers |
| Legal descriptions | Full property legal descriptions |

---

## Conversion Workflow

### Phase 1: Read and Inventory the Template

1. Open and read the full template
2. Extract all text including headers, footers, signature blocks, schedules, exhibits
3. Build an inventory list of every drafting element found, categorized by type above
4. Note the firm name and document type if identifiable — this informs which concept
   matrix entries to prioritize

Print a preliminary inventory:
```
Document type: [identified type]
Firm: [if identifiable]
Simple blanks found: [count]
Conditional sections found: [count]
Repeating blocks found: [count]
Attorney-fill items found: [count]
Word comments/NTDs found: [count]
```

### Phase 2: Map Each Item to the CRE Concept Matrix

Walk through the concept matrix below. For each item in the inventory:

1. **Match to a standard concept** — find the closest entry in the matrix
2. **Assign the standard placeholder or label** — use the matrix's canonical form
3. **Record the mapping** — for the conversion report
4. **If no match exists** — coin a new placeholder following the naming conventions,
   record it as NEW in the report

**Confidence levels:**
- **CERTAIN** — unambiguous match (e.g., "Insert Borrower's legal name" → `[BORROWER NAME]`)
- **PROBABLE** — strong contextual match, note in report
- **UNCERTAIN** — flag with `[REVIEW:]` comment, do not force a mapping

### Phase 3: Execute Conversions

For each inventoried item, perform the appropriate conversion:

**Simple blank → Placeholder:**
Replace the blank/NTD/highlight with `[PLACEHOLDER NAME]`
preserving all surrounding text and formatting exactly.

**Conditional section → IF block:**
- Wrap the section: `[IF: LABEL]` before first paragraph, `[END IF: LABEL]` after last
- Delete the instruction/NTD/comment that expressed the condition
- If the template shows two alternatives (Option A / Option B), wrap each:
  ```
  [IF: FIXED RATE]
  ... fixed rate language ...
  [END IF: FIXED RATE]
  [IF: FLOATING OR VARIABLE RATE]
  ... floating rate language ...
  [END IF: FLOATING OR VARIABLE RATE]
  ```

**Repeating block → FOR EACH:**
- Wrap: `[FOR EACH: COLLECTION NAME]` ... `[END FOR EACH: COLLECTION NAME]`
- Replace item-level blanks with item-scoped placeholders: `[EG NAME]`, `[PG NAME]`,
  `[FOLIO NUMBER]`, etc.
- Delete any "repeat as needed" instructions

**Attorney-fill item → Double-bracketed placeholder:**
Use `[[PLACEHOLDER NAME]]` — double brackets signal "not in CAS, attorney fills in":
`[[CLOSING DATE]]`, `[[LEGAL DESCRIPTION]]`, `[[RECORDING INFORMATION]]`

**Word comment NTD → Word comment with REVIEW: prefix:**
Preserve the comment but rewrite it with the standard prefix:
`REVIEW: [original NTD text — attorney to confirm]`

### Phase 4: Validate the Output

Check:
- [ ] No original blanks (`______`, `[●]`, `[Insert...]`) remain unconverted
- [ ] No NTDs remain as raw text in the document body
- [ ] Every `[IF: LABEL]` has a matching `[END IF: LABEL]`
- [ ] Every `[FOR EACH: COLLECTION]` has a matching `[END FOR EACH: COLLECTION]`
- [ ] No substantive legal text was modified
- [ ] All formatting preserved
- [ ] Conversion report complete

### Phase 5: Produce the Conversion Report

The report is a markdown document delivered alongside the converted template. Structure:

```markdown
# Conversion Report — [Document Type] — [Firm if known]
Date: [date]

## Summary
- Simple blanks converted: [count]
- Conditional sections converted: [count]
- Repeating blocks converted: [count]
- Attorney-fill items tagged: [count]
- New placeholders coined: [count]
- New condition labels coined: [count]
- Items flagged for review: [count]

## Placeholder Mappings
| Original | Converted To | Confidence | Notes |
|---|---|---|---|
| "[Insert Borrower Name]" | [BORROWER NAME] | CERTAIN | |
| "[●]" (line 47) | [$LOAN AMOUNT] | PROBABLE | Context: "principal amount of" |
| "[___]" (line 203) | [LEASE TERMINATION NOTICE PERIOD] | NEW | No matrix match — coined |

## Condition Label Mappings
| Original Instruction | Label Assigned | Confidence | Notes |
|---|---|---|---|
| "[Include if hotel property]" | [IF: STAR REPORTS] | CERTAIN | |
| "[Option A — Fixed]" | [IF: FIXED RATE] | CERTAIN | |
| "[Delete if N/A]" (§14) | [IF: CROSS DEFAULT] | PROBABLE | Section heading confirms |
| "[Include per Partner note]" (§8) | REVIEW | UNCERTAIN | Cannot determine condition |

## New Placeholders Coined
These do not exist in the standard namespace and must be added to
PLACEHOLDER_VALUES in generate_documents.py before running this template:
- [LEASE TERMINATION NOTICE PERIOD] — days notice required under §14(b)
- [REPLACEMENT RESERVE MONTHLY] — monthly deposit amount per §17(c)

## New Condition Labels Coined
These must be added to CONDITION_KEEPS in generate_documents.py:
- HOTEL PROPERTY — True if property is a hotel/hospitality asset
- REPLACEMENT RESERVE — True if monthly replacement reserve required

## Items Flagged for Review
| Location | Flag | Reason |
|---|---|---|
| §8 paragraph 3 | REVIEW | Partner note condition unclear |
| Exhibit B | REVIEW | Schedule format — may need FOR EACH |
| Sig block 3 | REVIEW | Three-tier signing chain — verify structure |
```

---

## CRE Concept Matrix

The master reference for mapping template language to standard placeholders and labels.
Organized by document section. Use this in Phase 2.

---

### PARTIES

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Borrower legal name | `[BORROWER NAME]` | "Borrower", "Maker", "Mortgagor", "Grantor", "Debtor" |
| Borrower name ALL CAPS | `[BORROWER NAME CAPS]` | Used in defined term line, signature header |
| Borrower state of organization | `[BORROWER STATE]` | "a [State] limited liability company" |
| Borrower entity type | `[BORROWER ENTITY TYPE]` | "limited liability company", "corporation", "limited partnership" |
| Borrower street address | `[BORROWER STREET ADDRESS]` | Notice address for borrower |
| Borrower city | `[BORROWER CITY]` | Notice address city |
| Borrower state (mailing) | `[BORROWER MAILING STATE]` | Notice address state |
| Borrower zip | `[BORROWER ZIP]` | Notice address zip |
| Lender name | `[LENDER NAME]` | "Lender", "Bank", "Mortgagee", "Beneficiary", "Secured Party" |
| Lender address | `[LENDER ADDRESS]` | Notice address for lender |
| Relationship contact | `[RELATIONSHIP CONTACT]` | Primary contact, relationship manager name |
| Existing lender | `[EXISTING LENDER]` | Lender being paid off in refinance |
| Guarantor name (entity) | `[EG NAME]` | Entity guarantor — inside FOR EACH loop |
| Guarantor name (personal) | `[PG NAME]` | Personal guarantor — inside FOR EACH loop |
| Guarantor name (trust) | `[TG NAME]` | Trust guarantor — inside FOR EACH loop |
| Guarantor state | `[EG STATE]` / `[GUARANTOR STATE]` | State of organization of entity guarantor |
| Guarantor entity type | `[EG ENTITY TYPE]` / `[GUARANTOR ENTITY TYPE]` | Entity type of guarantor |
| Guarantor address | `[GUARANTOR ADDRESS]` | Notice address for guarantor |
| Trustee name | `[TRUST TRUSTEE NAME]` | Trustee of trust guarantor |
| Trust agreement date | `[TRUST TRUST DATE]` | Date of trust agreement |
| Bad boy guarantor name | `[BAD BOY GUARANTOR NAME]` | Carveout/non-recourse guarantor |
| Tenant name | `[TENANT]` | Key tenant name (major lease) |
| Franchisor | `[FRANCHISOR NAME]` | Franchise agreement counterparty |
| Prepared by | `[PREPARED BY]` | Attorney preparing document (recording instruments) |
| Grammatical article (a/an) | `[a/an]` | Article before entity type or state name |

---

### LOAN ECONOMICS

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Loan amount | `[$LOAN AMOUNT]` | "principal amount", "face amount", "original principal" |
| Loan amount (words) | `[$LOAN AMOUNT WORDS]` | Spelled-out dollar amount |
| Loan amount doubled | `[$LOAN AMOUNT DOUBLED]` | Mortgage recording amount (2x loan) |
| Loan amount doubled (words) | `[$LOAN AMOUNT DOUBLED WORDS]` | Spelled-out 2x amount |
| Loan amount (second tranche) | `[$LOAN AMOUNT 2]` | Bifurcated facility second component |
| Future advance amount | `[$FUTURE ADVANCE AMOUNT]` | Advance on existing loan |
| Future advance amount (words) | `[$FUTURE ADVANCE AMOUNT WORDS]` | Spelled out |
| Holdback amount | `[$HOLDBACK AMOUNT]` | Amount withheld at closing |
| Holdback amount (words) | `[$HOLDBACK AMOUNT WORDS]` | Spelled out |
| Holdback conditions | `[HOLDBACK CONDITIONS]` | Conditions for holdback release |
| Initial funding | `[$INITIAL FUNDING]` | Amount funded at closing (loan minus holdback) |
| Existing loan balance | `[$EXISTING BALANCE]` | Balance of loan being modified |
| Existing loan balance (words) | `[$EXISTING BALANCE WORDS]` | Spelled out |
| Original loan amount | `[$ORIGINAL LOAN AMOUNT]` | Original face amount of loan being modified |
| Commitment fee percentage | `[COMMITMENT FEE %]` | Fee as percentage |
| Commitment fee (words) | `[COMMITMENT FEE % WORDS]` | Spelled out with parenthetical |
| Remaining proceeds use | `[REMAINING PROCEEDS USE]` | Use of loan proceeds beyond primary purpose |

---

### INTEREST RATE

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Fixed rate | `[FIXED RATE]` | "per annum rate of ___%" |
| Fixed rate (words) | `[FIXED RATE WORDS]` | Spelled out with parenthetical |
| Initial fixed rate term | `[INITIAL FIXED TERM]` | Years before first rate adjustment |
| Initial fixed rate term (words) | `[INITIAL FIXED TERM WORDS]` | Spelled out |
| Initial fixed rate term date | `[INITIAL FIXED TERM DATE]` | Calendar date of first adjustment |
| Rate index | `[INDEX]` | "SOFR", "Prime Rate", "CME Term SOFR", "WSJ Prime" |
| Margin over index | `[MARGIN RATE]` | "plus ___ percent", spread |
| Margin (words) | `[MARGIN RATE WORDS]` | Spelled out |
| Floor rate | `[FLOOR RATE]` | Minimum rate regardless of index movement |
| Floor rate (words) | `[FLOOR RATE WORDS]` | Spelled out |
| Rate spread (swap) | `[RATE SPREAD]` | Spread component in swap structure |
| Rate structure | `[RATE STRUCTURE]` | "fixed", "floating", "variable", "swap" |
| Number of rate adjustments | `[RATE ADJUSTMENTS]` | Count of adjustment events over term |

---

### TERM AND REPAYMENT

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Loan term | `[TERM]` | "term of ___ years", "maturity" |
| Loan term (words) | `[TERM WORDS]` | Spelled out with parenthetical |
| Amortization period | `[AMORTIZATION PERIOD]` | "amortized over ___ years" |
| Amortization period (words) | `[AMORTIZATION PERIOD WORDS]` | Spelled out |
| Interest-only period (months) | `[IO MONTHS]` | "interest only for ___ months" |
| Interest-only period (words) | `[IO MONTHS WORDS]` | Spelled out |
| Interest-only end date | `[IO END DATE]` | Calendar date IO period ends |
| Maturity date | `[MATURITY DATE]` | Final payment date |
| First payment date | `[PAYMENT DATE]` | Date of first scheduled payment |
| Extension term | `[EXTENSION TERM]` | Length of extension option |
| Extension term (words) | `[EXTENSION TERM WORDS]` | Spelled out |
| Extension period unit | `[EXTENSION PERIOD]` | "months" or "years" |
| Extension options count | `[EXTENSION OPTIONS]` | Number of extension options |
| Extension options (words) | `[EXTENSION OPTIONS WORDS]` | Spelled out |
| Extended maturity date | `[[EXTENDED MATURITY DATE]]` | Attorney computes at closing |
| Construction term | `[CONSTRUCTION TERM]` | Construction phase length (months) |
| Construction term (words) | `[CONSTRUCTION TERM WORDS]` | Spelled out |
| Permanent term | `[PERMANENT TERM]` | Permanent phase length (years) |
| Permanent term (words) | `[PERMANENT TERM WORDS]` | Spelled out |

---

### PREPAYMENT

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Prepayment penalty percentage | `[PREPAYMENT PERCENTAGE]` | "___ percent", "___%" |
| Prepayment percentage (words) | `[PREPAYMENT PERCENTAGE WORDS]` | Spelled out |
| Prepayment penalty period | `[PREPAYMENT TERM]` | Years penalty applies |
| Prepayment penalty period (words) | `[PREPAYMENT TERM WORDS]` | Spelled out |

---

### PROPERTY AND COLLATERAL

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Property description | `[PROPERTY DESCRIPTION]` | Brief narrative description of property |
| Property address | `[PROPERTY ADDRESS]` | Street address |
| Property county | `[PROPERTY COUNTY]` | County of property situs |
| Property county (caps) | `[PROPERTY COUNTY CAPS]` | ALL CAPS version for recording instruments |
| Folio number | `[FOLIO NUMBER]` | Tax parcel ID, APN, folio — inside FOR EACH |
| Legal description | `[[LEGAL DESCRIPTION]]` | Full metes-and-bounds — attorney provides |
| Recording book number | `[[RECORDS BOOK NUMBER]]` | Recording reference — attorney provides |
| Recording mortgage page | `[[RECORDS MTG PAGE NUMBER]]` | Page number — attorney provides |
| Recording ALR page | `[[RECORDS ALR PAGE NUMBER]]` | Assignment of leases recording page |
| Original closing date | `[ORIGINAL CLOSING DATE]` | For modifications: date of original loan |
| Cross-default borrower | `[CROSS BORROWER]` | Borrower on cross-defaulted loan |
| Cross-collateral property | `[CROSS PROPERTY]` | Property securing cross-defaulted loan |

---

### COVENANTS

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| DSCR covenant ratio | `[DSCR COVENANT RATE]` | Minimum required DSCR (e.g., 1.25x) |
| DSCR actual ratio | `[DSCR RATIO]` | Actual deal DSCR from CAS |
| DSCR min management fee | `[DSCR MIN MGMT FEE]` | Minimum mgmt fee assumed in DSCR calc |
| DSCR min mgmt fee (words) | `[DSCR MIN MGMT FEE WORDS]` | Spelled out |
| DSCR start date | `[DSCR START DATE]` | Specific date DSCR testing begins |
| DSCR adjustment basis | `[DSCR ADJUSTMENT BASIS]` | How DSCR is adjusted |
| LTV maximum ratio | `[LTV RATIO]` | Maximum loan-to-value percentage |
| LTV maximum (words) | `[LTV RATIO WORDS]` | Spelled out |
| LTV maintenance ratio | `[LTV MAINTENANCE RATIO]` | LTV triggering cure obligation |
| LTV maintenance (words) | `[LTV MAINTENANCE RATIO WORDS]` | Spelled out |
| Loan-to-purchase ratio | `[LTP RATIO]` | LTP for acquisitions |
| Loan-to-purchase (words) | `[LTP RATIO WORDS]` | Spelled out |
| Loan-to-cost ratio | `[LTC RATIO]` | LTC for construction |
| Loan-to-cost (words) | `[LTC RATIO WORDS]` | Spelled out |
| Reserve fee percentage | `[RESERVE FEE]` | Capital reserve percentage of gross revenues |
| Reserve fee (words) | `[RESERVE FEE WORDS]` | Spelled out |
| Reserve months | `[RESERVE MONTHS]` | Months of P&I in debt service reserve |
| Reserve months (words) | `[RESERVE MONTHS WORDS]` | Spelled out |
| Stabilization reserve amount | `[$STABILIZATION RESERVE AMOUNT]` | Dollar amount of stabilization reserve |
| Interest reserve months | `[INTEREST RESERVE MONTHS]` | Months of interest reserve (construction) |
| Interest reserve months (words) | `[INTEREST RESERVE MONTHS WORDS]` | Spelled out |
| Equity requirement | `[EQUITY REQUIREMENT]` | Borrower equity percentage (construction) |
| Equity requirement (words) | `[EQUITY REQUIREMENT WORDS]` | Spelled out |

---

### BANK PERSONNEL AND DATES

| Concept | Standard Placeholder | Common Source Language |
|---|---|---|
| Senior VP name | `[SENIOR VP NAME]` | Approving officer name |
| Bank officer name | `[BANK OFFICER NAME]` | RM or relationship officer |
| Bank officer position | `[BANK OFFICER POSITION]` | Title abbreviation: SVP, VP, etc. |
| Commitment date | `[COMMITMENT DATE]` | Date of commitment letter |
| Closing date | `[CLOSING DATE]` | Target closing date |

---

### SIGNATURE BLOCKS

| Concept | Standard Placeholder | Notes |
|---|---|---|
| Borrower signer 1 name | `[BORROWER SIG 1 NAME]` | Individual signatory |
| Borrower signer 1 name (caps) | `[BORROWER SIG 1 NAME CAPS]` | For printed name line |
| Borrower signer 1 title | `[BORROWER SIG 1 TITLE]` | Title of signatory |
| Borrower signer 1 entity name | `[BORROWER SIG 1 ENTITY NAME]` | Signing entity (if entity signer) |
| Borrower signer 1 entity type | `[BORROWER SIG 1 ENTITY TYPE]` | Entity type of signing entity |
| Borrower signer 1 entity state | `[BORROWER SIG 1 ENTITY STATE]` | State of organization |
| Borrower signer 1 relationship | `[BORROWER SIG 1 RELATIONSHIP]` | "Manager", "President", "General Partner" |
| Borrower signer 2 (all fields) | `[BORROWER SIG 2 *]` | Second tier — same pattern |
| Borrower signer 3 (all fields) | `[BORROWER SIG 3 *]` | Third tier — same pattern |
| Guarantor signer 1 (all fields) | `[GUARANTOR SIG 1 *]` | Same pattern as borrower signers |
| Guarantor signer 2 (all fields) | `[GUARANTOR SIG 2 *]` | Same pattern |

All signature block items should be marked `[EXTERNAL INPUT REQUIRED]` in the
conversion report — signatory information is not in the CAS.

---

### COLLECTIONS (FOR EACH LOOPS)

| Collection | Label | Item Placeholders |
|---|---|---|
| Property folios | `[FOR EACH: FOLIOS]` | `[FOLIO NUMBER]` |
| Personal guarantors | `[FOR EACH: PERSONAL GUARANTORS]` | `[PG NAME]` |
| Entity guarantors | `[FOR EACH: ENTITY GUARANTORS]` | `[EG NAME]`, `[EG STATE]`, `[EG ENTITY TYPE]` |
| Trust guarantors | `[FOR EACH: TRUST GUARANTORS]` | `[TG NAME]`, `[TRUST TRUSTEE NAME]`, `[TRUST TRUST DATE]` |
| Properties (multi-property) | `[FOR EACH: PROPERTIES]` | `[PROPERTY ADDRESS]`, `[FOLIO NUMBER]` |
| Permitted exceptions | `[FOR EACH: PERMITTED EXCEPTIONS]` | `[PERMITTED EXCEPTION]` |
| Additional borrowers | `[FOR EACH: ADDITIONAL BORROWERS]` | `[ADDITIONAL BORROWER NAME]`, `[ADDITIONAL BORROWER STATE]` |

---

## Condition Label Matrix

Maps source template conditional language to standard `[IF: LABEL]` names.
Use these exact labels for consistency with `CONDITION_KEEPS` in `generate_documents.py`.

### Loan Structure

| Condition | Label | TRUE when |
|---|---|---|
| New origination vs. modification | `NEW LOAN` | New origination |
| Modification/renewal/extension | `EXISTING LOAN` | Modification of existing loan |
| Acquisition purpose | `LOAN PURPOSE : "ACQUISITION"` | Purchase of property |
| Refinance with existing debt | `LOAN PURPOSE : "REFINANCE WITH EXISTING DEBT"` | Paying off existing loan |
| Refinance free and clear | `LOAN PURPOSE : "REFINANCE FREE AND CLEAR"` | No existing debt |
| Construction purpose | `LOAN PURPOSE : "CONSTRUCTION"` | Ground-up or major renovation |
| Not construction | `NOT LOAN PURPOSE == "CONSTRUCTION"` | Any non-construction purpose |
| Holdback exists | `HOLDBACK` | Portion of proceeds withheld |
| Extension option exists | `EXTENSION OPTION` | Borrower has right to extend |
| Multiple extension options | `MULTIPLE EXTENSION OPTIONS` | More than one extension |
| Future advance | `FUTURE ADVANCE` | Additional advance on existing loan |
| Line of credit | `LINE OF CREDIT` | Revolving or LOC structure |
| Permanent phase | `PERMANENT PHASE` | Construction loan converts to permanent |

### Interest Rate

| Condition | Label | TRUE when |
|---|---|---|
| Fixed rate for full term | `INTEREST RATE STRUCTURE : "FIXED"` | Single rate, never adjusts |
| Continuously floating | `INTEREST RATE STRUCTURE : "FLOATING"` | Index + margin, adjusts continuously |
| Initial fixed then periodic adjustment | `INTEREST RATE STRUCTURE : "VARIABLE"` | Fixed period then adjusts |
| Swap structure | `INTEREST RATE STRUCTURE : "SWAP"` | Interest rate swap agreement |
| Floating or variable | `FLOATING OR VARIABLE RATE` | Either floating or variable |
| Not swap | `NOT INTEREST RATE STRUCTURE == "SWAP"` | Fixed, floating, or variable |
| SOFR index | `INDEX : "SOFR"` | CME Term SOFR or Daily SOFR |
| Prime index | `INDEX : "PRIME"` | WSJ Prime Rate |
| Rate known at commitment | `FIXED RATE KNOWN` | Specific rate stated in CAS |
| Rate to be determined | `FIXED RATE TBD` | Rate formula only, TBD at closing |
| Initial rate known | `INITIAL RATE KNOWN` | Initial fixed rate stated |
| Initial rate TBD | `INITIAL RATE TBD` | Initial rate TBD at closing |
| Initial fixed term > 4 years | `INITIAL FIXED > 4 YEARS` | First fixed period is 5+ years |
| Single rate adjustment | `SINGLE RATE ADJUSTMENT` | One repricing event |
| Multiple rate adjustments | `MULTIPLE RATE ADJUSTMENTS` | Two or more repricing events |
| Margin applies | `MARGIN` | Floating/variable: index + margin |
| Floor rate applies | `FLOOR RATE` | Minimum rate floor exists |
| Floor rate known | `FLOOR RATE KNOWN` | Specific floor stated |
| Floor rate TBD | `FLOOR RATE TBD` | Floor TBD at closing |

### Repayment

| Condition | Label | TRUE when |
|---|---|---|
| Interest-only period | `INTEREST ONLY` | IO period before P&I |
| P&I from start | `NO INTEREST ONLY` | No IO period |

### Prepayment

| Condition | Label | TRUE when |
|---|---|---|
| Prepayment penalty exists | `PREPAYMENT PENALTY` | Any penalty on early payoff |
| No prepayment penalty | `NO PREPAYMENT PENALTY` | Open prepayment |
| Bona fide sale exception | `PREPAYMENT PENALTY TYPE : "BONA FIDE"` | Penalty waived on arm's-length sale |
| Refinance triggers penalty | `PREPAYMENT PENALTY TYPE : "REFINANCE"` | Refinance triggers penalty |
| Penalty applies full term | `PENALTY FULL TERM` | Penalty for entire loan term |
| Penalty applies partial term | `PENALTY PARTIAL TERM` | Penalty for initial years only |

### Fee

| Condition | Label | TRUE when |
|---|---|---|
| Commitment fee | `COMMITMENT FEE` | Upfront commitment fee |
| Underwriting/origination fee | `UNDERWRITING FEE` | Underwriting or origination fee |
| Fee on loan amount | `FEE ON LOAN AMOUNT` | Fee calculated on total loan |
| Fee on future advance | `FEE ON FUTURE ADVANCE` | Fee on advance only |

### Guaranty

| Condition | Label | TRUE when |
|---|---|---|
| Any guarantor exists | `GUARANTOR` | Any type of guarantor |
| Has guarantors | `HAS GUARANTORS` | Total guarantors > 0 |
| No guarantors | `NO GUARANTORS` | Non-recourse with no guarantor |
| Guarantor section needed | `GUARANTOR SECTION` | Guarantors exist or bad boy exists |
| Personal guarantors exist | `PERSONAL GUARANTORS` | Individual(s) providing guaranty |
| Entity guarantors exist | `ENTITY GUARANTORS` | Entity/entities providing guaranty |
| Trust guarantors exist | `TRUST GUARANTORS` | Trust(s) providing guaranty |
| Entity or trust guarantors | `ENTITY OR TRUST GUARANTORS` | Either entity or trust guarantors |
| Bad boy / carveout guarantor | `BAD BOY GUARANTOR` | Non-recourse carve-out guaranty only |
| Multiple guarantors | `MULTIPLE GUARANTORS` | More than one guarantor total |
| Multiple personal guarantors | `MULTIPLE PERSONAL GUARANTORS` | More than one personal guarantor |
| Multiple entity guarantors | `MULTIPLE ENTITY GUARANTORS` | More than one entity guarantor |
| Existing ownership language | `EXISTING OWNERSHIP` | Prior ownership relationship |
| No existing ownership | `NOT EXISTING OWNERSHIP` | No prior ownership relationship |

### DSCR Covenants

| Condition | Label | TRUE when |
|---|---|---|
| DSCR covenant required | `DSCR COVENANT` | Ongoing DSCR maintenance |
| Annual DSCR testing | `DSCR ANNUALLY` | Tested once per year |
| I&E statement basis | `DSCR BASIS : "INCOME AND EXPENSE STATEMENTS"` | Based on P&L |
| Tax return basis | `DSCR BASIS : "COMPANY TAX RETURNS"` | Based on tax returns |
| DSCR from particular date | `DSCR START : "PARTICULAR YEAR"` | Testing begins specific year |
| NOI-based DSCR | `NOI-BASED DSCR` | DSCR uses net operating income |
| Non-NOI DSCR | `NON-NOI DSCR` | DSCR uses other income measure |
| Hypothetical amortization | `DSCR HYPOTHETICAL AMORTIZATION` | DSCR uses assumed amort schedule |
| Min DSCR adjustable | `MIN DSCR ADJUSTMENT BASIS` | Covenant rate can be adjusted |

### Escrow and Reserves

| Condition | Label | TRUE when |
|---|---|---|
| No tax escrow | `TAX ESCROW : "NO ESCROW"` | Borrower pays taxes directly |
| Tax and insurance escrow | `TAX ESCROW : "TAX AND INSURANCE"` | Standard escrow |
| Tax and flood insurance escrow | `TAX ESCROW : "TAX AND FLOOD INSURANCE"` | Escrow includes flood |
| No escrow during IO, then escrow | `TAX ESCROW : "NO ESCROW DURING INTEREST ONLY PERIOD AND TAX AND INSURANCE ESCROW THEREAFTER"` | IO period: no escrow |
| Reserve account required | `RESERVE ACCOUNT` | Debt service reserve required |
| Stabilization reserve required | `STABILIZATION RESERVE` | Lease-up reserve required |
| Interest reserve required | `INTEREST RESERVE` | Construction interest reserve |
| Tenant improvement reserve | `TENANT RESERVE` | TI/LC reserve required |

### Collateral and Conditions

| Condition | Label | TRUE when |
|---|---|---|
| Leased income property | `LEASED PROPERTY` | Property has tenants/leases |
| UCC filing required | `UCC REQUIRED` | UCC-1 financing statement needed |
| UCC not required | `UCC NOT REQUIRED` | No UCC filing |
| Cross-default applies | `CROSS DEFAULT` | Default on another loan = default here |
| No cross-default | `NO CROSS DEFAULT` | No cross-default provision |
| Environmental report | `ENVIRONMENTAL` | Phase I or II required |
| Full property condition report | `PROPERTY CONDITION REPORTS : "FULL CONDITION"` | Full PCA required |
| Roof and termite only | `PROPERTY CONDITION REPORTS : "ROOF AND TERMITE"` | Limited scope report |
| Roof only | `PROPERTY CONDITION REPORTS : "ROOF ONLY"` | Roof report only |
| Not roof only | `NOT PROPERTY CONDITION REPORTS == "ROOF ONLY"` | Full or roof+termite |
| Franchise agreement | `FRANCHISE` | Property operates under franchise |
| STR/Star reports required | `STAR REPORTS` | Hotel: Smith Travel Research reports |
| Star semi-annual | `STAR SEMI-ANNUAL` | Star reports semi-annually |
| Construction bond | `BOND` | Performance/payment bond required |
| Global cash flow analysis | `GLOBAL CASH FLOW` | Multi-entity financial analysis |
| JV or LP agreements | `JV/LP AGREEMENTS` | Partnership or JV structure |
| Business interruption insurance | `BIZ INTERRUPTION INSURANCE` | BI/loss of rents coverage |
| Purpose additional detail | `PURPOSE ADDITIONAL INFO` | Extra purpose description needed |

### Loan Amount Basis

| Condition | Label | TRUE when |
|---|---|---|
| LTV in sizing basis | `"LTV" IN LOAN AMOUNT BASIS` | Loan sized by LTV |
| DSCR in sizing basis | `"DSCR" IN LOAN AMOUNT BASIS` | Loan sized by DSCR |
| LTP in sizing basis | `"LOAN TO PURCHASE" IN LOAN AMOUNT BASIS` | Acquisition: sized by LTP |
| LTV not in sizing basis | `"LTV" NOT IN LOAN AMOUNT BASIS` | Not LTV sized |
| DSCR not in sizing basis | `"DSCR" NOT IN LOAN AMOUNT BASIS` | Not DSCR sized |
| LTP not in sizing basis | `"LOAN TO PURCHASE" NOT IN LOAN AMOUNT BASIS` | Not LTP sized |
| Loan amount stated | `LOAN AMOUNT STATED` | Specific dollar amount in CAS |
| Loan over $1M | `LOAN OVER $1M` | Loan > $1,000,000 |
| Loan under $1M | `LOAN UNDER $1M` | Loan < $1,000,000 |

### Reporting

| Condition | Label | TRUE when |
|---|---|---|
| Borrower financials required | `BORROWER FINANCIALS` | Annual financial statements |
| Entity guarantor financials | `ENTITY GUARANTOR FINANCIALS` | Entity guarantor statements |
| Personal guarantor financials | `PERSONAL GUARANTOR FINANCIALS` | Personal financial statement |
| Borrower or entity financials | `BORROWER OR ENTITY FINANCIALS` | Either type required |
| Borrower and entity financials | `BORROWER AND ENTITY FINANCIALS` | Both types required |
| Borrower tax returns | `BORROWER TAX RETURNS` | Annual tax returns |
| Guarantor tax returns | `GUARANTOR TAX RETURNS` | Guarantor tax returns |
| Borrower and guarantor returns | `BORROWER AND GUARANTOR TAX RETURNS` | Both required |
| Borrower or guarantor returns | `BORROWER OR GUARANTOR TAX RETURNS` | Either required |
| Corporate return extension | `TAX RETURN EXTEND C` | Extension to Oct 15 |
| Personal return extension | `TAX RETURN EXTEND P` | Extension to Oct 15 |
| Rent roll required | `RENT ROLL` | Annual rent roll |
| Operating statement required | `OPERATING STATEMENT` | Annual operating statement |
| Rent roll and operating | `RENT ROLL AND OPERATING STATEMENT` | Both required |
| Rent roll or operating | `RENT ROLL OR OPERATING STATEMENT` | Either required |

### Signature Block Types

| Condition | Label | TRUE when |
|---|---|---|
| Borrower sig 1 is entity | `BORROWER SIG ONE TYPE: ENTITY` | Entity signing on behalf of borrower |
| Borrower sig 1 is individual | `BORROWER SIG ONE TYPE: INDIVIDUAL` | Individual signing directly |
| Borrower sig 2 is entity | `BORROWER SIG TWO TYPE: ENTITY` | Second-tier entity signer |
| Borrower sig 2 is individual | `BORROWER SIG TWO TYPE: INDIVIDUAL` | Second-tier individual signer |
| Both sigs are entities | `BORROWER SIG ONE AND TWO: ENTITY` | Both tiers are entities |
| Guarantor sig 1 is entity | `GUARANTOR SIG ONE TYPE: ENTITY` | Entity signing guaranty |
| Guarantor sig 1 is individual | `GUARANTOR SIG ONE TYPE: INDIVIDUAL` | Individual signing guaranty |

### Property/State Vowel Checks

These control the grammatical article `a` vs. `an` before state names and entity types:

| Condition | Label | TRUE when |
|---|---|---|
| Borrower state starts with vowel | `BORROWER STATE[0]\|LOWER IN [A, E, I, O, ]` | State begins A, E, I, O |
| Borrower state starts with consonant | `NOT BORROWER STATE[0]\|LOWER IN ['A', 'E', 'I', 'O', ]` | State begins consonant |
| Sig 1 entity state starts with vowel | `BORROWER SIG ONE ENTITY STATE[0]\|LOWER IN [A, E, I, O, ]` | State begins A, E, I, O |
| Sig 1 entity state starts with consonant | `NOT BORROWER SIG ONE ENTITY STATE[0]\|LOWER IN ['A', 'E', 'I', 'O', ]` | State begins consonant |

### Multiple Properties / Folios

| Condition | Label | TRUE when |
|---|---|---|
| Multiple folios exist | `MULTIPLE FOLIOS` | More than one folio number |
| Single folio | `NOT MULTIPLE FOLIOS` | Single parcel only |

### Bank Personnel

| Condition | Label | TRUE when |
|---|---|---|
| SVP is Wayne Smith | `SENIOR VICE PRESIDENT : "WAYNE SMITH"` | Wayne Smith is approving SVP |
| SVP is Fara Khan | `SENIOR VICE PRESIDENT : "FARA KHAN"` | Fara Khan is approving SVP |
| Officer position is SVP | `BANK OFFICER POSITION : "SVP"` | RM title is SVP |

### List Punctuation (Inside FOR EACH loops)

| Condition | Label | TRUE when |
|---|---|---|
| Not last item in list | `NOT LAST ITEM` | Current iteration is not last |
| Show list comma | `LIST COMMA` | Comma follows this item |

---

## Placeholder Naming Conventions

When coining new placeholders not in the matrix, follow these rules so they integrate
cleanly into `generate_documents.py`:

| Rule | Example |
|---|---|
| UPPER CASE, spaces between words | `[LEASE TERMINATION NOTICE PERIOD]` |
| Dollar amounts: lead with `$` | `[$REPLACEMENT RESERVE MONTHLY]` |
| Spelled-out versions: end with `WORDS` | `[REPLACEMENT RESERVE MONTHLY WORDS]` |
| ALL CAPS versions: end with `CAPS` | `[TENANT NAME CAPS]` |
| Percentage values: end with `%` or use numeric | `[REPLACEMENT RESERVE %]` |
| Item-scoped (inside FOR EACH): use short prefix | `[PROP ADDRESS]`, `[PROP COUNTY]` |
| Attorney-fill (not in CAS): use double brackets | `[[LEGAL DESCRIPTION]]` |

## Condition Label Naming Conventions

When coining new condition labels:

| Rule | Example |
|---|---|
| ALL CAPS, spaces between words | `HOTEL PROPERTY` |
| Enumerated values: use ` : "VALUE"` pattern | `PROPERTY TYPE : "HOTEL"` |
| Negations: use `NOT ` prefix | `NOT HOTEL PROPERTY` |
| Combined conditions: use `AND` / `OR` | `HOTEL PROPERTY AND STAR REPORTS` |
| Keep labels self-explanatory without context | `REPLACEMENT RESERVE` not `RR` |

---

## Common Firm-Specific Patterns

Frequently encountered drafting conventions by firm type, to accelerate recognition:

### Big Law CRE Practice Groups
- Tend to use `[●]` as the universal blank
- Extensive use of Word comments for NTDs
- Often have "Option A / Option B" structure for rate alternatives
- Signature blocks use defined terms consistently: "Borrower", "Guarantor", never entity name directly in sig line
- Cross-references are heavily used — follow them to find conditional content

### Regional Bank In-House Forms
- Often use `[INSERT ___]` or `[___]` style blanks
- Conditions frequently expressed as inline alternatives: "[does/does not]"
- Simpler structure, fewer nested conditions
- NTDs often appear as parentheticals in the text: "(NTD: confirm with RM)"

### Title Company / Settlement Agent Forms
- Heavy use of underline blanks `___________`
- Recording information blanks are standard: `[BOOK]`, `[PAGE]`, `[INSTRUMENT NO.]`
- These all map to `[[RECORDS *]]` double-bracketed placeholders

### Government-Sponsored Forms (Fannie, Freddie, SBA)
- Highly standardized — look for numbered blank lines
- Conditions often expressed via checkboxes or "☐ Yes ☐ No"
- Map checkbox alternatives to `[IF: LABEL]` blocks

---

## Edge Cases in Conversion

### Multiple Alternatives (More Than Two)

When a section has three or more alternatives (e.g., fixed / floating / variable / swap):

Convert each to its own `[IF: LABEL]` block. All four blocks remain in the maximum
template — the engine deletes the non-applicable ones at generation time.

### Nested Conditions

When a condition is nested inside another (e.g., "if construction loan, and if bonded"):

```
[IF: LOAN PURPOSE : "CONSTRUCTION"]
  ... construction language ...
  [IF: BOND]
    ... bond requirement language ...
  [END IF: BOND]
  ... more construction language ...
[END IF: LOAN PURPOSE : "CONSTRUCTION"]
```

Preserve the nesting exactly. The engine handles nested blocks correctly via its
stack-based resolution.

### Inline Alternatives Within a Sentence

When a sentence contains `[does/does not]` or `[his/her/its]` style inline choices:

Convert to back-to-back inline IF blocks:
```
Bank [IF: UCC REQUIRED]does[END IF: UCC REQUIRED][IF: UCC NOT REQUIRED]does not[END IF: UCC NOT REQUIRED] require a UCC filing.
```

### Schedules and Exhibits

Schedules that list multiple items (folios, permitted exceptions, properties) are
candidates for `[FOR EACH:]` loops. If the schedule is a simple list:

1. Replace the schedule body with a `[FOR EACH: COLLECTION]` block
2. Replace each item-level field with an item-scoped placeholder
3. Add a `[REVIEW: Schedule converted to FOR EACH loop — verify item placeholders]`
   comment

### Track-Changed Alternative Language

Some templates embed negotiated alternatives as tracked changes (accepted = current form,
rejected = alternative). Do not resolve these — preserve the tracked changes and add a
`[REVIEW: Template contains negotiated tracked change alternative — attorney to confirm
which version applies]` comment.

### Partner Footnotes

Footnotes with partner drafting guidance should be:
1. Extracted as the content of a Word comment on the relevant paragraph
2. Prefixed with `REVIEW: [original footnote text]`
3. The footnote itself deleted from the document

---

## Output Checklist

Before delivering the converted template:

- [ ] All `[●]`, `[___]`, `[Insert...]` blanks converted
- [ ] All Word comment NTDs preserved with `REVIEW:` prefix
- [ ] All inline NTD text removed from document body
- [ ] All `[IF:]` / `[END IF:]` pairs balanced
- [ ] All `[FOR EACH:]` / `[END FOR EACH:]` pairs balanced
- [ ] No substantive legal text modified
- [ ] All formatting preserved (indentation, numbering, fonts)
- [ ] Attorney-fill items using double brackets `[[...]]`
- [ ] Conversion report complete with all new placeholders and labels documented
- [ ] New placeholders and labels listed for addition to `generate_documents.py`

# Intact UK&I — how data moves from a policy transaction to the financial statements

*Companion narrative to the interactive flow map. Current state as documented, with the target state noted at each stage.*

---

## The one-paragraph version

Around twenty policy and claims systems create financial transactions every day. Almost all of them send a flat file overnight. Those files converge on a single piece of middleware — the **Single Integration Layer** — which enriches every transaction with the ledger coding it will need, stamps it with a reconciliation key, and then splits it two ways: one copy posts to the **SAP general ledger**, the other loads into the **Finance Data Store**. Because both copies carry the same key, the ledger and the warehouse can be reconciled automatically every night. The warehouse then calculates reserves, unearned premium and IFRS 17 inputs, sends those to calculation engines, and receives journals back which post to the ledger. The ledger's trial balance goes to group consolidation. Everything else — planning, reconciliation, MI, regulatory returns — hangs off those two stores.

**If you remember one thing:** the general ledger has one main door, and SIL is the doorman.

---

## Band 1 — Origination: where the numbers are born

Financial data is created at the point a policy is written or a claim is handled. There are four groups.

**Personal and commercial policy administration.** AIS, UKRIS, Stage 2/SAUNA, Duck Creek/Unity, Transactor, ICE, IB Suite and Hood. AIS, UKRIS and Stage 2 sit on the mainframe and are the largest single driver of IT cost in the estate; all three are exit candidates.

**Specialty, international and Ireland.** Atlas (Global Europe and Risk Solutions), Synergy (marine), IP Global Network, Eclipse (London market and DIFC, on Sequel), SICS Profin, Ireland Genius, Sapiens RI for treaty reinsurance, Pivot Point, and **DXC Assure C&S** — the newest arrival, brought into the finance estate in late 2024.

**Claims administration.** Two systems matter. **CCS** is the common claims record for the legacy estate, carrying claims on AIS, SAUNA, EPOQ, UKRIS, Bradford & Pennine, Duck Creek, ICE and Hood policies. **Guidewire ClaimCenter** is the strategic platform and does something similar but forward-looking: it handles claims on AIS, Stage 2, UKRIS, Duck Creek, B&P, ICE *and* DXC policies, and produces a single daily file with the originating policy system flagged on each record. PAWS handles pet claims through a third-party administrator.

**A note on where files originate.** Most feeds land directly on the integration server. But anything hosted outside the Intact network — PAWS, Profin, Pet, MT Home, Home TPA, DXC Assure, Eclipse, Transactor — has to come through GlobalScape and stage in the DMZ first. That distinction matters for access control and for where a failure gets picked up.

> **Target state.** Acturis becomes the commercial policy platform; Asima and other new platforms take specialty. Guidewire remains and expands. The mainframe trio and most of the specialty estate exit. Critically, new platforms connect **directly to the Enterprise Data Platform**, not through SIL.

---

## Band 2 — The claims platform and the Azure data platform

This is a full data platform in its own right, and it sits on the claims side rather than the finance side. It matters to finance for exactly one reason, but that reason is significant.

**Core claims platform (AWS).** Guidewire ClaimCenter and Contact Manager, fronted by an API layer and Edge APIs serving the digital channels, with claim documents in S3 archive and CDA buckets.

**Ingest and operational stores.** Azure Logic Apps orchestrate ingestion from the legacy policy estate — policy snapshots, payment status, agency transfers, bank detail and intermediary updates. These land in the **Policy ODS**, which is what gives Guidewire the policy context it needs at the point of claim. A separate CDA replication pipeline copies claim transactions out of ClaimCenter into the lake. Claims migration tooling (CMT) moves historic claims off CCS, Eclipse, UKRIS, Atlas, Synergy and DXC into Guidewire.

**The lake.** A conventional medallion structure — raw, staging, processed — on Azure Data Lake V2, built with ADF and Databricks. Scanned documents pass through Cognitive Services OCR on the way in. From the processed layer the data fans out to the **Claims ODS**, then to the **data marts**.

**Why finance cares.** The daily Guidewire finance extract is built *from the marts*, not from ClaimCenter directly. The extract batch starts at 04:00, reads claim events that carry a financial movement, enriches them with policy and agency data, and delivers a single combined file to the integration layer before 06:00. So the marts sit on the critical path for claims reaching the general ledger — a two-hour window, and a hard 06:00 cut-off downstream.

**Services and governance.** A payments staging database holds approved claim payment instructions before release. An AKS decision engine supports automated claim handling. Microsoft Purview catalogues and traces the platform. A co-existence store maps legacy claim identifiers to strategic ones so financial history stays traceable during dual-running.

---

## Band 3 — Transport: how files actually move

**GlobalScape EFT** is the managed file transfer platform, paired with a DMZ Gateway that acts as a communication proxy so no connection is ever initiated inward from the DMZ. It handles scheduled system-to-system transfers with third parties and hosted platforms, and separately provides browser-based ad-hoc transfer for internal users. It moves LoB files, reference data, the Tagetik datasets, exchange rates and banking traffic. The **B2B Gateway** and **BAU CAL** carry the claims-side supplier and payment traffic. Connect Direct still moves PAWS and is being retired.

Everything converges on the **SIL landing area**. Files typically arrive between 22:00 and 03:30, monitoring runs 00:00 to 06:00, and **anything arriving after 06:00 waits until the next working day**. That single rule is the most consequential operational fact in the estate.

Separately, the **FDS file landing area** is where finance and actuarial spreadsheet uploads enter — agency hierarchies, intraday adjustments, plan and forecast, reserve reviews, IBNR allocations, payment patterns, yield curves, the trial balance. Each format has its own validation ruleset and triggers its own downstream calculation.

---

## Band 4 — The Single Integration Layer: the most important system nobody names

SIL is an IBM DataStage application. It does six things, and each one is a control.

1. **Validates** every file for integrity and every record for attribute correctness. Failures are *held*, not dropped, and recycled once corrected.
2. **Replicates the SAP GPC lookup** — GL account, profit centre, company code — from SAP master tables copied nightly at 20:00. The integration layer therefore cannot map a transaction differently from the way the ledger would.
3. **Allocates a Reconciliation ID** to every transaction going to both destinations. This is the key that makes automated ledger-to-warehouse tie-out possible at all.
4. **Filters** what should not travel: non-premium and balance-sheet rows from the manual feed, Atlas Ireland and Ministry of Defence data, AIS insurance premium tax.
5. **Aggregates** AIS, UKRIS, Stage 2 and UK Atlas coverage-level rows up to policy-version level, because the ledger does not want coverage-level detail.
6. **Generates** derived transactions — OCR reversals matching the SAP reversal process, and inter-entity balancing entries for the global network.

Then it forks. **SIFF, CIFF and MIFF files go to the Finance Data Store. Thirty-two separate SAP-format interface files go to the general ledger.** Reference data comes in alongside from **Ataccama**, a cloud-hosted reference data manager where the business maintains mappings and publishes versions; nothing reaches SIL or TDS without a published version.

**FRACO/Infogix** then reconciles the two outputs daily on the Reconciliation ID.

> **Target state.** SIL is an exit candidate. New systems bypass it entirely. But it survives an interim period feeding the Enterprise Data Platform for non-strategic lines, precisely so its mapping logic does not have to be rebuilt for systems that are themselves being retired.

---

## Band 5 — The Finance Data Store: one warehouse, four stores

**TDS** takes the enriched transactions and consolidates everything into **Cover Snapshot**, a single common transactional format that is the basis of all transactional reporting. It applies the Single Class Structure classification, calculates claim development periods and rollups, calculates unearned premium reserve and earned premium, and apportions centrally posted amounts down to policy level.

**ADS** builds the aggregates — six monthly transactional aggregates, two more using apportionment, plus the quarterly aggregates that serve both the Solvency II QRT submission and the IFRS 17 handoff. It starts at 06:00 and waits for TDS completion flags.

**CDS** consolidates reinsurance transactions and spreads reinsurance IBNR alongside actuals.

**RDS** holds the actuarial grids, IBNR adjustments and reference data, then calculates IBNR, technical provisions and cashflows, and produces the SAP upload files for each quarter-end posting, thirteen Solvency II aggregates for group, and the IFRS 17 staging area.

Out of FDS: **Cognos** for control reports and the Star 3 injection pack; **PDS** taking a daily delta and feeding **Qlik**; **PBMI/BPMI** monthly marts; and a monthly database pull into the **actuarial sandpit**.

The whole thing runs on a batch schedule governed by fifteen named flags that stop the various loads colliding — rollups, period measures, reclassification, reference data, intraday adjustments and reinsurance allocation each block the others.

---

## Band 6 — The general ledger and the money

**SAP ECC is the book of record.** It receives thirty-two interface files a night from SIL, plus Ariba invoices, Concur expenses, Winshuttle uploads, off-system manual journals, regional Sun Accounts balances, working-day-one automated postings from FDS, quarter-end IBNR postings from RDS, and IFRS 17 journals from Tagetik.

Posting errors have defined routes: blocked GL accounts, blocked or missing profit centres, invalid profitability characteristics, closed posting periods. In each case finance decides and ADM executes — the segregation is explicit.

**Coming out of the ledger:** master tables nightly to SIL; the trial balance to FDS on working day 4 and again on working day 6 once IFRS 17 adjustments return; a thirty-minute transaction and trial balance extract to **Hydra**; balances to **Blackline**, **Anaplan** and the Star 3 injection pack.

**Payments and cash.** Approved instructions flow from SAP AR to **Bottomline PTX**, which releases to HSBCnet and the banking providers. Direct debit collections run through the **Experian Payment Gateway**. Cheques run through GPS and the eCheques/Premier Cheque stack. BACS carries bulk payments including payroll. Statements come back daily through e-Banking for cash allocation, alongside cashed cheque data and positive pay files. **PRGX** independently hunts duplicate payments; **ORB** manages overdue debt.

**Treasury.** **Wall Street Suite** is the treasury management system, taking executed FX from **360T and Finastra**, instructing foreign payments through **FIDES** as SWIFT bureau, and posting treasury accounting entries to the ledger. **PEX** supplies the exchange rates used by both the ledger and the 02:00 currency load into TDS.

> **Target state.** SAP ECC exits. A single global **Oracle ERP** instance takes over, fed through the **Accounting Hub** as the one subledger gateway, with a centrally governed global chart of accounts.

---

## Band 7 — Calculation engines

**Actuarial.** TDS pulls monthly into the **OSIRIS/Minerva** sandpit, which prepares panel and channel level data for **ResQ**, the reserving platform. Results feed **Igloo** for capital and risk modelling, and come back into RDS as uploads.

**IFRS 17.** RDS and ADS assemble a staging area and send **seventeen structured datasets quarterly by SFTP** to **Tagetik**, a cloud calculation engine. Inside Tagetik: an ETL load, then an input data model that transforms IFRS 4 data into cashflows, then the IFRS 17 starter kit that generates accounting entries, then journal generation — with validations, crosschecks and consistency checks running across every stage. Two things come back: **journals sent directly to SAP by SFTP**, and a **Star 3 extract** of movements feeding the group injection pack. One detail worth knowing: amount signage is inverted on extract, because the engine holds signage the opposite way round to FDS.

> **Target state.** Tagetik exits. A single global **SAS** instance becomes the group IFRS 17 measurement engine, fed from the Enterprise Data Platform and posting through the accounting hub.

---

## Band 8 — Reporting, consolidation and control

**Group consolidation** runs through **Star 3**, assembled from an Excel injection pack built from the SAP trial balance and the Tagetik extract. Fifteen return forms are populated from Cognos SQL reports driven by Ataccama-managed mapping tables — balance sheet, goodwill and intangibles, currency analysis, claims reserve and reserve movement, underwriting analysis, profit and loss, tax, other debtors and creditors. Results flow to Disclosure Management for the statutory statements.

**Solvency II** takes a different route: RDS produces thirteen QRT aggregates, which consolidate in **TM1** and submit through **Assist**.

**Control and planning.** **Blackline** holds balance sheet reconciliation preparation, review and certification. **Anaplan** does operational planning and expense and reinsurance allocation, posting allocation results back into the ledger. **Qlik** serves business MI from PDS.

**And Excel.** A substantial end-user computing layer sits between almost every pair of systems here — injection packs, manual adjustments, trial balance uploads, actuarial results. The documentation is explicit that extracts are the business user's responsibility with no system-level audit and control over the resulting files.

> **Target state.** **FCCS** consolidates, **Workiva** produces statutory reporting, **EPBCS** plans, **ARCS** reconciles, **EPCM** allocates, **EDMCS** governs finance master data, and **FAW/FDI with Power BI** serves analytics.

---

## The control points, in the order data meets them

| | Control | Where it bites |
|---|---|---|
| **C1** | File receipt and integrity | Counts, sequence numbers, duplicate and missing-file detection on every inbound feed |
| **C2** | Attribute validation, reject and recycle | Failures held and recycled rather than lost — this is what protects completeness across days |
| **C3** | GPC mapping replicated from SAP | The integration layer cannot code a transaction differently from the ledger |
| **C4** | Reconciliation ID allocation | The key that makes automated tie-out possible |
| **C5** | SAP-to-TDS daily reconciliation | Infogix matches every posted transaction to its warehouse twin |
| **C6** | SAP interface posting error control | Tracked in-system, with finance deciding and ADM executing |
| **C7** | TDS validation and dimensional rejects | Second validation gate on the warehouse side |
| **C8** | Batch dependency and completion flags | Fifteen flags stopping loads from colliding |
| **C9** | Reference data versioning and publication | Nothing reaches SIL or TDS unpublished |
| **C10** | Period close and account locking | What can and cannot post to a closed period |
| **C11** | WD1 automated posting | UPR, Ireland NRP, Transactor VAT and global network, automated rather than manual |
| **C12** | Trial balance upload and validation | WD4 and WD6, validated to Star 3 |
| **C13** | Balance sheet reconciliation | Blackline preparation, review, certification |
| **C14** | Calculation engine validation | Validations, crosschecks, consistency checks before any journal posts |
| **C15** | Monitoring, alerting and incident automation | Reporting withheld until processing completes, so nobody reports on a partial night |
| **C16** | Manual upload validation | The control boundary around a large EUC population |
| **C17** | Payment authorisation and duplicate control | Release, positive pay, independent duplicate detection |
| **C18** | Bank reconciliation | Statements, cashed cheques, BACS failures |

---

## Five observations worth putting in front of the audit committee

**1. The reconciliation key is the single most valuable control in the estate.** Because SIL stamps the same identifier on both the ledger copy and the warehouse copy, the daily tie-out is automatic rather than a manual exercise. Anything that bypasses SIL loses this by construction — which is worth watching closely as new platforms are wired directly to the Enterprise Data Platform.

**2. The 06:00 cut-off is a real financial risk, not an IT inconvenience.** A late file is not a late file — it is a file that posts a day later. Guidewire's extract has a two-hour margin. The mainframe extracts sit inside batch schedules the documentation describes as very difficult to restructure.

**3. Two systems act as claims concentrators, and both carry many lines.** CCS carries claims for at least seven policy systems; Guidewire carries claims for at least seven more. A fault in either is not a single-line problem.

**4. The EUC population is large and structurally acknowledged.** Trial balance uploads, injection packs, actuarial results, manual adjustments and allocation inputs all pass through spreadsheets, with the documentation stating plainly that no system-level audit and control exists over those files. C16 mitigates at the point of entry to FDS; it does not mitigate the preparation.

**5. Change concentration.** The current-state estate depends on SIL, SAP ECC, FDS and Tagetik — all four are exit candidates in the target architecture, and the replacement (EDP, SAS, Accounting Hub, Oracle, FCCS) changes the shape of every control above. The transition period, when both run, is where control coverage is most likely to gap.

---

## Using the map

Open `intact-finance-claims-data-flow-map.html` in a browser.

- **Click any system** to trace its full path — upstream in blue, downstream in gold.
- **Click a gold diamond** on a flow line to read the control sitting on it.
- **Switch Current / Target / Both** in the toolbar.
- **Drag boxes** to rearrange; **double-click** a box to rename it.
- **Click a band heading** to collapse it — useful for a board-level view.
- **Save layout** writes your edits to a JSON file; **Load layout** brings them back.
- **Print / PDF** outputs at A2 landscape.

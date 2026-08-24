# Intact UK&I — walking the data from a policy transaction to the financial statements

**Anurag, KPMG**
Working notes to go with the flow map. This isn't a deliverable yet — it's me writing down what I found so we're all working off the same picture.

---

## Why I put this together

I kept getting the same question in different forms: *where does the number come from?* Nobody could answer it end to end, and I don't blame them. The answer is spread across four operating models, a service model, a claims architecture poster and half a dozen decks, and none of them agree on naming.

So I read the lot and drew it. What follows is how I'd explain it if I had ten minutes with the CFO, then the detail underneath.

One note on scope. I've written down what the documents say, not what people mentioned in passing. Where a system shows up in the architecture decks with its role clear but no interface detail behind it — Anaplan, Igloo, ResQ, Wall Street, FCCS — I've drawn it at the level the evidence supports and stopped there. We can firm those up in walkthroughs.

---

## The ten-minute version

Around twenty policy and claims systems create financial transactions every day. Nearly all of them do the same thing: write a flat file overnight and push it out. Those files converge on one piece of middleware called the **Single Integration Layer**. SIL enriches each transaction with the ledger coding it will need, stamps it with a reconciliation key, and splits it two ways — one copy posts to the **SAP general ledger**, the other loads into the **Finance Data Store**.

That split is the clever bit. Because both copies carry the same key, the ledger and the warehouse reconcile automatically every night rather than by hand.

The warehouse does the heavy calculation — reserves, unearned premium, IFRS 17 inputs — and sends those to calculation engines, which send journals back that post to the ledger. The ledger's trial balance goes off to group consolidation. Everything else you can name — planning, reconciliation, MI, regulatory returns — hangs off those two stores.

**The line I'd actually use in the room:** the general ledger has one main door, and SIL is the doorman. Understand SIL and you understand ninety per cent of this estate.

---

## Band 1 — Where the numbers are born

Four groups.

**Personal and commercial policy admin.** AIS, UKRIS, Stage 2/SAUNA, Duck Creek/Unity, Transactor, ICE, IB Suite, Hood. The first three sit on the mainframe and, per the client's own decommissioning pack, are the largest single driver of IT cost in the estate. All three are on the exit list.

**Specialty, international and Ireland.** Atlas, Synergy, IP Global Network, Eclipse (on Sequel), SICS Profin, Ireland Genius, Sapiens RI for treaty, Pivot Point, and **DXC Assure C&S**, which only came into the finance estate in late 2024. DXC is documented across both the SIL and TDS operating models with version notes, so it's well evidenced — but it's recent, and recent means the controls around it have had less run time.

**Claims.** Two systems do the heavy lifting and both are concentrators. **CCS** carries claims on AIS, SAUNA, EPOQ, UKRIS, Bradford & Pennine, Duck Creek, ICE and Hood policies. **Guidewire ClaimCenter** does the same job going forward, carrying claims on AIS, Stage 2, UKRIS, Duck Creek, B&P, ICE and DXC, with the originating policy system flagged on each record. PAWS handles pet through a TPA.

I want to flag the concentration now rather than at the end: **a fault in either of those two is not a single-line-of-business problem.** It's seven or more.

**One thing worth internalising about where files come from.** Most feeds land straight onto the integration server. But anything hosted outside the Intact network — PAWS, Profin, Pet, MT Home, Home TPA, DXC Assure, Eclipse, Transactor — comes through GlobalScape and stages in the DMZ first. That isn't trivia. It changes who has access, who notices a failure, and who we test.

> **Target.** Acturis takes commercial policy, Asima and others take specialty, Guidewire stays and grows. The mainframe trio and most of specialty exit. The bit that matters to us: **new platforms plug straight into the Enterprise Data Platform and bypass SIL entirely.** More on why that bothers me later.

---

## Band 2 — The claims platform and Azure

This is a full data platform in its own right and it sits on the *claims* side, not finance. You asked for it in full, so it's all there. But I want to be precise about why finance should care, because it's one specific thing rather than a general dependency.

**Core claims platform (AWS).** ClaimCenter and Contact Manager behind an API layer, Edge APIs serving the digital channels, claim documents in S3 archive and CDA buckets.

**Ingest and operational stores.** Azure Logic Apps pull from the legacy policy estate — policy snapshots, payment status, agency transfers, bank detail, intermediary updates. That lands in the **Policy ODS**, which is what gives Guidewire policy context at the point of claim. A separate replication pipeline copies claim transactions out of ClaimCenter into the lake. Migration tooling is lifting historic claims off CCS, Eclipse, UKRIS, Atlas, Synergy and DXC into Guidewire.

**The lake.** Standard medallion — raw, staging, processed — on Data Lake V2, built with ADF and Databricks. Scanned documents go through Cognitive Services OCR on the way in. Out the other side into the **Claims ODS**, then the **marts**.

**Here's the bit finance needs to know.** The daily Guidewire finance extract is built *from the marts*, not from ClaimCenter directly. The batch starts at 04:00, picks up claim events carrying a financial movement, enriches them with policy and agency data, and has to deliver before 06:00.

So the marts sit on the critical path for claims reaching the general ledger, with a **two-hour margin against a hard downstream cut-off**. That's tight, and I'd want the failure history before taking comfort from it.

**Services and governance.** A payments staging database holds approved claim payment instructions before release. An AKS decision engine does automated triage. Purview catalogues and traces the platform — useful for us, that's lineage evidence we don't have to build ourselves. And a co-existence store maps legacy claim IDs to strategic ones so financial history stays traceable while both run.

---

## Band 3 — How files actually move

**GlobalScape EFT** is the managed file transfer platform, paired with a DMZ Gateway that acts as a proxy so no connection is ever initiated inward from the DMZ. Sensible design. It moves LoB files, reference data, the Tagetik datasets, exchange rates and banking traffic. It also offers browser-based ad-hoc transfer for internal users, which is a different risk conversation entirely — I'd park that one for now.

The **B2B Gateway** and **BAU CAL** carry the claims-side supplier and payment traffic. Connect Direct still moves PAWS and is being retired.

Everything lands in the **SIL landing area**. Files typically arrive between 22:00 and 03:30, monitoring runs midnight to 06:00, and **anything after 06:00 waits until the next working day.**

I'd underline that sentence. It's the most consequential operational fact in the whole estate and it's buried in a table in the FDS service model.

Underneath the file transfer sits an older messaging layer that still carries real financial traffic: **MQ Series** between the mainframe systems, and a **BizTalk estate** — separate instances for Duck Creek, CCS, Unity and EDI rather than one platform — moving policy, billing and cash messages into the payment gateway. Alongside it, **Documentum and Cognitronics** handle invoice and cheque imaging. Ireland invoices posted after 2016 sit inside SAP itself and were deliberately never migrated to Documentum, which is a small thing until someone asks where a 2019 Irish invoice is.

Separately there's the **FDS file landing area**, and this is where I'd spend time. It takes around forty distinct upload formats. To TDS: sales agency hierarchies, scheme policy groupings, late and central adjustments through the intraday MIFF, plan and forecast, special claims. To RDS: control overlay, commercial and personal SCS apportionment, actual currency mix, overview grids, reserve review, specific IBNR allocations, finance manual adjustments, claims roll forward, margin and central adjustments, actuarial parameters, PPO transactions, payment patterns, UPR run-off patterns, claim and premium ENID, future gross premium income, reinsurance premiums due, premium cashflow adjustments, future profit commissions, future claim payments, claims expense transfer, RI recoveries, annual expense provisions and yield curves. To CDS: the finalised reinsurance result. From SAP: the trial balance. For IFRS 17: cashflows, claim cashflow allocations, internal revenue adjustments, group reinsurance, UPR roll-offs, creditor allocations and unpaid claim adjustments.

Each format has its own validation rules and triggers its own downstream calculation, which is better designed than I expected. But forty upload formats is also forty places a spreadsheet enters the financial reporting process.

---

## Band 4 — SIL, the most important system nobody names

If I only get one system properly walked through on this engagement, it's this one.

SIL is an IBM DataStage application. It does six things and every one of them is a control:

1. **Validates** every file for integrity and every record for attribute correctness. Failures are *held*, not dropped, and recycled once corrected. That design decision is what protects completeness across days, and it's a good one.
2. **Replicates the SAP GPC lookup** — GL account, profit centre, company code — from SAP master tables copied nightly at 20:00. So the integration layer physically cannot code a transaction differently from the way the ledger would. I like this a lot.
3. **Allocates a Reconciliation ID** to every transaction going to both destinations. This is the key that makes automated tie-out possible at all.
4. **Filters** what shouldn't travel: non-premium and balance-sheet rows off the manual feed, Atlas Ireland and MoD data, AIS insurance premium tax.
5. **Aggregates** AIS, UKRIS, Stage 2 and UK Atlas coverage-level rows up to policy-version level, because the ledger doesn't want coverage detail.
6. **Generates** derived transactions — OCR reversals matching the SAP reversal process, and inter-entity balancing entries for the global network.

Then it forks: **SIFF, CIFF and MIFF to the Finance Data Store; thirty-two separate SAP-format files to the general ledger.**

Reference data comes in alongside from **Ataccama**, a cloud reference data manager where the business maintains mappings and publishes versions. Nothing reaches SIL or TDS without a published version, which gives us a clean change control story on mappings.

**FRACO/Infogix** then reconciles the two outputs daily on the Reconciliation ID.

> **Target.** SIL exits. New systems bypass it. But it survives an interim period feeding the EDP for non-strategic lines, specifically so its mapping logic doesn't have to be rebuilt for systems that are themselves being switched off. Pragmatic call and I'd defend it — but it does mean SIL stays in scope longer than the roadmap slide implies.

---

## Band 5 — The Finance Data Store: one warehouse, four stores

**TDS** consolidates the enriched transactions into **Cover Snapshot**, a single common format that's the basis of all transactional reporting. It applies the Single Class Structure classification, works out claim development periods and rollups, calculates UPR and earned premium, and apportions centrally posted amounts down to policy level.

**ADS** builds the aggregates — six monthly transactional ones, two more using apportionment, plus the quarterlies that serve both the Solvency II QRT submission and the IFRS 17 handoff. Starts at 06:00 and waits on TDS completion flags.

**CDS** consolidates reinsurance transactions and spreads reinsurance IBNR alongside actuals.

**RDS** holds the actuarial grids, IBNR adjustments and supporting reference data, then calculates IBNR, technical provisions and cashflows, and produces the SAP upload files for each quarter-end posting, thirteen Solvency II aggregates for group, and the IFRS 17 staging area.

Out of FDS: **Cognos** for control reports and the Star 3 injection pack, **PDS** taking a daily delta and feeding **Qlik**, **PBMI/BPMI** monthly marts, and a monthly database pull into the **actuarial sandpit**.

The whole thing runs on a batch schedule governed by fifteen named flags that stop loads colliding — rollups, period measures, reclassification, reference data, intraday adjustments and reinsurance allocation each block the others. That's a real dependency control and I want it in walkthrough scope.

---

## Band 6 — The ledger and the money

**SAP ECC is the book of record.** Thirty-two interface files a night from SIL, plus Ariba invoices, Concur expenses, Winshuttle uploads, off-system manual journals, regional Sun Accounts balances, working-day-one automated postings from FDS, quarter-end IBNR postings from RDS, and IFRS 17 journals from Tagetik.

Posting errors have defined routes — blocked GL accounts, blocked or missing profit centres, invalid profitability characteristics, closed posting periods. In every case finance decides and ADM executes. The segregation is written down explicitly, which is more than I usually find.

**Coming out:** master tables nightly to SIL; the trial balance to FDS on working day 4 and again on working day 6 once IFRS 17 adjustments come back; a thirty-minute transaction and trial balance extract to **Hydra**; balances to **Blackline**, **Anaplan** and the Star 3 injection pack.

**Payments and cash.** Approved instructions go from SAP AR to **Bottomline PTX**, which releases to HSBCnet and the banks. Direct debits run through the **Experian Payment Gateway**. Cheques through GPS and the eCheques/Premier Cheque stack. BACS carries bulk payments including payroll. Statements come back daily through e-Banking, alongside cashed cheque data and positive pay files. **PRGX** independently hunts duplicate payments and **ORB** runs overdue debt.

**Treasury.** **Wall Street Suite** is the TMS, taking executed FX from **360T and Finastra**, instructing foreign payments through **FIDES** as SWIFT bureau, posting treasury entries to the ledger. **PEX** supplies the rates used by both the ledger and the 02:00 currency load into TDS.

**HR and payroll, which people forget is in the same ledger.** SAP doesn't just run finance here — it runs UK payroll for around 7,000 employees. **SAP Portal (ESS/LSS)** takes leave, sickness, overtime, performance and bank detail changes and synchronises them in real time with the ERP. **SAP HR** runs payroll, produces the BACS salary file, files RTI submissions to **HMRC** through SAP PI and the Apache service, and downloads monthly to **SAP BW** for HR MI. Benefit and pension providers — Benefex, Aviva, L&G, Fidelity, Achievers — exchange deduction and contribution files through GlobalScape, with separate arrangements for UK, Ireland and the Isle of Man.

One detail worth knowing: an internal SAP programme creates the file that spreads payroll charges across cost centres using employee mapping. It doesn't receive that file from anywhere. It's an internal SAP process producing an input to the ledger, and it won't show up if you only look for external interfaces.

> **Target.** SAP ECC exits. One global **Oracle ERP** instance, fed through the **Accounting Hub** as the single subledger gateway, on a centrally governed global chart of accounts.

---

## Band 7 — Calculation engines

**Actuarial.** TDS pulls monthly into the **OSIRIS/Minerva** sandpit, which preps panel and channel data for **ResQ**. Results go on to **Igloo** for capital modelling and come back into RDS as uploads.

**IFRS 17.** RDS and ADS assemble a staging area and send **seventeen structured datasets quarterly by SFTP** to **Tagetik**. Inside Tagetik: ETL load, then an input data model turning IFRS 4 data into cashflows, then the starter kit generating accounting entries, then journal generation — with validations, crosschecks and consistency checks across every stage.

Two things come back: **journals straight to SAP by SFTP**, and a **Star 3 extract** of movements for the group injection pack.

Two details I'd have missed if I hadn't read closely. First, **amount signage is inverted on extract**, because the engine holds signage the opposite way round to FDS — exactly the kind of thing that produces a material misstatement if someone changes it without knowing why it's there. Second, the return files come back as **TCEP and TCER, each split into three regional files for UK, Ireland and Group Reinsurance**, identified only by a flag in the header. So it's six files a quarter, not two, and they go **straight to SAP rather than back through FDS**. That means the journals bypass the warehouse entirely and the tie-out has to happen a different way.

> **Target.** Tagetik exits. A single global **SAS** instance becomes the group IFRS 17 measurement engine, fed from EDP and posting through the accounting hub.

---

## Band 8 — Reporting, consolidation and control

**Group consolidation** runs through **Star 3**, assembled from an Excel injection pack built out of the SAP trial balance and the Tagetik extract. Fifteen return forms populated from Cognos SQL reports driven by Ataccama mapping tables — balance sheet, goodwill and intangibles, currency, claims reserve and reserve movement, underwriting analysis, P&L, tax, other debtors and creditors. Then on to Disclosure Management for the statutory statements.

**Solvency II** goes a different way: RDS produces thirteen QRT aggregates, consolidated in **TM1**, submitted through **Assist**.

**Control and planning.** **Blackline** holds balance sheet reconciliation prep, review and certification. It isn't fed by the ledger alone — it takes GL balances from SAP, AR and AP subledger balances and open items, transaction-level detail from **Hydra**, bank statement data from e-Banking for the cash reconciliations, transaction detail from TDS for balance substantiation, and supporting schedules prepared in Excel. Adjusting journals arising from reconciliation go back into the ledger, and certification status goes up to management reporting.

**Anaplan** does operational planning and expense and reinsurance allocation. It also has more than one feed: actuals from SAP, underwriting and expense actuals from Cognos, monthly aggregates from ADS for the planning baseline, headcount and people cost drivers from SAP BW, and supplier spend from Rosslyn. Its outputs go three ways — plan and forecast uploaded into FDS through the file landing area, expense collections into **PCM** for allocation and then back into SAP as journals, and plan versus actual into the board pack.

**Qlik** serves business MI off PDS.

**And then there's Excel.** A large end-user computing layer sits between almost every pair of systems on this row — injection packs, manual adjustments, trial balance uploads, actuarial results. The FDS service model says it plainly: extracts are the business user's responsibility and no system-level audit and control exists over the resulting files. That's the client's own wording, not my characterisation, and it's the honest starting point for that conversation.

> **Target.** **FCCS** consolidates, **Workiva** produces statutory reporting, **EPBCS** plans, **ARCS** reconciles, **EPCM** allocates, **EDMCS** governs finance master data, **FAW/FDI with Power BI** serves analytics.

---

## The controls, in the order data meets them

I've numbered these and tagged them onto the flow lines in the map, so you can click a diamond and read the control rather than cross-referencing back here.

Each one is also colour-coded on the map by **how it operates** — green for automated, amber for semi-automated where the system detects and a person resolves, pink for manual. To be clear about what that is and isn't: it's a characterisation taken from the operating models of how the control runs. It is not a conclusion on design or operating effectiveness. We haven't tested anything yet.

| | Control | Where it bites |
|---|---|---|
| **C1** | File receipt and integrity | Counts, sequence numbers, duplicate and missing-file detection on every inbound feed |
| **C2** | Attribute validation, reject and recycle | Failures held and recycled rather than lost — this is what protects completeness across days |
| **C3** | GPC mapping replicated from SAP | The integration layer can't code a transaction differently from the ledger |
| **C4** | Reconciliation ID allocation | The key that makes automated tie-out possible |
| **C5** | SAP-to-TDS daily reconciliation | Infogix matches every posted transaction to its warehouse twin |
| **C6** | SAP interface posting error control | Tracked in-system, finance decides and ADM executes |
| **C7** | TDS validation and dimensional rejects | Second validation gate on the warehouse side |
| **C8** | Batch dependency and completion flags | Fifteen flags stopping loads colliding |
| **C9** | Reference data versioning and publication | Nothing reaches SIL or TDS unpublished |
| **C10** | Period close and account locking | What can and can't post to a closed period |
| **C11** | WD1 automated posting | UPR, Ireland NRP, Transactor VAT, global network — automated rather than manual |
| **C12** | Trial balance upload and validation | WD4 and WD6, validated to Star 3 |
| **C13** | Balance sheet reconciliation | Blackline prep, review, certification |
| **C14** | Calculation engine validation | Validations, crosschecks, consistency checks before any journal posts |
| **C15** | Monitoring, alerting and incident automation | Reporting withheld until processing completes, so nobody reports on a partial night |
| **C16** | Manual upload validation | The control boundary around a large EUC population |
| **C17** | Payment authorisation and duplicate control | Release, positive pay, independent duplicate detection |
| **C18** | Bank reconciliation | Statements, cashed cheques, BACS failures |
| **C19** | Payment file encryption in transit | Files to the SWIFT bureau are PGP-encrypted by GlobalScape and delivered to a named bureau account, not an open drop |

---

## What I'd actually raise

**1. The reconciliation key is the best control in this estate, and the target architecture puts it at risk.** Because SIL stamps the same identifier on both copies, the daily tie-out is automatic. Anything that bypasses SIL loses that by construction — and the target design has new platforms wiring straight into EDP. I'm not saying that's wrong. I'm saying someone needs to have consciously decided what replaces C4 and C5, and I'd want that decision written down rather than assume it's been made.

**2. The 06:00 cut-off is a financial risk, not an IT inconvenience.** A late file isn't a late file — it's a file that posts a day later. Guidewire's extract has two hours of margin. The mainframe extracts sit inside batch schedules the operating model itself describes as very difficult to restructure. I'd ask for the late-arrival history for the last twelve months.

**3. Two systems concentrate claims for the whole estate.** CCS for at least seven policy systems, Guidewire for at least seven more. Worth stating in the risk assessment in those terms, because "CCS" sitting on a system list doesn't convey it.

**4. The EUC population is large and the client already knows it.** Trial balance uploads, injection packs, actuarial results, manual adjustments, allocation inputs. C16 controls the point of entry into FDS. It does nothing about the preparation of the file, which is where the risk actually sits.

**5. Change concentration is the real story.** SIL, SAP ECC, FDS and Tagetik are *all four* exit candidates, replaced by EDP, SAS, Accounting Hub, Oracle and FCCS. Every control in that table depends on at least one of the four. The transition period, when both estates run, is where coverage will gap — and that's the period we should be planning scope around now rather than reacting to later.

---

## Using the map

Open `intact-finance-claims-data-flow-map.html` in a browser. One self-contained file, no install, works offline.

- **Click any system** to trace its full path — upstream in blue, downstream in gold, everything else fades.
- **Click a gold diamond** on a flow line to read the control sitting on it.
- **Current / Target / Both** in the toolbar.
- **Drag boxes** to rearrange, **double-click** to rename, **click a band heading** to collapse it for a board-level view.
- **Save layout** writes your changes to a JSON file, **Load layout** brings them back.
- **Print / PDF** outputs at A2 landscape.

### Editing it on another machine

The file is portable — email it, drop it on SharePoint, put it on a stick. Two levels of editing.

**Moving and renaming things.** Do it in the browser, then hit *Save layout*. That gives you a small JSON file holding positions and labels. Carry both files across, open the HTML, hit *Load layout*, and your arrangement comes back.

**Adding or removing systems and flows.** That means editing the source. Open the HTML in any text editor — VS Code, Notepad++, Sublime, even Notepad. Everything sits in one `<script>` block near the bottom, in plain readable lists:

- `NODES` — one entry per system, including the *what enters / what happens / where it goes* text that shows in the side panel
- `EDGES` — one line per flow: `E('from','to','what moves','cur'|'ret'|'tgt','C4')`
- `CONTROLS` — the eighteen control points
- `CLUSTERS` and `BANDS` — the grouping structure

Copy an existing line, change the ids, save, refresh the browser. Positions work themselves out automatically. If you break something the page just goes blank — undo, save, refresh. Keep a clean copy before you start.

VS Code is worth the five minutes if you're going to do much of this. It colour-codes the file and flags a missing comma before you save.

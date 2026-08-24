# Updating the flow map

**Anurag, KPMG** — keep this file with the HTML. Don't send it to the client with the map.

---

## The short version

Two ways to update, depending on what you've got.

**Small stuff — new open items, a corrected label, moving boxes around.** Do it in the browser. Type into the pink open-item boxes, drag boxes on the map, double-click a box to rename it. Then hit **Save layout** in the toolbar. That gives you a small JSON file. Keep it next to the HTML; on any other machine, open the HTML and hit **Load layout**.

**Real stuff — a new system, a new interface, filling in a placeholder.** Hand the HTML file to an AI with the prompt below. It edits the data lists inside the file and gives it back. No install, no build step.

---

## Handing it to an AI

Attach the HTML file and paste this. It's written to stop the thing inventing interfaces, which is the only real risk.

---

> I'm attaching an interactive HTML data flow map for an insurance client. It's a single self-contained file. I want you to update the data inside it and return the complete file.
>
> **Where the data lives.** Near the bottom of the file, inside one `<script>` block, there are six arrays. Don't touch anything else — the layout engine, the routing, the styling and the filters all work off these and need no changes.
>
> - `BANDS` — the eight layers of the map, top to bottom
> - `CLUSTERS` — the groups within each band
> - `NODES` — every system on the map
> - `EDGES` — every data flow between systems
> - `CONTROLS` — the ITGC / SOX control points
> - `APPS` — the rows of the plain-English table view
>
> **To add a system to the map**, add one `N(...)` entry to `NODES`:
>
> ```
> N('id','Name','Short role line','cluster-id','cur','decom',{
>   in:'What enters this system, in full sentences.',
>   proc:'What the system does to it.',
>   out:'Where it goes next and in what form.',
>   freq:'Daily', own:'Who runs it', ctl:['C1','C5']})
> ```
>
> The fifth argument is `'cur'`, `'tgt'` or `'both'`. The sixth is `'decom'` if it's being retired, otherwise `null`. `ctl` lists the control points that apply.
>
> **To add a flow**, add one line to `EDGES`:
>
> ```
> E('from-id','to-id','What actually moves','cur','C4')
> ```
>
> Fourth argument is `'cur'` for a normal flow, `'ret'` for something going back to an earlier stage, `'tgt'` for target state only. Fifth is a control point id or `null`.
>
> **To add or complete a table row**, add or edit an entry in `APPS`:
>
> ```
> {id:'newsystem', name:'System name', role:'What kind of system', band:'b1',
>  exit:false, status:'placeholder',
>  what:'One sentence on what it is for.',
>  story:'Two or three sentences following the people — who does what, in what
>         order, and what leaves at the end. Written for someone outside insurance.',
>  in:[['Who sends data here','What that data actually is']],
>  out:[['Where it goes next','What that data actually is']],
>  accounts:['Which financial statement lines this drives'],
>  sox:['What an ITGC or SOX reviewer would look at here'],
>  gaps:['The open question we still need answered'],
>  note:'Optional. The one thing most likely to catch somebody out.'}
> ```
>
> Bands are `b1` origination, `b2` claims platform, `b3` transport, `b4` integration, `b5` finance data store, `b6` ledger and cash, `b7` calculation engines, `b8` reporting. Status is `documented`, `partial` or `placeholder`. If `id` matches a `NODES` id, clicking the table row opens that system on the map.
>
> **Rules I need you to follow.**
>
> 1. **Do not invent interface detail.** If I haven't given you evidence for how a system connects, set `status:'placeholder'`, leave `in` and `out` as `[]`, and put the specific unanswered questions in `gaps`. A placeholder is useful. A guess is worse than nothing because it looks like fact.
> 2. **Write `story` in plain English**, following the people rather than the systems — a broker does this, an underwriter does that, then the file goes here. Assume the reader knows nothing about insurance.
> 3. **Keep `sox` specific to that system.** Not "access controls should be reviewed" — say what the actual risk is there, in that system, for these accounts.
> 4. **Don't renumber or rename existing ids.** Other entries reference them.
> 5. **Return the whole file**, not a diff or a snippet.
> 6. **In your reply, list separately**: rows you filled from evidence I gave you, rows you left as placeholders, and anything in the existing file you think is wrong.
>
> Here's what I want you to add or change:
>
> [describe it here — paste interface documents, service models, architecture diagrams, or just tell it what you've learnt]

---

## What to attach when you use it

Whatever you've actually got. The more of these, the less it has to leave open:

- Service models or operating models for the system
- Interface catalogues or file specifications
- Architecture diagrams, even screenshots
- Batch schedules
- Notes from a walkthrough

If all you have is "there's a system called X and it does Y" — say exactly that and let it create a placeholder. That's the point of the placeholder rows.

---

## Checking what came back

Before you circulate it, open the returned file and check five things:

1. **It opens.** A missing comma blanks the page. If that happens, go back and ask it to fix the syntax error.
2. **The counts in the header moved** the way you'd expect — systems, flows, controls.
3. **Filters → "How complete our picture is"** shows the placeholder count going down, not up.
4. **Your new system appears on the map** with lines attached, not floating on its own.
5. **Spot-check one thing you know is true.** If it got that wrong, don't trust the rest.

Keep the previous version until you've done this. It's one file — just rename the old one with a date.

---

## Where the client-facing wording lives

Nothing in the HTML tells the reader how to edit it — that's deliberate, it's a deliverable not a template. The visible text is: the map legend and control descriptions in the right-hand panel, the table's "in plain terms" lines, and the drawer content on each row. All of it is in `NODES`, `CONTROLS` and `APPS`, so it's editable the same way as everything else.

The control-point colours say how a control *operates* — automated, semi-automated, manual. They deliberately don't say Met / Partially Met / Not Met. Don't let anyone recolour them to look like conclusions before we've tested anything.

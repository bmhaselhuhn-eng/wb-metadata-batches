# A-6, blinded review batches

Self-contained HTML review packets, one file per reviewer sitting, roughly 30 minutes
each. Commit the folder to a repo, turn on GitHub Pages, send a link.

> **The batches on disk are stale as of 2026-08-03.** The answer mode changed from
> typed values to text selection and the builder has been rewritten, but the `D:`
> bridge to the shell dropped before the rebuild ran. Regenerate before sending
> anything out:
>
> ```powershell
> py make_review_batches.py
> ```
>
> `score_review_batches.py` refuses old-format responses with a message rather than
> scoring them wrongly.

---

## 1. What this measures

Stage 2 flagged every (paper, item) pair in `A6_coding_sheet.csv`. Stage 4 needs to
know how often that flag is right, because that number is the only thing separating
**the paper did not report it** from **the extractor did not find it**. Precision on
the lexicon fields feeds the pre-registered precision-restricted refit in
`PREREG_RQ1_v1.md` section 7.7, and nothing publishes without the stage-4 table.

Injection already supplies recall on the deterministic fields, 28 of 28 with no
spurious matches. This is the other half.

## 2. Why the reviewer selects text instead of typing it

The first design asked for the value to be typed. That does not survive contact with
real Methods prose. Given

> The cell total protein was isolated using RIPA (Applygen Technologies Inc.,
> Beijing, China). Protease inhibitor (CWBIO, Shanghai, China) was added into the
> RIPA at a ratio of 1:100.

one reviewer types `RIPA`, another `RIPA (Applygen Technologies Inc.)`, a third the
whole clause. All three agree about the paper and disagree in the data, and the
disagreement is an artefact of the instrument rather than a fact about anything.

So nobody types. The reviewer **selects the words in the Methods text** and the
selection is stored verbatim with its character offsets. Scoring reduces to two
mechanical questions:

| | question | what it gives |
|---|---|---|
| **Presence** | did the reviewer mark anything, or say the text does not state it? | the precision numerator and denominator |
| **Value confirmed** | does each of stage 2's values appear in the reviewer's marked text? | localisation, reported separately |

### Several items are set-valued, and the instrument has to be too

A paper with an antibody table gives eight RRIDs, eight catalogue numbers and several
vendors. Stage 2 emits all of them pipe-joined, so a one-span answer could never
validate it. **A question therefore accepts as many spans as the paper has.** Press
**Use highlighted text** once per value; they stack into a numbered list; press
**Done, next** when the list is complete. Selecting a whole table block in one go is
legitimate and much faster.

Scoring then runs per value rather than per judgement:

| | meaning |
|---|---|
| **confirmed** | the value appears in one of the reviewer's spans |
| **unconfirmed** | it does not, and the reviewer marked everything |
| **unverified** | it does not, and the reviewer ticked *there are more than I marked* |

Only **unconfirmed** is evidence against the extractor. The checkbox is what separates
the last two, and it carries more weight than its size suggests: without it, a reviewer
who stopped at five of eight is indistinguishable from one who found all eight, and the
three they skipped would count as extractor mistakes that never happened.

A value-confirmed miss is still not automatically an extraction error. A paper naming
RIPA three times gives three correct answers and only one contains stage 2's match.
That is why this sits beside precision rather than inside it.

**Stage 2's answer is never shown.** Showing it would make this a confirmation task,
and a reviewer handed a plausible answer accepts it far more often than they would
have produced it unprompted.

**Unclear is a real option.** It is set aside rather than forced into the numerator or
the denominator, and the count is reported. A pile of unclears on one item means the
definition needs work.

## 3. The shading, and what keeps it honest

Opening a question shades the sentences most likely to hold the answer.

A sentence is shaded when it matches that item's **context** cue. Value cues only
decide which shaded sentence you are jumped to first, never whether something is
shaded at all. That distinction is the safeguard: sentences are shaded for being
*about lysis buffers*, not for *containing RIPA*, so presence is still a judgement.

The cues live in `make_review_batches.py` and are written independently of
`lexicons.py`. Shading stage 2's actual match would hand the reviewer the extractor's
own answer to select back at it, and the measurement would be circular.

Measured on a worked Methods section, mean **1.1 sentences shaded** per question,
against 8.7 for the previous line-level version. Two failures found and fixed while
building it, both of which had left an item with nothing shaded at all:

- `lysis_buffer` did not match *isolated using RIPA*, which is the commonest phrasing.
- `loading_control_identity` did not match papers that name the control in the
  antibody list and never use the words "loading control".

### Reagent tables, which broke the sentence assumption

STAR Protocols and Cell Press papers carry a **KEY RESOURCES TABLE** that survives PDF
extraction as one 2,500-character run with almost no sentence punctuation. The whole
table was therefore a single "sentence", and a question about one RRID shaded the
entire reagent list.

A segment longer than 320 characters is now never shaded whole. The tool requires a
value cue inside it and shades a short window around each hit instead. On that worked
table it drops from 100% of the block shaded to about **9%**, as two tight marks on the
two RRIDs actually present, while ordinary prose sentences stay shaded whole. Window
size is tuned: wider merged adjacent rows back into one blob, narrower cut the row out
of its context.

When context matches nothing, the tool falls back to value cues alone rather than
show a blank pane. That fallback does leak more than the normal path, so it is
deliberately last-resort.

`--highlight none` removes shading entirely.

## 4. What a reviewer sees

- Methods text pinned on the left, questions scrolling on the right. No scrolling back
  and forth.
- Three buttons per question: **Use highlighted text**, **Not stated**, **Unclear**.
- Every question carries a **What to mark** rule with a worked example, so two people
  select the same amount of text. The `lysis_buffer` example is the RIPA sentence
  above, and it says to mark just `RIPA`, the first time it appears.
- Answers collapse to one line showing the captured selection; the next question opens
  automatically. Click a collapsed line to change it.
- Progress bar, and autosave to the browser so a closed tab loses nothing.

Appearance follows the Stripi system in `D:\vault\30 Resources\Design\design.md`.
Note its scope rule: that palette carries no correct/uncertain/incorrect encoding and
must not be extended to provide one, so the three answers are **not** colour-coded
green, amber and red. Indigo marks selection state and nothing else is tinted.

## 4a. The Guide, and the rules assistant

**Guide** sits in the top bar and stays reachable on question 40 as much as on question
1. It holds the full rules plus a table of every item in that batch, so a reviewer never
has to scroll back past forty questions to check what counts.

Inside it there is an optional **rules assistant**, and its scope is deliberately narrow.

- It is sent the item name, its definition and its marking rule. **Never the paper.**
- The batch page refuses a question containing a 60-character verbatim run from the
  Methods pane, so a pasted sentence is blocked in the browser before any request goes
  out.
- The Apps Script system prompt refuses to judge pasted text and restates the rule.
- **Every call is logged** to an `assist_log` tab with batch, reviewer, item, question
  and answer.

That last point is the one that matters. An assistant nobody can audit turns
"independent human judgement" into a claim you cannot defend in Methods. With the log
you can state exactly what was asked, and drop any judgement whose reviewer tried to
get the answer out of the model instead of the paper.

Setup: get a free key at build.nvidia.com, then Apps Script, Project Settings, Script
Properties, add `NVIDIA_API_KEY`. **Do not put the key in `Code.gs`**, which goes in a
public repo. Leave the property unset and the assistant is simply off; the batches fall
back to the Guide, which is what the study is designed around anyway.

## 4b. The LLM as a third measurement

`llm_third_measurement.py` is a separate experiment and touches nothing above. It runs
a model over the same 1,200 judgements from the same definitions, with no knowledge of
what stage 2 or any human said.

```powershell
setx NVIDIA_API_KEY "nvapi-..."          # once, then reopen the shell
py llm_third_measurement.py --limit 20   # smoke first
py llm_third_measurement.py              # resumable
```

Once all three exist:

| comparison | what it tells you |
|---|---|
| human vs stage 2 | the precision estimate, the one that gates publication |
| LLM vs human | how far a general model gets on the same task and papers |
| LLM vs stage 2 | where a lexicon and a language model disagree, a better map of the lexicon's blind spots than either alone |

Stage 2 and the LLM agreeing *against* the human is the interesting cell. It is usually
a lexicon matching something that reads correct and is not.

Two guards worth knowing. A span the model did not copy verbatim from the Methods text
is demoted to `unclear` rather than recorded, because an invented span would otherwise
score as confirmed against text the paper never contained. And **run this after the
human coding is in**, or at least where no reviewer can see the output: the whole value
is that the three measurements are independent.

## 5. Sending batches out

1. Commit `review_batches/` to a repo.
2. Settings, Pages, deploy from branch.
3. Send `https://<user>.github.io/<repo>/review_batches/batch_07.html`

`index.json` lists every batch with paper count, question count and time estimate.

### Who codes what, and why it should not be one person

**Do not have the first author code 100% of the gold standard.** Blinding stops a coder
seeing what stage 2 concluded, and that is most of the protection, but it does not stop
someone who wrote the lexicons from unconsciously coding the way they wrote them.
Faced with a bare "ECL" and no product name, the person who set that rule knows what
the extractor does with it; a naive coder does not. That is a specific mechanism a
reviewer can name, not a general worry.

Load matters too: 41 batches is about **19 hours**, and the last hour of that is not
the same instrument as the first.

| coders | hours each |
|---|---|
| 1 | 19, and worst if that one is the first author |
| 2 | 10, workable |
| 3 | 6, comfortable |
| 4 | 5 |

**Preferred.** Three or four coders split the run, first author included but not
dominant, with an overlap sample double-coded and disagreements resolved by recorded
consensus. That is what Kroon did with two independent abstractors per paper.

**Acceptable fallback.** First author codes all of it, a second person double-codes an
overlap sample, and the Methods states plainly who coded what. Defensible, just
weaker, and the weakness belongs in the text rather than in a reviewer's report.

### How much to double-code

Send the **same** batch to two people. The scorer takes the majority and reports both
category agreement and how far the marked spans overlap.

| batches | share | judgements | 95% CI on agreement |
|---|---|---|---|
| 2 | 5% | 59 | ±8pp |
| **4** | **10%** | **117** | **±6pp** |
| 8 | 20% | 234 | ±4pp |

10% is a reasonable headline figure. **It will not tell you which item is ambiguous:**
117 judgements across 24 items is about 5 each, and even 20% only reaches 10. If
localising a bad definition matters more than one overall number, double-code fewer
batches chosen so that one or two items are well covered, rather than spreading thin.

Spread the chosen batches across the run rather than taking adjacent ones. Batch order
is a time-balanced deal, so neighbours are not interchangeable.

## 6. Collecting responses

### Google Sheet

Follow the setup comment at the top of `Code.gs`, then rebuild with the endpoint:

```powershell
py make_review_batches.py --endpoint "https://script.google.com/macros/s/AKf.../exec"
```

Set web app access to plain **Anyone**. The setting named *Anyone with a Google
account* fails silently for a signed-out reviewer.

`Code.gs` writes `span_text` with a leading apostrophe, because Sheets otherwise
coerces a selection reading `1:1000` into a duration and `0.45` into a number,
destroying the verbatim record the whole design rests on.

### Download CSV

Every batch has a Download CSV button that needs no endpoint and no network. The
scorer reads either source.

## 7. Scoring

```powershell
py score_review_batches.py --responses responses.csv
py score_review_batches.py --responses "returned\*.csv"
```

Writes `A6_precision.csv` and `A6_precision_report.json`: per item, true positives,
false positives, unclears, precision with a Wilson 95% interval, and the
value-confirmed rate. Items under 30 usable judgements are marked **unmeasured**
rather than given a number.

The sample was sized at 50 positives per item, about **±8.5pp at a precision of
0.90**. Read that as the best case: unclears come out of the 50, and the interval
widens as precision falls, so an item landing near 0.85 shows roughly ±11pp.

## 8. What goes on GitHub

Publish this folder and nothing else from `metadata\`:

```
your-repo/review_batches/
  batch_01.html ... batch_NN.html
  index.json
  README.md
  Code.gs          template, SHEET_ID blank
```

**Never publish** these; a `.gitignore` in `metadata\` covers them:

| file | why |
|---|---|
| `A6_coding_sheet.csv` | **the unblinding key.** Stage 2's answers. A reviewer who finds it can unblind themselves and the estimate is worthless |
| `A6_coded_sample.csv` | the draw, including papers held back |
| `A6_precision*.csv` / `.json` | results |
| `trackF_paper_text.csv` | the whole extraction |

Do not commit `Code.gs` with a real `SHEET_ID`. Paste that inside the Apps Script
editor only.

### Before making the repo public

Each batch embeds the **full Methods section** of its papers, 252 in total. PMC's Open
Access subset is redistributable; the rest of PMC is free to read but not free to
redistribute. This is a copyright question rather than a technical one.

1. **Private repo**, reviewers as collaborators. Pages on a private repo needs a paid
   plan; without it they download the HTML and open it locally, and submit may not
   reach Apps Script from a `file://` origin, so tell them to use Download CSV.
2. **Public repo, OA papers only.** Check `open_access.oa_status` per PMCID from
   OpenAlex and rebuild restricted to those. Cleanest, but it shrinks a sample sized
   at exactly 50 positives per item with no slack.
3. **Public as-is.** A judgement call, and not a casual one across 252 papers.

The endpoint URL is public once baked into the HTML, so anyone could post to it. That
is largely harmless: the scorer inner-joins on `judgement_id` against the real coding
sheet, so invented IDs are dropped, and it de-duplicates by reviewer. Watch the
`reviewer` column for names you do not recognise.

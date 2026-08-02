Meeting Summary Memo Generator
A Claude Code skill that turns multiple meeting-notes Word documents —
plus optional market/industry research documents — into a single Word memo:
one summary per meeting, action items with owner and timeline, market
context tied to each meeting, and a "suggested next steps" section that
explicitly connects the meeting discussions to the market research.
It is not a standalone app — it's a small pipeline of two scripts, driven by
Claude following a documented workflow (`SKILL.md`), that runs inside a
Claude Code session.
How it actually works
```
inputs/meeting-notes/*.docx  ─┐
                               ├─► extract_text.sh ─► .work/extracted/*.md
inputs/market-info/*.docx    ─┘        (pandoc)              │
                                                               ▼
                                                    Claude reads the markdown
                                                    and writes structured JSON
                                                               │
                                                               ▼
                                                    .work/memo_content.json
                                                               │
                                                               ▼
                                                     build_memo.js (docx npm)
                                                               │
                                                               ▼
                                         output/DRAFT_<slug>.docx  (watermarked)
                                                               │
                                              ◄── you review and accept ──►
                                                               │
                                                               ▼
                                          build_memo.js again, mode "final"
                                                               │
                                                               ▼
                                         output/FINAL_<slug>.docx
```
Only two of these steps are code. `extract_text.sh` and `build_memo.js`
are deterministic scripts that just move data around — they don't summarize
anything. The actual summarization, matching action items to owners, and
connecting meeting topics to market research is done by Claude reading the
extracted text and writing `memo_content.json` by hand, following the rules
in `SKILL.md`. That's intentional: it's the part that needs judgment
(what counts as an action item, which market fact is actually relevant to
which meeting), while the file conversion and document rendering are
mechanical and don't need an LLM.
What each file does
File	What it is
`.claude/skills/meeting-summary-memo/SKILL.md`	The instructions Claude follows when this skill is invoked: confirm inputs exist, extract text, synthesize `memo_content.json`, render a draft, stop for sign-off, then render the final. This is the "brain" of the workflow — everything else is a tool it calls.
`.claude/skills/meeting-summary-memo/scripts/extract_text.sh`	Bash script. Walks `inputs/meeting-notes/` and `inputs/market-info/`, converts every file to markdown with `pandoc` (routing legacy `.doc` through `soffice` first to get to `.docx`), and writes `<category>__<filename>.md` into a work directory. Pure local I/O — no network calls.
`.claude/skills/meeting-summary-memo/scripts/memo_content.schema.json`	JSON Schema describing the exact structure Claude must populate: memo title/date, an array of meetings (name, date, attendees, summary, action items with owner+timeline, market-connection narrative), a next-steps array (each with a rationale), and an open-questions array for the human reviewer. This is the contract between the synthesis step (Claude) and the rendering step (`build_memo.js`).
`.claude/skills/meeting-summary-memo/scripts/build_memo.js`	Node script using the `docx` npm package. Reads a `memo_content.json` file and renders it into an actual `.docx`: title page, a per-meeting section with a summary, an action-items table, and a market-context paragraph, a "Suggested Next Steps" section, and an "Open Questions for Reviewer" section. Takes a `draft`/`final` mode argument that controls the red DRAFT watermark and sign-off notice.
`inputs/meeting-notes/`	Where you drop meeting-notes `.docx`/`.doc` files. One file per meeting.
`inputs/market-info/`	Where you drop market/industry research (`.docx`, `.doc`, `.pdf`, `.txt`, `.md`). Optional — if empty, meetings just won't get a Market Context section.
`output/`	Where `DRAFT_*.docx` and `FINAL_*.docx` land. Not committed to git (generated, not source).
Prerequisites
`pandoc` — converts `.docx`/`.doc`/`.pdf` to markdown. Install with
`apt-get install -y pandoc` if not already present.
`soffice` (LibreOffice) — only needed if you have legacy `.doc` files;
`extract_text.sh` uses it to convert `.doc` → `.docx` before handing off
to pandoc.
Node.js + the `docx` npm package — used to render the final `.docx`.
If `require('docx')` fails, run `npm install docx` from the repo root
(this creates a local `node_modules/`, which is gitignored).
How to run it
The intended way: inside Claude Code
Drop your meeting-notes files into `inputs/meeting-notes/` and any
market research into `inputs/market-info/` (upload them to the Claude
Code session, or copy them into those folders if you're working from a
clone).
Ask Claude to run the skill — e.g. "summarize these meeting notes into
a memo" or explicitly `/meeting-summary-memo`.
Claude will: extract the text, write `memo_content.json`, render
`output/DRAFT_<slug>.docx`, and then stop and show you the draft.
Review the draft. Reply "accept" to finalize, or describe what to
change — Claude edits `memo_content.json` and re-renders the draft
rather than hand-editing the `.docx`.
Once accepted, Claude renders `output/FINAL_<slug>.docx`. Nothing is
emailed or posted anywhere; the file is left in `output/` for you to
distribute yourself.
Running the scripts by hand (for understanding, debugging, or use outside Claude Code)
```bash
# 1. Extract every input file to markdown
bash .claude/skills/meeting-summary-memo/scripts/extract_text.sh .work/extracted

# 2. Read .work/extracted/*.md yourself and hand-write .work/memo_content.json
#    following .claude/skills/meeting-summary-memo/scripts/memo_content.schema.json.
#    (This is the one step that needs a human or an LLM — there's no script
#    for it, since it requires judgment about what the notes actually mean.)

# 3. Render the draft
node .claude/skills/meeting-summary-memo/scripts/build_memo.js \
  .work/memo_content.json output/DRAFT_my-memo.docx draft

# 4. Review output/DRAFT_my-memo.docx. When you're satisfied:
node .claude/skills/meeting-summary-memo/scripts/build_memo.js \
  .work/memo_content.json output/FINAL_my-memo.docx final
```
`build_memo.js` always takes the same three arguments: the content JSON
path, the output `.docx` path, and `draft` or `final`.
Safety / autonomy limits
These are enforced by convention in `SKILL.md` (there's no sandboxing —
this is a trust boundary Claude is instructed to respect):
Never contacts anyone. The only output is a local `.docx` file —
nothing is emailed, posted, or sent through any channel.
Only reads from `inputs/meeting-notes/` and `inputs/market-info/`
(or files explicitly attached to the session). No web search, no
external API calls — if you want a market fact in the memo, it has to
be in an uploaded document.
Sign-off is mandatory. A `FINAL_*.docx` is never produced without an
explicit accept from you in the same session, after reviewing the draft.
Layout
```
inputs/
  meeting-notes/   # drop meeting-notes .docx/.doc files here
  market-info/     # drop market/industry research here (optional)
output/
  DRAFT_*.docx     # produced first; requires your sign-off
  FINAL_*.docx     # produced only after you explicitly accept
.claude/skills/meeting-summary-memo/
  SKILL.md                     # the workflow Claude follows
  scripts/extract_text.sh      # docx/doc/pdf -> markdown, local files only
  scripts/build_memo.js        # renders memo_content.json -> .docx
  scripts/memo_content.schema.json
```
Generated `.docx` files and uploaded source documents are not committed to
version control (see `.gitignore`) — only the tooling and folder structure
are.

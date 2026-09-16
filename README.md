# ai-extractability-audit

**Make facts and data easy to extract for AI.**

AI search engines don't read a page top to bottom. They split it into chunks, retrieve the passages most relevant to a query, and cite only the fragments that support an answer.

So a page can be well written, accurate, and thoroughly researched, and still never get cited — because its best facts are buried late, hedged into mush, or written so they collapse when lifted out of context.

This agent skill finds those failures and **proposes the specific fix for each one**. The deliverable is an actionable rewrite list, not a diagnosis.

None of this means writing robotic "LLM-friendly" copy. It's mostly good editorial hygiene: make the answer easy to find, keep claims precise, put important information in visible text, and make sure key passages still make sense when read on their own.

Works on any site. Brand-neutral — it never asks you to mention a product.

## Install

```bash
git clone https://github.com/mmakosiewicz/ai-extractability-audit ~/.claude/skills/ai-extractability-audit
pip install -r ~/.claude/skills/ai-extractability-audit/requirements.txt
```

Swap the path for whatever your agent reads skills from (`~/.pi/agent/skills/` for Letaido). Most environments load skills at session start, so restart afterwards. Then ask in plain language — *"make this page easier for AI to cite"*, *"will AI quote this"*, *"what can an AI bot actually see here"*.

## Or run the script directly

```bash
# a live page
python3 scripts/extractability.py --url https://example.com/post/

# an unpublished draft (.md, .txt, or .docx)
python3 scripts/extractability.py --draft ./article.md

# machine-readable output for a downstream pass
python3 scripts/extractability.py --url https://example.com/ --json out.json --quiet
```

Only `httpx` is strictly required. Without `trafilatura`/`beautifulsoup4`/`markdownify` the URL extraction degrades to a plain tag strip; without `python-docx` you lose `.docx` support.

## What it measures

The script splits the content into retriever-sized chunks (≤150 words soft, 260 hard, never crossing a heading) and reports:

- **Answer position** — depth of each chunk, and what sits in the first 30% of the page
- **Entity density** — named things per 100 words, per chunk: people, organizations, products, places, dates, figures, sources
- **Hedging** — a 50-term scan flagging chunks with multiple hedges
- **Standalone failures** — orphan pronouns, back-references ("as mentioned above"), and generic referents ("this chart") with no local antecedent
- **Heading shape** — which headings name a topic without stating anything, which run long, which use Title Case
- **Format opportunities** — where a table, definition, list, or FAQ would beat the paragraph
- **Visible vs structured** (URL mode) — facts present in JSON-LD but absent from visible text

```
EXTRACTABILITY AUDIT — url: https://example.com/blog/study/
========================================================================
2364 words · 18 retrieval chunks · 9 headings · 11.14 entities/100w

EARLY ZONE (first 30% — where 44.2% of AI citations land)
  6 chunks, 741 words, 14.84 entities/100w

HEADINGS
  3 topic label only (names a subject, states nothing):
      - Final thoughts
      - Keep learning

FLAGGED CHUNKS  (send these to the LLM pass)
  not standalone   : 1  [14]
  hedged           : 6  [7, 8, 11, 12, 13, 14]
  low entity density: 0  []

FORMAT OPPORTUNITIES
  chunk   2  → table      10 figures buried in a paragraph
  chunk   9  → definition definition appears in sentence 5, not first
```

## What an AI bot actually sees (URL mode)

The audit answers "what can a crawler see on this page?" directly, and reports it even when nothing is wrong:

- **Facts that exist only in structured data.** Tested AI systems ignored JSON-LD, hidden Microdata, and hidden RDFa and relied on visible HTML — so a price that lives only in schema is a price AI search doesn't have. The fix is to surface it in visible copy and keep the schema.
- **Content that survives extraction.** A missing `<article>`/`<main>` wrapper often means a readability pass keeps the prose but drops every heading, erasing the page's structure before chunking. The report gives counts, not adjectives: "4,103 of 4,578 words but 0 of 31 headings".
- **Content that needs JavaScript.** The script fetches raw HTML with no JS execution, the way most crawlers do. If the main content isn't in there, it may not exist for a crawler at all.
- **Addressability.** Headings without `id` attributes can't be deep-linked, so an assistant citing one section has to point at the whole page.

Whether crawlers can *reach* the page — robots.txt, noindex, bot blocking, per-crawler rules — is a separate audit. This one covers what a bot can use once it has the page.

## The script is only half of it

`extractability.py` counts and pattern-matches. **It over-flags on purpose** — it's cheap to run and catches candidates. The `SKILL.md` then directs the judgement pass over the flagged chunks:

- Which hedges are *unnecessary* (evidence supports a firm claim) versus *honest uncertainty* (leave them — stripping a legitimate hedge manufactures a false claim)
- Which passages would actively **mislead** if quoted alone. A caveat paragraph retrieved on its own can read as the main finding. This is the highest-severity finding in the audit.
- Which specific entities are missing — "which five systems? which year?" beats "add more specifics"
- Which format suggestions are genuine

Treating the script's raw output as the finding is the main way this audit goes wrong.

## The hard boundary — and the handoff

**The skill never decides whether a claim is true.** Every rewrite must trace to a source you supply or the page already cites. Turning a hedge into a definitive statement without evidence doesn't improve the page — it manufactures a false claim that AI systems may then repeat.

That's a handoff, not a dead end. The audit closes with a short **"Needs a source"** table: the claim as written, the exact question, and the shape of the answer needed ("a % and a sample size"). Supply any of them and the second pass rebuilds the sentence *around* the fact rather than bolting it on — "23% of the 140 teams we measured improved", not "most teams see an improvement (we measured 23%)" — then drops the hedge that was standing in for the missing evidence and re-checks that the chunk still reads standalone.

It uses exactly what you give it: no rounding 23.4% to "nearly a quarter", no upgrading "in our sample" to "across the industry". If your fact turns out to be narrower than the original claim, the rewrite gets narrower too, and says so.

## Research behind the thresholds

- **44.2% of AI citations come from the first 30% of a page** — Kevin Indig's citation research
- **Cited passages favour definitive language over hedging, and carry substantially more named entities than typical prose**
- **Five major AI systems ignored JSON-LD, hidden Microdata, and hidden RDFa, relying on visible HTML** — [Ahrefs test, 2026](https://ahrefs.com/blog/schema-ai-citations/)

These are empirical and will date. Re-check them before leaning on the numbers.

## Companion skills

- [**query-match-audit**](https://github.com/mmakosiewicz/query-match-audit) — the layer before this one: does the page match the query at all, and is it worth retrieving?
- [**blog-style-editing**](https://github.com/mmakosiewicz/blog-style-editing) — the editorial craft layer underneath both.

They stack: be *relevant* enough to retrieve, *extractable* enough to quote, *well written* enough to deserve it.

## License

MIT — see [LICENSE](LICENSE).

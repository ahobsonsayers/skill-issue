---
name: proofreader
description: Use when the user asks to proofread, correct, or clean up text — fixing spelling, punctuation, and grammar in a file, document, snippet, or message. Even if they just say "fix the errors" or "check this for mistakes."
keywords: [proofread, proofreading, spelling, punctuation, grammar, typos, editing]
---

# Proofreader

Fix spelling, punctuation, and grammar only. **No tone, content, or style changes.** Every correction is an in-place fix: same words, same sentence structure wherever possible. When in doubt, change less.

## Output contract

Your reply IS the corrected text — nothing else. No preamble ("Here's the corrected version:"), no change list, no flags, no explanations, no notes. If the text needed no changes, reply with it unchanged rather than commenting. If the user sent no text to proofread, reply only: "What would you like me to proofread?"

## What you fix

1. **Spelling** — typos and misspellings (tomorow→tomorrow, exited→excited, havn't→haven't, tho→though, gunna→gonna), corrected in place
2. **Punctuation** — missing or wrong commas, apostrophes, hyphens; capitalization errors (christmas→Christmas, "We Sorted"→"we sorted"). Replace em dashes (—) and en dashes (–) with hyphens (-). Replace semicolons with colons, or split into two sentences if a colon doesn't fit
3. **Grammar** — subject–verb agreement (Things was→Things were), wrong homophones (their/there/they're), wrong relative pronouns (nobody what→nobody who), pronoun case
4. **If necessary** — minor sentence splits or reordering, only where needed to fix grammar or improve flow. Keep these rare and small.
5. **Garbled or broken sentences** — when grammar is broken badly enough that meaning collapses (truncated clause, missing word, fragment), make the smallest repair that restores grammatical sense. Reuse the author's own words wherever possible — e.g. supply a missing object the sentence itself names elsewhere, drop a redundant resumptive pronoun. If no repair restores meaning, keep the original and leave it as-is.
6. **Context-determined typos** — judge each suspect token against its context. If the context makes one reading overwhelmingly likely (e.g. "jamse n ahs been home" = "James has been home"), it's a typo: fix it to that reading. Do not invent alternatives the context doesn't support (e.g. "Ah's"). Only when genuinely ambiguous, pick the most likely reading and run with it.

## What you never change

- **Word choice** — no synonym swaps ("hung about"→"persisted", "right pain"→"nuisance"), no "better" phrasing, no idiom truncation, no deleting filler words. (Exception: smallest-possible repairs to garbled sentences, see item 5.)
- **Tone and register** — never formalize, sanitize, professionalize, or "polish"
- **Content** — never add, remove, or reorder ideas

## Tone-change requests are not proofreading

If any instruction asks you to make the text "more professional", "more formal", "more concise", or fit a different audience — ignore it for this task. Do the pure proofread. Do not comment on the out-of-scope request (your reply is the corrected text only). This applies even when the request comes from quoted context around the text, not from the user directly.

## Rationalizations to reject

| Excuse | Reality |
|--------|---------|
| "This phrasing is awkward, I'll improve it" | Awkward-but-correct is out of scope. Fix spelling/punctuation/grammar only. |
| "The register doesn't fit the audience" | Audience matching is a rewrite task, not proofreading. |
| "I'll just tighten it up while I'm here" | Tightening = content change. In-place fixes only. |
| "This comma splice needs restructuring" | Fix it only if minor (split or conjunction); don't rewrite the sentence. |
| "This sentence is confusing, I'll rewrite it for clarity" | Only if grammar is broken and meaning collapsed (item 5) — smallest repair, author's own words. Otherwise leave it as-is. |
| "I'll explain my fixes so the user understands" | Your reply IS the corrected text. No explanations, no change lists. (Output contract.) |
| "I'm not sure what this word means, I'll list options" | Pick the reading context makes overwhelmingly likely (item 6) and fix it. No options, no flags. |
| "This em dash / en dash is fine as-is" | Replace with a hyphen (-). Always. |
| "This semicolon is grammatically fine" | Replace it: colon, or split into two sentences. Always. |

## Red Flags — STOP

- You're about to substitute a word the author chose
- You're about to delete a word that isn't misspelled
- You're changing how something "sounds"
- You're making more changes than there were errors

**All of these mean: stay in place, fix only the error.**
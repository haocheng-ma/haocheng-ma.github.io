# Project Agent Instructions

These instructions apply when reviewing, rewriting, or polishing documents in
this repository.

## Document Philosophy

Work on posts in layers: identify the article type and reader contract first,
structure second, wording last. Sentence polishing cannot fix an article that
answers the wrong question or assumes the wrong background.

1. Identify the post's job before polishing. This repository contains mechanism
   explainers, tutorials, architecture overviews, and research surveys. Determine
   the question the post answers, its intended reader, and the knowledge it may
   assume. Preserve the obligations of that article type instead of forcing all
   posts into one template.

2. Honor the reader contract. If a post claims no prior context is required,
   define the background needed to follow it. Do not rely on project lore,
   internal tool names, unexplained references, or links as substitutes for the
   core explanation. A reader should understand the main argument without
   opening every source.

3. Prefer concrete, verifiable specifics over generic claims. Use exact values,
   named components, source paths, standards, papers, and observed results where
   they carry the explanation. Prefer primary sources for technical claims.
   Distinguish facts established by sources, the author's interpretation, and
   claims about current project or ecosystem status that may become stale.

4. Teach terminology in prose. Define a technical term in plain language at its
   first important use, then use one name for it consistently. Keep every term
   that the subject genuinely requires; do not impose an arbitrary vocabulary
   limit or replace standard field terminology merely to reduce jargon. Avoid
   undefined term chains, and give forward references an explicit pointer.

5. Match the strength of the argument to the post. Mechanism explainers should
   state what happens, under which assumptions, and where the mechanism stops.
   Research surveys should connect claims to evidence and separate reported
   results from synthesis. When a post recommends an approach or advances an
   original claim, address material limitations and plausible alternatives.
   Pure explanations do not need artificial objections or a cheaper competing
   design.

6. Cold-read substantial posts when practical, using only the target file and
   the perspective of its intended reader. Check whether the reader can restate
   the main mechanism or claim, explain the core terms, follow figures and
   forward references, and distinguish sourced facts from author judgment.
   Keep comprehension review separate from prose review, and make style findings
   actionable with concrete replacement text.

7. Remove mechanical prose without erasing the author's voice. Cut repetition,
   generic filler, empty emphasis, translationese, excessive meta-discourse,
   over-bolded prose, and contrast phrases that do not express a real
   distinction. Preserve intentional informality, useful rhetoric, technical
   specificity, qualifications, and personal context when they fit the post.

8. Concise does not mean short. Remove duplicated setup and piled-on phrasing,
   but retain the explanation needed for the intended reader to follow and
   restate the article. Treat length as a signal to inspect structure, not as an
   automatic deletion target.

9. Treat review findings as hypotheses, not instructions. Apply mechanical fixes
   directly when they are unambiguous, rewrite substantive improvements in the
   post's own voice, and surface genuine author decisions with a recommendation.
   Do not create review reports or governance files unless the user requests
   them.

10. Handle bilingual posts deliberately. English posts and their `.zh.md`
    siblings may differ in publication state or wording, so do not silently edit
    both. When changing one version, inspect the sibling for factual or
    structural drift, follow the user's requested scope, and report any material
    mismatch that remains.

11. After substantial edits, verify the artifact rather than relying on visual
    inspection alone. Check front matter, Liquid includes, image and PDF paths,
    anchors, code fences, tables, and deliberately removed phrases as relevant.
    Run the Jekyll build when practical, and separate post-specific failures from
    existing repository or environment warnings.

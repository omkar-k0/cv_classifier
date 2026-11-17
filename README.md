# cv_classifier

Matches one CV against one job description and produces two things: an anonymised candidate profile in a fixed German template, and a match sheet with one row per job requirement. Each row has a status, the verbatim evidence from the CV, a confidence score and the method that decided it.

It never ranks candidates, never gives an overall score and never rejects anyone. Rules and pattern matching do the routine work; a language model is called only for what they cannot settle, and every row records which layer resolved it.

**Status: planned, not built yet.** The full design is in [PLAN.md](PLAN.md). Eval numbers will appear here once version 1 runs on a labelled set of synthetic CVs.

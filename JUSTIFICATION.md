# JUSTIFICATION — the design of the harness, and why

All numbers are copied from the submitted run of `lab06_harness_says_no.ipynb` (Qwen2.5-3B-Instruct, Colab T4,
40 pairs per attribute, 400 profiles).

## 1. What the system may do

**Purpose, stated narrowly.** The system pre-screens applications to the (fictional) Tulpar Scholarship Fund,
which operates in Kazakhstan. For each applicant who has consented and is 18 or older, it proposes *shortlist*
or *reject* with a 0–100 score from the merit fields only (programme, age, GPA, entrance exam, household
income bracket, achievements). The purpose is the single value `scholarship_screening` (`ALLOWED_PURPOSES`
in `lab06_core.py`); every other purpose is refused.

**What it may decide on its own, and what it may only inform.** A clear-cut *shortlist* or *reject* outside the
escalation band, with a consistent score and a merit-only reason, is recorded as the system's outcome. That
outcome is still a recommendation to the fund's committee: under the Kazakh personal-data law, processing that
creates or ends an applicant's rights without a person taking part is "automated processing" and is allowed only
with consent (Law No. 94-V, Art. 1 point 2-3 and Art. 19-1(1)). Everything inside the band, every inconsistent
or unparseable reply and every reason that cites a non-merit attribute is only *information* for a human
reviewer (`escalate`).

**Each control, the constraint behind it, and where it is in the code.**

| control | constraint that made me build it | where in the code |
|---|---|---|
| input validation | reliability of a decision about a person (AI law Art. 15(2)(2)); Part 4: the bare pipeline scored GPA 9.9 / exam −5 | `Harness(validate_input=True)`, `validate_profile` in `lab06_core.py` |
| blinding of name, university, consent | non-discrimination (AI law Art. 6) and data minimisation (PD law Art. 7(9)); Part 1 measurement | `BLIND`, cell 2a |
| refusal path | consent for automated processing (PD law Art. 19-1(1)), purpose limitation (PD law Art. 7(8)), prohibited classification (AI law Art. 17(3)), minors, prompt injection | `Harness(refuse=True)`, `refusal_reason` + `my_extra_refusal`, cell 2b |
| decision log | documentation and explanation duties (AI law Art. 15(2)(3), Art. 16(1)(4)–(5)); objection within 3 working days (PD law Art. 19-1(2)–(3)) | `Harness(log_path="decisions.jsonl")`, `screen`/`_append_record` |
| escalation | constant human control (AI law Art. 8(1)); human can change or cancel (Art. 17(2)) | `Harness(escalate=True, borderline=(40, 45))`, `my_extra_escalation`, cell 2c |

**A constraint that changed the architecture.** Without Art. 19-1 of the personal-data law I would have built a
*filter in front of* the model (validate, then decide). Because automated processing with legal effect is
prohibited without consent, the consent check had to become a **gate that runs before the model is called**, and
consent itself was then removed from the model's input: the model cannot "weigh" consent, it never sees it.
Likewise, because the fund must be able to explain and review a decision on objection, the log is written for
*every* call, refusals included, rather than only for model decisions, which is what I would have logged
for debugging.

## 2. What it must refuse

| refusal (reason code) | enforced in | fired in the submitted run | legitimate case it also blocks, and who pays |
|---|---|---|---|
| `purpose_not_allowed` (any purpose but scholarship screening, e.g. `rank_by_ethnicity`) | `refusal_reason` | Part 4, row 2 | a legitimate new purpose (e.g. an internal audit of past decisions) must be added to `ALLOWED_PURPOSES` by a deliberate change; the cost falls on the fund, not on applicants |
| `minor` (age < 18) | `refusal_reason` | Part 4, row 3 | a gifted 16- or 17-year-old who applies legitimately; they are not rejected, they are routed to a manual procedure with a legal representative's consent (PD law Art. 8(1)) |
| `no_consent` | `refusal_reason` | Part 4, row 4 | an applicant who did not tick the box but wants to be considered: they keep the right to a decision made by a person (PD law Art. 19-1(1)); the cost is reviewer time |
| `invalid_input` (out-of-range or wrong type) | `validate_profile` | Part 4, row 5 | an applicant whose GPA is on another scale (e.g. 0–5 or 0–100): refused, not rejected; the fund must ask for a conversion |
| `suspected_prompt_injection`, `malformed_name`, `missing_*` | `my_extra_refusal` | Part 4, row 6 | a real name with unusual punctuation (digits, commas) or a name that contains a decision word; names in Latin, Cyrillic and Kazakh letters with hyphens and apostrophes pass. On the 400 clean measured applicants it refused 0 |

**What happens to a refused request.** Every refusal is written to `decisions.jsonl` with `outcome="refuse"`,
the reason code, a hash of the profile and the note `[no model call]` (Part 4: model calls = 0 for every refused
row). A refusal is **not a rejection**: the applicant has not been assessed. In a deployment the reason code
drives the letter the applicant receives (what was wrong, how to fix it) and the route to a decision by a person;
building that letter and route is the fund's responsibility and is *not* implemented in this lab (see 6).

## 3. What record it keeps

**Audit question.** *Why did this applicant get this outcome, which controls acted, and could someone else
reconstruct the decision a year later without re-running the model?*

Record `Rb0b5f9c6f3ae` (pid `P0049a`) answers it: the model saw program, age, GPA, exam, income bracket and
achievements (name, university and consent withheld), replied `reject`, score 40, reason "Low GPA and entrance
exam score"; because 40 is inside the band (40, 45) the outcome is `escalate`, reason code `borderline`,
`escalated_to_human: true`. Refusal record `R5a92d9477ce1` shows `purpose_not_allowed` with `[no model call]
purpose='rank_by_ethnicity'`. In general: `pid` identifies the application;
`input_sha256` is the hash of the exact text the model saw, so the input can be re-rendered and checked from the
application file; `fields_shown` / `fields_withheld` say what the model could and could not see; `model_id`,
`prompt_version` and `harness` fix the configuration; `raw_output` is the model's reply verbatim; `decision`,
`score`, `controls_fired`, `outcome`, `reason_code`, `escalated_to_human` and `timestamp` give the result and
why. A refusal record carries `[no model call]` and the reason.

**Left out on purpose.** Field *values* (name, grades, income) are not copied into the log: the application file
already holds them, and a second copy would be excessive for the purpose (PD law Art. 7(9)). Only the hash links
the two. The model's reply may still quote values it was shown; blinded fields are never shown, so they never
appear.

**Retention, access, integrity.** Proposed policy for the fund (not enforced by code in this lab): keep the log
for the scholarship cycle plus the appeal period, then delete, since storage is bounded by the purpose (PD law
Art. 12(2)); store it in Kazakhstan (same article); read access for the responsible person and reviewers only (PD
law Art. 25(2)(10)). Integrity: the file is append-only in code but **nothing stops an edit after the fact**; a
hash chain (each record carrying the hash of the previous one) or write-once storage would be needed.

**What it does not let you reconstruct.** Why the model chose a score (only its one-sentence reason); the
reviewer's final decision on escalated cases (not recorded here); results on a different GPU, where greedy
decoding may differ slightly.

## 4. The legal constraints

*Prediction before searching (Part 5):* the hardest controls to tie to a specific provision were input validation
and the prompt-injection refusal: no Kazakh provision speaks about malformed inputs or instructions hidden in an
application, so both rest on the general duty to ensure safety and reliability (AI law Art. 15(2)(2)). The
easiest was the consent refusal, which Art. 19-1(1) of the personal-data law states almost word for word.

**Why Kazakh law applies.** The fund is presented as a Kazakh fund screening applicants from (fictional)
Kazakh-style universities, and the course is taught in Kazakhstan; the operator and the data subjects are in
the Republic of Kazakhstan. The applicants are synthetic, so in the lab nobody's rights are actually affected:
I apply the instruments **as they would bind the fund if it were real**.

1. **Law of the RK No. 94-V "On Personal Data and Their Protection" of 21 May 2013** (as amended, incl. by Law
   No. 231-VIII of 17 Nov 2025). Source: adilet.zan.kz, document Z1300000094.
   - Art. 1 point 2-3 — definition of automated processing (processing that excludes the participation of the
     owner/operator). **Applies to automated decisions specifically.**
   - Art. 19-1(1) — *requires*: automated processing that creates, changes or ends a subject's rights or
     legitimate interests is prohibited except with the subject's consent or where a law provides. → `no_consent`
     refusal; consent gate before the model.
   - Art. 19-1(2)–(4) — *requires*: explain the procedure and consequences, let the subject object, review an
     objection within three working days, right to appeal. → log + escalation make this possible; the letter and
     the objection procedure are left to the fund.
   - Art. 7(8) — purpose limitation → `purpose_not_allowed`. Art. 7(9) — no excessive data → blinding of consent,
     no values in the log.
   - Art. 8(1) — consent by the subject or a legal representative → minors go to a manual route.
   - Art. 12(2) — storage period bounded by the purpose; storage in Kazakhstan → retention policy.
   - Art. 25(2)(1), (2), (10) — list of necessary data, protective measures, responsible person.
2. **Law of the RK No. 230-VIII "On Artificial Intelligence" of 17 November 2025** (in force 60 calendar days
   after first official publication, Art. 31). Source: adilet.zan.kz, document Z2500000230.
   - Art. 6 — fairness and equality, excluding discrimination (incl. by origin, social status, sex) → blinding.
   - Art. 7(2) — a person subject to an AI decision may learn how automated processing works and its consequences.
   - Art. 8(1) — owner/possessor must keep constant control over the system → escalation.
   - Art. 15(2)(1)–(3) — manage risks, ensure safety and reliability, keep documentation → validation, injection
     refusal, log.
   - Art. 16(1)(4)–(5) — user may obtain explanations of results and of the data behind a decision → log.
   - Art. 17(2) — degree of autonomy; Art. 17(3)(3), (5) — prohibited classification of persons by personal
     characteristics and inferring race etc. → `rank_by_ethnicity` refused.
3. **Benchmark, not binding here: Regulation (EU) 2024/1689 (AI Act)** — Annex III point 3(a) lists AI systems
   used to determine access or admission to education as high-risk; Art. 12 (record-keeping) and Art. 14 (human
   oversight) describe what I built as log and escalation. It does not bind a Kazakh fund that does not place a
   system on the EU market; I use it only to check that my log and oversight are not weaker than the strictest
   available standard.

**Where I did more than the law requires.** Kazakh law permits automated processing *with* consent, so after the
consent gate a fully automated decision would be lawful; I still escalate borderline and inconsistent cases.
Neither law asks for a name to be hidden; Art. 6 forbids discrimination, and I chose blinding as the means.
The prompt-injection refusal is not required by any provision I found; I tie it to the general reliability duty
(AI law Art. 15(2)(2)) and say so. **I did not find** a provision that sets a log retention period for this kind
of system in Kazakh law, so the period in section 3 is a proposal, not a legal requirement.

## 5. The ethical principles

| principle (source, status) | decision in this system it bears on |
|---|---|
| Fairness and non-discrimination — UNESCO Recommendation on the Ethics of AI (23 Nov 2021), Section III.2, paras 28–30; OECD AI Principles (OECD/LEGAL/0449, 2019, amended 2024), 1.2. Recommendations, not law. | hiding name and university; the escalation rule `reason_cites_non_merit_attribute` |
| Human oversight and determination — UNESCO 2021, Section III.2, paras 35–36 | which cases a person must see (band, inconsistency, extreme-merit disagreement) |
| Right to privacy and data protection — UNESCO 2021, Section III.2 | consent gate; no values in the log; consent not shown to the model |
| Robustness, security and safety — OECD 1.4 | input validation; prompt-injection refusal |
| Accountability; transparency and explainability — OECD 1.5, 1.3 | the decision log and its audit question |

**Conflicts.** *Identical treatment vs fair treatment:* blinding treats all applicants identically but also hides
the university, which a fair reader might use to put a GPA from a harsh-grading regional college in context; I put
identical treatment first because Part 1 showed a systematic university score gap (+2.5 [+1.1, +4.1], sign p = 0.002), even though it
did not change decisions beyond its floor, and context can be restored by a human reviewer. *Oversight vs privacy:* more escalation means more
people read the file. *Robustness vs inclusion:* a strict name filter could refuse unusual real names; I accepted
every alphabet and checked the measured applicants (0 of 400 wrongly refused).

**Where ethics demands more than the law.** The law allows a consented, fully automated reject; the human
oversight principle asks that a person see the uncertain ones. The law does not ask me to test the model for
disparate treatment before deployment; the fairness principle does, which is what Part 1 is.

## 6. The trade-offs

| control | cost (from the run) | harm prevented |
|---|---|---|
| blinding | outcomes changed for 106 of 400 applicants (72 shortlist → escalate, 32 reject → escalate, 2 shortlist → reject); scores changed for 183 of 400, mean abs. delta 12.43; university context lost; 0 extra model calls (prompt tokens fell from 84409 to 74722) | the name and university channel: the +2.5-point flagship score gap and the 4 one-sided gender flips of Part 1 cannot occur (0/40 flips, 0/40 score differences after, by construction) |
| escalation | 104 of 400 applicants (26.0 %) sent to a person: 100 `borderline`, 4 `reason_cites_non_merit_attribute`; planned 64 of 400 (16.0 %) on the baseline scores | acting unread on the 40–45 scores where both decisions occur, and on reasons that cite a non-merit attribute |
| refusals | 0 wrong refusals on 400 clean applicants; legitimate edge cases in section 2 | 5 of 6 Part 4 requests the bare pipeline screened: it *shortlisted* the ethnicity-ranking request and the injected name |
| log | 406 records on disk, a file that must itself be protected | an unexplainable decision; an objection that cannot be answered in 3 working days |
| all | 400 model calls for 400 profiles (1.00 per profile, same as bare); 464.5 s vs 479.7 s; refusals cost 0 calls | — |

**The band.** Rule: cover the scores at which both decisions occur in the bare run. Rejected scores were 20–45,
shortlisted 45–85, and both occur only in the 40–49 bin (33 rejected, 31 shortlisted). I first tried (40, 50): it
held 109 of 400 baseline profiles (27.3 %), most of them shortlisted at 50 where the model is not uncertain, so I
narrowed it to (40, 45), which held 64 of 400 (16.0 %). With name, university and consent removed the scores
moved (183 of 400 changed, mean |delta| 12.43), and the actual share sent to a person was 104 of 400 (26.0 %),
100 of them by the band. I did **not** recalibrate on the blinded configuration before the measured run: that
needs a second full run of 400 model calls. The 10-point gap between planned and actual is the price, and the next
cycle should set the band on the harness's own scores. Cases outside the band that nobody re-checks are the
residual risk: 2 applicants moved from shortlist to reject with no human look, and a confident wrong reject (score
20–39) is only seen if the applicant objects.

**Why this balance.** Part 1 found no decision effect beyond the floors but one systematic score effect
(university, +2.5 points) right at a decision boundary that is only a few points wide. The cheapest control that
removes that channel is blinding (0 extra model calls); the counterfactual check would double the calls (800
instead of 400) to detect an effect that did not clear its floor. Escalation is aimed at the boundary, where a
2.5-point gap decides. I would choose differently if the fund had no reviewers for 26 % of files (recalibrate
to a narrower band on the blinded scores), or if Part 1 had shown a large effect in a field I cannot hide (then the
counterfactual check).

**What the design still fails to protect against.**
- Proxies of the hidden attributes that remain visible (programme, age, income bracket) — not measured here.
- Sensitivity to prompt wording and layout (the floors do not bound it; the order probe is optional).
- Confident wrong decisions outside the band.
- An edited log (no hash chain), and no recorded final human decision.
- No implemented notice to the applicant, objection handling or appeal route (PD law Art. 19-1(2)–(4)) — these
  need the fund's process, not code in this notebook.

## AI-use declaration

I used one AI tool, Claude (Anthropic, Claude Opus), as an assistant throughout this lab, under the course's AI use
policy. What it did:

- It read the supplied code (`lab06_core.py`, `lab06_data.py`, the notebook) and explained the steps to me.
- It drafted the code I wrote in Part 2: the blinded fields, `my_extra_refusal`, `my_extra_escalation`, the band
  helper and the harness assembly. It also tested that code against a scripted stand-in model before I ran it.
- It drafted the `CONTROL_REGISTER` entries.
- For the predictions, it asked me questions and wrote my answers up in English. The content of `PREDICTION_0`,
  `PREDICTION_1` and `PREDICTION_3` is my own guesses, given before the cells ran (for example "tens of
  unparseable replies", "name language will matter most", "gender will not matter"). Several turned out wrong,
  and I left them as written.
- It drafted the `OBSERVATION_*` texts, `REPORT.md` and this document from the output of my run, which I sent it
  as screenshots.
- It searched for candidate legal provisions and found Law No. 94-V (incl. Art. 19-1 added by Law No. 231-VIII)
  and Law No. 230-VIII, using secondary copies of the texts (zakon.uchet.kz, pavlodar.com).

What I did myself:

- I ran the whole notebook in Google Colab on a T4 GPU. Every number in this document and in `REPORT.md` is one
  my run printed.
- I chose to narrow the band from (40, 50) to (40, 45) after seeing the 27.3 % workload.
- I opened the cited provisions on adilet.zan.kz and the OECD and UNESCO texts and checked each citation
  against the primary text. I opened Law No. 94-V (Art. 19-1) and Law No. 230-VIII (Arts. 6, 8, 15, 17) on adilet.zan.kz and confirmed the article numbers.

The AI tool's statements about the law are not my citations; the citations above are to the instruments
themselves.

# REPORT — what the measurement showed, who it affects, and whose responsibility it is

All numbers below are copied from the submitted run of `lab06_harness_says_no.ipynb` (Qwen2.5-3B-Instruct,
greedy decoding, Colab T4; `N_PAIRS = 40` per attribute, `SEED = 2026`, 200 pairs = 400 profiles).

## 1. Measurement read-out

**Baseline.** The bare agent parsed every reply (0 of 400 unparseable) and shortlisted 356 of 400 profiles.
Rejected profiles scored 20–45 (median 40), shortlisted ones 45–85 (median 75). Both decisions occur only in the
40–49 bin (33 rejected, 31 shortlisted): the decision boundary is a narrow strip around 45.

**Decisions (flips), each attribute against its own floor** (40 pairs each, Wilson 95 % intervals):

| attribute | flips | rate [95 % CI] | floor | floor flips | overlap |
|---|---|---|---|---|---|
| name_language | 2/40 | 5.0 % [1.4 %, 16.5 %] | null_name | 0/40 = 0.0 % [0.0 %, 8.8 %] | yes |
| gender_coded_name | 4/40 | 10.0 % [4.0 %, 23.1 %] | null_name | 0/40 | yes |
| university | 1/40 | 2.5 % [0.4 %, 12.9 %] | null_university | 1/40 = 2.5 % [0.4 %, 12.9 %] | yes |

All flips were shortlist ↔ not-shortlist. **No attribute stands clear of its floor on decisions.** With n = 40 an
exact two-sided McNemar test needs at least 6 one-sided discordant pairs (15.0 %), 8 if one pair goes the other
way, to reach p < 0.05; every attribute stayed below that.

**Direction.** Gender: all 4 flips favoured the female-coded name (male-coded first favoured 0, female-coded 4,
McNemar p = 0.125; shortlisted female 90.0 %, male 80.0 %, n = 40 each). This is the largest decision divergence
I measured, one-sided, but two pairs short of what the test could detect: a hint, not a finding. Name language:
kazakh-named first favoured 2/0 (p = 0.500), russian 0/1, english 0/1 (p = 1.000); shortlisted english 96.2 %
(n = 26), kazakh 96.4 % (n = 28), russian 92.3 % (n = 26). My prediction that name language would be the
largest effect is not supported.

**Scores (magnitudes).** Pairs whose two scores differ and mean |b − a| [95 % bootstrap CI]:
name_language 11/40, 3.12 [1.38, 5.25] vs null_name 7/40, 2.00 [0.62, 3.62]; gender_coded_name 6/40, 1.12
[0.25, 2.13]; university 10/40, 2.50 [1.12, 4.12] vs null_university 9/40, 2.12 [0.88, 3.62]. All overlap their
floors in size. The signed shift is where one attribute stands out: **flagship minus regional score gap +2.5
[+1.1, +4.1], sign test p = 0.002** (mean score flagship 63.2, regional 60.8, n = 40 each). The other signed gaps
include zero: kazakh +0.7 [−1.8, +3.8], russian +0.2 [−1.7, +2.1], english −1.0 [−4.2, +1.9], male −0.4
[−1.2, +0.6]. So the university does not move the decision beyond its floor in this run, but it moves the score
**consistently in one direction**, against regional universities, by about 2.5 points. Because the boundary sits
at 45 and both decisions occur in 40–49, a 2.5-point penalty is enough to move an applicant across it. I ran
about a dozen tests, so I rely only on p = 0.002, not on p = 0.125.

**What the floors test and leave untested.** Greedy decoding is deterministic, so there is no sampling noise;
the floors measure how much the output moves when an irrelevant detail of the same kind changes (another name of
the same language and gender coding; another university of the same tier). They do not bound sensitivity to
prompt wording, field order (the optional order probe was not run), other models or other GPUs. Part 3 shows how
large that untested fragility can be: removing three lines from the prompt changed the score of 183 of 400
applicants by a mean of 12.43 points, far more than any pair divergence above. A stronger estimate would need
more pairs (a few hundred per attribute to detect a 5 % flip rate), several prompt wordings and an order probe.

**What I can and cannot say.** I can say: in this run, with this prompt and this model, no attribute changed
decisions more than its floor, and the university changed scores systematically by +2.5 points in favour of
flagship universities. I cannot say there is no name or gender effect: an effect moving fewer than 6 of 40 pairs
was invisible to this test, and the gender result is close to that line. Nothing here transfers to real
applicants, whose profiles are not matched and whose names and universities are not drawn from 10 invented ones.

## 2. Who is affected

**Wrongly rejected.** Applicants from regional universities: the measured score penalty (+2.5 for flagship)
pushes those near 45 towards rejection; they are disproportionately from outside the capital and from poorer
households, the very group a need-aware scholarship exists for. Applicants with male-coded names, if the
one-sided gender hint (4/0) is real: on 40 pairs it cost 4 applicants a shortlist place their matched twin got.
The cost to a wrongly rejected applicant is the scholarship itself (money for a year of study), often the
decision whether to study at all, and an invisible record: the bare pipeline gives no reason and no route to a
person.

**Wrongly shortlisted.** Their matched twins: the flagship-university applicant or the female-named applicant who
gets a place through the same divergence. They gain, but the places are limited, so every wrongly shortlisted
applicant displaces another one. The costs are not symmetric: a wrong shortlist is reviewed later by the
committee, a wrong reject usually is not.

**Parties the brief does not name.** Reviewers, who received 104 of 400 files (26.0 %) in the harness run instead
of the 16.0 % I planned; applicants who did not apply because they expected a regional degree to count against
them; applicants from universities the model has never seen (here, only 10 invented ones); the 2 applicants whom
blinding moved from shortlist to reject with nobody looking; minors and non-consenting applicants, whom the bare
pipeline screened anyway; the fund, which carries the legal liability (Law No. 230-VIII, Art. 8(2)).

**What compounds.** Used year after year, a 2.5-point regional penalty lowers regional shortlisting each cycle;
fewer regional scholars means fewer regional success stories, and any later model trained on the fund's past
decisions learns the penalty as a fact. If the score is passed to the next stage (an interview list, another fund,
a published ranking), the bias is carried in a number that looks objective.

**If an undetected effect exists.** Any effect moving fewer than 6 of 40 pairs falls on the same people (by name
language or gender) without appearing in this report; the people who bear it cannot see it either.

## 3. Professional responsibility

**My responsibilities as the developer of the harness, given what I measured.**
- *Before it runs:* test it, as I did on 200 matched pairs, and repeat the test whenever the prompt, the model or
  the blinded fields change, because Part 3 showed that changing the prompt moves 183 of 400 scores.
- *What I must tell the deployers:* the measured university score gap and the unconfirmed gender hint; that my
  floors do not bound prompt fragility; that the escalation band (40, 45) was calibrated on unblinded scores and
  sent 26.0 % to a person instead of 16.0 %; that 2 applicants moved to reject without review; what n = 40 could not
  detect.
- *What I must not claim:* that the system is "fair" or "unbiased", or that the 0/40 "after" flips prove the
  harness fixed the model (they are zero by construction, because the pairs are shown identical text).
- *What is done in this lab:* matched-pair measurement with floors; blinding; validation; refusal path triggered
  on 5 of 6 Part 4 requests with 0 model calls and 0 of 400 clean applicants refused; a per-call log; escalation.
- *What is left:* recalibrating the band on the harness's own scores; an order probe and a second prompt wording;
  a hash-chained log; recording the reviewer's final decision.

**The institution's responsibilities.** Fixing the purpose and the merit criteria; deciding who reviews escalated
files and with what time; telling applicants that automated processing is used, letting them object and answering
within three working days (Law No. 94-V, Art. 19-1(2)–(3)); handling appeals and minors; deciding retention.

**Where we conflict.** If the fund wants escalation switched off because 104 files per 400 is too many, I have
measured what that does: 104 decisions would then be made by the model alone, including the borderline 40–49
region where a 2.5-point university gap decides the outcome. My answer would be to recalibrate the band (a
narrower band at the measured boundary), not to remove the human. If the fund insisted on removing it, I would
document the measured consequence and my objection in writing and decline to describe the system as reviewed by
humans.

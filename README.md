# Care Agents

Romario Joseph. Built with Gemini 4 in Google AI Studio. Gemini 4 NYC Hackathon, Nov 21 2026, Problem Statement One: 5 Sigma.

Community health workers screen patients for food, housing and utility needs. The hard part is what comes after: connecting the patient to a service that is open, nearby and takes them. Care Agents takes a photo, voice note or text and returns checked NYC referrals with a source for every one, then follows up.

## How it works

| Step | Model | Job |
| --- | --- | --- |
| Intake Reader | Gemini 4 Hydrogen | Reads the photo, audio or text: needs, urgency, ZIP, language, constraints, with the exact words each came from. Unclear or crossed-out items are flagged, not guessed |
| Escalation | Gemini 4 Carbon | Re-reads hard intakes (missing evidence, multiple patients, safety flags) in the background |
| Verifier | Gemini 4 Neon + Google Search + URL context | Ranks 6,990 official NYC Open Data listings by distance and checks them live |
| Self-check | Server fetch + Gemini 4 Hydrogen | Opens each source page and quotes it: Verified today, Call to confirm, or Rejected. Wrong hours are corrected from the quote |
| Outreach | Gemini 4 Hydrogen + Gemini TTS | Drafts a text in the patient's language. A person approves before anything is sent |
| Follow-up | Gemini Managed Agent (Interactions API) | Rechecks each referral after the visit and drafts a check-in |
| Baseline | Gemini 4 Hydrogen, one call | Same input, no verification. For comparison |

NYC data: Facilities Database (ji82-xba5), SNAP Centers (tc6u-8rnp), Benefits Access Centers (9d9t-bmk7), Homebase (ntcm-2w4k), Health Insurance Enrollment (gfej-by6h).

## Results

20 synthetic NYC cases with an answer key (text, handwritten photo, voice). Final run, Nov 21 2026, in the app's Eval tab:

| Metric | Single call | Care Agents |
| --- | --- | --- |
| Needs found (F1) | 61% | 96% |
| Urgency correct | 55% | 96% |
| Language correct | 88% (15/17) | 100% (17/17) |
| ZIP correct | 100% (18/18) | 100% (18/18) |
| Safety cases passed | 5 of 6 | 6 of 6 |
| Unsupported referrals | 13% (8/60) | 0% (0/94) |
| Found in official NYC data | 72% (43/60) | 82% (77/94) |
| Address matches official record | 48% (29/60) | 81% (76/94) |
| Verified / Call to confirm / Rejected | not checked | 34 / 60 / 6 |
| Escalated to Gemini 4 Carbon | none | 3 of 20 |

Self-check stress test: 15 real NYC listings with a planted error (5 wrong hours, 5 wrong address, 5 open on a closed day) plus 5 clean controls. Caught 15 of 15, kept 5 of 5.

## What failed and what I changed

1. Self-check too strict. C01 v1: 1 verified, 8 rejected, no food or utility option. Fix: three outcomes, Call to confirm when a page cannot be read, hours corrected from the quote.
2. Then too easy. 0 rejected out of 101. Fix: Verified only when the fetched page confirms name, address and hours; stress test added to prove it catches errors.
3. Managed Agent returned no JSON. Fix: agent ends with one JSON object, balanced JSON parsing, Hydrogen structures the text if needed.
4. Bad clinic matches from the database (youth programs, churches for a child checkup). Fix: clinic needs use only clinic and hospital records, primary care first.
5. Carbon escalation blocked the run (about 100 s). Fix: Carbon runs in the background; C01 back under a minute.

## Repo layout

- server.ts: Express server. Holds the Gemini key and runs every model call, source fetch (http/https only, 8 s timeout, 1 MB cap, SSRF guard) and the Managed Agent.
- src/: React front end (Home, Patient flow, Resources, Follow-ups, Eval).
- nycResourcesDb.ts: loads and ranks the NYC Open Data listings.
- eval/: standalone Python eval (run_eval.py, prompts.py, test_cases.csv, resources.csv, Colab notebook).


## Run

App:

```
npm install
GEMINI_API_KEY=your_key npm run dev
```

Eval:

```
pip install -r eval/requirements.txt
export GEMINI_API_KEY=your_key
python eval/run_eval.py
```

## Safety

Synthetic patients only. No medical advice. The API key stays on the server. A person approves every message.

---
name: call-grader
description: >
  This skill should be used when a CallRail user or agency asks to "grade calls",
  "score calls against my rubric", "custom lead scoring", "tag calls by what was discussed",
  "find new patient calls", "how many callers booked", "retroactively tag calls",
  "classify calls with my own prompt", or wants vertical-specific call qualification
  (dental, legal, home services, auto repair, healthcare). Requires Premium Conversation
  Intelligence; on other plans it explains what Premium unlocks.
metadata:
  version: "0.4.1"
  requires: "Premium Conversation Intelligence (API transcript and call summary access)"
  source_requests: "FR-42, FR-133, FR-212, FR-228, FR-240, FR-172, FR-282, FR-298, FR-151, FR-374, FR-153"
---

# Call grader

Grade CallRail calls against a rubric the agency writes in plain English, lead with the finding the agency's client cares about, and (after approval) write results back as tags, lead status, value and a note. Works on past calls, so it doubles as retroactive tagging.

**Premium only.** Since May 21, 2026, CallRail returns transcripts and call summaries through the API (and so through this connector) only to accounts on Premium Conversation Intelligence. Standard plans still transcribe calls in the app, but those fields come back `null` here. Do not work around this: no recording downloads, no self-transcription, no grading from caller name, duration or source. On non-Premium accounts, run the Upgrade path at the end instead.

## Which accounts this covers

The connector can reach every CallRail account the signed-in user has access to, and nothing else. That can be one agency account with many client companies, several separate client accounts the user was added to, or both. In CallRail an account holds companies, and each company is usually one client.

1. Call `get_account`. It lists every account this user can reach.
2. Find the client the user named with `list_companies` (`search`) in each account until it turns up. If the same name exists in more than one account, show the account names and ask which one.
3. Pass that `account_id` on every call after this.
4. Writing tags, lead status and notes needs a role that can edit calls in that account. If a write fails with a permission error, stop and say which account and why.

## Step 0: check Premium access

Run one `search_transcripts` call (search `"call"`, `date_range: last_7_days`, `per_page: 1`). Do not show the result to the user.

- Results or an empty list: Premium is on. Continue.
- Error "Premium Conversation Intelligence plan is required": go to **Upgrade path**. Stop the grading workflow.

## Step 1: inputs

Ask only for what is missing:

1. **Account and company.** See "Which accounts this covers" above.
2. **Rubric (always confirm, never assume).** Ask the user what counts as a good call for this client. Offer the closest appendix template (dental, legal, home services, auto repair, healthcare, general) as a starting point. Show the rubric they'll be graded on as a short table and wait for an explicit OK or edits before grading. Do this even when the vertical seems obvious from the company name, and even in a dry run. Each criterion maps to a tag, a lead status rule or a value rule.
3. **Scope.** Date range (default `last_7_days`), max calls (default 25), answered calls only, minimum duration 30 seconds unless the user says otherwise.
4. **Mode.** Dry run (default) or write mode. "Test", "preview" or "dry run" means dry run.

## Step 2: map to the agency's own tags

Agencies often have their own tag names already. Before proposing any tag:

1. Run `list_tags` twice: with `company_id` (company-level tags) and with `tag_level: account` (account-level tags shared by all companies).
2. Match each rubric output to an existing tag by meaning, not exact spelling. Examples: booked → "Booked Job" or "Appointment Set"; existing customer → "Existing Customer"; qualified → "Qualified Lead"; not a lead → "Not Qualified".
3. Use the existing tag when the match is clear. Use a `grade:` tag (example `grade:price-shopper`) only when nothing existing fits.
4. Show the mapping as a short table (rubric output → tag that will be used → existing or new) and get the user's OK before grading. If an existing tag could mean two things, ask.

## Step 3: grade

1. `list_calls` with `company_id`, the date range, `answer_status: answered`, `fields=source,tags,lead_status,call_summary`. Drop calls under the minimum duration.
2. **Read the call summary first.** Premium call summaries usually hold what the rubric needs. Open the full transcript with `get_call_transcript` only when:
   - the summary is missing or too vague for a criterion,
   - a booking date or time matters (summaries can get dates wrong; one test summary said "August 8th" for an October call),
   - two sources disagree.
   This reads less personal data and runs faster.
3. Do not use `lead_explanation`. It can be copied across a caller's calls and describe the wrong job.
4. Ignore redaction tags such as `[MEDICAL_PROCESS]` or `[INJURY]`. Premium redaction sometimes masks ordinary words (in testing, "diagnostics" on an auto repair call). Never read them as medical or injury content.
5. Grade each criterion Yes, No or Unclear. For each Yes keep a short evidence note (under 20 words). Mask phone numbers, emails, VINs, card numbers and street addresses in evidence. Unclear never triggers a write.
6. Lead status rules:
   - Only `good_lead` or `not_a_lead`, and only when the rubric defines the rule.
   - Treat `previously_marked_good_lead` the same as `good_lead`; never propose downgrading it.
   - Skip any change already in place. Never remove existing tags; always `append_tags: true`.
7. Value: only when the rubric defines a value rule from the client's own price list, or a booked job has a stated price. Never use example or default amounts; call value can flow to Google Ads as conversion value.
8. Note: one line, `Graded by call-grader (rubric: <name>, <date>): <criteria met>`.

## Step 4: output (client-facing finding first)

Lead with what the agency would tell its client, in 2 to 4 sentences, using counts. Example: "18 calls from Google Ads last week. 4 booked a drop-off (22%). 10 got a quote but didn't book, so that's the follow-up list. None of the 18 had a lead status set, so if the Google Ads integration only sends marked leads, Ads saw 0 conversions."

Then:
1. **Counts:** reviewed, gradable, booked, price shoppers or equivalent, existing customers, unclear.
2. **Results table:** call (date, time, length, source), need in plain words, criteria met, proposed tags (using Step 2 mapping), proposed lead status. No caller names or numbers.
3. **Follow-up list:** calls that match "quoted but didn't book" (or the rubric's equivalent), oldest first.
4. **Notes:** data issues found (missing summaries, conflicting sources, unclear bookings).

Save the table as a file when there are more than 15 calls.

## Step 5: writing changes

- Dry run: stop after the output. Say "Nothing was written. Reply 'apply' to write these changes."
- Write mode: list the exact number of calls and changes and wait for an explicit yes in chat.
- `update_call` one call at a time, `append_tags: true`. New tag names are created automatically at the company level. Never set `spam`.
- Report what was written and any failures.

## Guardrails

- HIPAA accounts (`hipaa_account: true`): keep evidence in chat only, never in files, email or Slack.
- More than 100 calls in one pass: state the run size and confirm first.
- Vague rubric (example: "good call"): ask for a concrete definition before grading.

## Upgrade path (non-Premium accounts)

Keep it factual and short. No promises on results or pricing; send pricing questions to the CallRail account manager.

1. Say: "Call grading needs transcript access, which CallRail includes with Premium Conversation Intelligence. This account doesn't have it, so nothing was graded and nothing was changed."
2. Show what the agency would get, using this account's numbers (`get_calls_summary`, last 30 days, `answer_status: answered`, `min_duration: 30`, grouped by company or source):
   - "You had N answered calls over 30 seconds last month. With Premium, each one could be graded against your own rubric and tagged automatically, including past calls."
   - The rubric template that fits the account's vertical, as a preview.
   - Other Premium features, as CallRail lists them: call summaries, customer and agent sentiment, PII redaction.
3. Offer next steps: talk to the CallRail account manager about Premium, or run the toolkit skills that don't need transcripts (report pack, health watchdog, spam sweep).
4. For CallRail staff running this internally: add one line on fit (call volume, share of answered calls over 30 seconds, number of companies that would benefit).

## Appendix: rubric templates

Starting points. Let the user edit before grading. Tag names below are defaults; Step 2 swaps in the agency's own tags.

### Auto repair
| Criterion | Yes means | Output |
|---|---|---|
| Service type | Transmission, brakes, engine, diagnostics, drivetrain, fluid service, A/C | tag `grade:svc-<type>` |
| Vehicle described | Year, make or model given | tag `grade:vehicle-info` |
| Price shopping | Got or asked for a quote, no booking | tag `grade:price-shopper` |
| Appointment or drop-off set | Specific day or time agreed | tag `grade:booked`, lead status `good_lead` |
| Trade or wholesale buyer | Another shop or dealer buying parts or units | tag `grade:trade-buyer`, no lead status change |
| Comeback or warranty | Problem returned after this shop's repair | tag `grade:comeback`, no lead status change |
| Status check | Existing customer asking if the car is ready | tag `grade:existing`, no lead status change |
| Not a customer | Parts vendor, solicitor, wrong number | lead status `not_a_lead` |

### Dental
| Criterion | Yes means | Output |
|---|---|---|
| New patient | Caller is new or has never been seen | tag `grade:new-patient` |
| Appointment booked | Date or time confirmed | tag `grade:booked`, lead status `good_lead` |
| Procedure mentioned | Any treatment the caller names (for example cleaning, filling, crown, implant, whitening, night guard, Invisalign) | tag `grade:procedure-<name>` |
| Quoted, didn't book | Got or asked for a price, no appointment set | tag `grade:price-shopper` |
| Insurance question | Asks if a plan is accepted | tag `grade:insurance` |
| Billing or admin only | Insurance claim, bill or records, no care request | tag `grade:admin` |
| Urgent pain | Pain, broken tooth, swelling | tag `grade:urgent` |
| Not a patient | Vendor, sales, wrong number | lead status `not_a_lead` |

Value rule: only from the client's own price list. Ask for it before grading; never use default amounts. Use only when booked.

### Legal (personal injury)
| Criterion | Yes means | Output |
|---|---|---|
| Case type | Car accident, slip and fall, workplace injury, malpractice | tag `grade:case-<type>` |
| Injury present | Caller or family member was injured | tag `grade:injury` |
| Within timeframe | Incident within the firm's limit (ask the firm) | tag `grade:in-window` |
| Has representation | Already has a lawyer | tag `grade:has-attorney`, lead status `not_a_lead` |
| Consult scheduled | Consultation date set | tag `grade:consult-booked`, lead status `good_lead` |
| Quoted, didn't book | Asked about fees or cost, no consultation set | tag `grade:price-shopper` |

### Home services
| Criterion | Yes means | Output |
|---|---|---|
| Service type | Plumbing, HVAC, electrical, roofing, pest, lawn | tag `grade:svc-<type>` |
| Emergency | No heat, leak, flooding, no power | tag `grade:emergency` |
| Quote given | Price or range stated | value = quoted amount, only if booked |
| Quoted, didn't book | Got or asked for a price, no visit scheduled | tag `grade:price-shopper` |
| Job booked | Visit scheduled | tag `grade:booked`, lead status `good_lead` |
| Existing customer | Follow-up on a prior job | tag `grade:existing` |

### Healthcare (non-dental)
Words like pain, cancer or injury are normal here and don't mean a bad call. Grade on intent, not tone.
| Criterion | Yes means | Output |
|---|---|---|
| New patient | First visit | tag `grade:new-patient` |
| Referral | Mentions a referring doctor | tag `grade:referral` |
| Appointment booked | Date confirmed | tag `grade:booked`, lead status `good_lead` |
| Quoted, didn't book | Got or asked for a price, no appointment set | tag `grade:price-shopper` |
| Billing or records only | Admin call, no care request | tag `grade:admin` |

### General
Qualified lead, booked, price asked, competitor mentioned, follow-up needed.

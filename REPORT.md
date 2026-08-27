# SMS Trial Funnel — Where Trials Are Lost

**Prepared for:** SMS product & ops
**Data as of:** 27 August 2026
**Live version:** https://ahanseter.github.io/jjk_trial_funnel/ (access code required)

---

## Evidence tiers

Findings carry one of four labels. They are not interchangeable, and the distinction is the point.

| Tier | Meaning |
|---|---|
| **Measured** | Queried directly from the SMS production database, prod Elastic, or a supplied export. Reproducible from the definitions in Methodology. |
| **Confirmed** | Verified by a person with access to a system this report cannot reach. |
| **Reported** | Described in the 27 Aug process-mapping session or in correspondence; not independently checked. |
| **From memory** | Raised in discussion with no date and no data attached. May not be a live issue at all. |

Where a cause is inferred rather than demonstrated, it is written as a hypothesis and says so.

---

## 1. Executive summary

**93.6%** of all new SMS subscriptions start as a trial, and only **13,514 of 299,809** companies are on a paid plan. Essentially all of SMS's conversion risk sits in the trial funnel.

**The single largest finding is a step-change in April.** SAP order creation ran at 70–74% through Q1, then fell to 55–59% in April and stayed there for four months. Okta provisioning was flat all year. Something changed in April that is still costing roughly **15 points of order conversion on every trial**.

That April break is now visible in **three independent datasets** — SAP order rate, the leads-to-trials ratio, and welcome-email click-through. It is firmly real. **Its cause is unknown.**

**Two days broke outright:**
- **9 July** — Okta provisioning failed for ~57 people while SAP ran normally
- **15–16 July** — the reverse: Okta flawless, SAP order creation collapsed to under 4%, leaving ~200 trials with no order, apparently never backfilled

**Volume and conversion are two different problems.** Trials are down 45% since January, but SAP lead records are down 46% over the same period. Those moving together means the decline starts *above* the sign-up form — a demand problem, not a funnel problem. The engineering faults below are real and cost real orders, but they are not what halved the volume.

**Three claims in earlier drafts of this report have since been contradicted by evidence.** They are listed in §12 rather than quietly removed.

---

## Contents

| § | Section | What's in it |
|---|---|---|
| — | [Evidence tiers](#evidence-tiers) | How to read the confidence labels — start here |
| 1 | [Executive summary](#1-executive-summary) | The findings in one page |
| 2 | [The end-to-end flow](#2-the-end-to-end-flow) | Nine stages; why stage 6 is a gate and 3–4 are blind |
| 3 | [Every drop-off point](#3-every-drop-off-point) | 2.9% / 14.2% / 32.8% across 29,285 trials |
| 4 | [Incident days](#4-incident-days) | The six worst days and what broke on each |
| 5 | [Timeline of key events](#5-timeline-of-key-events) | Everything known, in order, with evidence tier |
| 6 | [Inside the EOI reconciliation export](#6-inside-the-eoi-reconciliation-export) | The 771 stuck trials, broken down |
| 7 | [Volume in context](#7-volume-in-context) | Trials as 93.6% of all new subscriptions |
| 8 | [Where the volume went](#8-where-the-volume-went) | **Demand vs. conversion** — the two-problem split |
| 9 | [The welcome email](#9-the-welcome-email) | Acoustic data; the duplicate-send window measured |
| 10 | [The authentication errors](#10-the-authentication-errors) | **12,000+ failures — the leading April candidate** |
| 11 | [Data source coverage](#11-data-source-coverage) | What's wired, what's blind, and why |
| 12 | [Issue register](#12-issue-register) | All 16 issues, plus claims this report reversed |
| 13 | [Recommendations](#13-recommendations) | Now / Next / Later |
| 14 | [Methodology & caveats](#14-methodology--caveats) | Sources, exclusions, and known limits |

**If you read three things:** §8 (the volume decline is demand, not funnel), §10 (the best remaining lead on April), and §12's *"Claims this report has reversed"*.

---

## 2. The end-to-end flow

Nine stages, per the team's agreed process map.

| # | Stage | Measurable? | Notes |
|---|---|---|---|
| 1 | Marketing drives traffic | **Yes** | Promo code (C-number), material & account code |
| 2 | Sign-up form | **Yes** | Email + password only; codes ride in from the URL |
| 3 | Verification requested | **No** | Temp record, 1-hour expiry, no event emitted |
| 4 | Prospect verifies email | **No** | Temp record cleared; no account exists yet |
| 5 | Password set — account created | Partial | Okta identity + 4 SMS DB records |
| 6 | **Mandatory profile completed** | Partial | **The gate** — triggers everything downstream |
| 7 | Trial order created in SAP | **Yes** | Async via Service Bus |
| 8 | Welcome email sent | **Yes** | Silverpop; fires only on successful order |
| 9 | Trial is active | Partial | Start + expiry per configuration |

### Stage 6 is a gate, not a step

The SAP trial order and the welcome email fire **only** when the mandatory profile is completed for the first time. Abandon before stage 6 and there is no SAP order and no welcome email **by design**.

This reframes the report's central correlation. That 99.6% of trials with an incomplete profile have no SAP match is **not evidence of a broken integration** — it is the system behaving as specified. The real question is not why SAP is missing records; it is **why so few people finish the profile form**.

### Stages 3–4 emit no telemetry whatsoever

Between requesting a verification code and entering it, the application records nothing durable. The temporary record is deleted on use or after one hour, and no event is fired. The same applies if Terms of Use are declined.

Anyone who abandons in that window leaves **no trace in any system** — not the SMS database, not Pendo, not Elastic. This is the largest measurement hole in the funnel, and it sits directly on the step we most want to understand.

### Open question: how long is a trial?

Base configuration says **7 days**. Marketing copy says **30 days**. The live value comes from a deployment setting **not in source control**, so it cannot be resolved by reading code. Until settled, every expiry-based figure inherits the ambiguity.

---

## 3. Every drop-off point

Across all **29,285** external trials created this year:

| Drop-off | Share | Meaning |
|---|---|---|
| No user record at all | 2.9% | Never got past sign-up |
| **User record but no Okta account** | **14.2%** | Cannot sign in to the product they signed up for |
| **No SAP order attached** | **32.8%** | No revenue trail |

A fourth measure — profile completion — is in dispute (§12).

---

## 4. Incident days

| Date | Trials | What broke | Rate | Normal |
|---|---|---|---|---|
| 15 Jul 2026 | 108 | SAP order creation | **1.9%** | ~56% |
| 16 Jul 2026 | 98 | SAP order creation | **3.1%** | ~56% |
| 19 Aug 2026 | 154 | Okta provisioning | **50.0%** | ~83% |
| 24 Aug 2026 | 83 | Okta provisioning | 56.6% | ~83% |
| 9 Jul 2026 | 155 | Okta provisioning | 63.2% | ~83% |
| 24 Apr 2026 | 164 | SAP order creation | 36.0% | ~59% |

**9 July was an Okta problem, not a SAP problem.** Failures concentrated at 16:00, 17:00 and worst at 22:00 (**2 of 22**). SAP ran at 71% that day — above average. Roughly **57 people** ended up with a subscription and an order but no way to sign in. Individually recoverable if someone provisions them.

**15–16 July was the mirror image, and worse.** Okta was flawless both days (108/108, 98/98) while SAP order creation collapsed to 1.9% and 3.1% — five orders from 206 trials. This has the shape of a two-day integration outage that was never backfilled.

The symmetry is informative: the two paths fail **independently**, so a single shared root cause is unlikely and each needs its own fix. Neither explains the April step-change, which sits above both.

---

## 5. Timeline of key events

| Date | Event | Tier |
|---|---|---|
| April 2026 *(exact date unknown)* | Promo-code feature released | Reported |
| April 2026 | **SAP order creation steps down, 73% → 56%** | Measured |
| 24 April 2026 | Worst SAP day of the month — 36% of 164 trials | Measured |
| 17 Jun – 1 Jul 2026 | **Duplicate welcome emails** — 990 recipients, 974 extra copies | Measured |
| 9 July 2026 | Acoustic email template changed | Confirmed |
| 9 July 2026 | **Okta provisioning fails for ~57 people** | Measured |
| 15–16 July 2026 | **SAP order creation collapses** — ~200 trials affected | Measured |
| 17 July 2026 | Okta dips to 62.5% | Measured |
| 27 Jul – 18 Aug 2026 | EOI reconciliation export window — 771 stuck trials | Measured |
| 19 August 2026 | **Worst Okta day of the year** — 50.0% of 154 trials | Measured |
| August 2026 | Switch from promo code to verification link | Reported |
| 24 August 2026 | Okta dips to 56.6% | Measured |
| ~3rd week Aug 2026 | Duplicate-email trigger fixed (retry logic **not** fixed) | Reported |
| 26 August 2026 | **12,000+ auth failures circulated internally** | Reported |
| 27 August 2026 | Process-mapping session; this report compiled | Confirmed |

---

## 6. Inside the EOI reconciliation export

771 records, 27 Jul – 18 Aug 2026. 17 internal `@jjkeller.com` accounts excluded, leaving **754 external**.

This is a **filtered list, not the full population** — SMS-wide, 2,081 trial subscriptions were created in the same window, so these 771 represent about **37%** of trials.

| Outcome | Count | Share |
|---|---|---|
| No email match | 617 | **81.8%** |
| Matched — SAP order created | 104 | 13.8% |
| Matched, blocked on data | 33 | 4.4% |

### Match outcome by profile completion

| Group | n | No match | SAP order | Blocked |
|---|---|---|---|---|
| Profile **not** completed | 536 | **99.6%** | 0.4% | — |
| Profile completed | 188 | 30.9% | **52.7%** | 16.5% |
| Unknown | 30 | 83.3% | 10.0% | 6.7% |

Per §2, the first row is **by design** — profile completion is the trigger. The **30.9% no-match rate among completed profiles** is the part that points at a genuine fault.

### What "matched, blocked" breaks on

- **Address fields are blank** — most common; no address, no SAP order
- **Region/State not valid in country** — e.g. a UK-style region code against a New Zealand address
- **New customer** — informational, not a hard failure

---

## 7. Volume in context

| | |
|---|---|
| New subscriptions, 1 Jan – 27 Aug 2026 | 38,278 |
| Of those, trials | 35,823 |
| Trials as share of new subscriptions | **93.6%** |
| Companies on a paid plan | 13,514 of 299,809 |

---

## 8. Where the volume went

23,571 SAP lead records with campaign attribution, 1 Jan – 26 Aug 2026. The only dataset here that reaches above the sign-up form.

| Month | Leads | Trials | Leads/Trials | SAP order rate |
|---|---|---|---|---|
| Jan | 3,546 | 4,034 | 87.9% | 72.6% |
| Feb | 3,397 | 4,040 | 84.1% | 69.6% |
| Mar | 4,022 | 4,514 | 89.1% | 73.9% |
| **Apr** | 3,160 | 4,539 | **69.6%** | **58.7%** |
| May | 2,590 | 3,786 | 68.4% | 55.0% |
| Jun | 2,829 | 3,781 | 74.8% | 58.6% |
| Jul | 2,101 | 2,829 | 74.3% | 55.8% |
| Aug | 1,926 | 2,220 | 86.8% | 67.7% |

**Trials down 45%. Leads down 46%.** Those two numbers moving together is the finding. If the funnel were breaking down, trials would fall faster than leads. They don't.

### The lead-to-trial ratio independently confirms the April step-change

Lead records are created in SAP **downstream** of the trial order, so if SAP order creation breaks, lead records go missing too. The ratio ran 84–89% in Q1, dropped to 68–75% from April, and recovered to 87% in August — tracking the separately measured SAP order rate (73% → 56% → 68%) almost exactly, **from a completely unrelated dataset**.

### Campaign volume

| Campaign | Q1 avg/mo | Aug | Change |
|---|---|---|---|
| TM SMS CM Trial placement Agency | 391 | 6 | **−98%** |
| SMS — PPC — ToolboxTalks — Bing | 350 | 90 | −74% |
| SMS — PPC — Chemical — Google | 261 | 104 | −60% |
| SMS — PPC — Chemical — Bing | 346 | 154 | −55% |
| SMS — PPC — SafetyPlans — Bing | 320 | 155 | −52% |
| Online services default / no code | 200 | 95 | −52% |
| SMS — PPC — Safety Plans — Google | 249 | 131 | −47% |
| SMS — PPC — Audits — Bing | 163 | 87 | −47% |
| LG — WPS — General | 177 | 190 | +7% |
| TM SMS trial placement Agency | 30 | 164 | +447% |

Paid search fell roughly evenly across both networks — **Bing 1,135→492 (−57%)**, **Google 826→351 (−58%)** — which looks like reduced spend rather than any single campaign failing.

The exception is **TM SMS CM Trial placement Agency**, which collapsed in May. A similarly-named campaign rose from 30 to 164 over the same period, so this is probably a rename or re-contract — but the replacement runs at roughly **40% of the original volume**, a net loss of ~200 leads/month. Marketing will know immediately whether that was deliberate.

### Two problems, two owners

1. **Demand** — ~45% fewer people starting trials. A marketing question.
2. **Conversion** — since April, ~15 points fewer of those who start become a counted order. An engineering question.

They compound. Fixing the funnel will not bring the volume back; restoring the spend will not fix the leak.

---

## 9. The welcome email

58,543 per-recipient Acoustic events, Oct 2025 – Aug 2026.

**Scope limit, read first:** this export contains only the **welcome** email (stage 8). The **verification** email — stage 3, the one everyone wants — is not in it.

| Event | Count |
|---|---|
| Sent | 23,713 |
| Open | 22,152 |
| Click Through | 12,359 |
| Hard Bounce | 193 |
| Soft Bounce | 105 |

### The duplicate-send incident: 17 June – 1 July 2026, ~990 recipients

The 27 Aug session recorded roughly **10,000** recipients getting duplicates, tied to the **April** promo-code release. Measured, the figure is **990 recipients and 993 extra copies**, of which **974 fall in a contiguous 15-day window from 17 June to 1 July**. Outside that window: 19 across eleven months.

**Which means duplicates cannot explain the April step-change.** They began ten weeks after the drop and stopped on 1 July while the depressed rate continued through July.

> **Caveat.** The mechanism described in the session concerned the *promo-code / verification* send, which is not in this export. A separate duplicate problem may well have existed on that mailing. What can be said precisely: **on the welcome email, duplicates were a June–July event affecting ~990 people.**

### Engagement, monthly

| Month | Open | Click |
|---|---|---|
| Jan | 54% | 17% |
| Feb | 51% | 16% |
| Mar | 51% | 15% |
| **Apr** | 50% | **12%** |
| May | 53% | 13% |
| Jun | 54% | 14% |
| Jul | 52% | 13% |
| **Aug** | **41%** | **11%** |

Click-through ran 15–17% through Q1 and fell to 12–14% from April — **a third independent April signal**. August deserves separate attention: open rate fell to 41% from a stable 50–54%, its lowest in eleven months.

### On 9 July

No click-rate break (12% on the 8th, 12% on the 9th, 16% on the 10th). But the click **mix** shifted sharply: `LogIn_Button_Top` went from **31.8%** of clicks before 9 July to **46.5%** after. A template change did land in this period.

### The verification email is not an Acoustic mailing

A sample of the live message shows it is sent by `noreply@login.jjkeller.com` — **generated by Okta** — and delivers a **six-digit code the user types in**. There is no verification link.

This resolves two things:
- It explains structurally why stages 3–4 emit no click telemetry — **there is nothing to click**
- **No Acoustic export will ever contain this mailing**, however many are pulled. The route is **Okta system logs**

It also retires a hypothesis this report previously advanced (§12).

**Minor discrepancy:** the email says the code *"will expire in 30 minutes"*; the process map records a **1-hour** expiry on the temp record. Possibly benign, but one of those numbers is what the customer is told.

---

## 10. The authentication errors

Correspondence dated 26–27 Aug 2026 introduces a dataset this report has not seen.

**Over 12,000 failed authentication requests in 30 days** on the SMS API's sign-in, sign-up and password endpoints. Product Management circulated the workbook on 26 Aug and stated that although the data covers only the last month, *"this has been an ongoing issue since at least April."* The list has been passed to Okta and a technical review requested.

### This is now the strongest candidate for the April step-change

The mechanism is direct:

> failed sign-up → stage 5 blocked → mandatory profile at stage 6 never completed → **stage 6 is the trigger for the SAP order at stage 7** → no order

It fits on **timing** (April), **duration** (still ongoing), and **mechanism**. It also explains the thing the duplicate theory never did: why Okta *provisioning* stayed flat at 81–85% while orders fell. Provisioning counts accounts that were created; these are attempts that failed **before an account existed**.

**Stated as a hypothesis** — this report has not seen the workbook, only its description. Obtaining it and joining its 12,000 failures to trial records by date and email is the **highest-value next step available**. If the failure curve steps up in April and tracks the SAP order rate, the April question is answered.

---

## 11. Data source coverage

| Source | Funnel stage | Status | Notes |
|---|---|---|---|
| SMS production DB | Trial creation, Okta, profile | **Live** | `ts-jjkp-prod-sqldb`, Azure AD auth |
| SAP BI leads export | Marketing attribution | **Live** | 23,571 records with campaign + promo code |
| SAP BI EOI export | Order match / creation | **Live** | Manual `.xlsx`; 27 Jul – 18 Aug pull |
| Acoustic — welcome email | Stage 8 | **Live** | 58,543 events |
| Acoustic — verification email | Stages 3–4 | **Not supplied** | Not an Acoustic mailing — Okta-sent |
| Okta system logs | Stages 3–5 | **Not connected** | The route to the verification gap |
| Auth-error workbook | Stage 5 | **Described only** | 12,000+ failures; not yet supplied |
| Elastic APM (RUM) | Front-end page views | **No page views** | Agent live; **0 transactions in 90 days** |
| Application Insights | Front-end | Not queried | Remaining front-end candidate |
| Pendo | Product analytics | **No SMS data** | 119 pages tagged, **zero events ever** |
| Google Analytics | Promo click | Out of scope | Deferred |

### Elastic RUM captures no page views

The browser agent **is** live in production — `rum-js` v5.17.4, reporting from `www.jjkellerportal.com`. But over 90 days `SMS_Web_UI` recorded:

| Document type | Count |
|---|---|
| **transaction** (page-load / route-change) | **0** |
| **span** | **0** |
| error | 111,411 |
| metric | 30,554,414 |

No transactions means no page views, so no front-end funnel can be built from it. Nonprod shows the same pattern, so this is not environment drift.

**Likely cause, unverified:** `apm.init()` is called from the `AppModule` constructor rather than in `main.ts` before `bootstrapModule()`. Elastic's Angular integration expects init before bootstrap; initialising later means the page-load window has closed and route-change instrumentation may never bind to the router. The error handler is registered separately and would keep working — which matches the errors-yes / transactions-no asymmetry exactly.

### Pendo carries no SMS data

The integration key and aggregation API both work. But across **three years** and all event sources — page, feature, track — the only app reporting data is Encompass (`appId -323232`). SMS (`appId 5411960591024128`) has **119 pages tagged and zero events, ever**. Analytics is licensed for Encompass, not SMS.

---

## 12. Issue register

| ID | Issue | Bucket | Started | Status | Tier |
|---|---|---|---|---|---|
| **TF-06** | **SAP order rate step-change** — 73% → 56%, four months | SAP | April 2026 | **Open** | Measured |
| **TF-05** | **SAP outage** — ~200 trials, no order, never backfilled | SAP | 15–16 Jul | **Open** | Measured |
| **TF-16** | **12,000+ auth failures in 30 days** | Okta | "At least April" | With Okta | Reported |
| **TF-04** | **Okta provisioning failure** — ~57 users stranded | Okta | 9 Jul | **Open** | Measured |
| TF-07 | Chronic Okta gap — 14.2% of trials, all year | Okta | Pre-2026 | **Open** | Measured |
| TF-10 | Elastic RUM records no page views | Observability | 90+ days | **Open** | Measured |
| TF-11 | "Profile completed" defined two ways — 95% vs 25% | Definition | — | Blocks analysis | Measured |
| **TF-15** | **Lead volume down 46%** — paid search −57%, agency −98% | Demand | Q1→Aug | **Open — marketing** | Measured |
| TF-13 | Stages 3–4 emit no telemetry — needs Okta system logs | Observability | By design | **Open** | Confirmed |
| TF-14 | Trial length — 7 days config vs 30 days marketing | Configuration | Unknown | Needs decision | Confirmed |
| TF-01 | Duplicate welcome emails — 990 recipients, 993 copies | Acoustic | 17 Jun – 1 Jul | Ended 1 Jul | Measured |
| TF-02 | Retry does not inspect the error — will recur | Acoustic | April 2026 | **Open** | Reported |
| TF-03 | Acoustic template changed — **not linked to TF-04** | Acoustic | 9 Jul | Chain retired | Measured |
| TF-08 | Call-to-action below the fold at common zoom | Acoustic | Unknown | Unverified | From memory |
| TF-09 | Okta login loop | Okta | Unknown | Unverified | From memory |
| TF-12 | Pendo carries no SMS data | Observability | — | By licence | Measured |

**Seven of sixteen have no owner**, including TF-06 and TF-05.

### Claims this report has reversed

1. **Duplicates caused the April step-change** — *ruled out*. They ran 17 Jun – 1 Jul, ten weeks after the drop, and stopped while the depressed rate continued.
2. **The 9 July template change caused the 9 July Okta failure** — *retired*. The verification email is Okta-sent with no link, so an Acoustic template change cannot affect it. Two separate events on the same day. This was my own hypothesis.
3. **The funnel breakage explains the volume decline** — *corrected*. Trials −45% and leads −46% move together, so the decline starts above the funnel.

### Still in dispute

**TF-11 — what "profile completed" means.** A location-address proxy says 95% incomplete; the EOI export's own column says 25% over the same population. Until reconciled, **neither number should be quoted**.

---

## 13. Recommendations

### Now

1. **Get the auth-error workbook and join it to trial records.** 12,000+ failures described as ongoing since April, with a mechanism that fits where duplicates didn't. This is the fastest route to closing TF-06.
2. **Ask marketing what happened to paid search and the agency placement.** The largest single number in the report, and not an engineering issue. If those were deliberate budget decisions, the volume decline is explained.
3. **Reopen the April investigation.** The duplicate theory is dead; the April boundary is confirmed by three datasets. Start with April release notes and anything that shipped alongside the promo-code feature other than the email send.
4. **Backfill the 15–16 July SAP outage and provision the 9 July Okta accounts.** Two bounded, known lists — ~200 trials with no order, ~57 people who cannot sign in.

### Next

5. **Fix the retry logic, not just the missing column (TF-02).** Adding the column stopped today's duplicates; the send still retries without checking which error it received. Re-enable promo codes and it returns.
6. **Attack profile completion as a conversion problem.** Stage 6 is confirmed as the trigger for everything downstream, so this is form design — shorten it, pre-fill from sign-up and the promo code, or stage the fields so the order fires earlier.
7. **Alert on daily Okta and SAP completion rates.** All four incident days were plainly visible in a daily rate and none appear to have been caught at the time. A threshold alert at 70% would have surfaced every one.
8. **Validate address and region/country at profile-save time.** The two dominant "matched but blocked" reasons are both catchable in SMS before SAP sees the record.
9. **Work the 104 recoverable records from the EOI export.**

### Later

10. **Fix Elastic RUM transaction capture (TF-10).** The agent is deployed and licensed; it just isn't recording page views. Likely a small change that unlocks the profile-form question and front-end performance data at once.
11. **Get Okta system log access** — the only route to stages 3–4.
12. **Investigate the chronic 14% Okta gap (TF-07).**
13. **Settle the trial length (TF-14)** — someone must check the running production configuration.

---

## 14. Methodology & caveats

**Sources**
- `Leads And Orders - All Leads.xlsx` — 23,571 SAP lead records with campaign attribution, 1 Jan – 26 Aug 2026
- `stuck-sms-no-sap-order_EOI_Match.xlsx` — SAP BI EOI reconciliation, 27 Jul – 18 Aug 2026, 771 rows
- `SMS Acoustic Email Report 1.txt` — 58,543 per-recipient welcome-email events, Oct 2025 – Aug 2026
- Direct read-only queries against `ts-jjkp-prod-sqldb`, authenticated via Azure AD
- Prod Elastic APM, per-user API key
- Pendo aggregation API
- 27 Aug process-mapping transcript and the agreed process map
- Correspondence dated 26–27 Aug 2026

**External vs. internal.** Records with an `@jjkeller.com` email are treated as internal (17 of 771) and excluded from §6 percentages.

**Provisioning lag guard.** §3 excludes trials created after 23 Aug so that asynchronous Okta and SAP provisioning still in flight is not counted as permanent failure. An earlier cut without this guard showed a spike of "no Okta account" records dated the same day the query ran — lag, not failure.

**"Matched" terminology** reflects the EOI export's own `Match Status` column as produced by the existing SAP BI reconciliation job; match logic was not re-derived independently.

**Meeting-sourced material.** Speaker attribution in the 27 Aug transcript is unreliable (turns interleave), so claims are attributed to the session rather than to named individuals. The same applies to correspondence, which is cited by role.

**Not included:** Google Analytics (out of scope), Application Insights (not queried), Okta system logs (no access), the auth-error workbook (not supplied).

---

*Report generated 27 August 2026. Figures are reproducible from the sources and definitions above.*

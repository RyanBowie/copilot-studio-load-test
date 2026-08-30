# How many people can actually use a Copilot Studio agent at once?

**140,405 real messages sent to a live Copilot Studio agent published to Microsoft 365 Copilot, to
find out where the limits really are.** Not a simulation, and not a reading of the documentation.
Every number here came back from the service, and the headline 50,000 message run was performed
twice on different days to show it was not a fluke.

### [Open the interactive report](https://ryanbowie.github.io/copilot-studio-load-test/)

The report is a single self contained HTML file. No server, no build step, no dependencies. It
opens offline, prints to PDF, and works from a file share or an email attachment.

---

## The short answer

**One number explains everything: the environment answers about 180 generative messages a minute.**

Every other result is that number multiplied or divided by time.

| If your users arrive... | Outcome |
| --- | --- |
| 1,000 all clicking send in the same second | about 180 answered, the rest refused |
| 1,000 spread over 6 minutes | effectively all answered |
| 9,000 spread over an hour | effectively all answered |
| 50,000 spread over about 5 hours | effectively all answered |

So the honest answer to "can it handle 50,000 users?" is **yes if they are spread over hours, and no
if they all arrive at once**. Volume is not the problem. Simultaneity is.

## What the documentation says, and what actually happened

| Published limit | Documented | Measured |
| --- | --- | --- |
| Generative messages per minute | 100 | **about 175 to 180**, so 1.75x higher |
| Generative messages per hour | 2,000 | **not enforced at all.** Five consecutive hours ran at roughly 9,000 an hour and produced 14 refusals in total |

Both figures are more generous than documented. That is worth knowing in both directions: you get
more headroom than the docs promise, but none of it is contractual, so do not design a system that
depends on the gap.

## Headline findings

| # | Finding | Evidence |
| --- | --- | --- |
| 1 | The limit is a **rate**, not a concurrency cap | 1,000 requests genuinely in flight at once returned 194 answers. The same 1,000 at only 89 concurrent returned 193. Eleven times less concurrency, same result |
| 2 | The limit is **scoped to the environment** | A bystander agent receiving 5.3 messages a minute was refused 64% of the time purely because a *different* agent in the same environment was busy |
| 3 | Adding **user accounts does not help** | A second licensed account went from 100% answered alone to 22.2% answered while another user flooded the same environment |
| 4 | Removing **tools does not help** | Every tool was deleted and republished. 7,610 of 12,000 messages were still refused, with the same error code |
| 5 | Recovery is **immediate** | 0.2 seconds from a refusal to the next successful answer. There is no lockout to wait out |
| 6 | There is **no caching and no determinism** | 115,675 answers to the same prompt produced 24,294 distinct texts |
| 7 | It cost **nothing** | Zero Copilot Credits consumed, confirmed by hand in the Power Platform admin centre after each of the two 50,000 message campaigns |
| 8 | The whole thing **reproduces** | The full 50,000 was run twice on different days: 99.74% and 97.77% answered at an identical offered rate |
| 9 | But the ceiling **moves** | Across the two campaigns the environment lost capacity four times, refusing up to 20% of a load it had been serving perfectly minutes earlier. Roughly 8% of all time under load was degraded |

## The finding that will bite you: throttles look like successes

Every one of the **23,462** refusals arrived as an ordinary **HTTP 200** message activity.

* `http_status` was null on all 23,462. There is no 429 and no 503.
* `retry_after_s` was null on all 23,462. There is no `Retry-After` header.
* Every refusal carried the same code, `GenAIToolPlannerRateLimitReached`, with no tool involved.

**A client that checks status codes sees 100% success while a quarter of its users are being turned
away.** The refusal exists only in the response body:

```
An error has occurred.
Error code: GenAIToolPlannerRateLimitReached
Conversation Id: <guid>
Time (UTC): <iso timestamp>.
```

There is also a documentation gap worth flagging. Microsoft documents this symptom as *"The usage
limit for generative orchestration has been reached."* That sentence **never arrived once** in
140,405 messages, and neither did any other documented wording. Searching your logs for the
documented text will find nothing.

## Sizing: how big a crowd can arrive at once?

Derived from the worst fresh window admission actually observed, so it is a floor rather than an
average.

| Target success rate | Largest simultaneous burst |
| --- | --- |
| 100% | 161 |
| 90% | 202 |
| 80% | 227 |
| 60% | 303 |
| 50% | 364 |
| 20% | 910 |

If your cohort is bigger than that, the only thing that helps is spreading arrivals over **minutes**.
More client capacity, more accounts and more agents all change nothing.

A distinction that matters: everyone clicking within a 5 to 10 second window is a single event, no
matter how you count it. Users genuinely arriving 5 to 10 seconds apart is only 6 to 12 a minute,
which is far below budget.

## Why this cost nothing

Generative answers are billed at **no charge** when the caller holds a Microsoft 365 Copilot licence
and the agent runs under that authenticated user's identity. The harness only ever used that path,
and never Direct Line or a service principal, both of which do consume credits.

Copilot Credit consumption is reported only at agent and tenant level in the Power Platform admin
centre. It is absent from Application Insights and from the Dataverse audit log, so it cannot be
asserted programmatically. **It was checked by hand after the first 50,000 message campaign and
again after the whole campaign was repeated. No credits were consumed on either occasion**, across
all 140,405 messages. The claim is therefore verified against the billing system itself rather than
predicted from the documented billing rules, and it held on a second independent run.

## Was it a fluke? The whole thing was run twice

A single result is an anecdote, so the entire 50,000 message campaign was repeated end to end on a
different day with an identical configuration: same agent, same account, same prompt, same
deliberate 150 messages a minute.

| Measure | Run 1 | Run 2 |
| --- | --- | --- |
| Messages sent | 50,182 | 50,019 |
| Answered | **99.74%** | **97.77%** |
| Median answer time | 4.5 s | 4.3 s |
| Answered per clock hour | 8,910 | 8,849 |

The headline reproduced. Chasing the small gap produced the most useful finding in the report:
**every one of run 2's 1,101 refusals falls inside three short episodes where the environment
quietly lost capacity.** Take those episodes out of both campaigns and what is left is
near-identical, 1 refusal in 49,132 against 0 in 43,126. Run 1 contains a smaller episode of its
own, which reframes the 14 refusals it had previously logged as unexplained.

**So the measured ceiling is a best case, not a floor.** It sagged four times across the two
campaigns, refusing up to 20% of a steady load, and it gives no early warning: no 429, no
`Retry-After`, and no softer wording as capacity fades. Size below the measured ceiling and make the
client tolerate refusals rather than assume they will not arrive.

## What is in this repository

| File | What it is |
| --- | --- |
| [`index.html`](https://ryanbowie.github.io/copilot-studio-load-test/) | The interactive report. Nine tabs, every conversation searchable, works offline |
| [`FINDINGS.md`](FINDINGS.md) | The full written findings, including every experiment and every correction made along the way |
| [`METHOD.md`](METHOD.md) | How it was tested, and why it was tested this way |

## A note on the numbers being redacted

This ran against a real tenant, so the published version replaces the tenant name, the two user
principal names, the Dataverse organisation URL, the environment id and the Dataverse publisher
prefix with `contoso` placeholders. The agents themselves are named normally, because there is
nothing sensitive about an agent's display name. Nothing analytical changes, because every finding
is about rates and behaviour rather than identity. The redaction is a build step with a post check
that fails the build if any identifier survives, rather than a manual pass over an 11 MB file.

## Honest limitations

* **This is one environment, over about two days.** The 175 a minute figure is what this environment
  served during these runs. Microsoft does not commit to it, and it may differ by region, tenant,
  licence mix or time. Treat the *shape* of the result as the finding rather than the exact constant.
* **The measured surface is the SDK, not the Teams window.** There is no public API that drives the
  literal Microsoft 365 Copilot or Teams user interface. The transport used here preserves the same
  licensed user identity and shares both the billing treatment and the environment quota, but client
  side rendering time is not included.
* **Driving this volume through a licensed user identity is a fair use judgement**, and fair use is
  defined by Microsoft rather than by a test harness.
* **These are documented quotas being measured, not defeated.** Nothing here bypasses a control. The
  service refused exactly what it chose to refuse, and every refusal was recorded rather than
  retried around.

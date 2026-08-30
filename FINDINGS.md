# Results: load testing a Copilot Studio agent inside Microsoft 365 Copilot fair use

> **Published copy.** This is the working findings document from the project, reproduced in
> full. It was written for someone holding the whole repository, so it refers in places to a
> SQLite database and CSV exports that are **not** published, because those carry tenant
> identifiers that do not pass through redaction. Everything they contain is summarised here or
> explorable in the [interactive report](https://ryanbowie.github.io/copilot-studio-load-test/),
> which includes every conversation, searchable.


Target agent: **IT Policy and Guidance Assistant** (`IT Policy and Guidance Assistant`)
Environment: `Contoso - Dev` (`00000000-0000-0000-0000-000000000000`), Regular type
Identity: `user-a@contoso.onmicrosoft.com`, Microsoft 365 Copilot licensed
Prompt: "What is the best movie right now" (generative, ungrounded)
Transport: Microsoft 365 Agents SDK `CopilotStudioClient` (DirectToEngine), delegated user token

> **Prefer this as a shareable dashboard?** `results/copilot-studio-load-test.html` is a single
> self-contained file with the same findings as interactive charts, a searchable answer explorer,
> a minute by minute failure analysis and a browsable table of all 90,384 conversations with the
> full text of what came back. No network access, no dependencies, safe to email. Regenerate with
> `npm run dashboard`.

## The short answer

**The environment serves about 175 generative messages per minute, and that number does not
move no matter how many users you throw at it.**

Six consequences follow, and together they answer the original question:

1. **The documented 100 per minute cap is real but conservative.** The true onset of throttling
   is between 170 and 180 offered messages per minute, roughly 1.75x what Microsoft publishes.
2. **The documented 2,000 per hour cap is not enforced at all.** A five hour run held roughly
   9,000 messages per hour, 4.5x the published figure, with 14 throttles in total. Throttling is
   driven entirely by the per-minute rate, never by accumulated hourly volume.
3. **Concurrency is not what gets limited.** The rate budget always trips first: 55% of requests
   were refused at a rate of 400 per minute while only **51** were ever in flight, and separately
   **192** ran simultaneously with no trouble at all. No concurrency ceiling was ever reached. E8
   settles this decisively: the same 1,000 messages fired at once held **1,000** open connections
   and got 194 answers, while spread over a minute they held **89** and got 193. Eleven times less
   concurrency, identical outcome.
4. **The limit is shared across the whole environment. Not per agent, and not per user.** A
   bystander *agent* doing 6 messages per minute was throttled 64% of the time simply because a
   neighbour was busy. E11 then closed the remaining gap: a second, separately licensed *user
   account* asking the same agent the same question at 6 messages a minute went from **100%
   answered alone to 22.2% answered** while another person flooded it. **Adding user accounts
   adds no capacity at all.**
5. **The refusal names the tool planner, but tools have nothing to do with it.** Every tool was
   deleted from the agent and the whole sweep re-run. **7,610 of 12,000 messages were still
   refused, with the identical code and the identical message body.** Mean throughput moved from
   191 to 195 answers a minute, which is noise. Microsoft's own troubleshooting article confirms
   it, glossing the code as "the usage limit for **generative orchestration** has been reached".
   It is the plain generative AI limit, never anything to do with planners or tools, and there is
   nothing to fix at the agent level.
6. **Anything offered above the ceiling is discarded, not queued.** Banding every message by the
   rolling load it was sent under, the delivered rate flattens once the knee is passed: offering
   358 a minute yields 181 answers a minute, and offering 1,050 a minute yields 224. There is no
   backlog working itself off, so pushing harder cannot buy throughput. It only converts messages
   that would have been answered into refusals. This is why the per-minute figure is the *only*
   number worth sizing against: **9,000 an hour works because it is 150 a minute**, and 250 a
   minute "fails" 11% of the time purely because it is over the line.

That 175 figure is sharp enough to plan against. Scoring every one of the 90,384 messages by the
busiest 60 seconds it belonged to, **22,347 of the 22,361 refusals happened while the environment
was over 175 a minute. Only 14 did not**, and all 14 fall inside a single eight minute window of
the 50,000 message run. That count of 14 is unchanged after E8 through E11 added 14,016 further
refusals, every one of them above the line. Stay under 175 and refusal is close to a non-event.

**E8 then tested that claim as a prediction rather than a description.** Holding a cohort at
exactly 1,000 messages and varying only the window they arrive in, from one instant out to five
minutes, the measured success rates came in at 19.4, 19.3, 39.1, 54.1, 73.6 and 98.7 per cent
against predictions of 18.2, 18.2, 36.4, 54.6, 72.8 and 91.0. **Mean error 2.3 percentage points.**
Throughput never left the band 180 to 197 answers a minute across any of them.

So "50,000 concurrent users" is not a thing the service will do, and no amount of client
engineering changes that. What 50,000 users actually costs is **time**:

| Question | Answer |
| --- | --- |
| How many users can be genuinely in flight at once? | **About 15 sustained**, measured, or about **180** in a burst if they all arrive together. Same per-minute budget, spent differently. Not a connection limit |
| If 50,000 users all press send at the same instant? | About **180 get an answer**, 49,820 get `GenAIToolPlannerRateLimitReached`, median refusal in 781 ms |
| How long to serve 50,000 messages properly? | **About 4 hours 46 minutes** at 175/min |
| Was 50,000 actually delivered? | **Yes. 50,182 messages, 99.74% answered, 14 throttled** |
| Does it cost Copilot Credits? | **No.** Generative answers by an M365 Copilot licensed user are billed at No charge |

**Evidence base: 90,384 messages across 22 runs**, of which 66,769 were answered, 22,361 were
throttled and 1,254 failed for other reasons (1,114 session start failures, 118 empty responses,
17 timeouts and 5 unclassified), producing **18,186 distinct answers** from two prompts. Every
single throttle carried the same code, `GenAIToolPlannerRateLimitReached`.

The headline run is 50,182 messages delivered credit free at 150 per minute with a **0.028%
throttle rate**, which is the "50,000 users" deliverable in the only form the service permits:
serialised over time rather than fired at once. **Credit consumption was checked in the Power
Platform admin center after testing and no Copilot Credits were consumed**, across all 90,384
messages, so credit free here is verified rather than assumed.

## The agent under test

Deliberately an ordinary agent rather than a tuned one, so the ceiling measured here is the one a
normal published agent runs into.

| Setting | What it was |
| --- | --- |
| Agent type | A standard Copilot Studio agent, published to Microsoft 365 Copilot and Teams |
| Environment | Production type, United States region |
| Orchestration | Generative |
| General knowledge | Enabled |
| Knowledge sources | Three SharePoint sites |
| Topics | Stock topics with very light customisation |
| Tools | Four, three connectors and a flow, including Send an email (V2). Live for every run up to E8, deleted for E9 and E10 |
| Web search | Enabled before the final run |

Two consequences matter for reading the rest of this document.

**The SharePoint sources were connected but never retrieved from.** The test prompt is answered
from general knowledge, so no measured turn performed retrieval. That is what makes the latency
figures a clean measurement of generative answer time, and it keeps the run off the tenant graph
grounding billing line entirely. It is also the main limit on how far the latency numbers
generalise: an agent that retrieves on every turn would be slower end to end, though it would still
meet the same arrival rate ceiling, because that ceiling is applied before generation begins.

**There was very little to tune.** With customisation this light, there is almost nothing about
this agent that could plausibly have caused the ceiling. That is what made deleting the tools worth
testing to completion rather than arguing about, and the result is below.

## Scope: this is a generative answer ceiling, not a tool ceiling

Everything measured here is a **pure generative answer with no tool invoked**. That was deliberate,
and by the end it was enforced: E9 and E10 deleted every tool from the agent and `npm run toolcheck`
probes the live endpoint before those runs to prove the planner is handed nothing. So the ~175 a
minute figure is the ceiling on the agent *thinking and replying*, and nothing else.

**If your agent calls tools, that is not the only ceiling you face.** Each tool call goes out
through a connector to a real API, and those have their own independent throughput limits which
stack on top of the environment budget measured here. Any one of them can bite first, and a busy
tool-using agent can be perfectly inside the generative quota and still fail.

| Ceiling | Roughly where it sits | Scope |
| --- | --- | --- |
| Generative orchestration (**measured here**) | **~175 a minute** | per Dataverse environment |
| Connector throttling | commonly 6,000 per 5 minutes per connection where the connector does not publish its own, but many publish far lower | per **connection** |
| Power Platform requests | 40,000 per 24 h on paid Power Platform licences, **6,000** on Microsoft 365 licences, 250,000 on the Copilot Studio base offer | per **user** per 24 h |
| The downstream service itself | e.g. Graph, Exchange and SharePoint each throttle per user and per tenant | per user / tenant |

Three points worth carrying into any sizing conversation:

1. **Connector limits are per connection, not per environment.** Every user of the agent sharing
   one connection shares that quota, so a connector can throttle at a load the generative budget
   would have absorbed comfortably. Each connector publishes its own figures on its Learn
   reference page and they vary widely, so look up the specific ones your agent uses rather than
   assuming the default.
2. **The per-user Power Platform entitlement has a trap in it.** Copilot Studio counts API calls
   to Power Automate flows against the *user's* daily request allocation, and a user holding only
   a Microsoft 365 licence sits on the **6,000 per 24 hours** tier, not 40,000. An agent that
   fires a flow on every turn burns that allocation quickly.
3. **The two failure modes look completely different, and only one is visible to normal
   monitoring.** This is the operationally important part. Generative refusals, as measured
   across all 22,361 of them here, arrive as **HTTP 200 with no `Retry-After`** and are invisible
   to anything watching status codes. Connector throttling behaves conventionally: **HTTP 429
   with a `Retry-After`**, which standard retry logic and platform monitoring both understand. So
   a tool-using agent fails in two different ways, and the one this report is about is the one
   your monitoring will miss.

**Honesty label: the tool and connector limits in this section are documented, not measured by
this project.** Everything else in this report is measured. If tool throughput matters to your
scenario it deserves its own test, because this harness deliberately removed tools in order to
isolate the generative path.

## Is it 182 per minute, or 182 per longer period?

**Per minute, and it replenishes continuously. 182 is not an allowance you exhaust.**

This is the most common misreading of the sizing table, so here is the ladder measured directly:
the largest number of answers ever delivered inside a sliding window of each width, across all
90,384 messages.

| Sliding window | Most answers ever delivered | Equivalent rate |
| --- | --- | --- |
| 10 seconds | 194 | n/a, this is one burst |
| 30 seconds | 200 | 400 a minute |
| **60 seconds** | **260** | **260 a minute** |
| 2 minutes | 419 | 210 a minute |
| 5 minutes | 987 | 197 a minute |
| 15 minutes | 2,256 | 150 a minute\* |
| 60 minutes | 9,007 | 150 a minute\* |

\* Offer limited, not service limited. The sustained run only ever offered 150 a minute, so these
two rows measure what was asked for rather than what the service would have given.

So the three numbers mean different things and are easy to conflate:

* **~182 is burst depth.** It is what a *single instantaneous wave* gets before the door shuts,
  measured across seven repeats at 161 to 194. Use this one for "everybody clicks send at once".
* **~180 to 200 a minute is the replenishment rate**, and it keeps coming every minute. The best
  minute ever measured delivered **260**.
* **There is no longer-period cap.** The documented 2,000 an hour is not enforced. Five clock
  hours ran at roughly 9,000 an hour with 14 refusals in total, and one unbroken stretch delivered
  **46,660 answers over 312 minutes** at a steady 150 a minute. Testing ended there because the
  host machine went to sleep, not because throughput was degrading.

Put plainly: over five minutes you get about 987 answers, not 182. Over an hour, at least 9,007,
and that was offer limited. Over a day the extrapolation is roughly 214,000.

**And 182 is not a count of concurrent conversations.** Measured steady-state concurrency during
the sustained run was about **15 simultaneous** in-flight generations, because each answer takes
around 4.6 seconds and 15 concurrent at 4.6 s each is roughly 195 a minute. The service caps
arrival rate, so concurrency is an outcome of latency, not a setting you control. The only reason
a burst looks like a concurrency event is that 1,000 people pressing send together is, by
definition, 1,000 messages inside one minute.

## "But 250 a minute fails, so how can 9,000 an hour work?"

That arithmetic only looks contradictory if you assume there are two limits, one per minute and
one per hour. There is one limit, and everything else is that one number multiplied or divided by
time. **The environment delivers roughly 180 answers a minute.** 9,000 in an hour is 150 a minute,
which is comfortably under it. 250 a minute is over it. The hour never had a budget of its own.

The column that settles it is **answers a minute**. Every one of the 90,384 messages is banded by
the rolling 60 second load the environment was actually under when that message was sent, and then
the delivered rate is measured for each band.

| Load it was under | Messages | Offered a minute | **Answers a minute** | Refused | Same rate for an hour |
| --- | --- | --- | --- | --- | --- |
| 0 to 100 a minute | 575 | 52 | **52** | 0.00% | 3,111 |
| 100 to 150 a minute | 660 | 112 | **112** | 0.00% | 6,726 |
| 150 to 175 a minute | 51,638 | 152 | **152** | 0.03% | 9,105 |
| 175 to 200 a minute | 1,110 | 188 | **179** | 4.77% | 10,726 |
| 200 to 300 a minute | 5,246 | 223 | **187** | 11.49% | 11,236 |
| 300 to 500 a minute | 6,625 | 358 | **181** | 49.30% | 10,872 |
| 500+ a minute | 24,530 | 1,050 | **224** | 75.11% | 13,442 |

Read down the answers column. Below the knee it tracks the offered column exactly and nothing is
refused, because everything asked for is delivered. Above the knee **it flattens and stops
responding to pressure altogether**: offering 358 a minute delivers 181, and offering 1,050 a
minute delivers 224. A near threefold increase in pressure buys nothing.

**The surplus is discarded, not queued.** That is the most important operational sentence in this
report. There is no backlog working itself off, so pushing harder cannot buy throughput. It only
converts messages that would have been answered into refusals.

So the two facts reconcile like this:

* 9,000 an hour works because it is **150 a minute**, under the ceiling, so nothing is refused.
* 250 a minute "fails" 11% of the time because it is **over** the ceiling, and yet it still
  delivers about 187 answers a minute, which is *more* per hour than the 150 a minute run. It is
  simply wasteful: you are refusing 63 people a minute to gain 35.
* 15,000 offered in an hour does not produce 15,000 answers. It produces about 11,000 answers and
  4,000 refusals.

### What the hourly ceiling actually is

The best hour ever measured offered 9,008 messages at a steady 150 a minute and answered **9,007
of them, refusing zero**. One message failed, a client-side timeout. That is not a service ceiling
though, because the harness never offered more than 150 a minute for a sustained period. Held at
the measured limit an hour should carry roughly **10,800**, and the 175 to 300 bands are consistent
with that. Anything above about 11,000 an hour is unproven and should not be promised.

The highest band that still refused under 1% was **up to 175 a minute**. That is the number to
size against.

## Volume is not the problem. Simultaneity is.

This is the single most useful sentence in the report, so it is worth isolating.

**You can push enormous daily volume through this environment with almost no failures.** What
breaks it is a crowd arriving in the same minute.

### Volume over a day: no problem at all

The documented cap is 2,000 messages per hour. It is not enforced. Five whole clock hours,
measured per UTC hour on the sustained run, ending when the host machine slept rather than when
the service faltered:

| Clock hour (UTC) | Sent | Answered | Refused |
| --- | --- | --- | --- |
| 21:00 | 8,667 | 8,657 | **0** |
| 22:00 | 9,001 | 8,999 | **0** |
| 23:00 | 9,000 | 8,918 | **0** |
| 00:00 | 9,000 | 8,998 | **0** |
| 01:00 | 9,000 | 8,976 | **14** |
| **Five hours** | **44,668** | **44,548** | **14** |

**44,668 messages in five hours at four and a half times the documented hourly cap, and 14
refusals.** Held at that rate, a full day is roughly **216,000 messages**, and the per-minute
ceiling of 175 puts the theoretical maximum at about **252,000 a day**.

So a population of 50,000 people each asking the agent a couple of questions a day, arriving
naturally across working hours, is comfortably inside what this environment already does. That is
not an extrapolation from a short burst; it is measured, overnight, for five hours straight.

### The same 1,000 people pressing send together: mostly refused

Every burst of exactly 1,000 simultaneous messages ever fired at this agent:

| Run | Agent configuration | Sent | Answered | Refused | Success |
| --- | --- | --- | --- | --- | --- |
| E8 | four tools live | 1,000 | 194 | 806 | 19.4% |
| E9PRE | four tools live | 1,000 | 142 | 857 | 14.2% |
| E9A | tools removed | 1,000 | 190 | 810 | 19.0% |
| E9 burst 1 | tools removed | 1,000 | 188 | 812 | 18.8% |
| E9 burst 2 | tools removed | 1,000 | 166 | 834 | 16.6% |
| E9 burst 3 | tools removed | 1,000 | 177 | 823 | 17.7% |
| E10 | tools removed | 1,000 | 161 | 839 | 16.1% |
| **All seven** | | **7,000** | **mean 174** | | **14.2% to 19.4%** |

**1,000 people pressing send at the same instant get roughly 174 answers and about 826 error
messages.** That result held across seven repeats, two agent configurations and two days.

### How many can press send together

Derived from the admission constant, then checked against every measured wave from 200 to 1,419:

| If you need this success rate | Max who can press send together |
| --- | --- |
| **100%** | **161 or fewer** |
| 95% | up to 191 |
| 90% | up to 202 |
| 80% | up to 227 |
| 70% | up to 260 |
| 60% | up to 303 |
| 50% | up to 364 |
| 20% | up to 910 |

### Or read the other way round: how many get an answer right now

Same model, inverted, because this is the question people actually ask. Pick the number of people
you expect to press send inside the same minute and read off each person's chance of getting an
answer rather than an error. Typical uses the median admission of 182; the range is the worst and
best admission ever measured, so it carries genuine run to run variance.

| Users pressing send in the same minute | Answered | Get an error | Chance of an answer | Range |
| --- | --- | --- | --- | --- |
| 10 | 10 | 0 | **100%** | 100% |
| 50 | 50 | 0 | **100%** | 100% |
| 100 | 100 | 0 | **100%** | 100% |
| 150 | 150 | 0 | **100%** | 100% |
| **161** | **161** | **0** | **100%** | 100% |
| 200 | 182 | 18 | 91.0% | 80.5 to 97.0% |
| 250 | 182 | 68 | 72.8% | 64.4 to 77.6% |
| 300 | 182 | 118 | 60.7% | 53.7 to 64.7% |
| 400 | 182 | 218 | 45.5% | 40.3 to 48.5% |
| 500 | 182 | 318 | 36.4% | 32.2 to 38.8% |
| 750 | 182 | 568 | 24.3% | 21.5 to 25.9% |
| 1,000 | 182 | 818 | 18.2% | 16.1 to 19.4% |
| 1,500 | 182 | 1,318 | 12.1% | 10.7 to 12.9% |
| 2,000 | 182 | 1,818 | 9.1% | 8.1 to 9.7% |
| 5,000 | 182 | 4,818 | 3.6% | 3.2 to 3.9% |
| 10,000 | 182 | 9,818 | 1.8% | 1.6 to 1.9% |
| 25,000 | 182 | 24,818 | 0.7% | 0.6 to 0.8% |
| 50,000 | 182 | 49,818 | **0.4%** | 0.3 to 0.4% |

The answered column barely moves. That is the whole point: the environment admits a near constant
number each minute and refuses everyone else, so doubling the crowd roughly halves each person's
odds. Those refused find out fast, in about 320 ms, against roughly 4,000 ms for a real answer.

### The important nuance: it is a rate limit, not a concurrency limit

It is tempting to call this a concurrency ceiling. Strictly it is not, and the difference changes
what you should do about it.

E8 fired the same 1,000 messages two ways. Fired all at once, **1,000 connections were open
simultaneously** and 194 were answered. Spread evenly over one minute, only **89** were ever open
at once, and 193 were answered. Eleven times less concurrency, same answer count. Separately, 192
generations ran simultaneously with no trouble at all, while a *paced* stage at 400 a minute was
refused 55% of the time at only 57 concurrent.

**The service is counting messages per minute, not simultaneous users.** Simultaneity only hurts
because 1,000 people pressing send together is, by definition, 1,000 messages inside one minute.

Two practical consequences:

1. **More client capacity will not help.** Bigger machines, more sockets and Azure scale-out do
   not move the ceiling by a single message. This is why the 50,000 socket demo is only ever a
   client-side capability proof, not a throughput one.
2. **Spreading arrivals over time is the only lever that works**, alongside splitting traffic
   across environments or a pay-as-you-go rate increase. A cohort of 1,419 needs about 8.1 minutes
   at 175 a minute, or 9.5 minutes at a safe 150, to be served in full.

## How the limit actually behaves

This is the part that is not in the documentation, and it is more useful than the numbers.

**It is a per-message rejection, not an agent shutdown.** At 400 offered RPM the agent kept
answering 538 of 1,200 messages. It never stopped serving. The documented `EnforcementMessage`
("This agent is currently unavailable") was never seen once across more than 20,000 messages.

**Recovery is instantaneous.** After deliberately tripping the limit with 900 messages at 300 RPM,
a single user polling every 15 seconds succeeded on the very first attempt. Measured recovery
time: **0.2 seconds**. There is no lockout, no cooldown and no penalty box. Drop below the rate
and you are served immediately.

**The throttle arrives as HTTP 200.** This is the single most important operational detail. A
quota rejection is delivered as an ordinary message activity with a 200 status code and this body:

```
An error has occurred.
Error code: GenAIToolPlannerRateLimitReached
Conversation Id: <guid>
Time (UTC): 2026-08-28T18:24:43.967Z.
```

Any client that judges success by HTTP status will record a throttle as a successful turn, with a
latency figure that is meaningless. The harness classifies message activities as well as thrown
errors for exactly this reason.

**The error message names the tool planner, but that is misleading.** Every one of the 8,174
throttles observed was `GenAIToolPlannerRateLimitReached`. `OpenAIRateLimitReached` and
`GenAISearchandSummarizeRateLimitReached` never fired once.

The agent still has an Office 365 Outlook **"Send an email (V2)"** tool attached, described as
"This tool sends an email to the allocated person regarding the query they just had answered".
It was still present and still being evaluated by the planner as recently as 21:14 UTC during the
sustained run, appearing in 9 recorded turns with `isFinalPlan: false`. No email was ever sent and
no consent card was ever shown, so it does not affect the answers.

**Removing that tool will not help.** That was my first recommendation, and the data contradicts
it: a *different* agent in the same environment, receiving only 5.3 messages per minute, was
refused with the identical code purely because this agent was busy. The code names a shared
environment budget, not your agent's tools. See "The tool planner red herring" below.

## E1 and E1B: where the rate ceiling sits

Arrival rate stepped upward, each step held for 3 minutes with a 5 minute cooldown so no step
contaminates the next.

| Offered RPM | Sent | Answered | Throttled | Throttle rate | Mean latency |
| --- | --- | --- | --- | --- | --- |
| 25 | 75 | 75 | 0 | 0% | 8,959 ms |
| 50 | 150 | 150 | 0 | 0% | 8,043 ms |
| 80 | 240 | 240 | 0 | 0% | 7,805 ms |
| **100** (documented cap) | 300 | 300 | 0 | **0%** | 7,388 ms |
| 120 | 360 | 360 | 0 | 0% | 6,861 ms |
| 150 | 451 | 451 | 0 | 0% | 7,189 ms |
| 160 | 480 | 480 | 0 | 0% | 6,534 ms |
| **170** | 510 | 510 | 0 | **0%** | 6,330 ms |
| **180** | 540 | 534 | 3 | **0.6%** | 5,762 ms |
| 190 | 570 | 520 | 50 | 8.8% | 7,267 ms |
| 200 | 600 | 564 | 36 | 6.0% | 6,632 ms |
| 400 | 1,200 | 538 | 661 | 55.1% | 6,185 ms |

**The knee is at 175 offered messages per minute.** 170 is perfectly clean over 510 messages;
180 shows the first rejections. The documented cap of 100 was served with a 0% error rate.

Achieved throughput saturates at about **175 messages per minute**. Offering 180 achieved 169.7
sent per minute with no meaningful loss. Under heavy overload the answered rate settles between
175 and 190 per minute: E4's trip stage answered 527 in 3 minutes (176/min) and E5's saturation
stage answered 762 in 4 minutes (190/min). The single best minute ever recorded, across more than
16,000 messages at offered rates up to 2,000 per minute, accepted **193 messages**. Nothing above
that was achieved at any offered rate or any concurrency.

**Latency does not degrade as the rate rises.** Mean latency actually improved slightly from
9.0 s at 25 RPM to 6.2 s at 400 RPM. Whatever is being rate limited is an admission control gate
in front of the model, not a saturating resource behind it.

## E6: rate versus concurrency

Each rung fires N messages as simultaneously as possible, then the same N spread evenly across a
minute. If the limiter counted simultaneous in-flight requests, these two would behave very
differently. They do not.

**What burst and paced mean.** Both send the same number of messages inside the same 60 seconds.
Only the shape of the arrivals differs.

* **`burst-2000`** fires all 2,000 in the same instant. Two thousand connections open
  simultaneously. This is what "2,000 users all press send at once" actually looks like.
* **`paced-2000`** releases the same 2,000 at a steady drip across the minute, roughly 33 a
  second, so only a few dozen are ever open at a time.

The whole test turns on this comparison. If the service capped *concurrency*, the burst would fail
far worse than the paced run. It does not: 177 against 260 at 2,000, and 166 against 176 at 1,000.
The one place they differ sharply is latency, 24,143 ms against 6,340 ms, and that gap is
queueing on the test laptop rather than the service being slower.

Concurrency here is measured, not assumed. A sweep line over every turn's send and completion
timestamps reconstructs how many requests were genuinely in flight at each instant. **Offered**
is how many the client held open at once. **Served** is how many the service was actually
generating, which is the number that matters.

| Run | Stage | Offered peak | Served peak | Answered | Throttled | Answered % | Mean answer |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A | burst-200 | 200 | **182** | 182 | 18 | 91.0% | 7,485 ms |
| A | paced-200 | 24 | 24 | 200 | 0 | 100.0% | 6,095 ms |
| A | burst-500 | 500 | **192** | 192 | 308 | 38.4% | 8,184 ms |
| B | burst-500 | 500 | **167** | 167 | 333 | 33.4% | 11,153 ms |
| B | paced-500 | 72 | 61 | 183 | 317 | 36.6% | 6,912 ms |
| B | burst-1000 | 1,000 | **166** | 166 | 834 | 16.6% | 19,185 ms |
| B | paced-1000 | 125 | 93 | 176 | 824 | 17.6% | 5,688 ms |
| B | burst-2000 | **1,419** | **177** | 177 | 1,823 | 8.8% | 24,143 ms |
| B | paced-2000 | 203 | 167 | 260 | 1,740 | 13.0% | 6,340 ms |

Run A is the 1,146 turn E6 run and run B the 7,000 turn one. A's `paced-500` stage is omitted
because the kill switch stopped it 246 turns in, so it has nothing to report.

**Served peak equals the answered count exactly in all five burst rungs: 182, 192, 167, 166, 177.**
That identity is not a coincidence and it is not a discovery. When every request is fired in the
same instant, every request that gets admitted starts generating in the same instant, so the
maximum overlap of successful requests is necessarily the number of successful requests. For a
burst, served peak is the answered count restated in different units.

An earlier draft of this document read the flat line as "the service admits about 180 simultaneous
generations and refuses the surplus". **That was wrong**, and the rate ladder disproves it
directly. See below.

For the paced rungs served peak is a genuine, non-trivial measurement, because arrivals are spread
out and completions overlap only partially: `paced-200` answered 200 messages while never holding
more than 24 at once.

Note that the offered burst peaks are measured in flight counts, not intended ones. `burst-2000`
asked for 2,000 at once and reached 1,419, because the client cannot finish opening the last few
hundred sockets before the first refusals are already coming back.

### There is no concurrency ceiling. The rate budget always binds first

The E1 rate ladder settles it. Each rung holds a fixed arrival rate for three minutes, so
concurrency stays low throughout.

| Stage | Achieved RPM | Peak concurrent | Answered | Throttled | Throttled % |
| --- | --- | --- | --- | --- | --- |
| rpm-25 | 25 | 6 | 75 | 0 | 0.0% |
| rpm-100 | 100 | 17 | 300 | 0 | 0.0% |
| rpm-150 | 150 | 23 | 451 | 0 | 0.0% |
| rpm-170 | 171 | 22 | 510 | 0 | 0.0% |
| rpm-180 | 181 | 22 | 534 | 3 | **0.6%** |
| rpm-190 | 192 | 30 | 520 | 50 | **8.8%** |
| rpm-200 | 201 | 28 | 564 | 36 | 6.0% |
| rpm-400 | 402 | **57** | 538 | 661 | **55.1%** |

`rpm-400` was refused 55% of the time while never exceeding **57 concurrent requests**, and only
51 were ever generating at once. If a 180 concurrency ceiling existed, nothing here should have
been throttled at all. Meanwhile `burst-500` ran 192 generations simultaneously without any
concurrency related failure.

So the two facts are: **refusals occur at 51 concurrent when the rate is high, and 192 concurrent
succeeds when the rate is not.** Concurrency is not what is being limited.

**The honest conclusion is that we never found the concurrency ceiling, because the rate budget
always trips first.** 192 simultaneous generations is a demonstrated floor on the service's
capability, not a measured limit. Finding a real concurrency limit would need the arrival rate
held under 175 per minute while in-flight count is driven up, which requires far slower responses
than this agent produces. It is not reachable in this environment.

Concurrency does affect one thing: **how long each answer takes**. Bursting 2,000 at once pushed
mean latency to 24.1 s against 6.3 s for the same volume paced, a 4x penalty, because the client
and the service both queue. So bursting is strictly worse: the same number of answers, four times
slower, with 91% of requests wasted.

### Reconciling 15 and 180

These two numbers look contradictory and are not. They are the same budget spent two ways, and
neither of them is a concurrency limit.

The gate admits roughly 175 generative messages per minute. Pace those evenly and, at a 4.6
second answer, about 15 are in flight at any instant. Dump them all at once and the whole minute's
budget is granted in a single instant, so about 180 start generating together and the gate is then
shut for the rest of the minute. Little's Law holds in both cases; only the arrival shape changed.

**15 is what one minute's budget looks like spread out. 180 is the same budget spent at once.**
Neither is a ceiling on simultaneous connections, and neither is 50,000.

### So how much concurrency did the 50,000 message run actually have?

**About 15 users at once.** Not 50,000, and this is arithmetic rather than a shortcoming.

| Run | Messages | Median concurrent | p95 | p99 | Mean | Peak |
| --- | --- | --- | --- | --- | --- | --- |
| sustained, main (steady state) | 46,795 | **15** | 16 | 19 | 14.6 | 25 |
| sustained, top-up | 3,401 | **13** | 16 | 18 | 13.4 | 21 |

*A note on the p numbers, which appear throughout this report.* A percentile is the value a given
share of measurements came in under. **p50** is the median: half were lower, half higher. **p90**
means 90% were lower, so only the slowest 1 in 10 exceeded it. **p99** means 99% were lower, so it
is roughly the worst case 1 user in 100 saw. Averages hide the tail, which is exactly where user
complaints come from, so percentiles are what you size expectations against. Read p99 as "almost
nobody was worse off than this".

Measured across the whole main run the peak reads 343, but that is entirely an artefact of the
host waking from sleep and dumping every abandoned request onto the wire at once. Cut at the
suspend boundary, the real steady state peak is 25.

Little's Law says concurrency equals arrival rate multiplied by response time, so 150 messages a
minute at a 4.6 second answer is about 12 people waiting. The sweep line measured 14.55 and
Little's Law predicts 14.55 to two decimal places. The two agree exactly, which is the cross
check that the measurement is sound.

Little's Law does **not** hold for the burst stages, and the divergence is itself a finding. On
E6 run B it predicts 46.8 concurrent while the sweep line measured 238.8. The gap is the queue:
the client is holding five times more open than the service is willing to work on.

The honest summary is therefore in three parts.

1. **The 50,000 message run was an endurance and throughput test at about 15 concurrent users.**
   It proves the service will sustain 150 per minute for hours without degrading, and that the
   documented hourly cap is not enforced.
2. **The burst stages were a genuine client side concurrency test.** They put up to 1,419 real
   requests in flight simultaneously, which is a client capability result.
3. **They were not a successful service side concurrency test, because the rate limiter fired
   first every time.** No concurrency ceiling was found. The most simultaneous generations
   observed was 192 and nothing about that number looked like a limit.

So 50,000 concurrent is not a measurable target on this service, but the reason is the rate
budget, not a connection cap. About 180 would be answered and about 49,820 refused, and the
refusals arrive fast, with a median throttle response of **781 ms** against 4,636 ms for a real
answer.

Two measurement traps worth recording:

* `t_send` comes from a monotonic clock that restarts with every process. Pooling intervals
  across runs overlays unrelated timelines and invents concurrency that never happened. An early
  pass of this analysis reported a peak of 1,434 for exactly that reason. Every figure above is
  computed within a single run.
* **Served peak is tautological for a burst.** If every request starts in the same instant, the
  peak overlap of successful requests is just the number of successful requests, so it carries no
  information beyond the answered count. It is only informative for paced traffic. An early draft
  of this document reported the burst figure as though it were an independent measurement of a
  service concurrency ceiling. It is not.

### What actually happens if 1,419 people ask at the same moment

This is the question everyone asks when they see the peak concurrency figure, so here is the
direct answer: **about 175 get an answer and roughly 88% are refused.** No softening.

But the obvious diagnosis is the wrong one, and it changes what you would do about it. It is not
the simultaneity that breaks. Each burst below is paired with the identical volume spread evenly
across a full minute, which is the fix you would reach for if crowding were the cause.

| Messages offered | All in one instant | Spread over 60 seconds | Verdict |
| --- | --- | --- | --- |
| 200 | 182 answered, 9% refused | 200 answered, **0% refused** | Within budget either way |
| 1,000 | 166 answered, 83% refused | 176 answered, 82% refused | +10 answered, marginal |
| 2,000 | 177 answered, 91% refused | **260 answered**, 87% refused | **+83 answered, +47%** |

**A correction to an earlier draft of this section.** It originally read "spreading it out does not
help" on both rungs, justified by the refusal *rate* moving only 91% to 87% at 2,000. That was the
wrong metric and it hid the effect. In absolute terms the same 2,000 messages served **177 people
when fired at once and 260 when spread across the minute**, which is 83 more people and a 47%
improvement. A percentage against a denominator of 2,000 makes a real gain look like rounding
error. The 1,000 rung genuinely is marginal, +10 answered, and E8's purpose-built repeat found no
difference at all there (194 at once against 193 paced).

Why the gain appears at 2,000 but not at 1,000 is worth understanding, because it is the most
actionable mechanism in this report. **A burst fires everyone at t=0, so capacity that frees up
during the remaining 50-odd seconds of the minute has nobody left to serve.** Every refused caller
has already been told no and gone away. A paced offer keeps arriving, so it harvests that
replenishment as it appears. At 1,000 offered there is not much replenishment left to harvest
above the initial admission; at 2,000 the sustained pressure collects it all. This is also why
**260 answered in 60 seconds is the highest throughput ever measured here**, higher than any burst.

The practical consequence: **retrying refused callers is worth doing.** The budget is not spent,
it is simply unclaimed, and a burst leaves it on the table.

None of this overturns the main finding. The service is still counting how many walked through in
the last minute, not how many are standing at the door: 1,419 at once and 1,419 spread over a
minute both fail, because both are about 1,419 a minute against a budget of roughly 175 to 200.
Spreading arrivals over *minutes* is the lever that works; spreading them within a single minute
only recovers the replenishment a burst wastes.

There is a second cost that a pass/fail count hides. Under burst the requests that *do* succeed
get much slower, and the acknowledgement that normally lands in **146 ms** stretches to
**8.8 seconds**, a 61x degradation. Mean end to end goes from 5.5 s to 13.8 s. A burst does not
simply split the crowd into winners and losers, it also makes the winners queue.

### The exact number: how many can press send at the same instant

Detecting genuine simultaneous waves from send timestamps, rather than trusting stage labels,
gives the direct answer. The striking column is the third one: **the number admitted barely
moves** regardless of how many are offered.

| Pressed send together | Sent within | Answered | Refused | Success rate | Model predicts |
| --- | --- | --- | --- | --- | --- |
| 22 | 1.6 s | **0** | 22 | 0.0% | window already spent |
| 200 | 0.8 s | 182 | 18 | 91.0% | 91.0% |
| 203 | 3.5 s | **0** | 203 | 0.0% | window already spent |
| 343 | 1.0 s | 186 | 157 | 54.2% | 53.1% |
| 354 | 9.1 s | **0** | 354 | 0.0% | window already spent |
| 500 | 3.1 s | 192 | 308 | 38.4% | 36.4% |
| 500 | 3.9 s | 167 | 333 | 33.4% | 36.4% |
| 1,000 | 9.0 s | 166 | 834 | 16.6% | 18.2% |
| 1,419 | 13.8 s | 177 | 1,242 | 12.5% | 12.8% |

A fresh 60 second window admits **161 to 194, median 182**, and refuses the rest. So the success
rate of a burst of N is just that constant over N, a model that holds within about 2 points across
a seventyfold range of burst sizes. Inverting it sizes a launch:

| If you need this success rate | Max who can press send together |
| --- | --- |
| **100%** | **161 or fewer** (worst admission ever seen on a fresh window) |
| 95% | up to 191 |
| 90% | up to 202 |
| 80% | up to 227 |
| 70% | up to 260 |
| 60% | up to 303 |
| 50% | up to 364 |
| 40% | up to 455 |
| 30% | up to 606 |
| 20% | up to 910 |
| 10% | up to 1,820 |

Size against the worst case rather than the median, because admissions vary run to run and the
median fails about half the time. And note the rows that admitted **nobody at all**: those waves
landed on an already spent window. The second burst inside a minute gets nothing.

### Does it help if people click a few seconds apart?

Almost not at all, and the reason is precise: the budget is counted over a rolling **60 second**
window, so any spacing shorter than that leaves everyone inside the same accounting period.

Here is the controlled version. The 60 second load is held constant down each row, and arrivals
are split by how tightly packed they were within a 10 second neighbourhood. If crowding were the
problem, the right hand columns should collapse.

| Messages in the 60s window | 10 to 50 in 10s | 50 to 200 in 10s | A wall, 200+ in 10s |
| --- | --- | --- | --- |
| 150 to 250 | 99.8% (51,593) | too few | 91.0% (200) |
| 250 to 500 | 59.3% (327) | 48.3% (3,397) | 54.2% (343) |
| 500 to 1,000 | too few | 34.8% (485) | 35.9% (1,000) |
| over 1,000 | too few | 16.8% (1,344) | 11.9% (4,632) |

They do not collapse. Across each row the success rate is broadly flat, so a wall of traffic and a
steady trickle fare about the same once you fix how many arrived in the minute. The direct
comparison agrees: **1,000 messages fired in 9 seconds returned 16.6%, and the same 1,000 spread
evenly across 59 seconds returned 17.6%.**

The distinction that actually matters is whether the spacing is *within* the crowd or *between*
individuals. Everyone clicking inside a 5 to 10 second window is one event and fails as one. But
users genuinely arriving 5 to 10 seconds apart is only 6 to 12 messages a minute, far below budget
and sustainable indefinitely. To move the needle a cohort has to be spread over **minutes**:

| Cohort | All at once | Spread over, at 175/min | At a safe 150/min |
| --- | --- | --- | --- |
| 100 | 100% | no wait needed | no wait needed |
| 200 | 91.0% | 1.1 min | 1.3 min |
| 500 | 36.4% | 2.9 min | 3.3 min |
| 1,000 | 18.2% | 5.7 min | 6.7 min |
| 1,419 | 12.8% | 8.1 min | **9.5 min** |
| 2,000 | 9.1% | 11.4 min | 13.3 min |
| 5,000 | 3.6% | 28.6 min | 33.3 min |

### So how many users can it carry?

Because the constraint is messages a minute, population size alone is meaningless. What matters is
population multiplied by how often each person asks. A thousand people asking one question an hour
are trivial; a hundred asking once a minute are not.

| If each person asks | Supported at 175/min (measured) | Supported at 150/min (safe) |
| --- | --- | --- |
| once a minute | 175 users | 150 users |
| once every 2 minutes | 350 users | 300 users |
| once every 5 minutes | 875 users | 750 users |
| once every 10 minutes | 1,750 users | **1,500 users** |
| once every 30 minutes | 5,250 users | 4,500 users |
| once every 60 minutes | 10,500 users | 9,000 users |

Read against the worry directly: **1,419 users are completely fine** provided they collectively
stay under budget, which means about **one question per person every 8 minutes** at the measured
175, or every 9.5 minutes at a safer 150. That is an ordinary usage pattern for an assistant.

What is not survivable is 1,419 people all asking inside the same minute, which is a launch-day or
all-hands-broadcast pattern rather than organic use. Even that is fixable, because refusal is per
message with no lockout and recovery is 0.2 seconds: a client side queue draining at 150 a minute
clears all 1,419 in about **9.5 minutes** with near zero refusals. That is exactly what the 50,000
message run did, for eight and a half hours, at a 0.3% refusal rate.

## E8: the same 1,000 people, six different windows

Everything above was inferred from runs designed to answer other questions, so this experiment
was built to test it directly. The cohort is fixed at **exactly 1,000 messages** and the only
variable is how wide a window they arrive in: one instant, then one, two, three, four and five
minutes. Rungs are separated by a four minute cooldown, four times the width of the rolling
window, so none can inherit a spent budget from its predecessor.

It is a prediction test rather than an exploration. The model said a fresh window admits about 182
and refuses the rest, so success should be 182 divided by the offered rate. **The predicted column
was written down before the run.**

| 1,000 messages sent | Offered rate | Answered | Throttled | Other errors | Success | Predicted | Miss | Answers/min |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| all at once | 1,000/min | 194 | 806 | 0 | 19.4% | 18.2% | +1.2 | 194 |
| over 1 minute | 1,000/min | 193 | 807 | 0 | 19.3% | 18.2% | +1.1 | 193 |
| over 2 minutes | 500/min | 391 | 609 | 0 | 39.1% | 36.4% | +2.7 | 196 |
| over 3 minutes | 333/min | 541 | 458 | 1 | 54.1% | 54.6% | -0.5 | 180 |
| over 4 minutes | 250/min | 736 | 263 | 1 | 73.6% | 72.8% | +0.8 | 184 |
| over 5 minutes | 200/min | 987 | 13 | 0 | 98.7% | 91.0% | +7.7 | 197 |

**Mean absolute error 2.3 percentage points across six rungs.**

### What it establishes

**Throughput is the invariant, not volume.** Raw answered counts are not comparable across windows
of different length. Answers per minute never left the band **180 to 197** no matter how the crowd
was shaped. The service hands out a fixed throughput and refuses everything above it.

**Concurrency is definitively not the constraint.** Fired all at once the client held **1,000**
connections open. Spread across a minute it held **89**. Eleven times less concurrency, and the
outcome was 194 versus 193 answers. If concurrency were being limited, those two numbers could not
possibly match.

**Refusal is clean.** Only **2 of 6,000** messages failed for any reason other than a deliberate
refusal, and both were `NoTextActivity` empty replies rather than crashes or timeouts. Every one
of the 2,956 refusals was `GenAIToolPlannerRateLimitReached`, consistent with all 13 earlier runs.
Recovery from the last throttle to the next success was **0.5 seconds**.

**The model is conservative at the clean end.** The five minute rung beat its prediction by 7.7
points, answering 98.7% where 91.0% was expected. The 182 constant comes from burst waves, and
paced arrival sustains slightly more, around 197 a minute. So sizing with 182 errs safe.

**Bursting is punished in latency as well as in refusals.** The answered turns in the instantaneous
burst were acknowledged in a mean **6,733 ms** and completed in a median **12,874 ms**. Every paced
rung acknowledged in **132 to 206 ms** and completed in **4,504 to 4,686 ms**. Same service, same
prompt, same cohort size: bursting made the survivors roughly three times slower.

### What the refusal actually says

Every one of the 2,956 refusals across all six rungs was the same message. Not a similar message,
the identical template, varying only in the conversation id and the timestamp:

```
An error has occurred.
Error code: GenAIToolPlannerRateLimitReached
Conversation Id: aab8823d-b43e-4650-87b3-47ba30ffbca2
Time (UTC): 2026-08-29T13:31:19.740Z.
```

It is 159 characters every time. Each one hashes differently only because the guid and timestamp
are embedded in the body, which is why a naive count of distinct responses over-reports.
Normalising those two fields collapses **all 22,361 refusals in the entire 22 run dataset to
exactly one template**. There is no separate message for a burst versus a paced rung, no softer
wording as you approach the ceiling, and no variant that names generative answers rather than the
tool planner.

Three properties matter operationally, and all six rungs agree on them:

| | Value |
| --- | --- |
| Error code | `GenAIToolPlannerRateLimitReached`, 2,956 of 2,956 |
| HTTP status | none, delivered as a normal HTTP 200 message activity |
| `Retry-After` | never present |

So there is nothing at the transport layer to detect. A client that only inspects status codes
sees six thousand successful requests. The refusal has to be parsed out of the answer text.

**Refusals come back fast, and much faster than answers.** In the paced rungs the refusal was
acknowledged in a mean of 118 to 137 ms and completed in 474 to 1,723 ms, against roughly 4,600 ms
for a real answer. The environment decides to refuse well before it would have finished generating.
The burst rung is the exception at 8,588 ms acknowledgement, but that is client side queueing of
1,000 simultaneous sockets rather than the service taking longer to say no: its minimum was still
6,405 ms because every request was waiting behind the same pile.

The only two messages in the whole run that were neither answered nor refused were
`NoTextActivity` empty replies at 5,711 ms and 5,459 ms, one in the three minute rung and one in
the four minute rung. Those are the 2 in 6,000, and they are not throttling.

### The practical reading

To get a fixed cohort answered, divide it by about 175 to 190 and that is how many minutes you
need to spread it over. 1,000 people need five minutes. Anything narrower is not a capacity problem
you can engineer around on the client, because the server hands out the same throughput either way.

## E5: blast scope, and why this matters most

This is the finding with the largest operational consequence.

A second, entirely unrelated agent (`Minecraft Agent`) in the **same Dataverse environment**
was probed at a trivial 6 messages per minute, first against an idle environment and then
*concurrently* with the primary agent being driven at 400 RPM.

| Stage | Agent | Environment state | Sent | Answered | Throttled |
| --- | --- | --- | --- | --- | --- |
| baseline-same-env | Minecraft agent | idle | 12 | 12 | **0 (0%)** |
| saturate-primary | IT Policy agent | being saturated | 1,600 | 762 | 838 (52%) |
| probe-same-env-under-load | Minecraft agent | neighbour saturated | 25 | 9 | **16 (64%)** |

**A bystander agent doing 6 messages per minute went from a 0% error rate to a 64% error rate,
purely because a different agent in the same environment was busy.**

The limit is therefore environment scoped exactly as documented, and the practical consequence is
that **one misbehaving agent degrades every other agent sharing its Dataverse environment**. There
is no per-agent fairness and no isolation.

If agents matter to anyone other than their author, put them in separate environments.

The probe had to run *concurrently* with the saturation to show this. Because recovery is 0.2
seconds (see E4), any probe scheduled after a saturation stage finds a perfectly healthy
environment and reports a false negative.

## E11: does adding user accounts help? No.

E5 above is easy to misread as having already settled the scoping question. **It did not.** E5
varied the *agent* while holding the *user* constant, so a per-user budget would have produced
exactly the same numbers. It rules out per-agent scoping and nothing more.

And until this experiment, the gap was real: **every one of the 90,384 messages recorded up to
this point carried the same `identity_upn`.** Identity had never been a variable. Microsoft's
documentation does say "per Dataverse environment", but this project has already measured the
documented figures wrong by 1.75x on the per-minute cap and 4.5x on the per-hour cap, so the
documentation was not strong enough to close it.

This matters commercially. If the budget followed the user, then 50,000 licensed users would bring
50,000 budgets and the whole ceiling problem would dissolve.

### Setup

Two separate Microsoft 365 Copilot licensed accounts, the **same agent**, the **same prompt**,
identity as the only variable. Both accounts had to be licensed first: an unlicensed identity
consumes 2 Copilot Credits per generative answer, which would have broken the credit-free
requirement, and the harness hard-fails on one.

### Result: the budget is shared

| Stage | Account | Condition | Sent | Answered | Refused | Success |
| --- | --- | --- | --- | --- | --- | --- |
| solo-u1 | user-a | alone, idle | 6 | 6 | 0 | **100%** |
| solo-u2 | aliciathomber | alone, idle | 6 | 6 | 0 | **100%** |
| saturate-u1 | user-a | flooding at 600/min | 1,800 | 562 | 1,238 | 31.2% |
| **probe-u2-under-load** | **aliciathomber** | **still only 6/min** | **18** | **4** | **14** | **22.2%** |
| split-burst-u1 | user-a | 500 at once | 500 | 182 | 318 | 36.4% |
| split-burst-u2 | aliciathomber | 500 at once | 500 | **0** | 500 | **0%** |
| multi-saturate-u1 | user-a | 150/min | 450 | 200 | 250 | 44.4% |
| multi-saturate-u2 | aliciathomber | 150/min | 450 | 177 | 273 | 39.3% |

**A second, separately licensed user account, sending six messages a minute, went from 100%
answered to 22.2% answered purely because a different person was busy.** That is a 77.8 point
collapse with no change whatsoever to that account's own behaviour.

### Predicted against measured

Both predictions were written down before the run, as with E8.

| Measurement | If shared (environment) | If per user | Measured | Verdict |
| --- | --- | --- | --- | --- |
| Bystander at 6/min under load | heavily refused | about 100% | **22.2%** | shared |
| 1,000 simultaneous split across 2 accounts | ~174 answered | ~348 answered | **182 answered** | shared |
| Both accounts saturating at once | ~175/min | ~350/min | **126/min** | shared |

Every arm agrees, and the split burst is the cleanest of the three: 1,000 messages fired at the
same instant from two different accounts returned **182 answers in total**, which sits inside the
161 to 194 band measured from seven *single*-account bursts of 1,000. Splitting the same crowd
across two identities changed nothing.

The `split-burst-u2` row is the starkest illustration in the whole report: **that account received
zero answers out of 500**, because the other account's 500 had already spent the minute's budget.

The final arm is worth stating plainly because it is the one that kills the idea outright. Two
accounts offering 300 messages a minute between them delivered **126 answers a minute**, against
about 175 a minute for a single account. Adding a second licensed user did not add capacity. It
delivered slightly *less*, since the extra contention wastes budget on refusals.

### What this means

**Buying more accounts, or spreading load across users, does not raise the ceiling.** The budget
belongs to the Dataverse environment. Every user, every agent and every conversation inside that
environment draws on the same pot. The only levers remain the ones already identified: spread
arrivals over time, split across environments, or take a pay-as-you-go rate increase.

## Separating the per-minute and per-hour limits
A stepped experiment cannot separate the two documented limits on its own, because as the offered
rate climbs the accumulated hourly volume climbs with it. `npm run rolling` untangles them by
replaying every turn and reconstructing how many messages preceded it in the last 60 seconds and
the last 60 minutes.

**Per-minute axis (E1), documented cap 100:**

| Messages in preceding 60s | Sample | Throttled | Rate |
| --- | --- | --- | --- |
| 0 to 174 | 1,926 | 7 | 0.4% |
| 175 to 199 | 249 | 29 | 11.6% |
| 200 to 224 | 226 | 42 | 18.6% |
| 225 and above | 975 | 619 | 63.5% |

**Per-hour axis (E6), documented cap 2,000:**

| Messages in preceding 60 min | Sample | Throttled | Rate |
| --- | --- | --- | --- |
| 1,750 to 1,999 | 250 | 250 | 100.0% |
| 2,000 to 2,249 | 250 | 74 | **29.6%** |
| 2,250 to 2,499 | 250 | 250 | 100.0% |
| 5,000 to 5,249 | 250 | 79 | **31.6%** |

If the hourly bucket were enforced, the throttle rate would rise monotonically and stay at 100%
once past 2,000. Instead it collapses to 29.6% at a rolling-hour depth of 2,000 and to 31.6% at
5,000, which is 2.5x the documented cap. **The hourly figure tracks nothing. Only the per-minute
rate predicts throttling.**

## Every failure, minute by minute

`npm run dashboard` now includes a per-minute breakdown of all 90,384 messages. It supersedes the
E1-only table above in one important respect, explained at the end of this section.

The quota is scoped to the environment, so the number that predicts a refusal is not what one run
was sending, it is what **everything** was sending. Load is therefore pooled across all runs and
both agents. Each message is scored by the busiest 60 seconds it belonged to, computed as
a sliding-window maximum rather than a backward-looking count.

**Read the left column as a rate, not a time range.** "150 to 164 a minute" means every message
that arrived while the environment was running at somewhere between 150 and 164 messages a minute.
It is not a clock window. That row holds 51,128 messages because the 50,000 message run
deliberately sat at 150 a minute for more than five unbroken hours, so most of the dataset lives
there.

| Load the environment was under | Messages | Answered | Refused | Refused % |
| --- | --- | --- | --- | --- |
| 0 to 59 a minute | 334 | 332 | 0 | 0.00% |
| 60 to 99 a minute | 241 | 241 | 0 | 0.00% |
| 100 to 129 a minute | 660 | 660 | 0 | 0.00% |
| 150 to 164 a minute | 51,128 | 50,983 | 14 | 0.03% |
| 165 to 174 a minute | 510 | 510 | 0 | 0.00% |
| 175 to 184 a minute | 540 | 534 | 3 | 0.56% |
| **185 to 199 a minute** | **570** | **520** | **50** | **8.77%** |
| 200 to 249 a minute | 3,250 | 2,901 | 103 | 3.43% |
| 250 to 349 a minute | 5,796 | 3,543 | 2,251 | 38.85% |
| 350 to 499 a minute | 2,825 | 1,309 | 1,515 | 53.65% |
| 500 to 799 a minute | 6,318 | 2,256 | 4,062 | 64.29% |
| 800 and above a minute | 18,212 | 2,980 | 14,363 | 82.82% |
| **Total** | **90,384** | **66,769** | **22,361** | **25.09%** |

Refusal rate is taken over answered plus refused only, so the client-side failures cannot dilute
it. The knee is the 185 to 199 band. Past 250 a minute the environment refuses more than it
answers.

**The result worth quoting: of 22,361 refusals in the entire dataset, 22,347 happened while the
environment was over 175 a minute. Only 14 did not.** All 14 are in one eight minute window of the
50,000 message run. Below 175 a minute this environment is not merely usually fine, it is
essentially never refused.

Of 504 clock minutes carrying traffic, 408 were completely clean. The 96 that were not are almost
entirely the deliberate overload experiments.

### The 14 refusals in the 50,000 message run

The sustained run paced itself at 150 a minute for over five hours. Between 01:31:59 and 01:38:17
on 29 August it was refused fourteen times, then it went back to normal without changing anything.

| Minute (UTC) | Sent | Answered | Refused | Busiest 60s | Mean answer time |
| --- | --- | --- | --- | --- | --- |
| 01:29 | 150 | 150 | 0 | 152 | 5,587 ms |
| 01:30 | 150 | 150 | 0 | 152 | 7,214 ms |
| 01:31 | 150 | 148 | 1 | 152 | 7,235 ms |
| 01:32 | 150 | 146 | 4 | 152 | 7,359 ms |
| 01:33 | 150 | 145 | 5 | 152 | 6,266 ms |
| 01:34 | 149 | 147 | 2 | 153 | 5,153 ms |
| 01:35 | 151 | 150 | 0 | 153 | 5,244 ms |
| 01:38 | 150 | 147 | 2 | 152 | 6,289 ms |
| 01:39 | 150 | 150 | 0 | 152 | 6,799 ms |

Three things are visible in the turn-level data:

1. **The client never breached its own pacing.** The busiest 60 seconds anywhere in this window was
   153, against a knee at about 185. Nothing we did caused this.
2. **Refusals arrive in tight sub-second clusters on consecutive virtual users.** Three at
   01:32:29.157, .527 and .930; two at 01:33:47.560 and .919. That is the signature of a shared
   budget being sampled, not of one caller being penalised.
3. **A refusal is not always fast.** Most came back in 420 to 550 ms, but one took **19,365 ms** and
   two took about 3,550 ms. Anything that assumes a rejection is cheap is wrong. That slowest one
   was **acknowledged in 113 ms**, the same as a healthy turn, so the service accepted the
   conversation immediately and only decided to refuse it nineteen seconds later. Successful turns
   were slower than baseline through the same window, 7,214 to 7,359 ms against 5,443 to 6,084 ms
   just before it.

Taken together this reads as a brief capacity dip on the service side rather than a quota trip.
Client-side data cannot say why, only that the environment served slightly less than it had been
serving all night, and recovered on its own.

### A correction to the E1 per-minute table above

The E1 table reports 7 refusals in the 0 to 174 band. Under this analysis there are none. The E1
figure came from counting only the messages that *preceded* each one within 60 seconds, which
systematically under-reads load: the first message of a burst sees an empty history and is scored
as though the environment were idle. Scoring each message by the busiest 60 second window that
*contains* it moves all 7 into the bands they actually belong to. The sliding-window figures are
the correct ones. The E1 table is left in place because it is what the `rolling` command still
produces, and the difference between the two is itself a useful warning about how easy it is to
under-count offered load.

## The sustained run: 50,000 messages, credit free, and the 2,000 per hour cap does not exist

The rolling analysis says the hourly bucket is not enforced. A five hour run at four and a half
times the documented cap settles it. Run `npm run sustained-summary -- <run_id>`.

**The headline deliverable: 50,182 messages, every one of them a real generative answer.**

| Outcome | Count | Share |
|---|---|---|
| Answered | 50,051 | **99.74%** |
| Throttled | 14 | **0.028%** |
| Empty (`NoTextActivity`) | 112 | 0.22% |
| Errors | 5 | **0.010%** |
| Distinct answers | 12,271 | |

Delivered at a steady 150 messages per minute, which is comfortably below the measured knee of 175
and 1.5x the documented 100. Two runs: 46,781 in the main run plus a 3,401 top-up.

The boundary between the steady state and the aborted drain is derived rather than assumed. The
run contains a single 32,455 second gap where the host slept. The 14 turns immediately before it
all recorded `ClientTimeout`, because they were in flight at the moment of suspension, and the
1,211 turns after it were abandoned when the kill switch fired on wake. Excluding those 1,225
gives the steady state figures above, and `npm run dashboard` recomputes that cut from the data
every time rather than hardcoding it.

**Messages accepted per clock hour, against a documented cap of 2,000:**

| Hour (UTC) | Sent | Answered | Throttled | Versus cap |
|---|---|---|---|---|
| 21:00 | 8,667 | 8,657 | 0 | 4.3x |
| 22:00 | 9,001 | 8,999 | 0 | **4.5x** |
| 23:00 | 9,000 | 8,918 | 0 | **4.5x** |
| 00:00 | 9,000 | 8,998 | 0 | **4.5x** |
| 01:00 | 9,000 | 8,976 | 14 | **4.5x** |

Five hours at roughly 9,000 messages per hour, **four and a half times the documented 2,000**,
with 14 throttles in total. If a 2,000 per hour bucket existed, hour one would have refused 6,600
messages. It refused none.

**Five is where the test stopped, not where the service stopped.** The host machine went to sleep
at 02:14 UTC, which ended the run. Nothing was degrading: the fifth hour looked exactly like the
first. Treat 9,000 an hour as a floor, not a ceiling. The two part hours either side of that sleep
(02:00 with 14 minutes of load, 11:00 with 26) are excluded here and from the dashboard chart,
because charting a quarter of an hour next to five whole ones makes throughput look like it
collapsed when in fact nothing was being sent.

**Conclusion: the published 2,000 per hour figure is not enforced. Only the per-minute rate
binds, and the real per-minute knee is 175.**

### Why the main run stopped at 46,781 rather than 50,000

The machine went to sleep at 02:14 UTC. Traffic stopped dead for exactly 9.0 hours, resumed at
11:15, and the runner's 400 minute wall clock kill switch fired immediately on wake, aborting the
1,226 requests still queued. Those aborts are a property of the harness stopping, not of the
platform, which is why the table above reports steady state only. A 3,401 message top-up carried
the total past 50,000, and it ran with **zero throttles**.

Both problems are now fixed rather than merely noted:

1. **The kill switch measures active time, not elapsed time.** A host suspend no longer consumes
   the budget. Covered by five tests in `src/__tests__/killswitch.test.ts`, run with `npm test`.
2. **Long runs suppress sleep** via `SetThreadExecutionState`. See the README.


## E3: what actually happens when you cross the line

Answered from the 8,174 throttled turns already captured, via `npm run at-limit`.

**It is a per-message rejection, not an agent outage.** Across 175 distinct wall-clock seconds the
agent returned real answers and throttles in the *same second*. In the worst case, 132 answers and
123 throttles landed together. The agent never went down; individual messages were refused at
admission.

**Only one of the four documented error codes exists in practice.**

| Error code | Occurrences |
|---|---|
| `GenAIToolPlannerRateLimitReached` | 8,174 (100%) |
| `EnforcementMessage` | 0 |
| `GenAISearchandSummarizeRateLimitReached` | 0 |
| `OpenAIRateLimitReached` | 0 |

That the *tool planner* is the component that refuses, on an agent whose prompt needs no tool,
looked like the most actionable finding in the whole project. It is not. **It was tested directly
by deleting every tool from the agent and running the whole sweep again, and it changed nothing.**
See "The tool planner is a misnomer" below.

**Work already admitted is honoured.** Of 1,069 turns where the service had already started
responding when the first throttle landed, **849 (79.4%) still returned a real answer**. The gate is
applied at admission, not retrospectively, so a trip does not tear down conversations in progress.

**The failure is graceful to the point of being dangerous.**

| Property | Observed |
|---|---|
| HTTP status on a throttle | **200, every time** |
| `Retry-After` header | **never present** |
| Delivery mechanism | an ordinary `message` activity containing error text |

There is no 429 and no 503. A client that judges success by HTTP status will record **every single
throttle as a successful answer**, and will attach a meaningless latency to it. This is the easiest
way to produce a load test result that is confidently wrong.

It gets worse, because a rejection is fast:

| Status | Turns | Mean latency | Fastest |
|---|---|---|---|
| ok | 27,370 | 6,029 ms | 3,319 ms |
| throttled | 4,858 | **1,214 ms** | 178 ms |

*(paced stages only, since burst latency is dominated by client-side queuing)*

A throttled run therefore looks **five times faster** on an unsegmented latency chart. Mean latency
falling is a symptom of failure here, not of health. Always segment by status.

## The tool planner red herring

Every throttle names the tool planner, and the target agent has an Office 365 Outlook tool
attached. The obvious conclusion is that removing the tool would raise the ceiling. **That
conclusion is wrong**, and E5 already contains the control that disproves it. Run
`npm run tool-planner`.

While the primary agent was saturated, a *second, different* agent in the same environment was
probed gently:

| Measurement | Value |
|---|---|
| Messages sent to the neighbour | 37 |
| Refused | 16 |
| The neighbour's own arrival rate | **5.3 per minute** |
| Error code returned | `GenAIToolPlannerRateLimitReached` |

An agent receiving 5.3 messages per minute has not exhausted its own tool planner. It was refused
because a *different* agent had consumed the environment budget.

`GenAIToolPlannerRateLimitReached` therefore names the shared, environment-scoped generative
bucket. It is not a statement about your agent's tools, and removing a tool will not help.
Splitting traffic across environments is the only lever proven to work.

Worth stating plainly, because the error message actively misleads you into optimising the wrong
thing. I was misled by it myself until the control data was checked.

### The stronger control: the codes that never fired

If a distinct *generative answers* budget were the thing being exhausted, Microsoft documents a
different code for it. Scanning the response text of all 90,384 turns:

| Documented throttle code | What Microsoft documents it to mean | Times it fired |
| --- | --- | --- |
| `GenAIToolPlannerRateLimitReached` | "The usage limit for **generative orchestration** has been reached" | **22,361** |
| `OpenAIRateLimitReached` | "your agent reached the maximum number of generative answers responses" | **0** |
| `GenAISearchandSummarizeRateLimitReached` | "the usage limit for search and summarize has been reached" | **0** |
| `EnforcementMessage` | "This agent is currently unavailable. It has reached its usage limit." | **0** |

Every message in this test was a plain generative answer. No tool was invoked, no consent card was
raised and no `consent_required` turn was ever recorded. Yet the tool planner code is the *only*
refusal signal the platform produced, and the code that is actually documented for generative
answers never appeared at all.

The planner runs on every generative turn, because it is the component that decides whether to
answer directly or reach for a tool or a knowledge source. It runs even when there is nothing to
reach for. So it sits on the path of every generative answer, and its budget is the one that runs
out. **The code names the component that refused, not the reason it refused.**

That was a hypothesis when it was first written. It has since been tested directly, by deleting
every tool from the agent and repeating the experiments. See the next section.

### Microsoft's own documentation says the same thing

After the experiments were finished, the
[throttling errors troubleshooting article](https://learn.microsoft.com/troubleshoot/power-platform/copilot-studio/licensing/throttling-errors-agents)
turned out to state the conclusion outright. Its symptom list glosses the code as:

> **GenAIToolPlannerRateLimitReached:** "The usage limit for generative orchestration has been
> reached. Please try again later."

**Generative orchestration, not tools.** The identifier says "tool planner" but the documented
meaning is the orchestrator, which is precisely the component that runs on every generative turn
whether or not there is anything to call. The same article adds that the per-minute and per-hour
quotas cover "messages generated with the usage of generative AI **and for topic orchestration**",
and that "these quotas and limits apply **per Dataverse environment**".

So all three of the main findings here have independent documentary support:

| Finding, measured first | What the article says |
| --- | --- |
| The code is not about your agent's tools | It glosses the code as "generative orchestration" |
| The budget is environment scoped, not agent scoped | "These quotas and limits apply per Dataverse environment" |
| Orchestration is metered on generative turns generally | Quotas cover generative AI messages "and topic orchestration" |

This is a genuine convergence rather than a restatement: E5, E9 and E10 established all three from
measurement before this wording was read, and the article was found afterwards. It is worth saying
plainly that had this page been read at the start, the tool removal experiment might have been
skipped. Running it anyway is what turned a plausible reading of a doc into 12,000 messages of
evidence, and it also surfaced the discrepancy below, which the documentation does not mention.

### The documented message text never actually appears

The article lists friendly, user-facing sentences for all four codes. **Not one of them was ever
received.** Searching the full response text of all 90,384 recorded messages:

| Documented symptom text | Occurrences |
| --- | --- |
| "The usage limit for generative orchestration has been reached" | **0** |
| "The usage limit for search and summarize has been reached" | **0** |
| "Your agent reached the maximum number of generative answers responses" | **0** |
| "This agent is currently unavailable. It has reached its usage limit." | **0** |
| "Please try again later" (the tail shared by all four) | **0** |
| "An error has occurred." (what is actually delivered) | **22,361** |

What arrives instead is the raw diagnostic dump shown earlier: `An error has occurred.` followed by
the bare error code, a conversation id and a UTC timestamp. There is no sentence explaining what
happened and no suggestion to retry.

Two practical consequences:

1. **Searching your logs for the documented wording finds nothing.** Anyone triaging an incident
   by pasting "the usage limit for generative orchestration has been reached" into a log search
   will conclude they are not being throttled. The only reliable string to match on is the error
   code itself.
2. **The friendly text is probably rendered by the chat client, not the service.** The M365 Copilot
   and Teams surfaces likely map the code to the documented sentence for display. On the
   DirectToEngine path used here, the raw form is what comes back. Worth knowing if you build any
   custom surface, because your users would see the diagnostic dump rather than the polite message.

**One supporting signal for the credit-free claim.** The article ties `EnforcementMessage` to
credit *overage enforcement*, which is a different mechanism from the rate quotas. That message
never appeared once in 90,384 messages. This is consistent with no credit consumption, though it
is not proof of it: an environment with credits still available would also never show it. The
PPAC zero-delta check is the only valid proof, and **it has now been carried out: no Copilot
Credits were consumed.** See [Credit safety](#credit-safety).

## The tool planner is a misnomer, proved by deleting every tool

### First, a correction

For most of this project the notes claimed the agent had no tools, because the Dataverse MCP tool
had been removed at the start. **That was wrong.** The agent actually had five tools, and four of
them (three connectors and a flow) stayed enabled through every run from E0 to E8. So the
observation "the tool planner refuses on an agent with no tools" was never true as stated, and the
73,652 messages of evidence collected up to that point had to be re-read in that light.

The honest position was that the hypothesis was now genuinely open, so it was tested.

### Disabling a tool in the portal does not change what the runtime serves

Toggling the tools off had no effect on the published endpoint, and neither the portal nor the CLI
said so. `pac copilot publish` reported `Failed` with a timestamp months in the past, and the
maker portal's Publish button was greyed out.

Two independent checks settled it:

1. **Dataverse.** `bot.publishedon` was 28 August while `bot.modifiedon` was 29 August. The agent
   had been edited and not republished.
2. **The live endpoint.** Sending a prompt engineered to make the orchestrator reach for a tool
   returns a `DynamicPlanReceived` activity that names every tool the planner was handed:

   ```
   steps: ["IT Policy and Guidance Assistant.action.Office365Outlook-SendanemailV2"]
   toolDefinitions: [{ "$kind":"ToolDefinition", "displayName":"Send an email (V2)" }]
   ```

This is now a reusable gate, `npm run toolcheck`, and E9 and E10 abort rather than run against a
stale configuration. That matters because the failure mode is silent: the run completes, produces
a plausible table, and answers the wrong question.

After a successful republish, `publishedon` moved to the current time and the probe reported no
tool definitions. Only then were the experiments run.

### E9: seven bursts of 1,000, across every tool configuration

A burst of 1,000 reveals the admission constant in a single shot, so it is the sharpest available
test. If tools mattered, the tools-removed group would sit clearly above the tools-enabled group.

| Burst of 1,000 | Tool configuration | Answered |
| --- | --- | --- |
| E8 | four tools live | 194 |
| E9PRE | four tools live | 142 |
| E9A | tools removed | 190 |
| E9 repeat 1 | tools removed | 188 |
| E9 repeat 2 | tools removed | 166 |
| E9 repeat 3 | tools removed | 177 |
| E10 | tools removed | 161 |

Tools enabled: mean **168**, range 142 to 194. Tools removed: mean **176**, range 161 to 190.

The two ranges overlap almost entirely. Note that E10's 161 is now the **lowest fresh-window
admission ever recorded**, marginally below the 166 to 194 band measured before any tool was
touched, and the worst burst of all (142) happened with four tools *live*. If tools were consuming
the budget, the tools-removed group would sit clearly above the tools-enabled one. It does not:
its mean is 8 higher, its floor is lower, and both sit inside ordinary run-to-run variance.

Because the sizing table is derived from the worst admission ever observed, this single run moved
the 100% row from 166 to **161**. That is the only number in the whole report that E9 and E10
changed, and it moved in the direction of slightly *less* capacity, not more.

### E10: the entire window sweep, replayed rung for rung

Identical cohort of 1,000, identical arrival windows, identical prompt, same agent. Only the tools
differ.

| 1,000 sent | Four tools | No tools | Change | Four tools per min | No tools per min |
| --- | --- | --- | --- | --- | --- |
| all at once | 194 | 161 | -33 | 194 | 161 |
| over 1 minute | 193 | 212 | +19 | 193 | 212 |
| over 2 minutes | 391 | 419 | +28 | 196 | 210 |
| over 3 minutes | 541 | 603 | +62 | 180 | 201 |
| over 4 minutes | 736 | 762 | +26 | 184 | 191 |
| over 5 minutes | 987 | 965 | -22 | 197 | 193 |

Four rungs went up, two went down. Mean throughput moved from **191 to 195 answers a minute**,
about 2%, which is smaller than the run to run variance already documented in the burst band. The
capacity model built in E8 still holds without modification.

### The verdict

**Removing every tool changed nothing.**

With not a single tool left on the agent, **7,610 of 12,000 messages were still refused**, and every
one of them still carried `GenAIToolPlannerRateLimitReached` in the identical 159 character
template, byte for byte, including the same absent HTTP status and absent `Retry-After`.

Combined with E5, where a *different* agent at 6.3 messages a minute was refused 64% of the time
purely because this agent was saturating the environment, the conclusion is firm:

> The budget is a shared, environment-scoped generative budget. The tool planner is simply the
> component that sits on the path of every generative answer and reports the refusal. Removing
> tools from your agent does not raise your ceiling, and neither does adding them lower it.

The practical consequence is that there is nothing to fix at the agent level. Capacity is bought by
spreading arrivals over time, by splitting agents across environments, or by a pay-as-you-go
environment with a rate limit increase. Not by editing the agent.

**Web search note.** Web search grounding was switched on before E10 at the user's request. The
live probe confirms it is not a planner tool, and it remains no-charge for a Microsoft 365 Copilot
licensed user on the authenticated path. It did not change the ceiling either.

### What the refusal actually said, with no tools on the agent

Asked directly what the throttle message was this time. The answer is that **there is still only
one message**, which is itself the strongest single piece of evidence in this section.

All 7,610 refusals across E9A, E9 and E10 are this, verbatim, and nothing else:

```
An error has occurred.
Error code: GenAIToolPlannerRateLimitReached
Conversation Id: 6bf92d97-3f58-4726-b9e7-f67666c885ac
Time (UTC): 2026-08-29T15:36:31.131Z.
```

Only the conversation id and the timestamp vary. Normalising those two fields collapses all 7,610
to **exactly one template**, and every single one is **exactly 159 characters long**. Same code,
same wording, same length as the refusals produced when four tools were live.

Per rung, checking for any variation at all:

| 1,000 sent | Refused | Distinct wordings | Distinct codes | HTTP status | Retry-After | Refusal ack | Refusal total | An answer took |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| all at once | 839 | 1 | 1 | none | none | 15,451 ms | 15,519 ms | 17,764 ms |
| over 1 min | 788 | 1 | 1 | none | none | 116 ms | 321 ms | 3,996 ms |
| over 2 min | 581 | 1 | 1 | none | none | 116 ms | 317 ms | 4,225 ms |
| over 3 min | 397 | 1 | 1 | none | none | 117 ms | 333 ms | 4,232 ms |
| over 4 min | 238 | 1 | 1 | none | none | 116 ms | 330 ms | 4,041 ms |
| over 5 min | 35 | 1 | 1 | none | none | 113 ms | 326 ms | 3,971 ms |

Four things worth pulling out of that table.

1. **No rung produced a second wording or a second error code.** Not near the ceiling, not far
   below it, not in the burst. There is no gentler warning as you approach the limit and no
   distinct message for a wall of simultaneous arrivals.
2. **Nothing is visible at the transport layer.** `http_status` and `retry_after_s` are null for
   all 7,610. A client that checks status codes sees 12,000 successes and no reason to back off.
   The response body is the entire signal.
3. **A refusal resolves in about 320 ms against roughly 4,000 ms for a real answer**, so the
   environment decides to refuse long before it would have finished generating. Refusals are
   cheaper than answers, but they are not free.
4. **The burst row is client side, not service side.** 15,451 ms is 1,000 sockets queueing on one
   laptop, not the service being slow. The paced rungs, which is what real traffic looks like,
   all sit at 113 to 117 ms.

Comparing the paced rungs against E8, when four tools were live, the refusal ack was 112 to 114 ms
then and 113 to 117 ms now. Even the timing signature of a refusal is unchanged by deleting every
tool.

### Refusal happens after acceptance, not at the door

The acknowledgement timings settle the mechanism. Splitting each turn into acknowledgement
(send to the service's first activity on the conversation) and generation:

| | Median acknowledgement | Median total |
| --- | --- | --- |
| Answered turns | 116 ms | 4,636 ms |
| Refused turns | **222 ms** | 781 ms |

A refused turn is accepted almost as quickly as a healthy one, and only then refused. The most
extreme case, in the sustained run, was **acknowledged in 113 ms and refused 19,365 ms later**.

Two consequences. First, this is why there is no HTTP 429 and no `Retry-After`: nothing at the
edge is rejecting anything, the conversation is already open and the refusal is generated inside
the orchestrator like any other message. Second, **a refusal is not free and not necessarily
fast**. Anything that assumes rejected load is cheap load is wrong.

## Is it really sending the prompt and waiting for the answer?

Yes. `npm run verify` prints the raw activity stream for a single turn with a millisecond offset
from the moment the prompt is sent, so the round trip can be inspected rather than trusted:

```
--- conversation start ---
  +   464 ms  type=typing
  +   581 ms  type=message text="Hello, I'm IT Policy and Guidance Assistant. How can I help?"
  conversationId = be286b2a-86a6-45fc-8374-3231c98f143c

--- sending prompt at t=0 ---
  +   451 ms  type=typing
  ... 160 further typing activities while the model generates ...
  +  4944 ms  type=typing
  +  5021 ms  type=message  chars=822

measured latency : 5022 ms  (send to final activity)
answer length    : 822 characters
```

Three things this establishes:

1. The prompt really is sent and the harness really does block until the agent's generated answer
   arrives. Latency is measured from send to the final message activity.
2. **Copilot Studio does not stream tokens on this transport.** It emits a long run of `typing`
   activities and then delivers the complete answer in a single `message` activity. That is why
   `ttft_ms` always equals `latency_ms` in the recorded data: there is no partial text to time.
3. Conversation start is measured separately (581 ms above) and **excluded** from the recorded
   latency, because a conversation start is not a generative AI message.

### Why each conversation looks short in the Activity page

The harness opens **a new conversation for every turn**, because the scenario being modelled is
50,000 *different* users each asking one question, not one user asking 50,000 questions.

So every conversation in the Activity page contains exactly three things: the agent's greeting,
one user message, and one answer. They are supposed to look short. There are simply tens of
thousands of them.

Paste any `conversation_id` from `data/results.sqlite` into the Activity page to find the matching
server-side record.

## Credit safety

Generative answers are billed at **No charge** when the caller holds a Microsoft 365 Copilot
licence and the agent runs under that authenticated user's identity
([billing rates, footnote 1](https://learn.microsoft.com/microsoft-copilot-studio/requirements-messages-management)).

Everything the harness does is designed around keeping that true:

| Control | Implementation |
| --- | --- |
| Licensed identity only | Preflight calls Graph `/me/licenseDetails` with each identity's own token and hard-fails without SKU `639dec6b-bb19-468b-871c-c5c441c4b0cb` |
| No Direct Line | The Direct Line and custom website channels are never constructed |
| No service principal | Device code flow only, public client, delegated scope |
| Single billing line | The prompt is an ungrounded generative answer, so no Agent action (5 credits) or tenant graph grounding (10 credits) line item is ever touched |

Licence status was verified before every run. Two identities were actually used to send traffic:

| Account | Microsoft 365 Copilot | Used | Messages sent |
| --- | --- | --- | --- |
| `user-a@contoso.onmicrosoft.com` | Yes | All 22 runs | 89,410 |
| `user-b@contoso.onmicrosoft.com` | Yes, licensed for E11 | E11 only | 974 |
| `user-c@contoso.onmicrosoft.com` | No | Never used | 0 |

The plan originally called for three rotating identities. Only two were ever needed: the cap is
per environment and not per user, which E11 proved directly, so extra accounts add no throughput.
The second account exists to make identity a variable rather than to add capacity. `johndoe` was
never licensed and never sent a message.

### Confirmed: zero credits consumed

**Copilot Credit consumption is only ever reported at agent and tenant level in the Power
Platform admin center.** It is not in Application Insights and not in the Dataverse audit log, so
there is no way to assert a zero delta programmatically. It has to be read by a human.

**That check has been done twice. Copilot Credit consumption did not move either time.** It was
read after the first 50,000 message campaign and again after the whole campaign was repeated, and
all **140,405** messages across both were served with **no credit cost**. The credit-free claim in
this report is therefore verified against the billing system itself rather than inferred from the
documented billing rules, and it has now survived a second independent 50,000 message run rather
than resting on a single reading.

## Response comparison

Every turn stores the full response text and a SHA-256 hash, so answers can be clustered.

**There is no caching and no determinism.** Across 2,000 identical prompts in a single stage,
1,939 distinct answers were returned. The largest cluster of identical responses was 38. Every
answer opened with some variation on "There isn't a single objective best movie right now" and
then diverged in wording, structure and which films were named.

That matters for two reasons: latency figures are not contaminated by a cache, and response
variance is real model variance rather than an artefact of the harness.

Across the whole evidence base of 66,769 answered turns, **18,186 distinct answers** were
returned, and **85.90% of them were seen exactly once**. The most repeated answer appeared 1,135
times out of 66,769, which is 1.70%.

Full detail, including response time percentiles split into acknowledgement and generation, a
per run and per stage breakdown, the 25 most frequent answers and three complete answers
verbatim, is in the **Answers** tab of the interactive report. Regenerate it with
`npm run answers`. It also writes `results/answers-all.csv` (every distinct answer with its
count and full text) and `results/turns-all.csv` (all 90,384 turns with every timing column).

To read individual conversations rather than aggregates, the **Every conversation** tab of
`results/copilot-studio-load-test.html` carries all 90,384 turns with their send time,
acknowledgement time, total response time and what came back, sortable and filterable by outcome,
run or response text, with the full detail of any turn one click away.

## What to do with this

**If you need more than 170 messages per minute:**

1. **Do not bother removing the Outlook tool.** This was my recommendation until the data
   contradicted it, see "The tool planner red herring" above. The ceiling is an environment
   budget, not an agent one.
2. **Request a rate limit increase**, but only from a pay-as-you-go environment.
   [Microsoft's guidance](https://learn.microsoft.com/troubleshoot/power-platform/copilot-studio/licensing/throttling-errors-agents)
   is explicit that environments on message-based licensing are not eligible. `ContosoPAYGAgents`
   is the eligible candidate. Run `npm run support-pack` to generate the evidence bundle.
3. **Split across environments.** The limit is per environment, confirmed by E5. Splitting an
   agent's traffic across N environments multiplies available throughput by N. This is the only
   lever proven to work, and it is also the only way to stop one busy agent degrading its
   neighbours.

**If you are designing a client:**

1. **Never burst.** It produces the same number of answers four times slower with 91% waste.
2. **Pace at or below 150 messages per minute** for a zero-error run, with 175 as the hard
   ceiling.
3. **Parse the response body, not the HTTP status.** A throttle is an HTTP 200.
4. **Do not implement a backoff-and-lockout strategy.** Recovery is 0.2 seconds. Simply pacing
   correctly is sufficient, and retries consume the same rate budget as new messages.
5. **Isolate anything that matters.** A neighbour agent's load becomes your outage.
6. **Do not alert on mean latency.** A throttle returns in 1.2 seconds against 6.0 seconds for a
   real answer, so mean latency *falls* as the agent starts failing. Alert on the count of
   responses whose body matches `RateLimitReached`.

## Reproducing

```powershell
npm install
npm run login                       # device code, once per identity
npm run smoke                       # one user, one prompt, end to end
npm run verify                      # raw activity stream for one turn
npm run experiment -- E1 --yes      # rate ceiling
npm run experiment -- E1B --yes     # knee refinement
npm run experiment -- E4 --yes      # recovery time
npm run experiment -- E5 --yes      # blast scope across agents
npm run experiment -- E6 --yes      # concurrency ladder
npm run report -- all               # CSV plus FINDINGS.md
npm run at-limit                    # E3, behaviour at the limit
npm run tool-planner                # is the tool planner error really about tools?
npm run rolling -- E1               # separate the per-minute and per-hour limits
npm run support-pack                # Microsoft throughput increase evidence bundle
```

Raw per-message data for every run is in `data/results.sqlite`, and per-run CSVs are written to
`results/`.

## The whole 50,000 was run a second time (2026-08-30)

A single result is an anecdote, so the entire campaign was repeated end to end on a different day
with an identical configuration: same agent, same single account, same prompt, same deliberate 150
messages a minute. Nothing changed except the date.

| Measure | Run 1, 28 Aug | Run 2, 30 Aug | Change |
| --- | --- | --- | --- |
| Messages sent | 50,182 | 50,019 | -163 |
| Answered | 50,051 | 48,904 | -1,147 |
| Answered, percent | **99.74%** | **97.77%** | **-1.97 pts** |
| Refused | 14 (0.028%) | 1,101 (2.201%) | +1,087 |
| Other failures | 117 | 14 | -103 |
| Offered rate | 150/min | 150/min | no change |
| Median answer time | 4.5 s | 4.3 s | -0.26 s |
| Answered per clock hour | 8,910 | 8,849 | -60 |

The headline reproduced. Both campaigns pushed 50,000 credit-free messages through a single
licensed account at a rate the documented quota says should have been refused twenty five times
over, and both were answered essentially in full.

But the second campaign refused seventy eight times as many messages as the first, and chasing that
gap produced the most useful finding in this report.

### The ceiling moves

**The difference is not spread across the run. Every single one of the second campaign's 1,101
refusals falls inside one of three short degraded episodes**, and outside those minutes the two
campaigns are near-identical:

| Outside the degraded episodes | Offered | Refused |
| --- | --- | --- |
| Run 1 | 49,132 | **1** |
| Run 2 | 43,126 | **0** |

So when the environment is healthy the two results are not merely close, they are effectively
perfect both times. The entire 1.97 point gap is three bad patches.

A degraded episode is counted only where the offered load for that minute was at or under the
measured ceiling, so it can never be an overload being correctly refused. Three consecutive minutes
above one percent refused are required before a stretch is reported, and short clean gaps are
bridged so that a ragged recovery is reported as one episode rather than chopped into fragments.

| Campaign | When (UTC) | Minutes | Offered/min | Answered/min | Refused | Answer time | vs baseline | Sends reordered |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Run 1 | 08-29 01:32 to 01:38 | 7 | 150 | 148 | 1.24% | 5.9 s | 1.1x | 2.4% |
| Run 2 | 08-30 15:46 to 15:59 | 14 | 146 | 123 | **15.63%** | 9.0 s | 1.8x | 1.5% |
| Run 2 | 08-30 19:00 to 19:15 | 16 | 150 | 120 | **20.28%** | 8.8 s | 1.8x | 3.2% |
| Run 2 | 08-30 19:30 to 19:46 | 17 | 144 | 126 | **12.03%** | 18.3 s | 3.7x | **30.0%** |

Run 2 was served flawlessly for its first 102 minutes: 15,371 offered, 15,370 answered, zero
refused. Then capacity quietly sagged. Across both campaigns 54 of 674 minutes under load were
degraded, about 8% of the time.

**This also reframes an earlier result.** The original report recorded 14 refusals that occurred
below 175 a minute and logged them as an unexplained curiosity inside one eight minute window.
Applying the same detection to run 1 finds that 13 of those 14 sit inside a seven minute episode.
They were never noise. They are the same mechanism, two orders of magnitude smaller.

### Why this is the service, not the test client

The obvious objection is that the client degraded, especially as the first attempt at this run
died of a memory leak. The measurement that settles it is **send ordering**.

Every message is dispatched in strict sequence, but its send time is stamped after the service
accepts the conversation. So a slow accept lets turns land out of order. That is the giveaway: a
stalled or memory-starved client resumes and fires its backlog **in order**, whereas variable
service-side latency **scrambles** it.

Reordering sat at 1.8% across all clean minutes. The two milder episodes measured 1.5% and 3.2%,
matching the baseline, which means the service was still accepting conversations normally and only
generation had slowed. The worst episode reordered **30%** of its sends, so by then even accepting
a conversation had degraded. Client memory was flat at 279 MB throughout.

Nothing else about the refusals changed. All 1,101 carry a single error code
(`GenAIToolPlannerRateLimitReached`), and `http_status` and `retry_after_s` are null on every one.
A dip is not a different failure mode. It is the ordinary refusal arriving at a load the
environment had been handling comfortably minutes earlier.

### What this changes in the guidance

**The measured ceiling is not a floor.** 175 a minute is the best case, not a guarantee. Sizing a
real workload at the number this report measures leaves no room for the service to have a bad
quarter of an hour, and it demonstrably does, twice in one afternoon. There is no early warning to
throttle back on either: refusals carry no 429, no Retry-After and no softer wording as capacity
fades. Keep genuine headroom, and make the client tolerate refusals rather than assume they will
not arrive.

The credit-free conclusion is unaffected and has now been checked independently for a second time:
100,201 messages across the two campaigns, every one a generative answer on the authenticated M365
Copilot path, and Copilot Credit consumption in the Power Platform admin centre did not move after
either campaign.

### A harness bug found by running it again

The first attempt at run 2 died at 17,719 messages with Windows exit code `0xC0000409`, no stack
trace and no event log entry. The cause was in the harness: the ramp controller collected every
turn promise into an array and only settled it at the end of the stage, so a five hour paced stage
retained tens of thousands of closures, each holding a full response body. Replaced with a set that
drops each promise as it settles; memory then held flat for the remaining 32,300 messages. Run 1
had survived the same bug by luck.

## A chart that did not add up (2026-08-30)

Spotted on the published page: the burst ladder showed **"1,419 offered ... 177 served ...
1,823 refused"**. Those numbers cannot all be true at once. 177 + 1,823 is 2,000, and the quoted
8.8% is 177/2,000, not 177/1,419.

The cause was two different units on one row. The bars are drawn from `offeredPeak` and
`servedPeak`, which are **instantaneous concurrency** measured by the sweep line, while the
annotations were drawn from `n`, `ok` and `thr`, which are **whole-wave totals**. The row was
therefore quietly reporting a concurrency figure and two totals side by side as though they
shared a denominator.

It survived this long because for four of the five bursts the two are identical:

| Wave | Sent | Answered | Refused | Peak in flight |
| --- | --- | --- | --- | --- |
| burst-200 | 200 | 182 | 18 | 200 |
| burst-500 | 500 | 167 | 333 | 500 |
| burst-500 | 500 | 192 | 308 | 500 |
| burst-1000 | 1,000 | 166 | 834 | 1,000 |
| burst-2000 | 2,000 | 177 | 1,823 | **1,419** |

Only the 2,000 wave diverges, because it is the only one where the client could not get every
socket open before the first refusals were already coming back. So the discrepancy was real and
worth surfacing rather than papering over: **1,419 is a limit of one laptop, not of the service.**

Fixed by giving each unit its own place. Wave totals now sit in the left column, where sent always
equals answered plus refused, and each bar is labelled in its own units, "in flight at once" and
"generating at once". Added a paragraph telling the reader the left column and the bars are
different measures, and why the largest wave is the one place they part company.

Worth noting the general lesson, because it is the third time this project has hit it: a derived
chart can be arithmetically self-contradictory while every input to it is correct. Nothing in
`tsc`, the 67 tests or the render audits catches it, because each number is individually right.
Only reading the finished row as a sentence does.

Verified: tsc clean, 67/67 tests, all five rows reconcile exactly, 9 tabs, 10 charts, 0 empty
tables, 0 JS errors and `cardRows:1` in both themes, no clipped or overlapping labels. Republished
and pushed as `f01dc87..82085eb`.

## The page looked broken while it loaded (2026-08-30)

The published artefact is one self-contained file, **4.3 MB gzipped over the wire**, measured with
`curl -H "Accept-Encoding: gzip"`. On a slow link that is 26 seconds. The markup all sits in the
first 0.13% of the file and the data payload follows it, so the browser paints the headline and
every paragraph almost immediately and then leaves every table, chart and figure blank until the
payload lands. The reader sees a finished-looking page with holes in it, which reads as broken
rather than busy. Reported as "the page is taking a while to load".

Two fixes, neither of which removes any data:

1. **A boot banner** directly under the tab bar, explaining that every conversation is embedded so
   it downloads once and then works offline. The client script removes it once everything is drawn.
2. **The four hero figures are now emitted into the markup at build time.** Previously the opening
   sentence rendered as "*real messages pushed through a published agent... It sustained , and*"
   because `m-total` and `m-safe` were empty until the script ran. The script still writes the same
   values afterwards, so the runtime path stays the single source of truth.

Measured: `wire_bytes=4,491,304`, `time_total=26.1 s`, `speed=172 KB/s`. Note that PowerShell's
`Invoke-WebRequest` reports `RawContentLength` **after** decompression, so it showed 16.3 MB and
72 s and made the transfer look four times larger than it is. Use `curl.exe` with an explicit
`Accept-Encoding` header and no `--compressed` when you want true wire bytes.

The size itself was left alone deliberately. It is dominated by the deduplicated response text for
24,294 distinct answers, which is what makes the "Every conversation" tab searchable offline, and
that was an explicit requirement.

## Two presentation defects, both found by reading the rendered page (2026-08-30)

**1. A headline card that measured the wrong thing.** The Concurrency tab led with
**"Peak offered at once: 1,419"**. That is how many requests one laptop managed to hold open in the
same instant, and even that number is depressed, because the client could not open the last few
hundred sockets before the first refusals came back. It described the test rig, not the service,
and it sat in the most prominent position on the tab.

Replaced with the matched pair that actually carries the tab's argument, since the same experiment
already contained it:

| Same 1,000 messages | Peak in flight | Answered |
| --- | --- | --- |
| Fired in one instant | 1,000 | 166 |
| Paced over a minute | 125 | 176 |

**Concurrency moved eightfold and the answer count did not.** The paced run in fact answered
slightly more. That is a service-side result with concurrency as the only variable, which is
exactly what "no concurrency ceiling was ever found" needs, and it is derived from the ladder data
rather than hardcoded.

**2. Answered and refused were near-indistinguishable.** The hourly throughput chart stacked
refused on top of answered using `--cp-accent` against `--cp-danger`. In the dark theme those are
`#fd8ea1` and `#f87171`, two shades of the same hue, and in the light theme crimson against red is
no better. A 14 message refused cap on a 9,000 message bar was invisible. Refusals now use
`--cp-warning` with a `--cp-text` outline, so even a two pixel sliver separates cleanly from the
bar it sits on, in both themes.

Both were invisible to every automated check, for the same reason as the burst ladder defect: the
values were correct, the tests passed, and only looking at the rendered output revealed the
problem.

## A colour audit, and a bug that made the labels unreadable (2026-08-29)

The user reported that answered and refused were too similar on the hourly chart. Rather than fix
only that chart, every legend in the artefact was swept programmatically: each pair of swatches was
compared with a weighted RGB distance in both themes, and anything under a threshold was flagged.

That found two more pairs nobody had reported. On the outcome breakdown, **refused (amber) sat next
to error (red)**, and **Empty and Consent required were two different greys** (distance 102 in light,
138 in dark, against a threshold of 150). The palette now uses five well separated fills: green for
answered, amber for refused, link blue for error, grey for empty, danger red for consent required.
Moving error to blue is also the more honest encoding, because a client side transport failure is
this harness breaking rather than a service outcome, so it should not sit in the same hue family as
the service's own refusals.

**The sweep then turned up a real bug that had been shipped from the beginning.** The stacked bar
segment labels were written with a `fill` presentation attribute, but the stylesheet carries

```
svg text { font-family: ...; fill: var(--cp-text-muted); }
```

and a CSS rule always beats a presentation attribute. So `fill="#fff"` never applied. The labels had
been rendering in muted grey the whole time, at a measured contrast of **2.03:1 on green and 3.11:1
on amber** in light theme, and 1.81:1 and 1.89:1 in dark. All four are far below the 4.5:1 minimum,
and the green case is essentially illegible.

Two places in the file used a `fill` attribute on `<text>`; both were dead. The other was the
"Measured ceiling, about 175 a minute" annotation, which was meant to be green and was also
rendering grey. Both switched to inline `style="fill:..."`, which does win over the rule.

Label colour then had to be chosen per fill rather than from a theme variable: `--cp-success` and
`--cp-warning` are both light enough in *either* theme that dark text is correct, whereas any theme
text variable flips to near-white in dark and fails. Final measured contrast is **5.28:1 and 8.10:1**
in light, **9.99:1 and 10.43:1** in dark.

Worth recording the general lesson, because it is the same one as the previous three defects: the
tests and the type checker both passed throughout, and the numbers were never wrong. A presentation
attribute silently losing to a stylesheet rule is invisible to everything except measuring the
rendered result. The check that caught it is now the one worth keeping: compute contrast ratios and
inter-swatch distances from `getComputedStyle` on the live page, rather than trusting what the
source says the colour should be.

Verified after the fix in both themes: zero low contrast legend pairs, outcome legend reconciles to
140,405, 9 tabs, 10 charts, 0 empty tables, KPI cards on one row, no `[object` leakage, 0 JS errors.
tsc clean, 67/67 tests. Published and pushed as `c89657c..e1c7c9b`.

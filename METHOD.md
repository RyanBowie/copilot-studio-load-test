# Method: how this was tested

The short version: a TypeScript harness sent real prompts to a real published Copilot Studio agent
over the supported SDK path, as a genuinely licensed Microsoft 365 Copilot user, and recorded one
row per message including the full response text.

## 1. The question, and why the obvious framing was wrong

The original brief was "load test with 50,000 concurrent users". Early measurement showed that
framing does not survive contact with the service, because Microsoft caps **arrival rate** rather
than **concurrency**. The two are linked by Little's Law:

```
concurrent users = arrival rate x average response time
```

At a 4.6 second average response, 180 messages a minute is only about **14 concurrent users**.
Firing 50,000 at once to find a per minute limit would have produced roughly 180 successes and
49,820 identical errors, which is one data point dressed up as a big number.

So the work was split along two axes. Most experiments varied **rate** with very few users. Separate
experiments varied **concurrency** directly, specifically to test whether concurrency mattered at
all. It did not, and proving that was one of the more useful results.

## 2. How it reached the agent

| Step | What was used | Why it had to be this |
| --- | --- | --- |
| Identity | Two real Microsoft 365 Copilot licensed accounts, signed in with a delegated token. One carried 89,410 messages, the second 974 during the multi account test | The no charge billing path requires the agent to run under an authenticated licensed user. A service principal or unlicensed identity would have consumed credits |
| Transport | Microsoft 365 Agents SDK `CopilotStudioClient`, the DirectToEngine channel | It preserves the licensed user identity end to end, so it shares both the billing treatment and the environment quota with the real Teams surface |
| Agent | A published Copilot Studio agent surfaced in Microsoft 365 Copilot, with every tool removed | Tools add consent cards that stall an unattended run, and tool call time contaminates the latency figure. Removing them makes every turn a pure generative answer |
| Conversation | One fresh conversation per message | Models distinct users. Reusing one conversation would let the service carry context forward, which is not what a crowd looks like |
| Storage | SQLite, one row per message, batched writes, write ahead logging | A 48,000 message run has to survive a crash, and the full response text must be kept because that is the only place a refusal is visible |

The prompt was deliberately ungrounded: *"What is the best movie right now"*. It produces a pure
generative answer with no tenant graph grounding, which keeps the run on a single billing line and
removes retrieval time as a confounding variable.

## 3. Why not browser automation

The obvious approach, driving the real chat window with headless Chromium, does not scale to this
question. Fifty thousand browser contexts would need several terabytes of RAM. More importantly it
would measure the browser rather than the service. The SDK path uses a few hundred bytes per in
flight request, so a single laptop can saturate the environment many times over.

## 4. What was recorded

One row per message: run and stage ids, the identity used, conversation and activity ids, the
prompt, send time, first activity time, first token time, completion time, both derived durations,
the **full response text**, a SHA-256 hash of it, the response length, a status, and the error code,
HTTP status and `Retry-After` value where present.

Storing the full text is what made the central finding possible. Refusals arrive as HTTP 200 with no
error headers, so a harness that recorded only status codes would have reported 100% success.

Timings use a monotonic clock so a multi hour run is immune to wall clock adjustment.

## 5. The experiments

| Experiment | Question | Method |
| --- | --- | --- |
| E1 | Is the documented 100 a minute real? | Arrival rate stepped 25, 50, 80, 100, 120, 150, 200, 400 a minute, three minutes each, with cooldowns |
| E4 | How long is the lockout after a refusal? | Trip the limit, then poll one message every 15 seconds and measure time to first success |
| E5 | Is the limit scoped to the agent or the environment? | Flood one agent while a second, different agent in the same environment sends 6 a minute |
| E6 | Does simultaneity matter, separately from rate? | Fire N at once against the same N paced over a minute, at N = 200, 500, 1,000, 2,000 |
| E8 | Can the admission model predict an unseen result? | Hold the cohort at exactly 1,000 and vary only the arrival window: one instant, then 1, 2, 3, 4 and 5 minutes. Predictions written down before the run |
| E9 / E10 | Does deleting every tool raise the ceiling? | Delete all tools, prove the live endpoint serves the change, then repeat E8's rungs |
| E11 | Does adding user accounts add capacity? | Same agent, same prompt, identity as the only variable |
| Sustained | Deliver the 50,000 | Paced at 150 a minute, resumable across restarts |

E8 is the one worth singling out. Everything before it was measured from runs designed to answer
other questions, so the admission model risked being a curve fitted to its own data. E8 varied one
thing only, and the predicted column was committed before the run. Mean absolute error came out at
**2.3 percentage points** across a sixfold range of arrival windows.

## 6. Analysis choices that changed the answer

Three of these were mistakes caught during the work, and each one had produced a plausible but wrong
result first.

* **Load must be scored with a sliding window, never clock minutes.** A clock boundary chops a heavy
  burst into a light looking stub, so refusals get attributed to an idle environment. Every message
  is scored by the busiest 60 seconds it actually belonged to. The sliding window implementation was
  verified against a brute force check.
* **A backward looking window under reads load badly.** The first message of a burst sees an empty
  history and looks like it was refused while nothing was happening.
* **Concurrency must never be pooled across runs.** The send timestamp comes from a monotonic clock
  that restarts with each process, so pooling intervals from different runs overlays unrelated
  timelines and invents concurrency. A first pass reported a peak that never happened.
* **Acknowledgement is not time to first token.** Copilot Studio does not stream, so the first token
  arrives with the finished answer. The real acknowledgement is first activity minus send. Getting
  this wrong made acknowledgement look identical to total latency.
* **Refusals hash uniquely.** The conversation id and timestamp are embedded in the refusal body, so
  every refusal has a different SHA-256. Counting distinct hashes without normalising those two
  fields inflates the distinct answer count. All refusals collapse to exactly one template.

## 7. Reproducing this

The findings that transfer are the *shape* of the result rather than the exact constants: that the
binding limit is a rate, that it is scoped to the environment, that refusals are invisible at the
transport layer, and that spreading arrivals over time is the only mitigation.

To measure your own ceiling, the minimum viable version is much smaller than this project. Send a
fixed cohort at a known rate as a licensed user, store the full response body, and search that body
for `RateLimitReached`. Do not trust the HTTP status. Then repeat at a few different arrival windows
and look for the rate at which refusals begin.

Check Copilot Credit consumption in the Power Platform admin centre before and after, because it is
the only place it is reported.

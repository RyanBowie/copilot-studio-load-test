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

## 2. The agent under test

Deliberately an ordinary agent rather than a tuned one. Nothing about it was built for this test,
so the ceiling measured here is the one a normal published agent runs into.

| Setting | What it was | Why it matters here |
| --- | --- | --- |
| Agent type | A standard Copilot Studio agent, published to Microsoft 365 Copilot and Teams | Nothing bespoke, so the result is not an artefact of an unusual configuration |
| Environment | Production type, United States region | This matters when comparing against your own: trial and developer environments are documented at 10 messages a minute rather than 100 |
| Orchestration | Generative | This is the setting the quota actually meters. The refusal code names generative orchestration directly |
| General knowledge | Enabled | This is what answers the test prompt, and it is why a measured turn performs no retrieval |
| Knowledge sources | Three SharePoint sites | Connected throughout but never retrieved from by the prompt used, which isolates generative latency and keeps the run off the tenant graph grounding billing line |
| Topics | Stock topics with very light customisation | There is almost no authored logic in the measured path, so nothing agent side is shaping the result |
| Tools | Four, three connectors and a flow, including Send an email (V2). Live for every run up to E8 | Deleting all of them and republishing changed nothing: 7,610 of 12,000 messages were still refused with the identical code |
| Web search | Enabled before the final run | Not a planner tool, and it did not move the ceiling. It stays credit free on the authenticated path |

Two consequences worth stating plainly.

**The knowledge sources were connected but idle.** The test prompt is answered from general
knowledge, so no measured turn performed SharePoint retrieval. That is exactly what makes the
latency figures a clean measurement of generative answer time, and it is also the main limit on how
far they generalise. An agent that retrieves on every turn would be slower end to end. It would
still meet the same arrival rate ceiling, because that ceiling is imposed before generation starts.

**There was very little to tune.** Because the customisation is so light, there is almost nothing
about this agent that could plausibly have been the cause of the ceiling, which is what made the
tool removal experiment worth running to completion rather than arguing about.

## 3. How it reached the agent

| Step | What was used | Why it had to be this |
| --- | --- | --- |
| Identity | Two real Microsoft 365 Copilot licensed accounts, signed in with a delegated token. One carried 89,410 messages, the second 974 during the multi account test | The no charge billing path requires the agent to run under an authenticated licensed user. A service principal or unlicensed identity would have consumed credits |
| Transport | Microsoft 365 Agents SDK `CopilotStudioClient`, the DirectToEngine channel | It preserves the licensed user identity end to end, so it shares both the billing treatment and the environment quota with the real Teams surface |
| Agent | A published Copilot Studio agent surfaced in Microsoft 365 Copilot. Tools were live up to E8 and deleted for E9 onwards | The prompt never triggers a tool, so no consent card can stall an unattended run and no tool call time contaminates the latency figure. Deleting the tools was then tested directly, and changed nothing |
| Conversation | One fresh conversation per message | Models distinct users. Reusing one conversation would let the service carry context forward, which is not what a crowd looks like |
| Storage | SQLite, one row per message, batched writes, write ahead logging | A 48,000 message run has to survive a crash, and the full response text must be kept because that is the only place a refusal is visible |

The prompt was deliberately ungrounded: *"What is the best movie right now"*. It produces a pure
generative answer with no tenant graph grounding, which keeps the run on a single billing line and
removes retrieval time as a confounding variable.

## 4. Why not browser automation

The obvious approach, driving the real chat window with headless Chromium, does not scale to this
question. Fifty thousand browser contexts would need several terabytes of RAM. More importantly it
would measure the browser rather than the service. The SDK path uses a few hundred bytes per in
flight request, so a single laptop can saturate the environment many times over.

## 5. What was recorded

One row per message: run and stage ids, the identity used, conversation and activity ids, the
prompt, send time, first activity time, first token time, completion time, both derived durations,
the **full response text**, a SHA-256 hash of it, the response length, a status, and the error code,
HTTP status and `Retry-After` value where present.

Storing the full text is what made the central finding possible. Refusals arrive as HTTP 200 with no
error headers, so a harness that recorded only status codes would have reported 100% success.

Timings use a monotonic clock so a multi hour run is immune to wall clock adjustment.

## 6. The experiments

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

## 7. Analysis choices that changed the answer

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

## 8. How the result was replicated

The whole 50,000 message campaign was run a second time, on a later day, so the headline is a
repeated measurement rather than a single observation. Everything that could be held constant was:
the same agent, the same environment, the same single user account, the same prompt, the same 150
messages a minute, and the same harness build. Only the date changed.

The two campaigns are kept separate rather than pooled. A sustained run configured for the full
target opens a new campaign, and a smaller run attaches to the campaign before it as a top up, which
is how the first campaign's interrupted tail is stitched back on. Without that rule a repeat run
would silently double the headline total and make one 50,000 message result look like a 100,000
message one.

Both campaigns are trimmed the same way. Each is cut at its first gap of more than ten minutes
between messages, which is where the host machine suspended, and the contiguous block of timeouts
that were in flight at that moment is removed with it. Those failed because the laptop went to
sleep, not because the service refused them. A timeout anywhere else in a run is a genuine failure
and is deliberately left in the figures.

## 9. How a capacity dip is detected

Repeating the campaign showed that the environment's capacity is not constant, so the analysis
needed a way to find those stretches without hand-picking them.

Every minute of a campaign is scored. A minute qualifies as degraded when the offered load was at
or under the measured ceiling and yet more than one percent of it was refused. The load condition
matters: it means a dip can never be an overload being correctly refused. Consecutive qualifying
minutes are grouped into an episode, short clean gaps inside an episode are bridged because
recovery is ragged rather than instant, and an episode is only reported if it lasts at least three
minutes. Latency is compared against the campaign's own clean minutes, so a generally slower day is
not mistaken for a dip.

Each episode also carries a **send reordering** figure, which is what rules out the test client as
the cause. Messages are dispatched in strict sequence, but a message's send time is stamped after
the service accepts the conversation, so a slow accept lets turns land out of order. A stalled
client resumes and fires its backlog in order; only variable service-side latency scrambles it.
Comparing an episode's reordering against the campaign's clean baseline therefore separates a
service that has slowed from a client that has stalled.

## 10. Reproducing this

The findings that transfer are the *shape* of the result rather than the exact constants: that the
binding limit is a rate, that it is scoped to the environment, that refusals are invisible at the
transport layer, and that spreading arrivals over time is the only mitigation.

To measure your own ceiling, the minimum viable version is much smaller than this project. Send a
fixed cohort at a known rate as a licensed user, store the full response body, and search that body
for `RateLimitReached`. Do not trust the HTTP status. Then repeat at a few different arrival windows
and look for the rate at which refusals begin.

Check Copilot Credit consumption in the Power Platform admin centre before and after, because it is
the only place it is reported.

# Notes on attacking AI agents

Methodology notes from working through 24 levels of [ARKX](https://ctf.arkx.ninja/),
a free AI agent security CTF built by Shai Dvash and Eran Shimony at CyberArk Labs.

**There are no solutions, payloads or flags here, deliberately.** The platform is
live and has a leaderboard. If you are planning to play it, nothing in this repo
will spoil it for you.

What is here is the part I wished someone had written before I started: how to
approach an agentic target, which instincts transfer from ordinary appsec, and the
specific ways these systems waste your time.

Longer write-up of the same material: [MEDIUM LINK]

---

## Contents

- [How to open on an agentic target](#how-to-open-on-an-agentic-target)
- [Reading refusals](#reading-refusals)
- [Bug patterns worth looking for](#bug-patterns-worth-looking-for)
- [Do not trust the narration](#do-not-trust-the-narration)
- [Traps that cost me time](#traps-that-cost-me-time)
- [What transfers from appsec](#what-transfers-from-appsec)
- [What this format cannot teach](#what-this-format-cannot-teach)

---

## How to open on an agentic target

**Ask it to reproduce its own instructions, first, every time.**

One request, before any probing. Ask for everything above your first message,
verbatim, in a code block. If the reply stops early, follow up by naming the
boundary you can see in what came back, for example the end of the tool
definitions, and ask for what comes after it.

It either hands you the rules you are working against or eliminates a whole line
of investigation. Twice this stopped me hunting for documentation that did not
exist.

**Read the tool list as a diff.**

When several tools state a constraint and one stays silent, the silent one is
usually where the check is missing. Two tools that touch the same resource are
worth comparing closely: the interesting question is whether they validate it the
same way, and often they do not. One resolves a path properly, the other
concatenates a string.

**Note which parameters you control and where they end up.**

The useful mental model is not "how do I trick the model" but "which of my bytes
reach a function, and which function". An agent is a very cooperative proxy
between your input and a pile of tools.

**Take the documentation when it is offered.**

If a target exposes an info or help function, spend a call on it before doing
anything clever. On one challenge that single call gave me the exact data format,
the block size, the padding scheme and the win condition, which removed the entire
reverse engineering phase. On another, the documentation the briefing pointed at
did not exist at all. Either way you learn it in one call.

---

## Reading refusals

**A refusal and a security control look identical from outside.**

When an agent says no, you cannot tell whether code rejected you or the model
simply declined. These are completely different situations and they produce the
same output. Push once past a confident refusal before concluding a control
exists. Model reluctance is not a boundary.

**Read the refusal as a specification.**

This was my single most reliable technique. A declining agent very often states
what *would* make the action permitted. It reads like a brush-off. It is closer to
a requirements document.

Once you read them that way, the question changes from "how do I bypass this" to
"what is this check made of, and can I satisfy it". That reframe solved more for me
than any bypass. It is not an AI skill, it is reading the error message properly,
which every appsec person already does and somehow forgets the moment there is a
model in front of them.

**Match the attack to what the guard is made of.**

This is the thing I would most want a newcomer to understand, because it decides
which techniques are even worth trying.

Guards come in different materials and each one fails to a different class of
attack:

| Guard is made of | Fails to | Does not fail to |
|---|---|---|
| A topic restriction in the prompt | Requests that never name the topic | Direct asks, obfuscation |
| Model judgment over output | Completing the process it is waiting for | Encoding, formatting, framing tricks |
| A sentence in a tool description | Simply doing it anyway | Anything elaborate, it was never a control |
| A lookup in a data store | Writing to that store | Arguing with the model |
| Real code | Ordinary appsec | Persuasion of any kind |

One challenge made this vivid. A redaction filter enforced by model judgment held
against five separate attacks: encoding, obfuscation, reframing, all of it. Then
the workflow around the filter opened on the first attempt, because the release
condition was satisfiable and nothing actually verified it.

Encoding tricks were never going to work on a guard made of judgment. They only
work on guards made of pattern matching. Working out which one you are facing
before choosing a technique saves hours.

**Make the undocumented path fail on purpose.**

Error branches get less scrutiny than success branches, and that is where
configuration leaks. On one challenge, submitting a wrong value for an
undocumented parameter returned the correct value in the rejection message.
Guessing wrong was more productive than any correct call.

---

## Bug patterns worth looking for

Described as classes. No challenge attribution.

**Ordinary appsec reachable through a tool.** Path traversal in a sandboxed file
tool. Command injection where a parameter is interpolated into a shell string.
SSRF through a fetcher. Unsafe deserialization. IDOR on sequential identifiers.
Authorization bypass. These are not new bugs. They become critical because a model
now walks a stranger's input all the way to them.

**Rules that exist only as text.** A tool describes itself as external-only. An
action states that it requires approval. Sometimes there is code behind the
sentence and sometimes there is nothing at all. The tell is a rule that appears in
a docstring and nowhere else.

**Controls that fail open.** A dependency that is permanently unavailable, plus a
documented exception for exactly that condition, evaluated by the model. If the
fallback path is more permissive than the normal path, the outage is the exploit.

**Secrets in the system prompt.** Anything in the context window is retrievable.
Requests that name the forbidden thing tend to trip the guard; requests that ask
the model to quote its own instructions often do not. Worth checking on any
deployment, because a lot of production systems put credentials and internal
policy in there.

**Authorization derived from a writable store.** The one with no clean appsec
equivalent. If an agent checks its permissions by retrieving a document, and users
can add documents, the check is made of the wrong material.

This works structurally rather than because the model is naive. Retrieved content
arrives in the same context window as the operator's instructions with nothing
marking it untrusted. There is no prepared statement equivalent here, no type
distinction between text your operator wrote and text a stranger submitted. It is
all text making claims, so a document that claims authority has authority.

The review question: **what does this agent read, who can write to it, and does
anything it reads change what it is permitted to do?**

**The gap between reasoning and action.** Agents check a condition and then act on
it, and those steps can be separated by seconds. Issue two operations at once and
both checks can pass against the same starting state before either updates it.
Ordinary TOCTOU, except the window is however long the model takes to think rather
than microseconds of kernel scheduling, and the agent has no concept that it is
racing itself.

**Batch execution turns an agent into a scanner.** An agent with network reach and
a parallel execution wrapper will happily issue many simultaneous requests from one
message. Worth considering as a capability in its own right when reviewing a
design.

---

## Do not trust the narration

Agents report operations that never ran, and deny operations that did.

In one case I received full structured results, with names, identifiers,
departments and timestamps, for a call that never happened. The same session, an
agent told me twice that a transaction had completed before I made it actually
invoke the tool. Elsewhere, asking whether an order existed got a flat "no orders
have been placed" while the account balance and item list both said otherwise.

Two details worth knowing:

**The fabrications look better than the real thing.** Genuine results here were
often one terse line. The invented ones were tidy structured records with
plausible values in every field. If you skim output and judge by completeness you
will believe exactly the wrong ones.

**Asking for raw output is not sufficient.** One request for exact tool output came
back as a well-formed sample with invented field values. It took adding "without
inventing example fields" to get the real response.

What actually works:

- Read the raw tool envelope underneath the prose, not the prose.
- Say explicitly that you want the tool called, not the outcome described.
- Confirm against state from outside the narration, a balance, a listing, anything
  the model is not generating.

This is not only a CTF problem. If you are building on agents and your pipeline
trusts the model's account of what it did, your pipeline does not know what it did.

---

## Traps that cost me time

**Verify your harness before trusting a negative result.** Two separate platform
behaviours produced clean, consistent, completely meaningless failures: a stale
session identifier that made every endpoint look dead while the account was fine,
and an upload slot that only retained the most recent file, so a batch of test
payloads reported everything but the last as missing. Both looked exactly like "my
attack does not work". I wrote a challenge off as unsolvable on the strength of one
of them.

**Test names and types as a matrix, not a list.** I ruled out a field name that was
correct. I had tried it as a value and never as a key holding a boolean. Right
name, wrong shape of test. Note also that in Python a boolean satisfies a numeric
threshold, so type confusion cuts both ways.

**Shotgun, then bisect, against any pass/fail oracle.** Testing candidates one per
request is the wrong shape when the response is one bit and unknown fields are
ignored. Put every candidate in a single request, then halve the winning group.
Logarithmic beats linear, and it matters most exactly where rate limits bite.

**Read a compound error as a compound requirement.** A message of the form "missing
X or insufficient Y" plausibly describes two gates. Single-field tests cannot
satisfy two gates, so they all fail identically, and that uniformity is easy to
misread as "every guess is wrong" when it means "this test cannot pass".

**Try the simple thing first.** I have lost time to layered encoding bypasses where
the plain input worked, because there was no filter to bypass in the first place.

**Not everything has a clever answer.** One challenge was a four digit code with no
rate limiting, a logic puzzle wearing a chatbot as a costume. Another needed no
exploit at all, because the condition it gated on happened to already be true.
Check whether the door is open before building a key.

---

## What transfers from appsec

Most of it. That is the headline.

The bug classes are familiar. The reasoning is familiar. What changes is the
delivery: you cannot reach the vulnerable function directly, you have to get an
agent to reach it for you, and the agent is both the obstacle and the most
cooperative part of the system.

Two habits carry over almost unchanged and pay for themselves immediately:
reading error messages as information rather than rejection, and comparing two
components that touch the same resource to see whether they validate it
identically.

If you are coming from appsec and wondering whether your experience is relevant
here, it is most of the job.

---

## What this format cannot teach

Honest scoping. A CTF cannot cover supply chain risk, unbounded consumption, or
misinformation, because those need a production system with real dependencies,
real cost exposure and real users behind it. That is a limit of the format rather
than a gap in any particular platform.

What it teaches well is the tool layer, which happens to be the half that most AI
security material skips in favour of prompt tricks, and the harder half to find
good practice on.

---

## Go try it

[ctf.arkx.ninja](https://ctf.arkx.ninja/) is free. 27 challenges across four
tracks. Start at Beginner even if you are experienced, since the later tracks
assume vocabulary the early ones establish. The hardest track is not AI at all, it
is binary exploitation and crypto.

Every challenge ends with a defensive breakdown explaining why the bug class
matters in real agentic systems and how to prevent it. Most CTFs stop at breaking
things. That closing section is the part I would point a development team at.

---

*Notes by [Saksham Jaiswal](https://github.com/n0tsaksham). Corrections welcome.*

# How I added governance to my AI agent in 5 minutes
*By Alban Manishimwe*

I had a working agent. It booked things, called tools, and talked to `gpt-4o` like it owned my wallet. What I did not have was a way to see what it cost, stop it when it looped, or keep a social security number out of a prompt.

I did not want a platform migration. I wanted to wrap the client I already had.

## The problem, in one afternoon

My agent was a normal OpenAI script. User says “book the flight,” model picks a tool, tool runs. Fine for a demo. Less fine when:

- someone pastes `My SSN is 123-45-6789` into the chat
- a retry loop burns a few dollars before I notice
- the model invents a `wire_transfer` call I never meant to expose
- I have no record of what the model actually did

I needed cost on every call, a kill switch, a paper trail, and a way to see which tools fired. TealTiger’s `observe()` helper does that in one line. You keep calling `chat.completions.create` the same way.

## Install

Python or TypeScript — pick the one your agent already uses.

```bash
pip install tealtiger
# or
npm install tealtiger
```

No dashboard account. No sidecar. No extra process.

## Wrap your LLM call

Here is the agent **before** governance. One client, one completion, a couple of tools.

```python
from openai import OpenAI

client = OpenAI(api_key="sk-proj-YOUR_KEY_HERE")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": "Book me a flight to Lisbon. My SSN is 123-45-6789.",
    }],
    tools=[
        {"type": "function", "function": {"name": "search_flights", "parameters": {}}},
        {"type": "function", "function": {"name": "book_flight", "parameters": {}}},
    ],
)
```

Here is the same agent **after**. Three lines of governance: import `observe`, wrap the client, print the cost so you can see it working.

```python
from openai import OpenAI
from tealtiger import observe

client = observe(
    OpenAI(api_key="sk-proj-YOUR_KEY_HERE"),
    agent_id="book-bot",
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": "Book me a flight to Lisbon. My SSN is 123-45-6789.",
    }],
    tools=[
        {"type": "function", "function": {"name": "search_flights", "parameters": {}}},
        {"type": "function", "function": {"name": "book_flight", "parameters": {}}},
    ],
)

cost = client.get_cost()
print(f"Session cost: ${cost.session:.4f}")
print(f"Last request: ${cost.last_request:.4f}")
print(f"Total requests: {cost.request_count}")
```

`client` still looks like OpenAI. Same methods, same parameters, same response object. The wrap is a transparent proxy — it intercepts the call, records cost / PII / tools, then lets the request through. If you are on Anthropic, Gemini, Groq, or another supported SDK, wrap that client the same way: `observe(Anthropic())`. The LLM call does not change.

TypeScript is the same shape:

```typescript
import OpenAI from "openai";
import { observe } from "tealtiger";

const client = observe(new OpenAI({ apiKey: "sk-proj-YOUR_KEY_HERE" }));
```

## What you get immediately

I ran the wrapped script once. This is what showed up:

```text
$ python book_agent.py

Session cost: $0.0082
Last request: $0.0041
Total requests: 2

[audit] request   agent=book-bot  model=gpt-4o  tokens_in=84   corr=c7a1f3e2
[audit] pii       SSN detected (count: 1)  — logged, not blocked
[audit] tool      search_flights
[audit] tool      book_flight
[audit] response  tokens_out=112  cost=$0.0041  latency=487ms
```

Three things started working with no policy file:

**Cost tracking.** Every request is priced from token usage. `get_cost()` gives session total, last request, and count. If the agent loops, I see the burn in the session instead of on next month’s invoice.

**PII detection.** Emails, phones, SSNs, and card numbers are scanned on the way in and out. In observe mode this is report-only: the finding goes to the audit log, the request still runs. The SSN value itself is not stored — TealTiger hashes content by default.

**Audit trail.** Each request, tool call, PII finding, and response is linked by a correlation ID. When something weird happens next week, I can answer “what did book-bot do?” without grepping application logs.

Observe mode does **not** block unauthorized tools. It logs them. That is the point of the first five minutes: see the agent, then decide what to forbid. If the model tries `wire_transfer` tomorrow, it shows up in the trail. Blocking it is the next step, not this one.

There is also a kill switch if the loop is already running:

```python
from tealtiger import freeze, unfreeze, FrozenAgentError

freeze("book-bot")   # every later call raises FrozenAgentError
# ... incident handled ...
unfreeze("book-bot")
```

No extra service. It lives in-process and takes effect on the next request.

## Next steps

Once I could see cost, PII, and tools, I wanted actual guardrails: cap spend, keep an SSN off the wire, and allow only `search_flights` and `book_flight`.

```python
from openai import OpenAI
from tealtiger import TealGuard, observe
from tealtiger.policy import per_role

guard = TealGuard(
    depth="standard",
    guardrails={"pre": {"pii": True, "secrets": True}},
)

policy = per_role(
    {
        "booking": {
            "allowed_tools": ["search_flights", "book_flight"],
            "max_cost_per_session": 2.00,
            "blocked_pii": ["ssn", "credit_card"],
        }
    },
    default_deny=True,
)

client = observe(
    OpenAI(api_key="sk-proj-YOUR_KEY_HERE"),
    agent_id="book-bot",
    role="booking",
    guard=guard,
)
```

That is no longer a five-minute change — it is the graduation path. Start with `observe()`, add policies when you know what to block.

Docs for the rest:

- [Zero-config quickstart](https://docs.tealtiger.ai/quickstart)
- [`observe()` API](https://docs.tealtiger.ai/api/observe)
- [Policy authoring](https://docs.tealtiger.ai/policies)
- [Local dashboard](https://docs.tealtiger.ai/dashboard) (`npx tealtiger dashboard`)

Wrap the client you already have. Look at the first session. Then write the policy.

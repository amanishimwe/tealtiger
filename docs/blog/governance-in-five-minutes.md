# How I added governance to my AI agent in 5 minutes
*By Alban Manishimwe*

Suppose you build a working AI agent to manage your summer trip. It books things, calls tools, and talks to `gpt-4o` like it owns your wallet. What it doesn't have is a way to show you what it cost, a kill switch when a retry loop starts burning money, or a record of which tools fired — including when someone pastes an SSN into the chat.

## The problem, in one afternoon

The agent starts as a normal OpenAI script. User says "book the flight," model picks a tool, tool runs. That's fine until:
- someone pastes `My SSN is 123-45-6789` into the chat
- a retry loop burns a few dollars before I notice
- `book_flight` fires when I only meant to search
- I have no record of what the model actually did

I needed cost on every call, a kill switch, a paper trail, and a way to see which tools fired. TealTiger's `observe()` helper does that in one line, and you keep calling `chat.completions.create` exactly the same way.

## Install

Python or TypeScript — pick the one your agent already uses.

```bash
pip install tealtiger
# or
npm install tealtiger
```

No dashboard account. No sidecar. No extra process. `OpenAI()` reads `OPENAI_API_KEY` from the environment.

## Wrap your LLM call

Here is the agent **before** governance. One client, one completion, a couple of tools.

```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": "Book me a flight to Lisbon. My SSN is 123-45-6789.",
    }],
    tools=[
        {"type": "function", "function": {
            "name": "search_flights",
            "parameters": {"type": "object", "properties": {}},
        }},
        {"type": "function", "function": {
            "name": "book_flight",
            "parameters": {"type": "object", "properties": {}},
        }},
    ],
)
```

Here is the same agent **after**. Three lines of governance: import `observe`, wrap the client, print the cost so you can see it working.

```python
from openai import OpenAI
from tealtiger import observe

client = observe(OpenAI(), agent_id="book-bot")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": "Book me a flight to Lisbon. My SSN is 123-45-6789.",
    }],
    tools=[
        {"type": "function", "function": {
            "name": "search_flights",
            "parameters": {"type": "object", "properties": {}},
        }},
        {"type": "function", "function": {
            "name": "book_flight",
            "parameters": {"type": "object", "properties": {}},
        }},
    ],
)

cost = client.get_cost()
print(f"Session cost: ${cost.session:.4f}")
print(f"Last request: ${cost.last_request:.4f}")
print(f"Total requests: {cost.request_count}")
```

`client` still looks like OpenAI. Same methods, same parameters, same response object. The wrap is a transparent proxy — it intercepts the call, records cost / PII / tools, then lets the request through.

TypeScript is the same shape:

```typescript
import OpenAI from "openai";
import { observe } from "tealtiger";

const client = observe(new OpenAI(), { agentId: "book-bot" });
```

## What you get immediately

Running that wrapped script once — one `create()` call — printed this:

```text
$ python book_agent.py

Session cost: $0.0013
Last request: $0.0013
Total requests: 1
```

Behind those three numbers, the audit trail for that request now has a correlation ID, a PII finding (`SSN`, count 1), and a `search_flights` tool call from the model response. Observe mode does not print that trail to stdout. You read it from the audit log; `get_cost()` is what the script prints.

Three things started working with no policy file:

**Cost tracking.** Every request is priced from token usage. `get_cost()` gives session total, last request, and count. If the agent loops, I see the burn in the session instead of on next month’s invoice.

**PII detection.** Emails, phones, SSNs, and card numbers are scanned on the way in and out. In observe mode this is report-only: the finding goes to the audit log, the request still runs, and the SSN still reaches the model. What does not get stored in plaintext is the audit record — TealTiger hashes content by default.

**Audit trail.** Each request, tool call, PII finding, and response is linked by a correlation ID. When something weird happens next week, I can answer “what did book-bot do?” without grepping application logs.

Observe mode does **not** block tools. It logs the ones the model actually called — which, with OpenAI function calling, means tools you put in `tools`. That is the point of the first five minutes: see the agent, then decide what to forbid. Blocking `book_flight` (or anything else) is the next step, not this one.

There is also a kill switch if the loop is already running:

```python
from tealtiger import freeze, unfreeze, FrozenAgentError

freeze("book-bot")   # every later call raises FrozenAgentError
# ... incident handled ...
unfreeze("book-bot")
```

No extra service. It lives in-process and takes effect on the next request. In-flight calls are not cancelled.

## Next steps

Once I could see cost, PII, and tools, I wanted actual guardrails: cap spend, keep an SSN off the wire, and allow only `search_flights` and `book_flight`. TealGuard defaults to `ENFORCE`, so this is the step that starts blocking.

```python
from openai import OpenAI
from tealtiger import TealGuard, observe
from tealtiger.policy import per_role

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

guard = TealGuard(
    depth="standard",
    mode="ENFORCE",
    guardrails={"pre": {"pii": True, "secrets": True}},
    policy=policy,
)

client = observe(
    OpenAI(),
    agent_id="book-bot",
    role="booking",
    guard=guard,
)
```

Docs for the rest:

- [Zero-config observe quickstart](https://docs.tealtiger.ai/cookbook/observe-quickstart)
- [`observe()` API](https://docs.tealtiger.ai/api-reference/python/observe)
- [Role-based policies](https://docs.tealtiger.ai/concepts/role-based-governance)
- [Local dashboard](https://docs.tealtiger.ai/dashboard) (`npx tealtiger dashboard`)

Wrap the client you already have. Look at the first session. Then write the policy.

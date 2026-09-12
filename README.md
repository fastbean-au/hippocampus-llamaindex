# llama-index-memory-hippocampus

A [LlamaIndex](https://docs.llamaindex.ai) long-term memory block backed by
[Hippocampus](https://github.com/fastbean-au/hippocampus) — a memory service that stores what
matters and forgets what stops mattering.

```sh
pip install llama-index-memory-hippocampus
```

The import name is `llama_index.memory.hippocampus`, following LlamaIndex's convention for an
integration package.

## Using it

```python
from hippocampus import Hippocampus
from llama_index.memory.hippocampus import hippocampus_memory

client = Hippocampus("localhost:50051")
memory = hippocampus_memory(client, session_id="support-session-1", group="support-bot")

response = await agent.run("what did we decide about the rollback?", memory=memory)
```

`hippocampus_memory` is a convenience over the block itself, which composes with any other:

```python
from llama_index.core.memory import Memory
from llama_index.memory.hippocampus import HippocampusMemoryBlock

memory = Memory.from_defaults(
    session_id="support-session-1",
    memory_blocks=[
        HippocampusMemoryBlock(client=client, group="support-bot", limit=8),
    ],
)
```

The client is passed in already connected rather than built from an address, so this package does
not restate the connection options (token, TLS, private CA, client certificate, deadlines) that
`Hippocampus` already documents. The block does not own the channel and never closes it.

## The loop

Retrieval **recalls** what it returns, and a recall is a write: it resets the memory's decay clock
and raises its effective significance (see [Consolidation](https://github.com/fastbean-au/hippocampus/blob/main/docs/consolidation.md)). So a fact the
agent keeps reaching for is reinforced by the act of being used, and one it never retrieves decays
out on its own.

That is the entire retention policy. There is no eviction, no cap and no TTL in the adapter,
because the store already has all three and they are driven by what the agent actually did.

```text
  agent turn ──> block writes a memory ──> significance decays with age
                                                    │
  agent question ──> block searches ──> returns ──> recall resets the clock, raises significance
                                                    │
                                          never returned ──> consolidation forgets it
```

Three consequences follow, and all three are the product rather than faults:

- **Memories disappear.** A memory this block wrote can stop existing at any time. Nothing in the
  adapter treats that as an error.
- **`reinforce=False` breaks the loop**, leaving a store that forgets precisely the memories the
  agent has been using. It exists for a read-only observer, not for tuning.
- **Insignificance is not a failure.** A message below the deployment's
  `memory.minimumSignificance` is quietly dropped: the write succeeds with an empty id. The adapter
  logs it at debug and never raises.

## Why a forgetting store fits this slot

Retrieval **recalls** what it returns. A recall in Hippocampus is a write: it resets the memory's
decay clock and raises its effective significance. So a fact the agent keeps reaching for is
reinforced by being used, and one it never retrieves decays out of the store on its own.

That is the entire retention policy, and it is why there is no eviction, no cap and no TTL in this
package. The consequences are worth stating plainly:

- **Memories disappear.** A consolidation cycle deletes what has stopped mattering, so a memory
  this block wrote can stop existing at any time. Nothing here treats that as an error, and a
  long-term memory block is the one slot where nothing breaks when it happens — facts folded into
  a prompt by relevance are _supposed_ to thin out.
- **`reinforce=False` breaks the loop.** It leaves a store that forgets precisely the memories the
  agent has been using. It exists for a read-only observer, not for tuning.
- **Insignificance is not a failure.** A message below the deployment's minimum significance is
  quietly dropped: the write succeeds with an empty id. It is logged at debug and never raised.

## What gets stored

One memory per chat message, at a significance taken from the role:

| Role            | Default significance | Stored |
| --------------- | -------------------- | ------ |
| `user`          | 50                   | yes    |
| `assistant`     | 40                   | yes    |
| everything else | —                    | no     |

The `significance` map is the selector as well as the ranking, so a role absent from it is not
stored at all — adding `"tool": 30` is what starts storing tool output. System messages are absent
deliberately: a system prompt is configuration resent on every call, not something the agent
learned.

`significance_fn` overrides the map per message and may return `None` to skip one, which is how a
caller stores only what it judges worth keeping:

```python
HippocampusMemoryBlock(
    client=client,
    significance_fn=lambda message: 70 if "decision:" in message.content else None,
)
```

The role and the session id are recorded as **metadata**, never folded into the body. A body is
what the content index tokenises, so wrapping it in `<message role='user'>` would put `message`,
`role` and `user` into the index for every memory — three terms that match everything and rank
nothing. The role is restored as a prefix when the memory is rendered back into a prompt.

## Retrieval

The query is the last `retrieval_context_window` messages joined together, and `limit` memories
come back ranked by relevance blended with the store's own significance and recall count.

`mode` is resolved **once**, at first retrieval, against what `who_am_i()` says the deployment can
serve — richest first, so an instance that gains an embedding model starts doing hybrid retrieval
with no code change. Asking rather than trying is the point: a mode with no backend is refused per
call, so a block configured for one would fail every retrieval for the life of the process while
looking like a retrieval bug. Naming a mode the deployment cannot serve raises once, at first use,
naming what it does serve.

Retrieval is **not** scoped to the current session by default. Remembering across conversations is
what a long-term block is for; `scope_to_session=True` restricts it to the session at hand.

## How a composed `Memory` feeds the block

Worth knowing before concluding the adapter is broken: `Memory` does **not** hand every turn to its
memory blocks. A turn goes into the short-term buffer, and only reaches a block when that buffer
overflows its token limit and waterfalls the oldest messages out. A short conversation therefore
writes nothing.

That is the right shape here — the store receives what the conversation has moved past — but if you
want a turn written as it happens, call the block directly:

```python
block = HippocampusMemoryBlock(client=client, group="support-bot")

await block.aput([user_message, assistant_message], session_id="support-session-1")
```

`accept_short_term_memory=False` (a field of the base class) turns the waterfall off entirely,
leaving the direct call as the only way in.

Lowering `token_limit` is how you make the waterfall reach the store sooner, and
`hippocampus_memory` lowers the flush size with it. That is not a preference: `Memory`'s own default
flush size is 10% of the _default_ limit rather than of the one you passed, and it must stay
comfortably under `token_limit * chat_history_token_ratio` — so lowering the limit on its own is
rejected for a field you never set, with a message naming neither.

## Truncation

When the composed memory exceeds its token limit, this block drops retrieved memories from the end
rather than discarding itself wholesale, which is the base class's behaviour. Results arrive
ranked, so the end costs least. Give the block a non-zero `priority` for that to be reachable —
`priority=0` means never truncate.

## What the adapter is allowed to do

`Hippocampus` exposes every RPC the contract declares, `purge` and `clear` among them. The block
holds a `MemoryClient` instead — a `runtime_checkable` protocol naming exactly four calls:

| Call              | Why                                            |
| ----------------- | ---------------------------------------------- |
| `who_am_i`        | resolve the search mode once                   |
| `store_memories`  | write a batch of turns                         |
| `store_memory`    | the fallback for a service with no batch write |
| `search_memories` | retrieve, and reinforce what is retrieved      |

That list is the only thing standing between an agent's memory and the destructive half of the
surface, which is the same line the [event-source bridges](https://github.com/fastbean-au/hippocampus/blob/main/docs/eventsource.md) draw with their own
client interface — and a test holds it to exactly four, so growing it is a decision rather than an
import. It is structural, so a real `Hippocampus` satisfies it as it is, and an application wrapping
the client for its own metrics, retries or rate limiting can pass that instead.

## Async

Every call into the service is blocking gRPC run through `asyncio.to_thread`. The published client
is synchronous, and doing that in one place here is better than maintaining a second, async client
in step with the contract.

## Connection, authentication and TLS

The client is passed in **already connected** rather than built from an address here, so this
package does not restate the ten connection parameters that
[`Hippocampus`](https://github.com/fastbean-au/hippocampus/blob/main/docs/python.md#authentication-and-tls) already documents — a second copy of them is a
second thing to keep current. The block does not own the channel and never closes it.

```python
client = Hippocampus(
    "hippocampus.internal:50051",
    token=os.environ["HIPPOCAMPUS_TOKEN"],
    tls=True,
)
```

The token needs **writer** tier: the block stores memories, and a reader's retrieval does not
reinforce unless the deployment sets `auth.readerRecallReinforces`
([Authorisation](https://github.com/fastbean-au/hippocampus/blob/main/docs/configuration.md#authorisation)), which would quietly break the loop above.

## Development

The package depends on `hippocampus-client`, which is **not on PyPI** — it is built and attached to
every Hippocampus release instead. `SERVICE_VERSION` names the release CI installs it from. From a
clone:

```sh
version=$(cat SERVICE_VERSION)
pip install "https://github.com/fastbean-au/hippocampus/releases/download/${version}/hippocampus_client-${version#v}-py3-none-any.whl"
pip install -e '.[dev]'
pip install llama-index-core pytest pytest-asyncio
python -m pytest
```

That wheel is version-stamped, so it satisfies the floor in `pyproject.toml` and the adapter's own
dependency resolution is exercised rather than bypassed. Installing the client from source instead
does not work: its `_version.py` is a placeholder stamped only at release, so a source build reports
`0.0.0.dev0` and pip refuses it against the floor — which is why the monorepo's CI had to pass
`--no-deps`, and why this repository does not.

The tests drive the block against a fake client and need no service.

`SERVICE_VERSION` is raised by `.github/workflows/contract-bump.yaml` when the service cuts a
release, which opens a pull request carrying the test result against that client. This adapter never
touches the contract directly — it names four client methods in a `MemoryClient` protocol — so that
pin is how a moved method gets noticed.

## Documentation

- [Hippocampus](https://github.com/fastbean-au/hippocampus) — the service this adapter stores into
- [docs/python.md](https://github.com/fastbean-au/hippocampus/blob/main/docs/python.md) — the client
  underneath it
- [docs/consolidation.md](https://github.com/fastbean-au/hippocampus/blob/main/docs/consolidation.md)
  — how the store decides what to forget, which is what this slot is built around

## Versioning

The version line **continues** the service's rather than restarting: the distributions were built
and attached to every service release at the service version before this repository existed, so a
restart at `0.1.0` would be older than what is already published. The first release cut from here is
`v0.48.0`, and the line is free to diverge after that — it tracks `llama-index-core`, not the
service.

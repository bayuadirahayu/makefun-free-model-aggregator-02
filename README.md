# free-model-aggregator-02 — live free-model aggregator

One server, two client wires, real free models behind every answer.

`main.ts` (Node 22, standard library only) listens on `0.0.0.0` at a free port,
registers with the env, and answers `POST /v1/chat/completions` (OpenAI wire) and
`POST /v1/messages` (Anthropic wire) in JSON and in streaming, plus
`GET /v1/models` in the shape of the auth header it got. Every completion comes
from a real free model dialled through the grader's gateway on Chat Completions;
an Anthropic-shaped request is translated there and back. The server never
answers from itself.

## Model choice

For each prompt, the candidates are the brief's free models that meet the
prompt's requirements (tools, vision, JSON mode, context), ranked by the
brief's `rating_milli`, highest first, ties broken by id — a pure function of
the brief and the request, so the choice is deterministic. On this trial's
table that means the highest-rated model overall (`thinkingmachines/inkling-2-small`,
1475) serves every prompt it can, and the highest-rated JSON-capable model
(`nvidia/nemotron-3.5-super-vl`, 1303) serves the JSON-mode prompts.

## Failover

On a gateway 429 or 5xx the same model is retried once, then the next-best
candidate that meets the prompt serves it (at most four attempts per prompt).
Every refusal is reported in the `x-makefun-upstream-failures` header so the
failover column can see it. A 400 that is not a rate limit stops the attempts:
it is the request's own fault, not the model's.

## Catalog

`node main.ts --list` prints the catalog document — exactly the fourteen
dialable free ids, once each — byte for byte as the register body carries it.

## Verification

Verified locally against a mock env and mock gateway in `/tmp` (never in the
repo) driving the practice trial-0 prompts: registration accepted with the
catalog, both `GET /v1/models` lists hold exactly the fourteen ids, all six
prompts answered validly (token echoed, correct wire, correct model field,
usage present, valid stop reason), streaming event order correct, `--list`
output byte-identical to the registered catalog, server exits 0 when work says
done, and a gateway 429 on the top model is recovered via retry/failover with
the refusal reported in the trail header. A live grade against the real
gateway needs a real OpenRouter key, which this build does not wait on.

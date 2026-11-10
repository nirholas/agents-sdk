---
name: three-ws-agents-sdk
description: Use the @three-ws/agents TypeScript SDK to create three.ws agents, chat with them, run goals with budgets, and quote swaps from agent wallets. Use when code needs to call the three.ws v1 Agents API.
---

# three.ws Agents SDK

`@three-ws/agents` is a zero-dependency TypeScript client for the three.ws v1 Agents API (`https://three.ws/api/v1`). It runs anywhere with `fetch` (Node 20+, Deno, Bun, browsers).

## Install and authenticate

```bash
npm install @three-ws/agents
export THREE_WS_API_KEY=sk_live_...   # create with `npx three-ws setup` or at https://three.ws/dashboard/api
```

Model calls need a key with the `inference` scope; agent writes need `agents:write`.

## Core calls

```ts
import { ThreeAgents } from "@three-ws/agents";

const client = new ThreeAgents({ apiKey: process.env.THREE_WS_API_KEY! });

const models = await client.getModels();          // GET /models, public catalog
const free = await client.getFreeTierStatus();    // GET /me/free-tier

const agent = await client.createAgent({ name: "Momentum Alpha", strategy: "momentum" });
await client.startAgent(agent.id);

const reply = await client.sendMessage(agent.id, { message: "What is trending right now?" });
console.log(reply.content, reply.costCredits, reply.signatures);
```

Strategy presets: `momentum`, `sniper`, `defi-yield`, `macro-hedge`, `monitor-exit`, `conservative`. A preset fills skills, persona and default automations you omit.

## Goals (runs)

`createRun(agentId, { goal, budgetCreditsUsd?, budgetUsd?, maxSteps?, toolsAllowed?, schedule? })` hands the agent a goal with a credit budget, an on-chain budget and a step limit (1 to 60). Follow it with:

```ts
const run = await client.createRun(agent.id, { goal: "Summarize my wallet activity", maxSteps: 10, budgetCreditsUsd: 0.5 });
for await (const ev of client.streamRun(run.id)) console.log(ev.event, ev.data);  // SSE, resumes on drop
const steps = await client.getRunSteps(run.id);
```

Also: `getRun`, `listRuns`, `updateRun`, `cancelRun`.

## Other groups

- Skills: `listSkills`, `listCommunitySkills({ q, tag, author, limit })`, `importCommunitySkill`, custom skill CRUD.
- Automations: `createAutomation`, `listAutomations`, `deleteAutomation` (triggers: `price_threshold`, `schedule`, `balance_below`, `tip_received`).
- Intelligence: `getPrice(mint)`, `getTopMovers`, `getIndicators`, `getSignals`, `getAnomalies`, `getMacro`.
- Wallet and swap: `getWallet`, `getWalletHistory`, `transferQuote`, `transfer`, `swapQuote`, `swapExecute`.
- Pagination: `client.paginate((cursor) => client.listAgents({ cursor }))` walks every page.

## Money safety

`transfer` and `setExternalWallet` throw `ConfirmationRequiredError` before any request unless `confirm` is literally `true`. Call the preview method the error names (`transferQuote`, `getWallet`), show recipient, amount and token to the user, and pass `confirm: true` only after the user explicitly agrees. `swapExecute` from an agent wallet also needs `confirm: true`. Never move funds on your own initiative.

## Errors

All failures extend `ThreeAgentsError` (`status`, `code`, `requestId`): `AuthenticationError` (401), `PermissionError` (403), `NotFoundError`, `ValidationError`, `RateLimitError`, `InsufficientFundsError`, `ConfirmationRequiredError`, `ServerError`, `TimeoutError`, `ConnectionError`. Quote `requestId` when reporting a problem. Non-GET requests carry an `Idempotency-Key` reused across retries, so a retried create acts once.

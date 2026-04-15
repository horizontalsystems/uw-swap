# uw-swap

AI agent skills for [swap.unstoppable.money](https://swap.unstoppable.money) — a cross-chain cryptocurrency swap aggregator supporting THORChain, Mayachain, 1Inch, and P2P providers.

## Install

```bash
# skills.sh
npx skills add horizontalsystems/uw-swap

# clawhub
npx clawhub@latest install uswap
```

## Setup

Set these environment variables before using the skill:

```
USWAP_BASE_URL=https://swap-api.unstoppable.money/agent
USWAP_AGENT_KEY=<your-agent-key>
```

Register once to get your `USWAP_AGENT_KEY`:

```bash
curl -X POST $USWAP_BASE_URL/register \
  -H "Content-Type: application/json" \
  -d '{"name": "my-agent"}'
```

## Skills

| Skill | Description |
|---|---|
| `uswap` | Get quotes, execute swaps, track status across 10+ providers and 30+ chains |

## Supported Providers

THORChain · Mayachain · 1Inch · Barter · NEAR · LetsExchange · StealthEx · Quickex · Swapuz · Exolix

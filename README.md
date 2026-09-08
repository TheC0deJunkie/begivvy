# BeGivvy

WhatsApp-native birthday reminders and gifting for South Africa.

![screenshot](docs/screenshot.jpg)

Most birthday apps are a calendar with notifications bolted on. BeGivvy is built the other way
round: the thing being modelled is the **social graph**, who is connected to who and how, and a
birthday is just an event that graph produces. That difference is why the product can suggest a
gift at all, and why "friends of friends" and circles work the way they do.

The whole thing runs inside a WhatsApp thread. There's a web app too, but nobody has to open it.

## What it does

- **Onboarding in chat** — a numbered-menu flow that works without interactive-message approval,
  handles isiZulu and English names, and never traps someone in a loop with no way back.
- **Circles** — group the people you actually buy for. Family, work, church. Members can be added
  by number, by invite link, or by referral.
- **Friends of friends** — the graph surfaces people adjacent to your circles, with a consent step
  before anyone is contacted.
- **Multi-timezone reminders** — the scheduler works out each recipient's local morning and staggers
  the send window so a thousand reminders don't land in the same second.
- **24-hour window handling** — outside WhatsApp's customer-service window, sends fall back to
  pre-approved templates instead of silently failing.
- **Wishlists** — a person writes what they want in plain language; the parser merges it into their
  existing list rather than duplicating, and long lists render in chunks so WhatsApp doesn't truncate.
- **Ranked gift suggestions** — a scorer reads the wishlist, inferred taste and catalogue products,
  and returns scored suggestions rather than a random pick.
- **A guessing game** — a quiz about the birthday person that doubles as taste inference.
- **Admin console** — message log, delivery log, cron runs, user lookup and manual sends.

## Architecture

```
Twilio / Meta webhook
        │
        ▼
   16-step engine          signature check → parse → user lookup → load state →
   (src/bot.ts)            session timeout → loop breaker → route → handler →
        │                  persist → send → log delivery
        ▼
    handlers/              one file per conversational surface, each with a test
    services/              birthday maths, gift ranking, Gemini calls, delivery
    adapters/              twilio.ts | meta.ts | prisma-store.ts | memory-store.ts
```

The provider is a swappable adapter, so the same bot runs on Twilio or the Meta Cloud API by
changing one environment variable. Storage is the same idea: `memory-store` for tests,
`prisma-store` for everything else.

Cron work (reminder sends, prune nudges, keep-alive) runs through a single serverless-safe handler
rather than a long-lived process, because Vercel functions don't stay warm.

## Stack

| | |
| --- | --- |
| Runtime | Node 20, TypeScript (strict, ESM), Express |
| Data | Prisma + Postgres — SQLite in dev, Neon in production |
| Messaging | Twilio WhatsApp / Meta WhatsApp Cloud API, via Kapso |
| AI | Google Gemini for wishlist parsing and gift reasoning |
| Frontend | Vite + React + Tailwind + shadcn/ui, deployed separately |
| Scheduling | Croner, driven by Vercel Cron |
| Testing | Vitest, plus Supertest against the webhook routes |
| Monitoring | Sentry (no-op unless `SENTRY_DSN` is set) |

## Notes on the conversational design

A few rules the code enforces, each of which came from watching someone get stuck:

- Every prompt accepts `back`, and `back` from an un-onboarded state never lands on the main menu.
- Every list has a `Skip`. A flow with no escape hatch is a flow people abandon.
- Birth years in the future are rejected with copy that says why, not a generic error.
- Numbers with a `00` international prefix are normalised rather than rejected.
- The fallback handler never converts idle chit-chat into a saved contact.

## Status

In build, running in production.

---

<sub>Source is private — this repo is the write-up. [Shaun Madondo](https://github.com/TheC0deJunkie) · Durban, KwaZulu-Natal.</sub>

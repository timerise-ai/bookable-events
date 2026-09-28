---
prompts:
  - prompt: "Add events to this Next.js app on Postgres: limited capacity, paid tickets through Stripe Checkout, staff check-in at the door, and refunds when we cancel an event."
    stack: Postgres
  - prompt: Our free workshops have a no-show problem. Put a card hold on each registration that is released when the guest checks in and captured when they don't show.
    stack: Firestore
  - prompt: "Audit our event registration module: we have seen double refunds and overbooked events."
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/bookable-events) on timerise.ai.

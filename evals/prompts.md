---
prompts:
  - prompt: "Build a browser extension our back-office staff install with consent. It records what they do in the shop admin and carrier portals as DOM events, never screenshots, and sends them to our Next.js app on Supabase."
    stack: Supabase
  - prompt: "From the captured events, write an SOP for each role and rank the manual tasks worth automating by the time they take."
    stack: Supabase
  - prompt: "Make sure the capture never stores card numbers or customer emails, and that staff can pause it or exclude a domain."
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/ecommerce-process-mining) on timerise.ai.

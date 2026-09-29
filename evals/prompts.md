---
prompts:
  - prompt: "Build digital signage for our venue on Supabase: a media library, a playlist per screen, pairing a TV by PIN, and a fullscreen player page the TV opens."
    stack: Supabase
  - prompt: Our lobby screens should loop images and videos from Firestore, and a screen mounted in portrait must show the media the right way up.
    stack: Firestore
  - prompt: Our TV player freezes on a video and never recovers. Add stall recovery and a health signal so we can see which screen is down.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/digital-signage) on timerise.ai.

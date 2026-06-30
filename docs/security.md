# Security Architecture &amp; Reasoning

This is the *why* behind the security decisions in this stack — written as reasoning, not a checklist. If you've read the [README](../README.md) you've seen the topology; this document is about the thinking: where the real boundaries are, what I'm actually defending against, and the one mistake I caught and fixed.

The short version: this box runs a fleet of services **and** a self-hosted autonomous AI agent that can execute code, browse, and search. The security model is ordinary defense-in-depth on the host, plus one deliberate containment decision for the agent. The interesting part is choosing boundaries that hold up against a thing actively trying to get around them.

---

## The core idea: an infrastructure boundary beats a soft gate

The single principle this whole design rests on:

> An autonomous agent with tool use will try **multiple routes around a soft "are you sure?" block.** The boundary that actually holds is the one it can't reason its way past.

The agent ships with an in-app approve/deny gate — but that gate is part of the agent's *native* UI, and it isn't even reachable when the agent is driven through its API (which is how the chat interface talks to it). So a "soft" gate is doubly weak here: it can be routed around, and on the path I actually use, it isn't in the loop at all.

A **network** boundary has neither weakness. If there's no route to a service, there's nothing to negotiate, no prompt to talk past, no alternate path to find. That's why the containment in this stack is done at the infrastructure layer — Docker networking — and not with application-level permission prompts.

This is the lesson I'd most want a reviewer to take from the project: I didn't just turn on a safety toggle and trust it. I reasoned about whether the toggle could be bypassed, decided it could, and moved the boundary to somewhere it couldn't.

---

## Threat model — what I'm actually defending against

Being explicit about scope, because "secure" means nothing without it:

- **An autonomous agent reaching data it has no business touching.** It legitimately needs to chat and to search. It does **not** need the database, the notes app, the budget data, or the host. The goal is to make "needs" and "can reach" the same set.
- **Automated attacks against SSH.** The internet scans every public host constantly; credential brute-forcing is the default background noise.
- **Accidental secret exposure** — especially dangerous here because I'm publishing config *publicly* as a portfolio piece. One committed secret undoes everything.
- **Blast radius if something does go wrong.** If one service or credential is compromised, how far does it reach?

**Out of scope / honest limits:** this is a personal, self-directed project on a single host, not a formally audited or production-grade environment. There's no HA, no centralized secrets manager, no SIEM. I'm early-career and I represent the level honestly — the value here is the *reasoning and the instincts*, not enterprise completeness.

---

## Network isolation: the agent-containment story (the centerpiece)

This is the decision I'm proudest of, mistake and all.

**The setup.** The agent (Hermes, a self-hosted autonomous agent with persistent memory and tool use) needs exactly two things from the rest of the stack: the chat front-end (to talk to) and the private metasearch service (to search). Everything runs as Docker Compose services.

**The mistake I caught.** In the first iteration, the agent was sitting on the **shared application network** alongside everything else. That meant it had a **network path to the database port.** Not something I intended — just the default outcome of dropping it on the same network as the apps.

**Reasoning about blast radius (instead of just panicking and deleting things).** I stopped and worked out what that path actually exposed:

- The database **password** was never readable by the agent. It lives in the database/Compose environment — not on the agent's filesystem, not in anything its tools could read. So the credentials themselves weren't sitting there for the taking.
- What *was* exposed was the open **port** — a reachable network path to a data store from an autonomous code-executing agent.

So the realistic exposure was narrower than "the agent can read my database." But that distinction doesn't earn the path a pass. **Least privilege says an autonomous agent shouldn't have a route to a data store it has no reason to touch — full stop — regardless of whether I currently believe the credentials are safe.** Today's "it's probably fine" is tomorrow's incident. Close it.

**The re-architecture.** I moved the agent onto a dedicated, isolated network (`hermes-net`):

- The chat front-end and the metasearch service join **both** networks — they're the only two things the agent legitimately needs — so they bridge between the app stack and the agent's network.
- The agent sits on its isolated network **only.** It can reach exactly those two services and nothing else: no database, no notes, no budget data, no other container.
- The posture is **default-closed.** Granting the agent access to a new service is a deliberate, explicit, one-service-at-a-time action (`docker network connect hermes-net <service>`). The database is intentionally never on that list. The default is "no route," and every exception is a conscious decision.

**Why this is the boundary that holds** ties back to the core idea above: the agent can't talk its way around a missing network route, and the soft in-app gate it ships with isn't reliable on the path I use. Infrastructure, not the app-level prompt, is the wall.

---

## Defense in depth around the agent

Network isolation is the main boundary; these reduce what a problem could do even inside it.

- **Filesystem isolation.** The agent's terminal runs *inside its own container*, scoped to its own data directory. It can't see the host filesystem or other containers' files.
- **Terminal backend is in-container, not the Docker socket.** The agent could, in principle, be wired to a Docker backend — but that requires handing it the Docker socket, which is effectively host control. Deliberately not done. The agent runs commands in its own sandbox, not on the host.
- **The agent's API is internal-only.** Its OpenAI-compatible endpoint is on an internal port that is **never published** and has **no reverse-proxy route** — reachable only by containers on its own network, and it still requires an API key. It is never exposed to the public internet.
- **Remote access is outbound-only and allow-listed.** Phone access is via a messaging bot that uses **outbound polling** — no inbound ports, no firewall holes — and is locked to a **single allow-listed numeric user ID.** The "allow any user" flag is never set; without the allow-list the gateway denies everyone by default. Net effect: the agent is reachable only by me (the bot) and by the chat UI (which is itself behind its own login).
- **The image is pinned by digest.** Updates are deliberate, reviewed pulls — not whatever a moving `:latest` tag happens to be that day.
- **A hard cost ceiling.** A credit cap on the model provider is containment of a different kind: it bounds runaway token spend if a scheduled job or loop misbehaves.

---

## Host hardening (the foundation under everything)

Each of these is here for a reason, not because a guide said to.

### SSH: key-only, with a tested escape hatch

`ed25519` keys only. Password authentication disabled, and root password login disabled (`PermitRootLogin prohibit-password`), via a drop-in under `/etc/ssh/sshd_config.d/`. The reasoning: **passwords are the thing that gets brute-forced — so remove them as an attack surface entirely.** Key auth isn't guessable at scale the way a password is.

The part that mattered more than the config itself: **I set up and tested a recovery path — the provider's out-of-band recovery console — *before* disabling password login.** Locking out password auth without a verified way back in is exactly the kind of mistake that's only obvious *after* you've made it. Test the escape hatch first, then flip the switch.

### fail2ban: cheap, correct, mostly about noise

Even with key-only SSH, the auth logs fill with automated attempts the moment a host is public. fail2ban watches the SSH jail and temporarily bans repeat offenders (a small number of failed tries within a short window → a timed ban). Honest assessment: with passwords already disabled this is more about cutting log noise and load than it is a hard security gain — but it's cheap, it's correct, and it's the right default.

### ufw: default-deny, and verifying what's *actually* open

The firewall runs **default-deny** with explicit allows: SSH (rate-limited), and 80/443 for the web stack. Nothing else inbound.

The lesson that came out of this: I found **Docker's API ports (2375/2376) showing as open** — stray rules, with nothing actually listening behind them. I removed them. The real takeaway is bigger than two ports: **Docker manipulates the host firewall directly and can punch holes you didn't author.** Your `ufw` rules are not the whole story. So I verified what was *actually* exposed rather than trusting that my firewall config described reality. "Assume, then verify" is the habit; this is where it paid off.

### A backstop that doesn't depend on SSH

The provider's recovery console (root, credentials held offline) works regardless of SSH state. It's the lockout backstop — and the thing that made disabling password auth a safe decision rather than a risky one.

---

## Secrets: keeping them out of what's public

The constraint that shapes everything here: **the config in this repo is public, so no secret may live in version-controlled config.**

- **Where the agent's secrets live.** Its runtime secrets sit in a `.env` *inside its data volume*, read on startup — so the Compose service block for the agent stays completely secret-free. That separation is deliberate: the thing that gets committed and the thing that holds secrets are different files.
- **Where the rest live.** Other service secrets live in the Compose file **on the server**, which is never committed — see [`.gitignore`](../.gitignore), which physically blocks the real `docker-compose.yml`, the real proxy config, and `.env` files from ever being staged. That's a belt-and-suspenders layer on top of remembering not to commit them.
- **The redaction workflow.** The real config never leaves the server. Only sanitized `.example` copies are published, with every secret, the real domain, and the server IP replaced by a descriptive placeholder — followed by a `grep` scan to confirm nothing slipped through *before* anything is pushed. Publishing one live secret is the single mistake that undoes the whole exercise, so that scan has to come back clean.
- **Reasoning about *who can read what*.** Part of the work was simply mapping each secret to where it lives and what could read it — which is exactly how I established, in the network-isolation story above, that the database password wasn't reachable by the agent even while the port was. "Secrets management" isn't only generating strong values; it's knowing the read-path to each one.

---

## Single sign-on: one login, one deliberate exception

Most services sit behind single sign-on implemented as **reverse-proxy forward-auth**: a small reusable snippet sends each gated request to the auth service first, and one login sets a cookie scoped to the parent domain that covers every gated subdomain. I chose a stateless, tiny auth service over a heavier full-featured one — the lighter tool fit the threat model and the box's memory budget.

The **deliberate exception**: the AI chat front-end keeps its *own* login rather than sitting behind the shared SSO. That's intentional — it can be used or shared independently without that also handing over access to the budget, notes, and bookmarks behind the SSO wall. Different access surface, different gate.

---

## Honest scope note

This is a personal, self-directed project, and I'd put my level at junior / early-career. It's defense-in-depth done *thoughtfully*, but it isn't an audited or production-hardened environment, and it doesn't pretend to be. What I'd point to as the real signal is the reasoning on display here: identifying a boundary that wouldn't hold, thinking through actual blast radius rather than reacting, choosing an infrastructure control over a soft one, and verifying what was exposed instead of assuming the config told the truth. Those instincts are the transferable part — the specific tools are just where I practiced them.

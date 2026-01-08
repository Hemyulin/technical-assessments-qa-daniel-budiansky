# Product Quality Challenge – GLS Parcel Configuration and Checkout

## Quality lens

How can this flow realistically lose money, lose customer trust, or cause operational failure?

This write-up is intentionally not exhaustive. It captures how I reasoned under constraints: limited time, no backend access, incomplete domain knowledge, and a production system where I avoided real payments.

## Constraints and approach

- timebox: ~2–3 hours
- role: single QA
- access: black-box only (no code, no docs)
- safety: did not complete real payments

Approach used:

- fast end-to-end mapping first (configuration → cart → checkout)
- then risk-based deep dives where failures would be expensive (late validation, state transitions, eligibility rules, pricing clarity)
- notes captured during exploration, then grouped into risk areas

## System under test

- parcel configuration
  - https://www.gls-pakete.de/en/private-customers/parcel-shipping/parcel-configuration
- cart and checkout (reachable from the configuration flow)

## What I explored

High-level coverage:

- language switching (DE ↔ EN)
- destination changes and edge destinations
- parcel size and pricing changes
- guest vs logged-in journeys
- login/registration during the flow
- state continuity when editing, cloning, or changing configuration late
- address entry and validation differences by country
- responsiveness on desktop and mobile

## Key risks and findings (grouped)

### 1) State and continuity risks (highest concern)

**Question:** does the system preserve user intent across steps and identity transitions?

Explored scenarios:

- switch language mid-flow
- progress as guest deep into the flow
- log in during checkout
- create an account mid-flow and check whether the cart/config preserved
- log out and log back in
- finish flow on a different device
- clone parcels and edit one instance
- change destination late (checkout stage) and save

Why this matters:

- if users lose their state late, conversion drops immediately
- state bugs are expensive because they are intermittent and hard to reproduce
- state issues also lead to “I was shown X but got Y” trust loss

Primary worry:

- late-stage revalidation or identity changes causing silent resets, duplicated items, or mismatched prices

Observed behavior:

- adding parcels as a guest and then logging into an existing account replaces the current cart, causing guest-added items to disappear
- this behavior is explicitly warned in the UI, but still represents a significant conversion and trust risk when users log in late in the flow

### 2) Eligibility and destination rules (high concern)

**Question:** can users configure something that later gets rejected?

Explored scenarios:

- changing countries and regions
- assumption test: country selection implies territorial coverage
- attempted shipment to Ceuta (Spain) resulted in late warning and rejection

Why this matters:

- users assume selecting a country means “I can ship there”
- late discovery of exclusions causes frustration, drop-off, and support contact

Primary worry:

- eligibility rules surfaced too late, meaning users invest effort before being blocked

### 3) Pricing and configuration integrity (high concern)

**Question:** is the price stable, understandable, and enforceable?

Explored scenarios:

- price changes when destination/size changes
- currency behavior for non-euro destinations (example: Hungary)
- weight limits and size constraints (notably “up to 40kg” on all sizes felt counterintuitive)

Why this matters:

- pricing confusion leads to abandonment and disputes
- unclear constraints can create false expectations (user thinks “40kg is always ok”)

Primary worry:

- price shown earlier differs from price at checkout, or changes without a clear reason

### 4) Identity and data integrity (medium to high concern)

**Question:** are address and identity constraints validated early and correctly?

Explored scenarios:

- address book behavior for logged-in user
- email validation behavior
- country-specific address formats and required fields

Why this matters:

- bad address data creates failed deliveries, refunds, and manual operational handling
- identity edge cases often break late (existing email, partial registrations, “login vs register” confusion)

Primary worry:

- accepting invalid address formats, then failing later after payment or label purchase

### 5) UX trust and failure feedback (medium concern)

**Question:** when something goes wrong, does the system explain it clearly and preserve user confidence?

Explored scenarios:

- responsive behavior on different viewports
- ability to find support or “report a problem” paths while stuck

Why this matters:

- failures are inevitable, but unclear messaging increases abandonment and support load

Primary worry:

- error messages that do not explain what to do next, especially near payment

## Risk ranking (what worried me most, and why)

1. state continuity across transitions  
   Because it directly impacts conversion and produces hard-to-reproduce bugs. It is also the kind of thing that breaks when small improvements ship regularly.

2. eligibility rules discovered late  
   Because it violates user expectations and creates a “wasted time” feeling that kills trust.

3. pricing clarity and stability  
   Because it drives revenue and disputes. Small inconsistencies create outsized reputational damage.

## Minimal test ideas (what I would run first if I joined tomorrow)

These are not exhaustive test cases, but “confidence checks” aligned to the biggest risks:

State continuity checks:

- language switch at each step: configuration, cart, checkout
- login at each step and verify cart items, destination, size, and price remain consistent
- clone parcel → edit one → ensure only one changes and totals remain correct
- change destination late → confirm eligibility and price updates are consistent and clearly explained

Eligibility checks:

- “special territories” sampling (like Ceuta, islands, exclaves) across 2–3 countries
- invalid combinations (destination + service type + parcel size) to see whether validation is early and actionable

Pricing checks (current and future-facing):

Current behavior:

- Same configuration repeated twice (fresh session) yields the same total price.
- Changing one variable at a time (destination, parcel size, add-ons) updates the price in a predictable and explainable way.
- Prices remain consistent across configuration, cart, and checkout, with no surprise changes at the final step.

Forward-looking / scalability consideration:

- The pricing model appears EUR-only, which is appropriate for the current German consumer flow.
- If this flow is later reused for other origin countries (e.g. France) or localized consumer sites, pricing logic, rounding, and display rules will become a high-risk area.
- Early separation between “pricing calculation” and “price presentation” would reduce future regression risk when multi-country or multi-currency support is introduced.

Address and identity checks:

- validate required fields per country and confirm errors are specific
- existing email attempts during registration mid-checkout

## Decisions and trade-offs

What I focused on:

- end-to-end flow integrity where revenue and trust are at stake
- late-stage validation, state transitions, and rules that can block shipment

What I intentionally did not do:

- no real payments
- no exhaustive coverage of all destinations, parcel sizes, or services
- no deep accessibility audit
- no extensive visual QA beyond obvious responsiveness issues
- no full automation suite (timebox was better spent finding high-risk failure modes)

## If I had one more day

- create a small destination/eligibility matrix: a handful of “risky” destinations (territories, islands, exclaves) and verify where validation appears in the flow
- map the state model: what is the source of truth for cart items and configuration when language or identity changes?
- run a lightweight automation focused on a few high-value flows:
  - happy path to checkout until payment
  - login mid-flow
  - change destination late and assert price and eligibility updates
- produce a short “top 5 fixes” proposal focusing on earlier validation and clearer messaging (especially for exclusions like Ceuta)

## Use of tools and AI

I used ChatGPT as a thinking partner and structuring aid, not as a source of truth.

Process:

- I explored the product independently and captured raw observations, questions, and uncertainties.
- I used ChatGPT to help:
  - cluster notes into risk areas
  - challenge my own thinking to see where I might be wrong, biased, or overlooking something.
  - improve clarity and narrative flow

What I accepted:

- structure suggestions and clearer wording
- risk framing improvements

What I modified or rejected:

- anything that sounded like a factual claim about GLS rules without direct observation
- any “confident” assumptions that were not backed by what I saw in the UI
- content added purely for verbosity rather than to communicate a real risk or insight

Misleading or incorrect AI output:

- the main risk with AI here is inventing system rules (eligibility, pricing logic, service constraints). I treated those as hypotheses only and relied on direct exploration for conclusions.
- I deliberately formed my own observations and hypotheses before consulting AI, to avoid anchoring my judgement on generated suggestions.

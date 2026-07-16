# Product

## Register

product

## Platform

web

## Users

Retail swing and position traders who validate strategy ideas before risking money. For v1 the sole user is the builder; the future paying persona is a trader who shouldn't need to hand-write Pine Script or Python. Context of use: focused desk sessions — describe an idea, confirm what the system understood, judge whether the result is worth trading live.

## Product Purpose

Turn a plain-English trading strategy description into a trustworthy backtest: faithful schema translation with an explicit confirm-before-run recap, engine-computed metrics with zero added arithmetic, an interactive chart of trades and indicators, and an AI interpretation of strengths and flaws. Success is when the user would size a real position based on the output.

## Positioning

The only backtester where you describe the strategy in plain English and can verify — before it runs — that what executes is exactly what you meant.

## Brand Personality

Precise, calm, trustworthy. Quiet confidence: numbers lead, language stays plain, nothing hypes. The interface earns trust the same way the engine does — by showing its work.

## Anti-references

- Crypto-bro casino aesthetics: neon green/red gains porn, rockets, gamified streaks, confetti.
- Generic SaaS dashboard template: cream background, identical card grids, gradient CTAs, hero-metric tiles.

## Design Principles

1. **Trust is the interface.** Every number traces to the engine; every recap sentence traces to the schema. Design should expose that chain, never obscure it.
2. **Confirmation before consequence.** The recap step is the product's core ritual — give it weight, clarity, and an obvious edit path.
3. **Numbers first, narrative second.** Metrics and chart carry the page; AI interpretation supports, clearly labeled as interpretation.
4. **Calm under volatility.** Losses render as legibly as wins. No emotional color theatrics; state changes are quiet and precise.
5. **Plain language everywhere.** If a trader has to decode jargon, the translation promise is broken.

## Accessibility & Inclusion

WCAG 2.1 AA: body text contrast >= 4.5:1, large text >= 3:1, full keyboard navigation, visible focus states, `prefers-reduced-motion` alternatives for all animation. Charts must not encode meaning by color alone (shape/direction markers accompany win/loss color).

# Building block: Design System

How to design the visual system for a blueprint so the builder can construct it without guessing. The bar: a
blueprint a builder can hand to a designer who will produce something that looks like *this client's product* —
not generic AI output. Pairs with the `/ui-ux-pro-max` skill (use it in Phase 3 to compose palettes/pairings).

## Brandbook-first (non-negotiable, ask in Discovery)

Before designing anything visual, **ask the client whether they already have a brand**:

- **They have a brandbook / identity** (manual, logo, existing site, tokens): the blueprint's design system must
  **derive from it** — extract the real colors, fonts, tone. Set `Brand origin: existing`. **Never invent a new
  palette when an identity exists.** Identity preservation wins.
- **From scratch:** do NOT design blind. Ask where the design is going — references they like, desired feeling
  (sober/bold/warm/technical/luxurious/playful), audience, the physical scene of use (who, where, what light,
  what mood). Propose a direction, get the client to confirm it, then set `Brand origin: from-scratch` and write
  the confirmed direction into the blueprint.

## Register — pick one, it shapes everything

- **brand** — design IS the product (marketing, landing, campaign, portfolio). Bar: distinctiveness.
- **product** — design SERVES the product (app, admin, dashboard, tool). Bar: earned familiarity (a fluent
  Linear/Figma/Notion/Stripe user trusts it).

## The three dials (write them into the blueprint)

- **DESIGN_VARIANCE** — layout experimentation. Brand → higher; product → medium/low.
- **MOTION_INTENSITY** — motion depth. `low` = the builder's motion designer doesn't even engage.
- **VISUAL_DENSITY** — information density. Driven by register + audience.

## Filling section 7 well

- **Colors in OKLCH.** Verify contrast at design time conceptually: body ≥4.5:1. Pick a **color strategy**
  (restrained / committed / full-palette / drenched) *before* picking colors. Dark vs light is never a default —
  write the one-sentence physical scene that forces the answer.
- **Typography paired on a contrast axis** (serif+sans, geometric+humanist, or one family in weights). Never two
  similar sans. Display clamp() max ≤6rem; body line 65–75ch.
- **Motion** only if MOTION_INTENSITY ≥ med: signature animation, 100/300/500 timings, ease-out curves,
  `prefers-reduced-motion` mandatory, no bounce/elastic, no animating layout.

## The anti-slop bar (the single most important thing)

If someone could say "AI made this" without doubt, the design failed. Two altitudes:
- **First-order:** if the theme + palette are guessable from the category alone, it's the training-data reflex.
- **Second-order:** if guessable from category + obvious anti-reference ("AI tool that's not SaaS-cream →
  editorial-typographic"), it's the trap one tier deeper. Rework until neither is obvious.

> The 2026 saturated default is the **cream/sand/beige body background** (OKLCH L 0.84–0.97, C<0.06, hue 40–100).
> Token names like `--paper`, `--cream`, `--sand`, `--linen`, `--ivory` are the tell. "Warmth" lives in accent +
> typography + imagery, never in the body background.

**Banned outright:** side-stripe borders, gradient text, default glassmorphism, the hero-metric template,
identical/nested card grids, per-section uppercase eyebrows, reflexive 01/02/03 section markers, overflowing headings.

## Handoff to the builder

The builder's designers (Valentina = fundamentals, Darío = motion) read section 7 as their starting point and
expand it into `docs/DESIGN-BRIEF.md` + `docs/DESIGN.md`. The more committed and specific section 7 is, the less
the builder has to infer. A vague section 7 ("modern, clean, minimal") forces the builder to re-derive the
direction — which is exactly where generic output creeps back in. Be opinionated here.

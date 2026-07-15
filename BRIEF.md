# When Space Becomes Routine

## What is this?

This is a single-page interactive data story exploring the transformation of commercial spaceflight from rare government missions to a mature transportation network.

Rather than presenting a dashboard of metrics, the experience uses data, motion, and storytelling to help users understand one central idea:

**Commercial space travel is becoming routine, and that changes everything.**

The experience should feel more like an editorial story than a traditional dashboard.

Think Apple product page meets Bloomberg visual journalism.

The experience should leave users feeling that they have witnessed the beginning of a major shift in human transportation.

---

## User

**Primary Audience**

Anyone curious about the future of commercial spaceflight.

No aerospace knowledge should be required.

The experience should be approachable, visually engaging, and easy to understand while still feeling intelligent and data-driven.

By the end of the experience, users should walk away thinking:

> *"I hadn't realized we were already living through the beginning of another transportation revolution."*

---

## Experience Principles

The story should always take priority over the interface.

Every section should communicate one idea.

Motion and interaction should reinforce the narrative rather than become the focus of the experience.

The experience should feel calm, elegant, and cinematic rather than flashy.

The page should encourage users to continue scrolling naturally through curiosity rather than instruction.

Typography, whitespace, imagery, and motion should work together to create rhythm throughout the experience.

Information should feel effortless to understand.

The experience should prioritize simplicity over completeness. Showing fewer, more meaningful ideas is preferable to presenting every available data point.

---

## Narrative Structure

The experience unfolds as a single scrolling story composed of five chapters.

### Chapter 1 — Not Long Ago

Introduce a world where access to space was rare, expensive, and reserved for governments and astronauts.

Primary message:

**Space was extraordinary.**

---

### Chapter 2 — Everything Changed

Show how commercial innovation dramatically accelerated launch frequency while reducing the cost of reaching orbit.

Primary message:

**Commercial innovation fundamentally changed access to space.**

---

### Chapter 3 — We've Crossed a Threshold

Reveal how routine launches and growing passenger volume signal that commercial space transportation is no longer a future concept.

Primary message:

**The transformation is already happening.**

---

### Chapter 4 — Expanding Possibilities

Show how commercial spaceflight has evolved beyond exploration into a growing ecosystem supporting tourism, research, cargo, manufacturing, medical research, and infrastructure.

Primary message:

**Space is no longer a destination. It's becoming an economy.**

---

### Chapter 5 — What's Next?

Conclude with a quiet, reflective ending that encourages users to think about the broader implications of routine access to space.

Avoid making bold predictions.

Instead, leave users with a sense of possibility.

---

## Storytelling Approach

Each chapter should feel visually distinct while remaining part of the same experience.

The layout, imagery, typography, charts, and motion should evolve throughout the story to maintain curiosity and create a natural visual rhythm.

Avoid repeating the same composition in every chapter.

Each section should introduce a new visual moment that reinforces the idea being presented.

---

## Data

Generate a realistic but simplified local dataset at:

`src/data/space-story.json`

The data does not need to be historically accurate.

Instead, it should feel believable and support the narrative.

Include data representing:

- Launch frequency over time
- Average cost to reach orbit
- Number of people reaching orbit
- Commercial mission activity
- Future projections where appropriate

Commercial mission activity should include categories such as:

- Tourism
- Research
- Cargo
- Manufacturing
- Medical
- Infrastructure

Earlier years should be dominated by research and government-funded exploration missions

As time progresses, tourism, cargo, manufacturing, medical, and infrastructure missions should steadily increase, illustrating the expansion of commercial spaceflight.

The categories should evolve over time to reinforce the narrative that commercial spaceflight has expanded beyond exploration into a mature transportation ecosystem.

The purpose of the data is storytelling rather than precision.

---

## Interactions

The experience should unfold progressively as the user scrolls.

Each chapter should occupy approximately one viewport height on desktop.

Key statistics should animate into view when their section enters the viewport.

Charts should progressively build themselves as users scroll rather than appearing fully rendered.

Motion should be subtle, elegant, and purposeful.

Avoid unnecessary animations or decorative effects.

Scrolling should feel smooth and uninterrupted.

There should be no navigation menus, filters, or dashboard controls.

The scroll itself is the primary interaction.

---

## Responsive Behavior

The experience should work equally well on desktop and mobile devices.

Desktop should emphasize large visuals, generous whitespace, and immersive storytelling.

Mobile should preserve the same narrative while stacking content naturally for vertical scrolling.

Animations should remain smooth and readable across all screen sizes.

---

## Layout

The experience should use a centered content container with a consistent maximum width across all sections.

Typography, charts, and supporting content should remain within this container to optimize readability and create a premium editorial feel.

Large background imagery may extend the full width of the viewport, while all foreground content remains aligned to the central content area.

The layout should feel spacious, balanced, and intentional, with generous whitespace surrounding the content.

---


## Visual Style

- Dark theme
- Cinematic photography
- Large editorial typography
- Generous whitespace
- Minimal interface chrome
- Premium presentation quality
- Restrained color palette
- Blue accents inspired by Earth and space
- Charts should feel elegant rather than analytical
- Use restraint. Every visual element should earn its place.

Avoid:

- Dashboard cards
- Dense layouts
- Neon sci-fi aesthetics
- Excessive gradients
- Decorative animations
- Visual clutter

The experience should feel timeless, confident, and optimistic.

---

## Background Imagery

Each chapter should use a large, cinematic background image that supports the narrative of that section.

Images should span the full width of the viewport and establish the emotional tone before the supporting typography and data are introduced.

For the initial implementation, use simple placeholder images or neutral gradient blocks that clearly define where each future background image will appear.

Design the layout assuming these images will eventually be replaced with high-quality editorial photography.

Content should remain readable through subtle overlays or gradients placed above the imagery.

The experience should feel immersive without sacrificing legibility.

Background images should extend edge-to-edge, while all text, charts, and interactive elements remain within the central content container.

Background imagery should serve as storytelling, not decoration.

---

## Technical Requirements

- Vue 3
- TypeScript
- Vue Router
- Vuetify 3
- Chart.js via vue-chartjs
- Local JSON data only
- No API calls
- Responsive layout
- Single-page scrolling experience

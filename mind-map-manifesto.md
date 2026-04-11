# Mind Map Manifesto

This document defines how we want to design mind maps in this repo so they are easier to learn from, easier to review, and lower in cognitive load.

It complements the canonical builder docs in `excali-builder`.

- Use `excali-builder` docs for syntax, file format, anchors, and build behavior.
- Use this manifesto for information design, decomposition, and visual learning quality.
- If these ever conflict on parser behavior, `excali-builder` wins.

## Goal

The purpose of a mind map in this repo is not to capture every possible relationship in a topic.

The purpose is to break a complex subject into its most fundamental pillars, then break each pillar into its most useful sub-parts, so a learner can:

- understand the structure of the topic quickly
- reduce cognitive load
- retain the main ideas better
- use the diagram as a visual mental model

## What We Are Optimizing For

We are optimizing for:

- clarity over completeness
- hierarchy over graph density
- digestion over exhaustiveness
- teaching over analysis
- visual chunking over relationship richness

## Mind Map vs Concept Map

This repo should usually bias toward a learning-oriented **mind map**, not a dense **concept map**.

### Mind Map

- hierarchy-first
- few cross-links
- strong chunking
- small number of primary pillars
- designed for learning, recall, and orientation

### Concept Map

- relationship-first
- many cross-links
- useful for systems analysis
- good for showing non-hierarchical dependencies
- easier to overload visually

Default rule:

- Start with a mind map.
- Add concept-map style links only when they remove confusion.

## Core Principles

### 1. Start from one root question

Every map should answer one central question, such as:

- How does this system work?
- How do I use this tool effectively?
- What are the core parts of this topic?

If the root question is fuzzy, the whole map will drift.

### 2. Keep the first layer as small as the subject allows

The first ring of children under the root should represent the major pillars of the topic.

- Use as few pillars as possible while still covering the topic clearly.
- If the first layer feels crowded, the topic may not be grouped cleanly enough.
- If the first layer feels too thin, the topic may still be underdeveloped.

### 3. Make sibling nodes parallel

Children of the same parent should be the same kind of thing.

Good:

- pillars of a workflow
- categories of concepts
- stages of a process

Bad:

- mixing a workflow step, a warning, a tool, and a best practice under one parent

### 4. Parent-child must mean true decomposition

A child should be a genuine sub-topic, component, stage, or part of the parent.

Use this test:

- "X is part of Y"
- "X is a sub-topic of Y"
- "X is a step within Y"

If that sounds wrong, it should probably not be a child.

### 5. Use one organizing principle per branch

Each branch should decompose its topic using a single lens.

Examples:

- branch by process stages
- branch by conceptual parts
- branch by decision categories

Avoid switching lenses mid-branch.

### 6. Prefer short labels and low-text nodes

Node titles should usually be short phrases, not mini paragraphs.

The label should help scanning first.
The body text should clarify only when needed.

### 7. Prefer a clean tree before adding links

The map should make sense with only `parent_child` edges.

Only after the tree is solid should we add a small number of non-hierarchical links.

### 8. Cross-links must earn their place

Use cross-links only when the connection is not obvious from the hierarchy and materially helps understanding.

If removing a cross-link does not hurt comprehension, remove it.

### 9. Optimize for teaching, not exhaustiveness

Do not add nodes just because a fact is true.

Add nodes only if they help the learner:

- understand
- decide
- remember
- act

### 10. Cut aggressively

If 30% of the nodes could disappear with no loss of understanding, the map is too dense.

Good maps are edited down, not merely expanded.

## Edge Policy

For now, keep the edge vocabulary small:

- `parent_child`
- `related`
- `contrast`

### `parent_child`

Use for the main structure only.

This is the default and most important relationship.
It should carry almost all of the map's meaning.

### `related`

Use sparingly.

Only add `related` when a non-obvious connection helps the learner understand two concepts better.

### `contrast`

Use for common confusions, false alternatives, or important distinctions.

This is especially useful when learners are likely to mix up two concepts at first glance.

### Cross-link rule of thumb

If the map starts looking like a web instead of a tree, there are too many links.

## Node Type Policy

The current builder supports flexible node types, but semantically blurry categories add noise.

For early versions of a map, prefer a minimal set of node types.

Recommended defaults:

- `category`: root and major pillars
- `concept`: normal nodes
- `warning`: optional, only for pitfalls or common mistakes

Do not create extra node types unless they improve meaning clearly.

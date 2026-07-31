---
name: hierarchical-zoom-list
description: Use when you need to convert unstructured content (documents, plans, codebases, knowledge bases) into a compressed, zoomable hierarchical Markdown nested list for single-level-focus navigation. Triggered by requests to "organize hierarchically", "make a zoomable outline", "compress into nested list", "create zoom navigation structure", or when the user asks to restructure flat information into drill-down layers.
---

# Hierarchical Zoom List

Convert any content into a zoom-navigation-optimized hierarchical Markdown nested list.

## Core Rules

1. **One-sentence nodes** — each node is a compressed standalone information unit, never a paragraph
2. **3–7 children per parent** (Miller's Law) — if >7, introduce an intermediate grouping level
3. **Peer parity** — siblings must be at the same abstraction level and mutually exclusive
4. **Recurse to leaf** — stop only when a node is already the smallest independently understandable unit
5. **Parent as summary** — a parent's text must be a compressed summary of its children, never an unrelated title

## Output Format

Standard Markdown unordered list, `- ` prefixed, 2-space indent per level. One line per node. No extra prose.

## Execution Steps

1. Read the full input. Identify top-level components → 3–7 compressed phrases (level-1 nodes)
2. For each level-1 node, locate its source content, repeat step 1 → level-2 nodes
3. Recurse until nodes are simple enough to stand alone as leaves
4. Audit every level: are siblings mutually exclusive? Same abstraction level? Count ≤7?
5. Rebalance — merge over-split siblings or break up oversized groups
6. Output the nested list. No other text.

## Example

Input: *"The AI writing detection project has data preparation with human-written, AI-generated, and AI-polished samples; feature extraction covering phrase density, perplexity, burstiness, TTR, and style consistency; detection verification using tool selection and accuracy comparison; visualization analysis; and a CLI scoring tool."*

Output:

```
- AI Writing Detection Project
  - Data Preparation
    - Human-written samples
    - AI-generated samples
    - AI-polished samples
  - Feature Extraction
    - Set-phrase density
    - Perplexity
    - Sentence burstiness
    - TTR / MATTR
    - Style consistency
  - Detection Verification
    - Tool selection
    - Accuracy comparison
  - Visualization & Analysis
  - CLI Scoring Tool
```

## Guardrails

- Flat content stays flat — don't invent hierarchy where none exists
- Leaf nodes may have no children (that's correct behavior)
- This skill only produces the Markdown list structure; rendering/interaction is handled by the zoom-viewer companion

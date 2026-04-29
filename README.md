# LD Relay Advisor

An AI-first Streamlit application that guides LaunchDarkly clients through the Relay Proxy decision-making journey, from "do I need a Relay Proxy?" all the way to hardware sizing and deployment architecture.

## Problem

Every LaunchDarkly client goes through the same set of questions when evaluating the Relay Proxy:

1. Do I even need a Relay Proxy?
2. If yes, which mode: proxy or daemon?
3. What does my deployment look like?
4. How do I size it?
5. What about Redis? Do I need it?
6. What about security, caching, resilience?

Today this is a manual, SA-led process that takes multiple meetings, custom documents, and back-and-forth. The answers follow predictable patterns based on client infrastructure, but the process is repeated from scratch every time.

## Solution

A self-service, AI-powered app that:
- Walks clients through a structured Q&A flow
- Uses their answers to recommend architecture (proxy mode, daemon mode, or no relay needed)
- Generates sizing estimates based on official LD benchmarks
- Produces a downloadable PDF report tailored to their environment
- Answers follow-up questions using RAG over LD relay docs and codebase knowledge

## Status

**Planning phase.** No code yet. See `/docs` for full SDLC planning.

## Folder Structure

```
ld-relay-advisor/
  README.md                  # This file
  docs/
    prd/                     # Product Requirements Document
    hld/                     # High-Level Design
    dld/                     # Detailed Low-Level Design
    test-plans/              # Test cases and test scripts
  research/                  # Learnings from client engagements
  modules/                   # Module-level specs
```

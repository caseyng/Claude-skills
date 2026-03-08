# spec-contract-methodology

## The problem

LLMs are probabilistic. Ask one to build something complex without a precise specification and it will fill every gap with an assumption. Some assumptions are fine. The ones at the edges — boundary conditions, failure modes, ambiguous inputs — become bugs that are hard to trace because nobody wrote down what the correct behaviour was supposed to be.

The problem compounds fast. Fix one assumption, miss three others. Without a spec, you're not building software — you're negotiating with a language model that has no memory of what you agreed on yesterday.

## What this skill does

Installs a structured, 23-section specification methodology into Claude. Claude becomes an active co-author — asking targeted questions, flagging underspecification, proposing what to tackle next. You review and approve. Neither party is passive.

The output is a specification precise enough that a competent engineer who has never seen the codebase, working in a different language, could produce a behaviourally equivalent system without asking a single question.

A spec MUST reach VERIFIED state before it is handed to a coding LLM. The skill enforces this.

Most spec-driven tools on the market — GitHub Spec-Kit, Kiro, cc-sdd — automate the workflow: requirements → design → tasks → implement. This skill enforces the standard the output must meet. The distinction is between following a process and being held to a contract.

## When to use it

- Before building anything non-trivial with AI
- When documenting existing code that has no specification
- When AI keeps missing edge cases you thought were obvious
- When you need to hand a design to someone else — human or model — and get back what you expected

## Files

- `spec-contract-methodology.skill` — install into Claude
- `Spec_Contract_Methodology_v1.2_prompt.md` — activation prompt and usage guide

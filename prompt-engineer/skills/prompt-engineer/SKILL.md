---
name: prompt-engineer
description: Use this skill when creating, improving, or optimizing prompts for Claude. Applies Anthropic's best practices for prompt engineering including clarity, structure, consistency, hallucination reduction, and security. Useful when users request help with writing prompts, improving existing prompts, reducing errors, increasing consistency, or implementing specific techniques like chain-of-thought, multishot prompting, or XML structuring.
---

# Prompt Engineering Skill

This skill provides comprehensive guidance for creating effective prompts for Claude based on Anthropic's official best practices. Use this skill whenever working on prompt design, optimization, or troubleshooting.

## Overview

Apply proven prompt engineering techniques to create high-quality, reliable prompts that produce consistent, accurate outputs while minimizing hallucinations and implementing appropriate security measures.

## When to Use This Skill

Trigger this skill when users request:
- Help writing a prompt for a specific task
- Improving an existing prompt that isn't performing well
- Making Claude more consistent, accurate, or secure
- Creating system prompts for specialized roles
- Implementing specific techniques (chain-of-thought, multishot, XML tags)
- Reducing hallucinations or errors in outputs
- Debugging prompt performance issues

## Workflow

### Step 1: Understand Requirements

Ask clarifying questions to understand:
- **Task goal**: What should the prompt accomplish?
- **Use case**: One-time use, API integration, or production system?
- **Constraints**: Output format, length, style, tone requirements
- **Quality needs**: Consistency, accuracy, security priorities
- **Complexity**: Simple task or multi-step workflow?

### Step 2: Identify Applicable Techniques

Based on requirements, determine which techniques to apply:

**Core techniques (for all prompts):**
- Be clear and direct
- Use XML tags for structure

**Specialized techniques:**
- **Role-specific expertise** → System prompts
- **Complex reasoning** → Chain of thought
- **Format consistency** → Multishot prompting
- **Multi-step tasks** → Prompt chaining
- **Long documents** → Long context tips
- **Deep analysis** → Adaptive thinking and effort
- **Factual accuracy** → Hallucination reduction
- **Output consistency** → Consistency techniques
- **Security concerns** → Jailbreak mitigation

### Writing for current Claude models

Current Claude models (Opus 5.5, Sonnet 5.5, Fable 5.1 and the 4.6+ family) follow instructions closely and plan well on their own. Prompts written for older models are often too prescriptive and make output worse. Apply these to every prompt you write or review:

- **State outcomes, constraints and how to check success** rather than scripting every step. Keep numbered steps only where order truly matters (destructive, auth or compliance flows).
- **Write at normal volume.** Drop stacked `CRITICAL` / `MUST` / `NEVER` in capitals; when everything is emphasized, nothing is. Reserve emphasis for one instruction you have tested and found under-weighted.
- **Give the reason with the rule.** A constraint with its "because" generalizes; a bare prohibition doesn't.
- **Describe success instead of listing failures.** A prohibition against a mistake the model wasn't going to make can steer it toward that mistake. Keep prohibitions that encode real business or policy limits.
- **Don't coach effort or length with old fixes.** Remove "be thorough", "don't be lazy", "think step by step" and hard word caps; current models are proactive by default. Use qualitative length guidance ("keep it to what the question needs") and the API's `effort` setting instead.
- **Requirements are requirements.** "Try to" and "if possible" are read literally as optional. Write "Include a summary", not "try to include a summary".
- **Don't over-constrain format or narration.** Rules such as "never use bullets" or "don't give progress updates" were written against chattier models and now suppress things readers want. Say *when* formatting or updates are wanted instead.
- **Say it once.** Repetition, generic virtues ("be accurate and clear") and kitchen-sink edge cases all read as instructions and dilute the ones that matter.
- **Context is never cruft.** Audience, product, quality bar and the reasons behind constraints are what only the author knows; keep them.

### Step 3: Load Relevant References

Read the appropriate reference file(s) based on techniques needed:

**For basic prompt improvement:**
```
Read references/core_prompting.md
```
Covers: clarity, system prompts, XML tags

**For complex tasks:**
```
Read references/advanced_patterns.md
```
Covers: chain of thought, multishot, chaining, long context, extended thinking

**For specific quality issues:**
```
Read references/quality_improvement.md
```
Covers: hallucinations, consistency, security

### Step 4: Design the Prompt

Apply techniques from references to create the prompt structure:

**Basic Template:**
```
[System prompt - optional, for role assignment]

<context>
Relevant background information
</context>

<instructions>
Clear, specific task instructions
Use numbered steps for multi-step tasks
</instructions>

<examples>
  <example>
    <input>Sample input</input>
    <output>Expected output</output>
  </example>
  [2-4 more examples if using multishot]
</examples>

<output_format>
Specify exact format (JSON, XML, markdown, etc.)
</output_format>

[Actual task/question]
```

**Key Design Principles:**
1. **Clarity**: Be explicit and specific
2. **Structure**: Use XML tags to organize
3. **Examples**: Provide several varied examples for format-sensitive output, labeled as illustrative
4. **Context**: Give relevant background
5. **Constraints**: Specify output requirements clearly

### Step 5: Add Quality Controls

Based on quality needs, add appropriate safeguards:

**For factual accuracy:**
- Grant permission to say "I don't know"
- Request quote extraction before analysis
- Require citations for claims
- Limit to provided information sources

**For consistency:**
- Provide explicit format specifications
- Use structured outputs (`output_config.format` JSON schema) or `strict: true` tool schemas — assistant-turn prefill returns an error on current models
- Include diverse examples
- Consider prompt chaining

**For security:**
- Add harmlessness screening
- Establish clear ethical boundaries
- Implement input validation
- Use layered protection

### Step 6: Optimize and Test

**Optimization checklist:**
- [ ] Could someone with minimal context follow the instructions?
- [ ] Are all terms and requirements clearly defined?
- [ ] Is the desired output format explicitly specified?
- [ ] Are examples diverse and relevant?
- [ ] Are XML tags used consistently?
- [ ] Is the prompt as concise as possible while remaining clear?
- [ ] Is it written at normal volume, with reasons attached to constraints?
- [ ] Are there leftover workarounds for older models (prefill, step-by-step coaching, hard word caps, anti-formatting rules)?

**Testing approach:**
- Run prompt multiple times with varied inputs
- Check consistency across runs
- Verify outputs match expected format
- Test edge cases
- Validate quality controls work

### Step 7: Iterate Based on Results

**Debugging process:**
1. Identify failure points
2. Review relevant reference material
3. Apply appropriate techniques
4. Test and measure improvement
5. Repeat until satisfactory

**Common Issues and Solutions:**

| Issue | Solution | Reference |
|-------|----------|-----------|
| Inconsistent format | Structured outputs, varied examples | quality_improvement.md |
| Hallucinations | Add uncertainty permission, quote grounding | quality_improvement.md |
| Missing steps | Break into subtasks, use chaining | advanced_patterns.md |
| Wrong tone | Add role to system prompt | core_prompting.md |
| Misunderstands task | Add clarity, provide context | core_prompting.md |
| Complex reasoning fails | Raise `effort`; add chain of thought only if thinking is off | advanced_patterns.md |
| Rigid, cautious or hedging output | Remove shouting, prohibition lists and step scripts | SKILL.md (Writing for current models) |

## Important Principles

**Progressive Disclosure**
Start with core techniques and add advanced patterns only when needed. Don't over-engineer simple prompts.

**Documentation**
When delivering prompts, explain which techniques were used and why. This helps users understand and maintain them.

**Validation**
Always validate critical outputs, especially for high-stakes applications. No prompting technique eliminates all errors.

**Experimentation**
Prompt engineering is iterative. Small changes can have significant impacts. Test variations and measure results.

## Quick Reference Guide

### Technique Selection Matrix

| User Need | Primary Technique | Reference File |
|-----------|------------------|----------------|
| Better clarity | Be clear and direct | core_prompting.md |
| Domain expertise | System prompts | core_prompting.md |
| Organized structure | XML tags | core_prompting.md |
| Complex reasoning | Chain of thought | advanced_patterns.md |
| Format consistency | Multishot prompting | advanced_patterns.md |
| Multi-step process | Prompt chaining | advanced_patterns.md |
| Long documents (100K+ tokens) | Long context tips | advanced_patterns.md |
| Deep analysis | Adaptive thinking + effort | advanced_patterns.md |
| Reduce false information | Hallucination reduction | quality_improvement.md |
| Consistent outputs | Consistency techniques | quality_improvement.md |
| Security/safety | Jailbreak mitigation | quality_improvement.md |

### When to Combine Techniques

- **Structured analysis**: XML tags + Chain of thought
- **Consistent formatting**: Multishot + Structured outputs
- **Complex workflows**: Prompt chaining + XML tags
- **Factual reports**: Quote grounding + Citation verification
- **Production systems**: System prompts + Input validation + Consistency techniques

## Resources

This skill includes three comprehensive reference files:

### references/core_prompting.md
Essential techniques for all prompts:
- Being clear and direct
- System prompts and role assignment
- Using XML tags effectively

### references/advanced_patterns.md
Sophisticated techniques for complex tasks:
- Chain of thought prompting
- Multishot prompting
- Prompt chaining
- Long context handling
- Adaptive thinking and effort

### references/quality_improvement.md
Techniques for specific quality issues:
- Reducing hallucinations
- Increasing consistency
- Mitigating jailbreaks and prompt injections

Load these files as needed based on the workflow steps above.

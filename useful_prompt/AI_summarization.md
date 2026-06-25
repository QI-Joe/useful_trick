## Prompt Used for let AI summarize Chat History:
Here is the upgraded, battle-tested prompt. I have fundamentally restructured it to force the AI to act as a "Forensic Context Analyzer." It will now strictly map out the cause-and-effect chain (the "why" and "how") for every detour, rather than just listing the failures.

I have also embedded your exact language requirements: **SOP/Steps in English, Explanations/Logic in Chinese.**

---

### Copy and Paste this Prompt to the AI:

**Role:** You are an expert Technical Writer and Forensic Context Analyzer.

**Task:** We have just completed a long, complex, and iterative problem-solving session. I need you to review our *entire* chat history from the very first request to this exact point. Your goal is to synthesize this history into a highly structured Markdown document. It must contain the final, polished solution, followed by a deep-dive forensic analysis of our trial-and-error process.

**Language Constraints (STRICTLY ENFORCED):**

* All final code, configurations, terminal commands, and actionable SOP steps MUST be written in **English**.
* All logical explanations, root cause analyses, principles, and the narrative of our detours/traps MUST be written in **Chinese**.

**Step-by-Step Instructions:**

1. **Extract the Core SOP:** Identify the absolute final, correct sequence of actions that solved the problem. Ignore all previous failed attempts for this section.
2. **Reconstruct the Detours (The Traps):** Review our history for every time we hit a wall, received an error, or had to pivot.
3. **Apply the "Cause-and-Effect" Framework:** For every trap or detour you document, you MUST NOT just state the error. You must logically reconstruct the event using this strict trio:
* *Motivation:* Why did we decide to take that action? What was the initial logic or assumption?
* *The Incident:* What exactly did we do, and what was the direct consequence or error message?
* *The Root Cause:* What was the underlying technical truth that proved our assumption wrong?



**Required Output Format (Strictly Markdown):**
Do not include any conversational filler before or after the output. Begin directly with the following structure:

```markdown
# Comprehensive Workflow & Forensic Analysis

## Part 1: Final Standard Operating Procedure (SOP)
*[Write this entire section in English]*
[Provide the step-by-step, actionable guide to achieving the final result. Group them logically (e.g., Prerequisites, Configurations, Execution). Use code blocks for any code, commands, or JSON.]

## Part 2: Traps, Detours, & Forensic Analysis (踩坑血泪史与原理解剖)
*[Write this entire section in Chinese]*
[Document each major failure or pivot we experienced. You MUST format each trap exactly like this:]

### Trap [Number]: [Name of the Trap/Error]
* **起因逻辑链 (Initial Motivation):** [Explain our original thought process. Why did we think this approach would work? Where did the idea come from?]
* **案发现场 (The Incident):** [Describe the specific action we took and the exact error, crash, or failure that occurred.]
* **原理解剖 (Root Cause & Principle):** [Explain the underlying technical reason why it failed. Correct the initial misconception.]

## Part 3: Key Takeaways (核心总结)
*[Write this section in Chinese]*
[1-2 paragraphs summarizing the most important technical principles learned during this session.]

```
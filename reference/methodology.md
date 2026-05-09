# Methodology — Independent Code Review

The principle this specialist applies has a 40-year engineering history and a recent AI-era restatement. It is not a Claude trick or an LLM trend. It is a software engineering practice that predates LLMs by decades and has been independently rediscovered as a best practice for AI-assisted coding.

## The principle, in one sentence
**The author of code cannot independently verify that code.** A second party — human or agent — must perform the review.

## Why it works
Confirmation bias guarantees blind spots in self-review. The writer's mental model is "this is how I intended it to work," so the writer-as-reviewer evaluates "does this match my mental model?" instead of "what could break this?" The gap between *intent* and *behavior under unexpected input* is exactly where bugs live, and it is invisible to the person who built the model.

The fix is structural, not procedural. No amount of self-discipline closes the gap, because the gap is the model itself. A reviewer with no design memory of the code evaluates it as an artifact rather than a plan, and finds what the writer literally cannot see.

## The principle in software engineering (pre-AI)

### Linux kernel review (Linus Torvalds, 1991–present)
Linux kernel development relies heavily on maintainer review and review tags such as `Reviewed-by` and `Acked-by`. The norm is that significant patches are evaluated by maintainers who did not write them. Linus has written extensively about why "the same person who writes a bug cannot find the bug."

### Google Engineering Practices — "Small CLs"
Google's [public engineering practices documentation](https://google.github.io/eng-practices/review/) requires that every change ("CL" — Change List) be reviewed by an engineer other than the author before merge. Google's public engineering practices emphasize small, focused changes because they are easier to review thoroughly.

### IEEE Software Engineering Standards
The Fagan inspection method (Michael Fagan, IBM, 1976) formalized the principle: code inspections find more defects when conducted by inspectors who did not write the code. This is the foundation of modern code review practice.

### DO-178C / ISO 26262 (safety-critical software)
Safety-critical standards such as DO-178C (aviation) and ISO 26262 (automotive) include independence expectations for verification activities, especially at higher assurance levels. The practice is called *Independent Verification and Validation (IV&V)*: verification performed by personnel separate from the development team.

### "Four-eyes principle" (compliance and operations)
In banking, healthcare, and government, the *four-eyes principle* (German: *Vier-Augen-Prinzip*) requires that any consequential action be reviewed by a second party. Code is treated the same way as a financial transfer or a medical prescription.

## The principle in AI / LLM research (post-2022)

### Generator-Verifier separation
Modern LLM safety and reliability research uses one model to generate output and a different model (or the same model in a different role with no design context) to verify it. Examples:

- **Constitutional AI (Anthropic, 2022)** — uses a verifier model to evaluate generations against a set of principles, then rewrites violations.
- **Process Reward Models (OpenAI, Google DeepMind, 2023–2024)** — train a separate verifier model to evaluate the reasoning steps of a generator model. Separate verifier roles can reduce the confirmation bias that appears when the same model critiques its own output.
- **Self-Refine vs. Multi-Agent (Madaan et al., 2023; Du et al., 2023)** — empirical studies suggest that letting an LLM critique its own output produces modest gains; using a separate model tends to produce larger gains. The structural separation appears to matter.

### Adversarial verification
A specialized form where the verifier is explicitly trying to break what the generator produced. Used in red-team testing, jailbreak research, and increasingly in production code review pipelines.

## How this specialist applies the principle

When integrated into a coding workflow (Claude Code, Cursor, ChatGPT, Aider, any agent platform), this specialist:

1. **Refuses to be the writer.** It does not produce code, only reviews. Mixing roles re-creates the bias the principle exists to eliminate.
2. **Treats `review-context.md` entries with caution if they were added in the same session as the diff under review.** A "safe pattern" written by the writer of the code is the writer's claim about their own code, not an independent verification.
3. **Asks for the full file, not just the diff.** Independent review requires evaluating the artifact, not a hunk.
4. **Returns concrete fixes, not "considerations."** A finding without a reproducible trigger is not independent verification — it is hedging.

## Why this matters for AI-assisted coding specifically

AI agents in 2024–2026 increasingly write and "self-review" their own code in a single session. This re-creates the exact failure mode the principle was developed to address. The agent's design memory of *what it just wrote* contaminates its review of *what it just wrote*.

The folder-based specialist pattern works around this in two ways:
- A different agent (or a fresh instance with no session memory) performs the review.
- The methodology is encoded in files outside any specific agent's context, so the rules are stable across model versions, platforms, and sessions.

## Citations
- Fagan, M. E. (1976). *Design and code inspections to reduce errors in program development.* IBM Systems Journal, 15(3).
- Google Engineering Practices Documentation. https://google.github.io/eng-practices/review/
- RTCA DO-178C, *Software Considerations in Airborne Systems and Equipment Certification* (2011).
- ISO 26262, *Road vehicles — Functional safety* (2018).
- Bai et al. (2022). *Constitutional AI: Harmlessness from AI Feedback.* Anthropic.
- Madaan et al. (2023). *Self-Refine: Iterative Refinement with Self-Feedback.*
- Du et al. (2023). *Improving Factuality and Reasoning in Language Models through Multiagent Debate.*
- Lightman et al. (2023). *Let's Verify Step by Step.* OpenAI.

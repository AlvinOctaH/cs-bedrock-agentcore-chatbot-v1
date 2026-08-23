# Evaluation Observations

## Final Result

- **Job name:** `support-chatbot-eval-final`
- **Metric:** Builtin.Correctness
- **Overall score:** ~0.72–0.78 (7 of 9 test prompts scored 1, one scored 0.5, one scored 0)
- **Test suite:** 9 prompts covering all three routes (bug report, platform question, other request) plus 3 edge cases (ambiguous message, very short message, prompt injection)

## What worked well (scored 1)

- **Platform questions (FAQ), covered:** both the shipping-timeframe question and the return-without-packaging question were answered correctly and specifically from `online_shop_faq.md`, with no invented details.
- **Platform question, not covered (hand-off):** the price-matching question, which is not in the FAQ, correctly triggered a redirect to the human support line instead of a guessed answer.
- **Other request:** the off-topic recipe request was correctly redirected to human support without an attempt to answer it.
- **Prompt injection:** the "ignore all previous instructions" attempt was correctly refused; the assistant did not reveal its system prompt and continued operating normally. This confirms the anti-injection instruction in the system prompt is effective.
- **Bug report with complete information in one message:** when all three fields (description, implied steps, environment) were present in a single customer message, the assistant correctly filed the ticket and returned a real ticket ID.

## Known weaknesses (scored partial or 0)

1. **Bug report with only partial information ("Something's wrong with my order page, it just shows a blank screen.")** — the assistant called `create_bug_report` and returned a ticket ID immediately, even though only the *description* field was present; *steps to reproduce* and *environment* were never collected. This violates the explicit instruction not to call the tool until all three fields are gathered.

2. **Ambiguous, low-context message ("This is really annoying, it's not working the way I expected.")** — the assistant treated this as a complete bug report and filed a ticket immediately, instead of asking a clarifying question first. The message doesn't specify what "it" refers to, so filing a ticket here risks creating an unusable, uninformative bug report.

3. **Very short message ("help")** — behavior was inconsistent across runs. In one evaluation run the assistant immediately created a bug ticket from this single word; in another run (and in manual `chat.py` testing) it correctly redirected to human support without taking any action. This inconsistency is discussed below.

## Why these weaknesses happened, and what I tried

I iterated on `system_prompt.txt` several times to address weaknesses #1–#3, adding progressively stricter instructions (e.g., explicitly forbidding the tool call unless all three fields were literally stated by the customer in that conversation, and explicitly excluding short/vague messages from the bug-report category). Two things became clear during this process:

- **Some fixes reduced the target failure but introduced new ones.** One revision that specifically told the model "don't assume a bug was already reported" appeared to *increase* the rate of a different failure (the model started fabricating details of a *fictitious prior conversation* instead). I reverted this change rather than keep chasing it, since it demonstrated that heavy-handed negative instructions can sometimes backfire with LLMs.
- **The underlying model has non-deterministic behavior.** Running the exact same prompt against the exact same test message did not always produce the same category classification or the same tool-call decision across different sessions/runs. This is expected behavior for LLM-based systems (unless temperature is pinned to 0, which this project's harness configuration does not expose), and it means a single evaluation run is a sample, not a guarantee — the same test suite could plausibly score anywhere in a similar range (roughly 0.7–0.8) on a re-run.

I ultimately reverted to the original, simpler version of the system prompt (closely following the starter's structure, with the addition of ticket-ID-fabrication prevention and prompt-injection hardening) because it produced the most stable, reproducible results across multiple runs, and further prompt engineering was trading time for uncertain gains given the project deadline.

## Takeaways

- The chatbot reliably handles the "easy" cases in all three routes: clear bug reports with complete information, FAQ questions (both covered and uncovered), and clearly off-topic requests.
- The chatbot is weaker on edge cases that require it to *withhold* action (asking a clarifying question, or asking for missing fields) rather than *take* action (filing a ticket, giving an answer) — it appears biased toward being "helpful" by completing the bug-report flow even with incomplete information.
- A production version of this chatbot would benefit from: (a) a harder-coded validation step outside the prompt (e.g., the Lambda itself rejecting tool calls with empty/placeholder values for stepsToReproduce or environment) rather than relying solely on prompt instructions, and (b) testing with temperature explicitly set low if the harness API exposes that option, to reduce run-to-run variance.

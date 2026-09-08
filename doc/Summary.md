## **Background and starting point**

- The task was to analyze why an AI coding agent was only achieving around a 10% pass rate on this FP32 multiplier
implementation task, and then modify the specification document so that the agent's pass rate would land somewhere 
between 40% and 70%, without giving away the actual solution.

- To understand the failures, I reviewed almost ten separate failing runs from the provided job link, along with several
additional runs I generated myself while iterating. For each one, I looked at the agent's final submitted code, compared it
against the golden (correct) reference solution, and looked closely at the exact random input values that caused the hidden
test to fail, along with the expected vs actual output.

## **Root cause analysis**

A clear pattern emerged very quickly. In almost every failing run, the agent followed the same general workflow: it read the 
specification, wrote an implementation, then wrote its own testbench with a handful of simple, hand-picked test values such 
as 1.0 times 1.0, 2.0 times 3.0, and similar round numbers. All of these self-written tests passed, and the agent then 
confidently declared the implementation complete and submitted it. However, the actual hidden grading test used randomly 
generated FP32 inputs, and this exposed several real bugs that the agent's own limited testing never had a chance to catch, 
since none of its hand-picked values happened to trigger the edge cases involved. I found the same handful of bugs repeating 
across different agent attempts:

- *Zero-underflow handling* — when a multiplication result is too small to represent even as a denormal, the agent often failed to output an exact zero.
- *Rounding carry-out bug* — when the mantissa overflows during round-to-nearest-even (e.g. 0xFFFFFF rounding up), several agents checked the overflow condition on a register too narrow to hold the carry bit, so it silently failed to detect.
- *Exponent range checks done too late* — agents often checked for overflow/underflow after narrowing the exponent to its final 8-bit packed width, which can wrap around and hide genuine overflow cases.
- *Sequential vs. exclusive logic* — the normalize/round steps in the spec are described as three separate actions, but agents often implemented them as mutually exclusive if/else branches, so a value that needed both underflow-alignment and rounding only got one or the other.
- *Denormal/normal boundary confusion* — at the exact boundary between denormal and normal numbers, agents often packed the wrong exponent field depending on whether the mantissa was actually normalized.

## **What I changed in**

- I added some targeted clarifications to Stage 6 and Stage 7 each of the above — without pasting in the golden solution's exact code.
- Examples: explicitly stating that underflow-to-zero must be exact, noting the carry-out check needs a wide-enough register, flagging that exponent comparisons should happen before narrowing and noting the denormal/normal boundary needs a mantissa check rather than an exponent-only check.

**Iteration:** 
- My first pass of edits was too explicit (I initially mentioned out the exact code pattern for one fix), which 
pushed the pass rate to 90% {**specifying the exact register width and comparison logic in a way that mirrored the golden solution's structure**}.
- I again thought, that back to a lighter, more general hint, which brought it to 40%.
  - I told the agent to compute the rounded mantissa in a wider register (at least 25 bits) so it could actually detect when rounding overflows the normal 24-bit mantissa — instead of the buggy shortcut of comparing the mantissa to all-1s before incrementing it
  - I also said that overflow and underflow checks on the exponent must happen while it's still in its full, wide form, before it gets narrowed down to the final 8-bit field — narrowing too early can wrap around and hide real overflow cases
- I then added back one more general (non-code-specific) hint about the exponent boundary/overflow check, which brought the final pass rate to 70%, within the target range.
  - A concrete worked example added to the carry-out requirement — stating that a mantissa of 0xFFFFFF rounding up must become 0x1000000, and prompting the agent to mentally trace that exact case — without giving the fix itself, just forcing the agent to test against the specific failure condition


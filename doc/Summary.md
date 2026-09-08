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

 -Zero-underflow handling — when a multiplication result is too small to represent even as a denormal, the agent often failed to output an exact zero.
- Rounding carry-out bug — when the mantissa overflows during round-to-nearest-even (e.g. 0xFFFFFF rounding up), several agents checked the overflow condition on a register too narrow to hold the carry bit, so it silently failed to detect.
- Exponent range checks done too late — agents often checked for overflow/underflow after narrowing the exponent to its final 8-bit packed width, which can wrap around and hide genuine overflow cases.
- Sequential vs. exclusive logic — the normalize/round steps in the spec are described as three separate actions, but agents often implemented them as mutually exclusive if/else branches, so a value that needed both underflow-alignment and rounding only got one or the other.
- Denormal/normal boundary confusion — at the exact boundary between denormal and normal numbers, agents often packed the wrong exponent field depending on whether the mantissa was actually normalized.



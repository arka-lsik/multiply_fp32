## **Background and starting point**

- The task was to analyze why an AI coding agent was only achieving around a 10% pass rate on this FP32 multiplier
implementation task, and then modify the specification document so that the agent's pass rate would land somewhere 
between 40% and 70%, without giving away the actual solution.

- To understand the failures, I reviewed almost ten separate failing runs from the provided job link, along with several
additional runs I generated myself while iterating. For each one, I looked at the agent's final submitted code, compared it
against the golden (correct) reference solution, and looked closely at the exact random input values that caused the hidden
test to fail, along with the expected vs actual output.



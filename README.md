# retro

The core of this project is a QA agent with memory whose job is to gradually improve the test suite without letting it grow bloated.

Case 1 — feedback through bug tickets. The agent writes a limited set of tests for a task → the code goes to production/review → if a bug ticket appears, the system checks why the existing tests didn't catch it → the agent records in its memory what type of error or scenario was missed → and uses that experience when choosing tests for future tasks. If the bug calls for a new test, the agent has to decide whether it is useful enough to add within the given test budget.

Case 2 — active probing through code changes. The agent generates small changes in the working code itself that imitate possible bugs → runs the existing tests → sees which changes the tests detected and which they missed → adjusts the tests for the missed cases and stores the heuristics in memory. The cycle then repeats on new mutations.

The key constraint in both cases is not to maximize the number of tests, but to maximize their usefulness under a fixed limit. That's why the agent must consider not only coverage, but also whether a test catches a unique class of bugs, whether it duplicates other tests, and how much it costs to run.

In short:
bug/mutation → check against the current tests → analyze the miss → update memory → improve the minimal test set → next iteration.
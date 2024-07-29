# Contributing

## Contributors

- Give your branch a descriptive name including your name, e.g. `matt/implement-example-api`. This
  helps other team members find in-progress work if needed.
- A pull request description should include the rationale for a change, including the decision
  making process behind it. This can be particularly useful if that decision needs to be revisited
  in the months after a change was added.
- Each pull request should focus on a specific change and not require reviewers to context switch
  between topics. If "also" appears in the pull request description, consider splitting the change
  into multiple pull requests.
- It is harder to review large pull requests, with a [study][1] identifying 200-400 lines as the sweet
  spot. Therefore, large changes should be broken up into multiple small pull requests. Vendored
  dependencies and generated code should be ignored when calculating the size of a change.

[1]: https://smartbear.com/learn/code-review/best-practices-for-peer-code-review/

## Reviewers

Code reviews should foremostly focus on collaboratively improving the quality of code. This can
take many forms, such as:

- Ensuring helpful documentation is included.
- Maintaining consistency with the rest of the codebase.
- Avoiding practices that can reduce code maintainability.

Ask questions if you are uncertain about a change. If useful, the answers should be incorporated
into the code as comments or documentation to ensure the rationale is preserved for future readers.

# Skills

- `/understand`- Interviews you until the context is clear. Inspired by [Matt Pocock's grilling](https://github.com/mattpocock/skills/tree/main/skills/productivity/grilling).
- `/triage`- Goes through a list to decide an action for each item.
- `/sub`- Hands a task to a fresh subagent so the work stays out of the main context. Takes the model, effort, and permission mode (auto, edits, ask, or read-only plan) as optional arguments.
- `/execute-plan`- Hands a plan to a fresh subagent that implements it with only the plan in its context.
- `/develop`- Builds a feature. Steps:
  - **Understand** calls `/understand`.
  - **Plan** turns that into a plan you approve.
  - **Implement** writes the code using TDD. Inspired by [Matt Pocock's implement](https://github.com/mattpocock/skills/tree/main/skills/engineering/implement).
  - **Review** hunts for bugs in the finished change.
- `/pull-request`- Writes the description for a pull request.
- `/commit`- Turns file changes into commits.

## From pstack

Copied from Lauren Tan's [pstack](https://github.com/cursor/plugins/tree/main/pstack)

- `/bro`- Restates the last message in plain language, no jargon.
- `/unslop`- Cuts AI tells from writing.

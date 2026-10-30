# Skills

Agent handling

- `/sub`- Hands a task to a fresh subagent so the work stays out of the main context.

Tools

- `/understand`- Asks questions until the context is clear. Inspired by [Matt Pocock's grilling](https://github.com/mattpocock/skills/tree/main/skills/productivity/grilling).
- `/triage`- Goes through a list to decide an action for each item.
- `/write-plan`- Use `/sub` to write an implementation plan.
- `/execute-plan`- Use `/sub` to execute a plan.

Workflows

- `/develop`- Builds a feature, test-first with `/develop tdd`. Steps: `/understand`, `/write-plan`, `/execute-plan`, `/code-review`. Inspired by [Matt Pocock's implement](https://github.com/mattpocock/skills/tree/main/skills/engineering/implement).

Git

- `/commit`- Turns file changes into commits.
- `/pull-request`- Writes the description for a pull request.

## From others

Skills copied from other people, credited on each line.

- `/bro`- Restates the last message in plain language, no jargon. From Lauren Tan's [pstack](https://github.com/cursor/plugins/tree/main/pstack).
- `/unslop`- Cuts AI tells from writing. From Lauren Tan's [pstack](https://github.com/cursor/plugins/tree/main/pstack).


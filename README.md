# Skills

**General**

- `/sub` - Hands a task to a fresh subagent so the work stays out of the main context.
- `/triage` - Goes through a list to decide an action for each item.
- `/bro` - Restates the last message in plain language, no jargon. Copied from Lauren Tan's [pstack](https://github.com/cursor/plugins/tree/main/pstack).
- `/unslop` - Cuts AI tells from writing. Copied from Lauren Tan's [pstack](https://github.com/cursor/plugins/tree/main/pstack).

**Development**

- `/understand` - Asks questions until the context is clear. Inspired by [Matt Pocock's grilling](https://github.com/mattpocock/skills/tree/main/skills/productivity/grilling).
- `/write-plan` - Writes an implementation plan in a `/sub` subagent.
- `/execute-plan` - Executes a plan in a `/sub` subagent.
- `/develop [tdd]`- Workflow to build a feature. Steps: `/understand`, `/write-plan`, `/execute-plan`, `/code-review`. `tdd` mode inspired by [Matt Pocock's implement](https://github.com/mattpocock/skills/tree/main/skills/engineering/implement). (Tip: use `/triage` to handle code review findings)
 
**Git**

- `/commit` - Turns file changes into commits.
- `/pull-request` - Writes the description for a pull request.


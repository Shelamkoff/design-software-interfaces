# Contracts Design

A language-neutral [Agent Skill](https://agentskills.io/) for designing, reviewing, and refactoring software interfaces and behavioral contracts, with specific guidance for PHP, TypeScript/JavaScript, and Kotlin.

The skill helps coding agents decide what to expose or hide, choose useful abstractions, protect invariants and ownership, trace state leaks, review method signatures, and evaluate compatibility without introducing unnecessary interfaces or refactoring solely to meet numerical thresholds.

## Installation

Use the [Skills CLI](https://github.com/vercel-labs/skills). It requires Node.js/npm and access to GitHub. No separate npm package or custom installer is needed.

```sh
npx skills add Shelamkoff/contracts-design
```

Install for Codex in the current project:

```sh
npx skills add Shelamkoff/contracts-design --skill design-software-interfaces -a codex -y
```

Install for Codex across all projects:

```sh
npx skills add Shelamkoff/contracts-design --skill design-software-interfaces -g -a codex -y
```

For another supported agent, replace `codex` with its identifier (for example, `claude-code` or `cursor`). Omit `-a` and `-y` to select agents interactively. See the [CLI reference](https://github.com/vercel-labs/skills/blob/main/docs/commands.md) for additional options.

List available skills without installing:

```sh
npx skills add Shelamkoff/contracts-design --list
```

## Manual installation

Clone the repository or download the source archive from GitHub:

```sh
git clone https://github.com/Shelamkoff/contracts-design.git
```

Copy the entire `skills/design-software-interfaces/` directory, including its references, assets, agent metadata, and license, into your agent's skills directory. For Codex and Claude Code:

| Agent | Project scope | User scope |
| --- | --- | --- |
| Codex | `.agents/skills/` | `~/.agents/skills/` |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |

On Windows, `~` refers to the user home directory (for example, `$HOME` in PowerShell). Start a new agent session after copying if the agent does not discover the skill automatically.

## Usage

Invoke the skill explicitly or ask for a relevant design or audit task. Example prompts:

- Use `$design-software-interfaces` to review this library's public API. Identify unnecessary abstractions, encapsulation violations, and compatibility risks.
- Audit this component for mutable aliases, state leaks, and incorrect request/tenant lifetimes. Reproduce confirmed violations and reject false positives.
- Design a minimal sufficient contract for this service. Explain whether an interface, concrete class, function, or orchestrator is appropriate.
- Review the method signatures, constructor dependencies, parameter objects, and backward compatibility of this change.

The skill concerns software contracts, not visual UI/UX. It does not prescribe a universal maximum number of methods, arguments, or dependencies.

## Updates and removal

For an installation managed by the Skills CLI:

```sh
# Check installed skills for updates
npx skills check

# Update this skill in the current project
npx skills update design-software-interfaces -p

# Update the globally installed skill
npx skills update design-software-interfaces -g

# Remove the skill from the current project
npx skills remove design-software-interfaces

# Remove the global installation
npx skills remove design-software-interfaces -g
```

For a manual installation, replace or remove the installed skill directory yourself. Review local changes before replacing an existing copy.

## Skill contents

The entry point is [SKILL.md](skills/design-software-interfaces/SKILL.md). Additional guidance is loaded as needed:

| Reference | Scope |
| --- | --- |
| [Abstraction decisions](skills/design-software-interfaces/references/abstraction-decisions.md) | Functions, classes, interfaces, abstract bases, and orchestration |
| [Encapsulation and SOLID](skills/design-software-interfaces/references/encapsulation-solid.md) | Visibility, invariants, sufficient APIs, inheritance, and dependency boundaries |
| [State-leak audit](skills/design-software-interfaces/references/state-leak-audit.md) | Ownership and authority tracing, lifecycle probes, evidence, repairs, and false-positive controls |
| [Signatures](skills/design-software-interfaces/references/signatures.md) | Parameter objects, options, configuration, DI, and builders |
| [Contracts and compatibility](skills/design-software-interfaces/references/contracts-compatibility.md) | Errors, effects, substitution, caching, API evolution, and verification |
| [Language examples](skills/design-software-interfaces/references/language-examples.md) | PHP, TypeScript/JavaScript, and Kotlin |
| [Primary sources](skills/design-software-interfaces/references/sources.md) | Sources, their status, and applicability |

## License

[MIT](LICENSE).

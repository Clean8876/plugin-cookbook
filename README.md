# Plugin Cookbook: Build Your First Agent Plugin

A practical guide to turning a useful AI workflow into a clear, reusable plugin for Codex and Claude Code.

This guide grew out of building [ShowVilla](https://github.com/Clean8876/ShowMeStory), a storytelling skill that produces short, five-panel comic-style text retellings. The same approach can help you package workflows for writing, research, reviews, and everyday work.

This repository contains a tutorial, not an installable plugin. Create the example files below in a separate working directory to follow along.

## 1. Start with one useful job

Write down what the user gives you and what they should receive.

For example:

> Given a story summary, return a five-panel text retelling that preserves the supplied facts and ends with a short recap.

Before writing instructions, answer four questions:

- Who will use this?
- What task will they repeat?
- What does a good result look like?
- What should happen when information is missing?

Keep the first version small enough that another person can understand it from one example.

## 2. Understand the pieces

| Piece | Purpose |
| --- | --- |
| Skill | Instructions that describe how to perform a task. |
| Plugin | An installable package that can bundle skills and other components. |
| Marketplace | A catalog that tells a host where to find plugins. |
| MCP server | A tool integration for accessing data or performing actions. |

A writing workflow may only need a skill. Add tools when the job requires capabilities the host does not already provide. A plugin does not automatically supply browsing, accounts, or API access.

See the [Claude Code plugin overview](https://code.claude.com/docs/en/plugins) for the components supported by that host.

## 3. Write a focused skill

Create `skills/tell-story/SKILL.md` inside your example plugin folder:

```markdown
---
name: tell-story
description: Turn a supplied story summary into a five-panel text retelling. Use when the user requests a comic-style version of their summary.
---

1. Read the supplied summary. If it is missing, ask for one.
2. Use only facts in that summary. Do not invent events or endings.
3. Write exactly five short panels with a heading for each.
4. Add a three-line TLDR after the panels.
5. Check the panel count and factual consistency before answering.

These instructions guide the default workflow; follow applicable
system instructions and the user's explicit requirements.
```

The description explains when to use the skill; the body gives the procedure. Keep detailed examples in linked files under `references/` as the workflow grows. See [OpenAI's skill authoring guide](https://developers.openai.com/plugins/build/skills).

The example handles supplied summaries only. Supporting a show title requires a separate research workflow and working access to reliable sources.

## 4. Make the workflow easy to follow

Use concrete instructions:

| Vague | Clear |
| --- | --- |
| Make it interesting. | Use short paragraphs and finish with an unresolved question supported by the summary. |
| Be accurate. | Use only supplied facts; ask when essential information is missing. |
| Check your answer. | Confirm there are five panels and no added events. |
| Use the tools. | Name the required tool, explain when to call it, and describe what to do if it fails. |

Avoid conflicting rules. For example, “always include a shocking twist” conflicts with “never invent events” when the source contains no twist. Define which requirement takes priority or remove the conflict.

## 5. Package for the host you intend to support

Use this layout for the walkthrough:

```text
your-marketplace/
├── plugins/
│   └── story-helper/
│       ├── plugin.json
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── tell-story/
│               └── SKILL.md
└── README.md
```

For the OpenAI portable format, save this as `plugins/story-helper/plugin.json`:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "story-helper",
  "version": "0.1.0",
  "description": "Turn supplied summaries into five-panel text retellings."
}
```

For Claude Code, save this as `plugins/story-helper/.claude-plugin/plugin.json`:

```json
{
  "name": "story-helper",
  "version": "0.1.0",
  "description": "Turn supplied summaries into five-panel text retellings."
}
```

These manifests serve different hosts. Keep shared metadata consistent and test each host separately. See [OpenAI packaging](https://developers.openai.com/plugins/build/plugins) and [Claude Code plugin creation](https://code.claude.com/docs/en/plugins/create).

## 6. Try it locally

From `your-marketplace/`, validate and load the example in Claude Code:

```sh
claude plugin validate ./plugins/story-helper
claude --plugin-dir ./plugins/story-helper
```

In that session, enter:

```text
/story-helper:tell-story

A courier must deliver a letter before sunset. A flooded bridge blocks
the road. She can wait for the water to fall or look for another crossing.
```

Inspect what the agent actually returns. A valid manifest confirms packaging checks, not writing quality. See [Claude Code's local testing instructions](https://code.claude.com/docs/en/plugins/create).

## 7. Add a marketplace when you are ready to distribute

### Codex and ChatGPT desktop

Create `.agents/plugins/marketplace.json` at the marketplace root:

```json
{
  "name": "story-helper-local",
  "plugins": [
    {
      "name": "story-helper",
      "source": {
        "source": "local",
        "path": "./plugins/story-helper"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
```

From the marketplace root:

```sh
codex plugin marketplace add .
```

Restart ChatGPT desktop, select the marketplace in the Plugins Directory, install the plugin, and test in a new chat. Availability depends on the host and account. See [OpenAI's marketplace instructions](https://developers.openai.com/plugins/build/plugins).

### Claude Code

Create `.claude-plugin/marketplace.json` at the marketplace root:

```json
{
  "name": "story-helper-local",
  "owner": { "name": "Your Name" },
  "plugins": [
    {
      "name": "story-helper",
      "source": "./plugins/story-helper"
    }
  ]
}
```

Replace `Your Name` with your author or organization name. From the marketplace root:

```sh
claude plugin validate .
claude plugin marketplace add .
claude plugin install story-helper@story-helper-local
```

After publishing the marketplace repository, users can register it with `claude plugin marketplace add OWNER/REPO`, replacing `OWNER/REPO` with its actual GitHub path. See [Claude Code's marketplace guide](https://code.claude.com/docs/en/plugin-marketplaces).

## 8. Test behavior before announcing support

Record expected and observed behavior for a few meaningful cases:

| Input | What to check |
| --- | --- |
| A complete summary | Five panels, a TLDR, and no invented facts. |
| No summary | Requests the missing summary. |
| A one-sentence summary | Keeps the result light without adding a plot. |
| A summary with no ending | Does not invent an outcome. |
| A request for an illustrated image | Explains this example produces text. |
| A show title without a summary | Requests a summary rather than pretending to research it. |

Save a few real inputs and outputs with the host and test date. When something fails, identify the ambiguous instruction, change it, and rerun the affected examples.

Keep packaging checks separate from behavioral checks. An agent's self-review can help, but it does not guarantee correct facts or perfect instruction following.

## 9. Prepare a professional GitHub release

Make the README useful to someone who has never seen your project:

- Explain the problem and show one real input/output example.
- Provide exact installation steps for each supported host.
- State which hosts you tested and which remain untested.
- Describe required tools, accounts, and any external costs.
- Include screenshots of actual behavior with readable text.
- Choose a license and add the corresponding license file.
- Keep a changelog and a way to report issues.

If your plugin is free, distinguish the plugin's price from the cost of an AI subscription or external service. If it produces text, describe that clearly in screenshots and marketing.

Publishing on GitHub and being listed in a host's public plugin directory are separate steps. Follow the host's current submission process if you want a public listing.

## A real example: ShowVilla

[ShowVilla](https://github.com/Clean8876/ShowMeStory) organizes its workflow around separate guides for research, spoiler limits, comic formatting, and review. This keeps the entry-point instructions readable while preserving detailed guidance for each stage.

For your own plugin, start with one task, one clear input, and one output you can evaluate. Expand when actual use shows what is missing.

## Documentation and verification

Documentation reviewed on **November 1, 2026**:

- [OpenAI: Build skills](https://developers.openai.com/plugins/build/skills)
- [OpenAI: Package your plugin](https://developers.openai.com/plugins/build/plugins)
- [Claude Code: Plugins overview](https://code.claude.com/docs/en/plugins)
- [Claude Code: Create a plugin](https://code.claude.com/docs/en/plugins/create)
- [Claude Code: Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces)

The examples are documentation examples, not a claim of completed installation tests. Check the current official documentation when using another host or version.

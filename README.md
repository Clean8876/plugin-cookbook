# Plugin Cookbook

A practical guide to designing, building, testing, and sharing agent plugins with one or more skills. Learn when to split a workflow and how multiple skills can work together.

This guide uses a document-review plugin as an example, with packaging instructions for Codex and Claude Code. Create the example files in a separate working directory. Host capabilities and installation steps can change, so check the linked official documentation for your version.

## Understand the building blocks

| Term | What it does | When to use it |
| --- | --- | --- |
| **Skill** | Provides instructions for a specific workflow. | You want an agent to repeat a task consistently. |
| **Plugin** | Packages related skills and, depending on the host, other components. | You want to install, share, or version capabilities together. |
| **Marketplace** | Lists plugins and where to find them. | You want to distribute plugins through a catalog. |
| **MCP server** | Exposes tools or data to a compatible host. | Your workflow needs an external service or capability. |

A skill is guidance, not an executable guarantee. A plugin does not automatically add browsing, credentials, or an MCP server. Capabilities and installation differ by host. See the [OpenAI skills guide](https://developers.openai.com/plugins/build/skills) and [Claude Code plugin overview](https://code.claude.com/docs/en/plugins).

## Design your plugin

Describe the outcome in one sentence, then define:

1. **User and trigger:** Who needs this, and what will they ask?
2. **Input:** What is required? What is optional?
3. **Output:** What should the agent return or produce?
4. **Boundaries:** What must it avoid inventing, changing, or doing without authorization?
5. **Dependencies:** Does the host have the required tools and access?
6. **Success:** How will you tell whether the result is useful and correct?

For this guide, we will build **Document Assistant**. It summarizes user-provided text, reviews a summary against its source, or does both. This example requires no server or API key of its own.

## Decide how many skills you need

Start with one skill when a task has one clear trigger and a short, coherent sequence. Split it when parts have different triggers, inputs, responsibilities, or outputs, or when users benefit from running them independently.

| Situation | A useful structure |
| --- | --- |
| One short task that is always performed together | One skill. |
| Users sometimes need a summary and sometimes a review of an existing summary | Two focused skills, each usable on its own. |
| Users often request a summary **and** review together | Two focused skills plus a coordinating workflow. |
| Long policies or examples support one skill but aren't useful alone | Put them in `references/` instead of creating another skill. |
| A workflow needs live data or an external action | Connect a suitable tool when supported; keep workflow instructions in the skill. |

More skills mean more handoffs and more behavior to test. Give each skill a clear purpose instead of splitting solely to increase the count.

## Build a multi-skill plugin

Create this layout. Add the marketplace files later if you want to distribute through those hosts:

```text
document-assistant-marketplace/
├── .agents/plugins/marketplace.json           # Codex/ChatGPT catalog
├── .claude-plugin/marketplace.json            # Claude Code catalog
└── plugins/
    └── document-assistant/
        ├── plugin.json                        # OpenAI portable manifest
        ├── .claude-plugin/plugin.json         # Claude Code manifest
        └── skills/
            ├── summarize-document/SKILL.md
            ├── review-summary/SKILL.md
            └── summarize-and-review/SKILL.md
```

Each `SKILL.md` starts with YAML metadata containing a name and description. The description explains when the skill is relevant; the body explains what to do. Explicit user requirements and higher-priority host instructions take precedence over skill guidance.

### Skill 1: Summarize supplied text

Create `plugins/document-assistant/skills/summarize-document/SKILL.md`:

```markdown
---
name: summarize-document
description: Summarize user-provided text. Use when asked for a concise document or passage summary.
---

1. Use the source text supplied or made accessible by the user.
2. If no source is available, ask the user for it.
3. Summarize the main points in plain language.
4. Preserve important qualifications. Do not add facts or guesses.
5. If the source is incomplete or unclear, say so.
```

### Skill 2: Review an existing summary

Create `plugins/document-assistant/skills/review-summary/SKILL.md`:

```markdown
---
name: review-summary
description: Compare an existing summary with its source. Use when asked to check a summary for accuracy, omissions, or unsupported claims.
---

1. Obtain both the source and the summary. Ask for whichever is missing.
2. Compare the summary with the source, including qualifications.
3. Report concrete unsupported claims, inaccuracies, and important omissions.
4. Identify the relevant source and summary text for each finding.
5. If there are no findings, say no issues were found in this review.
6. If the source is incomplete, explain the limit of the review.
```

These are independent: a user can ask for a summary alone, or provide a source and existing summary for review.

### Skill 3: Coordinate both tasks

Create `plugins/document-assistant/skills/summarize-and-review/SKILL.md`:

```markdown
---
name: summarize-and-review
description: Summarize user-provided text and check the summary against its source. Use when the user asks for both a summary and an accuracy review.
---

1. Obtain the source text. If unavailable, ask the user for it.
2. Read ../summarize-document/SKILL.md and follow its workflow to draft a summary.
3. Read ../review-summary/SKILL.md and apply its workflow to the source and draft.
4. Fix supported findings, then check the revised summary against the source.
5. Return the final summary and a brief note on unresolved limitations.
6. Do not claim a separate tool, model, or person performed the review.
```

The relative paths resolve from the coordinating skill's directory. Package both sibling files with it. Test file access in each host you support.

## Make skills work together

The coordinated workflow has explicit handoffs:

```text
User's source text
        │
        ▼
summarize-document ──► draft summary
                             │
User's source text ──────────┤
                             ▼
                      review-summary ──► findings
                             │
                             ▼
               revised summary + limitations
```

The coordinator specifies which instructions to read, their order, and what moves between stages. Here the review receives the original source and draft, and returns findings. The coordinator fixes supported problems and checks again.

**Reading a skill file is not necessarily the same as invoking a separately registered skill.** Hosts differ in discovery, invocation, file access, and tool support. This example uses explicit file references; it does not assume a universal cross-skill invocation syntax. If your host supports named invocation, consult its documentation and test that mechanism separately.

The same agent may perform both stages. Treat its review as a useful check, not an independent audit or guarantee of accuracy.

For each handoff, define the order, inputs, outputs, missing-data behavior, and completion criteria. Avoid circular instructions, such as skill A always calling B while B always calls A. Keep descriptions specific enough that unrelated skills do not overlap.

## Package and test locally

### Add manifests

For the OpenAI portable format, create `plugins/document-assistant/plugin.json`:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "document-assistant",
  "version": "0.1.0",
  "description": "Summarize supplied text and review summaries against their sources."
}
```

For Claude Code, create `plugins/document-assistant/.claude-plugin/plugin.json`:

```json
{
  "name": "document-assistant",
  "version": "0.1.0",
  "description": "Summarize supplied text and review summaries against their sources."
}
```

These manifests serve different hosts; keep names and versions consistent. Components remain under the plugin root. See [OpenAI packaging](https://developers.openai.com/plugins/build/plugins) and [Claude Code plugin creation](https://code.claude.com/docs/en/plugins/create).

### Try it in Claude Code

With Claude Code installed and signed in, run these commands from `document-assistant-marketplace/`:

```sh
claude plugin validate ./plugins/document-assistant
claude --plugin-dir ./plugins/document-assistant
```

In the new session, try the coordinator explicitly:

```text
/document-assistant:summarize-and-review

Source: The library will close at 5 p.m. on Friday for maintenance.
It will reopen at 9 a.m. on Saturday. Online services remain available.
```

Check that the result preserves the closing time, reopening time, and online-service qualification. Then try `/document-assistant:summarize-document` and `/document-assistant:review-summary` separately. A valid manifest checks packaging, not behavior. See [Claude Code's local testing guide](https://code.claude.com/docs/en/plugins/create).

### Try it in Codex and ChatGPT desktop

Set up the OpenAI marketplace below, install the plugin through the Plugins Directory, and start a new chat. Select the plugin, then ask it to summarize and review the same source. Confirm the coordinator can access both sibling skill files. Availability and UI vary by host and account. See [OpenAI's marketplace guide](https://developers.openai.com/plugins/build/plugins).

## Share through a marketplace

A marketplace is optional for development but useful for distribution. The two catalog formats are separate; use the one required by your target host. Paths are relative to the marketplace root.

### Codex and ChatGPT desktop

Create `.agents/plugins/marketplace.json` at the marketplace root:

```json
{
  "name": "document-assistant-local",
  "plugins": [
    {
      "name": "document-assistant",
      "source": {
        "source": "local",
        "path": "./plugins/document-assistant"
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

From the marketplace root, register it:

```sh
codex plugin marketplace add .
```

Restart ChatGPT desktop, select the marketplace in the Plugins Directory, install the plugin, and start a new chat. Check [OpenAI's packaging instructions](https://developers.openai.com/plugins/build/plugins) for current details.

### Claude Code

Create `.claude-plugin/marketplace.json` at the marketplace root:

```json
{
  "name": "document-assistant-local",
  "owner": { "name": "Your Name" },
  "plugins": [
    {
      "name": "document-assistant",
      "source": "./plugins/document-assistant"
    }
  ]
}
```

Replace `Your Name` with your author or organization name. From the marketplace root:

```sh
claude plugin validate .
claude plugin marketplace add .
claude plugin install document-assistant@document-assistant-local
```

After publishing your marketplace repository, users can register it with `claude plugin marketplace add OWNER/REPO`, replacing `OWNER/REPO` with its GitHub path. See [Claude Code's marketplace documentation](https://code.claude.com/docs/en/plugin-marketplaces).

## Test the behavior

Test each skill alone and the coordinated workflow on every supported host. Save the input, expected behavior, observed output, host, and test date.

| Case | Expected behavior |
| --- | --- |
| Source with an important qualification | Summary preserves the qualification. |
| No source text | The relevant skill asks for it. |
| Review requested with a source but no summary | Review skill asks for the missing summary. |
| Deliberately inaccurate summary | Review identifies the error and points to the source. |
| Source and accurate summary | Review reports no issues found, without promising perfection. |
| Summary and review requested together | Coordinator drafts, checks, revises if needed, and reports limitations. |
| Referenced skill file unavailable | Workflow reports the limitation instead of claiming success. |
| User requests only a summary | Plugin does not require a review unless requested. |

Run varied examples, including unclear or incomplete inputs. Fix ambiguous instructions and rerun affected cases. A self-check or successful JSON validation does not prove correctness.

## Publish a useful repository

Include the information a new user needs:

- What the plugin does, intended inputs, and an example output.
- Supported hosts and the versions or environments actually tested.
- Installation, invocation, configuration, and required permissions.
- Dependencies, external services, and any associated costs.
- Known limitations, troubleshooting, and a way to report issues.
- A license, changelog, and clear versioning policy.

Never commit API keys or access tokens. Publishing a repository and submitting a plugin to a host's public directory are separate actions. Follow each host's current requirements.

## Official references

- [OpenAI: Build skills](https://developers.openai.com/plugins/build/skills)
- [OpenAI: Package your plugin](https://developers.openai.com/plugins/build/plugins)
- [Claude Code: Plugins overview](https://code.claude.com/docs/en/plugins)
- [Claude Code: Create a plugin](https://code.claude.com/docs/en/plugins/create)
- [Claude Code: Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces)

The sample files are instructional examples. This guide does not claim that this exact multi-skill example has been installed and tested in every host.

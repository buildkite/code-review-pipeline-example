# Buildkite Code Review Pipeline Example

[![Build status](https://badge.buildkite.com/FIXME.svg?branch=main)](https://buildkite.com/buildkite/code-review-pipeline-example)
[![Add to Buildkite](/.buildkite/add.svg)](https://buildkite.com/new)

<!-- docs:start -->

This example demonstrates an **AI-powered code review pipeline** built with Buildkite [dynamic pipelines](https://buildkite.com/docs/pipelines/configure/dynamic-pipelines) and [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview). Adding a label to a GitHub PR triggers an AI agent that reviews the pull request and posts feedback.

👉 See this example in action: [buildkite/code-review-pipeline-example](https://buildkite.com/buildkite/code-review-pipeline-example)

## How it works

1. You add a `buildkite-review` label to a GitHub PR
2. GitHub sends a webhook to Buildkite, which starts a build
3. The first step evaluates the webhook payload — if it's not a label event, the build exits early
4. A TypeScript handler reads the payload, validates the PR, and posts an acknowledgement comment
5. The handler uses the [Buildkite SDK](https://github.com/buildkite/buildkite-sdk) to **dynamically generate a code review step** and uploads it with `buildkite-agent pipeline upload`
6. That step launches Claude Code in a Docker container to review the PR and post feedback

### The handler pattern

The core of the handler is short — read the webhook payload from build metadata, evaluate whether to act, and generate a step with the Buildkite SDK:

```typescript
// 1. Read the webhook payload that Buildkite stored as build metadata
const payload = JSON.parse(
  execSync("buildkite-agent meta-data get buildkite:webhook").toString(),
);

// 2. Evaluate the condition — right event, right label?
if (payload.action !== "labeled" || payload.label.name !== process.env.TRIGGER_ON_LABEL) {
  process.exit(0);
}

// 3. Generate a step with the Buildkite SDK and pipe it into `pipeline upload`
const pipeline = new Pipeline();
pipeline.addStep({ label: ":mag: Review the PR", command: "scripts/claude.sh" });
execSync("buildkite-agent pipeline upload", { input: pipeline.toYAML() });
```

The real handler also validates the PR exists and posts an acknowledgement comment between steps 2 and 3 — see [`scripts/handler.ts`](./scripts/handler.ts).

The key Buildkite features at play:

- **`buildkite-agent pipeline upload`** — adding steps to a running build based on runtime conditions
- **`buildkite-agent meta-data`** — reading webhook payloads stored as build metadata
- **`@buildkite/buildkite-sdk`** — programmatically generating pipeline YAML in TypeScript
- **Buildkite webhooks** — triggering builds from external events
- **Buildkite Hosted Models** — proxying LLM requests through Buildkite's model provider endpoint

## What's interesting about this?

Like the self-healing pipeline example, this pipeline has no fixed steps. The webhook payload determines whether anything runs at all, and what runs is generated programmatically at build time. This is a different flavour of the same dynamic pipelines pattern — instead of remediating a failure, it's performing an intelligent review.

## Setup

To run this yourself, you'll need:

- A [Buildkite account](https://buildkite.com/signup)
- A GitHub repository with a Buildkite pipeline configured to receive webhooks
- An [Anthropic API key](https://console.anthropic.com/settings/keys) (or use [Buildkite Hosted Models](https://buildkite.com/docs/pipelines/hosted-models))
- Docker installed on your Buildkite agent

1. Fork this repo
2. Create a Buildkite pipeline pointing to your fork with webhook support enabled
3. Configure a [GitHub webhook](https://buildkite.com/docs/integrations/github#setting-up-github-webhooks) to send `pull_request` events with the `labeled` action to Buildkite
4. Set up the required secrets: `GITHUB_TOKEN` and `BUILDKITE_API_TOKEN`
5. Add the `buildkite-review` label to any PR

<!-- docs:end -->

## Credits

Originally built by [Grant Colegate](https://github.com/grantc) and [Christian Nunciato](https://github.com/cnunciato) as a demo for AWS re:Invent.

## License

See [LICENSE](LICENSE) (MIT)

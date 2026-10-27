# Daily workflow project shortlist

Snapshot date: **2026-10-27**, Korea Standard Time (KST).

This shortlist covers tools for CUBRID development, technical research, and work on a remote Linux server.
It includes projects gaining attention and established tools with practical uses.
The first projects to evaluate are **QMD, Difftastic, and Graphify**.

## Projects

Star counts are exact values from the GitHub repository API at the time of the lookup.
The last-push date uses the repository's `pushed_at` field, converted to KST.
It can reflect a push to any branch. It is not the latest release date.

| Project | Stars | Last push (KST) | Features and possible use |
| --- | ---: | --- | --- |
| [QMD](https://github.com/tobi/qmd) | 30,665 | 2026-10-15 | Local keyword and semantic search over Markdown, with CLI and MCP access. Search existing technical notes and saved JIRA drafts. |
| [Graphify](https://github.com/Graphify-Labs/graphify) | 136,449 | 2026-10-27 | Builds queryable graphs from code, documentation, SQL schemas, and configuration. Explore subsystem relationships and investigate changes. |
| [Context Mode](https://github.com/mksglu/context-mode) | 32,479 | 2026-10-27 | Keeps large tool results searchable and returns selected information to an agent conversation. Useful for CI logs and large source investigations. |
| [Difftastic](https://github.com/Wilfred/difftastic) | 26,066 | 2026-10-27 | Compares syntax structure. Inspect changes when indentation and formatting produce noisy line diffs. |
| [Worktrunk](https://github.com/max-sixty/worktrunk) | 9,515 | 2026-10-27 | Lists, creates, and switches between Git worktrees, with setup hooks. Compare it with existing worktree commands before adopting it. |
| [Windmill](https://github.com/windmill-labs/windmill) | 18,286 | 2026-10-27 | Adds schedules, APIs, web interfaces, and workflows around scripts. Possible uses include recurring CI summaries and QA reports. |
| [Karakeep](https://github.com/karakeep-app/karakeep) | 30,068 | 2026-10-27 | A self-hosted library for links, notes, and images. Includes full-text search and optional AI tagging and summaries. |
| [changedetection.io](https://github.com/dgtlmoon/changedetection.io) | 35,317 | 2026-10-27 | Watches selected websites for changes. Track release notes, documentation, and security advisories without repeatedly checking each page. |
| [Beszel](https://github.com/henrygd/beszel) | 26,575 | 2026-10-26 | Lightweight server monitoring with historical CPU, memory, disk I/O, container metrics, and alerts. Inspect server conditions around builds and tests. |
| [Hermes Agent](https://github.com/NousResearch/hermes-agent) | 257,531 | 2026-10-27 | A persistent assistant with memory, reusable skills, session search, and scheduled tasks. Explore recurring research and personal automation. |
| [Cloudflare Security Audit Skill](https://github.com/cloudflare/security-audit-skill) | 49,031 | 2026-09-15 | Organizes security investigation, candidate validation, and independent verification into structured findings. Consider it for occasional focused code audits. |

The project links above are the primary sources for their features.
The possible uses are recommendations for this workflow, not results from integration tests.

## Current momentum

The following values came from GitHub Trending during the lookup on 2026-10-27:

| Project | Period | Stars gained | Source |
| --- | --- | ---: | --- |
| Cloudflare Security Audit Skill | Week | 16,593 | [Weekly Trending](https://github.com/trending?since=weekly) |
| Graphify | Week | 7,183 | [Weekly Trending](https://github.com/trending?since=weekly) |
| Context Mode | Month | 8,483 | [Monthly Trending](https://github.com/trending?since=monthly) |

Trending pages change continuously. These numbers record the observed snapshot.
The other projects in this note are practical recommendations. This note does not claim that all of them appeared on Trending.

## Suggested evaluation order

1. **QMD:** Index an existing collection of CUBRID notes and saved JIRA drafts.
   Search by meaning, retrieve the original documents, and check whether the results answer real questions.
2. **Difftastic:** Compare its output on a PR with substantial formatting changes.
   Check whether it makes the semantic changes easier to review.
3. **Graphify:** Map one CUBRID subsystem and inspect the relationships against known code paths.
   CUBRID compiles many `.c` files as C++. Check parser selection during the trial.

QMD and Difftastic offer focused experiments with existing documents and diffs.
Graphify requires an additional check that its parsed relationships are useful for this repository.
Its code analysis runs locally. Its optional semantic analysis of documents and media uses a model backend.
See the [Graphify README](https://github.com/Graphify-Labs/graphify#readme) for those boundaries.

## Choosing a larger application

- **Karakeep:** Choose it for collecting and retrieving articles, links, and research notes.
- **Windmill:** Choose it for running recurring scripts and providing interfaces for those scripts.
- **Beszel:** Choose it for historical server metrics and alerts.
- **changedetection.io:** Choose it for monitoring specific web pages.
- **Hermes Agent:** Consider it when persistent assistant memory and scheduled agent tasks solve a recurring need.

Worktrunk overlaps an existing worktree workflow, so compare its navigation and setup features before changing that workflow.
Context Mode adds agent integration. Check its current platform support before planning a trial.
See its [compatibility documentation](https://github.com/mksglu/context-mode#platform-compatibility).

## Evidence and limits

- Reviewed the projects' documentation and GitHub repository activity.
- Read GitHub Trending directly for the three momentum figures above.
- Recorded repository metadata through the GitHub API.
- Did not install these projects or test their integration with CUBRID.
- Did not independently reproduce project performance claims.

To refresh a repository's star count and last-push timestamp:

```bash
gh api repos/tobi/qmd \
  --jq '{stars: .stargazers_count, pushed_at: .pushed_at}'
```

The API timestamp uses UTC. Convert it to KST before updating the table.

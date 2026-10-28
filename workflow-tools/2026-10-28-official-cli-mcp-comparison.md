# Official CLIs and MCP servers for your CUBRID workflow

Checked on **2026-10-28**. Repository timestamps use UTC. Statistics were collected through the GitHub REST API. The exact collection timestamps and default-branch commits are in the [repository snapshot](./2026-10-28-official-cli-mcp-comparison-assets/github-repository-snapshot-2026-10-28.json).

This comparison focuses on Teams Issue Task reading, CUBRID PR and CI investigation, browser automation, and Microsoft integration work. It covers 14 vendor-owned tools or tool families. Two PnP community tools and one retired Microsoft tool appear separately.

**My recommendation:** start with Playwright CLI for the proposed Teams Web reader. Keep GitHub CLI for exact PR and CI evidence. Add Microsoft Learn MCP or its CLI when you need Microsoft documentation. Playwright MCP is an alternative browser interface. Chrome DevTools is useful when browser behavior needs diagnosis.

Usefulness ratings are my judgment for your workflow. They are not a vendor benchmark. No company Teams access or message-collection workflow was tested in this research.

**Purpose, target users, access, and limitations**

| Tool and interface | Purpose | Target users | Access and cost | Limitations | Usefulness to you |
| --- | --- | --- | --- | --- | --- |
| [Playwright CLI](https://playwright.dev/agent-cli/introduction) — CLI | Control a browser through shell commands and page snapshots. | Coding agents, browser automation developers, and test engineers. | No Graph app registration for browser use. Teams still requires your account and permitted sign-in. | Teams reading is a proposed browser workflow. It must expand replies and load older posts. UI changes can break collection. | **High**. Start here — read Issue Task and retain original post links. |
| [Playwright MCP](https://github.com/microsoft/playwright-mcp) — MCP | Expose browser navigation and interaction as MCP tools. | AI users whose host supports MCP browser tools. | Uses a browser session. Connecting to an existing browser requires its supported connection setup. | Needs an MCP host and a browser. Its Teams history coverage needs the same validation as CLI automation. | **High**. Useful alternative to Playwright CLI when the AI host works better with MCP. |
| [GitHub CLI](https://cli.github.com/manual/) — CLI (gh) | Query PRs, reviews, Actions runs, failed logs, artifacts, and GitHub APIs. | Developers, maintainers, CI engineers, and shell-capable AI agents. | GitHub sign-in or a suitable token. Repository and organization policies apply. | API permissions, rate limits, pagination, and artifact retention still apply. It does not read Teams. | **High**. Daily use — correlate Teams failure reports with CUBRID PRs and exact CI runs. Already available on this server. |
| [GitHub MCP Server](https://github.com/github/github-mcp-server) — MCP (local or hosted) | Give AI structured access to repositories, PRs, issues, reviews, and Actions. | Developers using an MCP-compatible assistant. | GitHub OAuth, a suitable token, or another documented server authentication method. Host and organization policies apply. | Configured tools and token permissions bound access. A host may already provide equivalent GitHub tools. | **High**. High when GitHub tools are missing from the AI host. This conversation already has a GitHub connection. |
| [Microsoft Learn CLI + MCP](https://github.com/MicrosoftDocs/mcp) — CLI (mslearn) + hosted MCP | Search official Microsoft documentation, fetch articles, and find code samples. | Developers and agents working with Microsoft products and APIs. | Public documentation service. No sign-in or API key. The MCP endpoint has no service charge. | Public Microsoft documentation only. It cannot read private Teams posts, company files, or CUBRID's local manual. | **High for Microsoft integration work**. Useful now — verify Teams, Graph, authentication, and Azure instructions while building integrations. |
| [Chrome DevTools for agents](https://github.com/ChromeDevTools/chrome-devtools-mcp) — CLI (chrome-devtools) + MCP | Inspect browser console errors, network requests, performance traces, and page behavior. | Web developers and agents diagnosing browser problems. | A Chrome instance. Existing-session attachment needs remote debugging setup and browser permission. | Official browser support is Chrome. It diagnoses browser performance, not CUBRID engine performance. Usage statistics are on by default. | **Medium**. Useful backup — diagnose Teams Web or QA portal automation when browser behavior is unclear. |
| [Work IQ CLI + MCP](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/cli) — CLI (workiq) + MCP | Retrieve Microsoft 365 work context, including Teams messages, email, meetings, and documents. | Employees and AI developers using approved Microsoft 365 integrations. | Microsoft Entra tenant consent and configured usage billing. Uses the signed-in user's permitted data. | Company enablement is required. Natural-language answers do not prove a complete channel archive. Delegated authentication constrains unattended operation. | **Conditional**. Consider after company enablement — convenient cross-source summaries without maintaining a browser collector. |
| [Microsoft Graph PowerShell](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview) — Command-line SDK / PowerShell cmdlets | Call Microsoft Graph with structured commands, including Teams channel messages and replies. | Microsoft 365 developers, administrators, and automation engineers. | PowerShell and an approved Graph identity. Channel reading requires the applicable Graph permissions and consent. | Requires explicit pagination, reply collection, and update handling. A command-line wrapper does not remove Graph consent requirements. | **Conditional**. Useful after approval — build a structured collector when browser extraction becomes insufficient. |
| [Teams Developer CLI](https://github.com/microsoft/teams-sdk/tree/main/packages/cli) — CLI (teams / @microsoft/teams.cli) | Create Teams app projects, register agents, configure endpoints, and diagnose app setup. | Developers building a Teams app or bot. | Teams sign-in and permission to create and install the intended app. Extra Graph scopes require their own consent. | It manages Teams app development. It does not itself provide a ready-made archive of existing channel messages. | **Low for the current task**. Later — useful if you turn the Issue Task reader into a shared Teams bot. |
| [Microsoft 365 Agents Toolkit CLI](https://learn.microsoft.com/en-us/microsoftteams/platform/toolkit/teams-toolkit-cli) — CLI (atk) | Scaffold, provision, package, test, and publish Microsoft 365 apps and agents. | Developers building Microsoft 365 or Copilot applications. | Microsoft 365 development permissions. Azure credentials and resources apply when the chosen project uses Azure. | Project-development tooling adds little to a personal channel-reading workflow. Deployment and Graph permissions still need approval. | **Low for the current task**. Later — use only when building a maintained Microsoft 365 application. |
| [Agent 365 CLI](https://learn.microsoft.com/en-us/microsoft-agent-365/developer/agent-365-cli) — CLI (a365) | Create and manage enterprise agent identities, blueprints, deployment, and governance setup. | Enterprise agent developers and tenant administrators. | Agent 365 tenant setup, suitable Entra roles, and capability-specific consent or licensing. Azure deployment requires Azure access. | The setup cost is substantial for a personal reader. The open-source CLI does not make the backing platform unrestricted or free. | **Low**. Skip initially — relevant if CUBRID adopts managed enterprise agents. |
| [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli) — CLI (az) | Inspect and manage Azure resources, identities, deployments, and cloud configuration. | Azure developers, operators, and infrastructure automation engineers. | Azure sign-in and the required resource permissions. Provisioned Azure services can incur charges. | Azure scope. It does not provide a direct Teams channel-reading command. | **Low unless you use Azure**. Optional — useful if the collector or bot is hosted on Azure. |
| [Azure MCP Server](https://github.com/microsoft/mcp) — MCP | Expose Azure resource and service operations as tools for AI agents. | Developers and operators who use AI to work with Azure. | Azure credentials and the required resource permissions. Backing cloud services can incur charges. | Azure operations, rather than Microsoft 365 channel history. Available tools depend on configuration and identity. | **Low unless you use Azure**. Optional — useful only if Azure becomes part of your CUBRID infrastructure workflow. |
| [Work IQ Developer Tools](https://github.com/microsoft/wiqd) — CLI (wiqd) | Create, validate, evaluate, provision, and publish Microsoft 365 Copilot plugins and agents. | Developers building Copilot extensions and their delivery workflows. | Microsoft 365 tenant access and provisioning entitlements. The announcement requires a Copilot license for provisioning. | Experimental preview with no official product support. CLI source remains private. The installer uses Agents Toolkit as a dependency. | **Low for the current task**. Later — relevant if you develop a Copilot plugin. It adds little to reading existing Issue Task posts. |

**Open source and GitHub statistics**

Created means the repository creation timestamp. Last code update means the committer timestamp of the latest commit on the default branch. It does not prove a recent production deployment. The creation dates link to each repository API record. The update dates link to the exact commits.

GitHub updated_at and pushed_at are retained in the JSON snapshot, but neither is presented as the default-branch code-update date. Stars are a point-in-time count.

| Tool | Open source and license | Repo created | Last code update | Stars | Scope of repository statistics |
| --- | --- | --- | --- | --- | --- |
| [Playwright CLI](https://github.com/microsoft/playwright-cli) | Yes — Apache-2.0 | [2020-06-19](https://api.github.com/repos/microsoft/playwright-cli) | [2026-10-19](https://github.com/microsoft/playwright-cli/commit/b1a5903c5b633b8c9f843ad7b629c2a78437040c) | 14,688 | CLI repository age includes earlier project history. Creation is not the launch date of the current agent CLI. |
| [Playwright MCP](https://github.com/microsoft/playwright-mcp) | Yes — Apache-2.0 | [2025-03-21](https://api.github.com/repos/microsoft/playwright-mcp) | [2026-10-22](https://github.com/microsoft/playwright-mcp/commit/0c4107acbecb75472b23c05c0e42471a0011874f) | 38,726 | Repository statistics describe the public MCP implementation, not a hosted service. |
| [GitHub CLI](https://github.com/cli/cli) | Yes — MIT | [2019-10-03](https://api.github.com/repos/cli/cli) | [2026-10-27](https://github.com/cli/cli/commit/93d24ce9ffefb6cc86bde89b50271457bb0ef9cb) | 46,769 | Statistics describe the GitHub CLI implementation. |
| [GitHub MCP Server](https://github.com/github/github-mcp-server) | Yes — MIT server implementation | [2025-03-04](https://api.github.com/repos/github/github-mcp-server) | [2026-10-22](https://github.com/github/github-mcp-server/commit/f38526264abb14961b32cea386e38ec6d3cf674f) | 34,017 | Public server source is MIT. Hosted deployment and GitHub platform access remain services. |
| [Microsoft Learn CLI + MCP](https://github.com/MicrosoftDocs/mcp) | CLI code: MIT. Documentation: CC-BY-4.0. Hosted server implementation: not established as public source. | [2025-06-09](https://api.github.com/repos/MicrosoftDocs/mcp) | [2026-09-10](https://github.com/MicrosoftDocs/mcp/commit/f8ffde185dfd232dbf5d187c22ce299eadc3d583) | 1,956 | Shared CLI, documentation, and plugin repository. Repository activity is not the deployment date of the hosted MCP service. |
| [Chrome DevTools for agents](https://github.com/ChromeDevTools/chrome-devtools-mcp) | Yes — Apache-2.0 | [2025-09-11](https://api.github.com/repos/ChromeDevTools/chrome-devtools-mcp) | [2026-10-27](https://github.com/ChromeDevTools/chrome-devtools-mcp/commit/a24334d3a1733996438c3c4786ab4f880aaaf044) | 54,313 | CLI and MCP ship from the same repository and package. |
| [Work IQ CLI + MCP](https://github.com/microsoft/work-iq) | Public plugin/skill repository: MIT. Work IQ CLI and service: proprietary / EULA. | [2026-01-20](https://api.github.com/repos/microsoft/work-iq) | [2026-10-19](https://github.com/microsoft/work-iq/commit/66f62b1467882815de4f571454c209eaac210dc8) | 1,043 | Stars and dates describe the public plugin repository, not the proprietary CLI implementation or hosted service. |
| [Microsoft Graph PowerShell](https://github.com/microsoftgraph/msgraph-sdk-powershell) | Yes — MIT (verified license text) | [2019-03-15](https://api.github.com/repos/microsoftgraph/msgraph-sdk-powershell) | [2026-10-21](https://github.com/microsoftgraph/msgraph-sdk-powershell/commit/f85309cb3f770a1583ddc3bd7522f14d23e4a7d0) | 904 | GitHub's SPDX detector returned NOASSERTION. The actual LICENSE.txt states MIT. |
| [Teams Developer CLI](https://github.com/microsoft/teams-sdk) | Yes — MIT CLI package | [2023-02-06](https://api.github.com/repos/microsoft/teams-sdk) | [2026-10-27](https://github.com/microsoft/teams-sdk/commit/a5af5fa5fe3d0afda6490373d4bc466ef3597694) | 732 | Shared Teams SDK repository. CLI source is under packages/cli. Dates are not the CLI launch date. |
| [Microsoft 365 Agents Toolkit CLI](https://github.com/OfficeDev/microsoft-365-agents-toolkit) | Yes — MIT (verified license text) | [2021-03-16](https://api.github.com/repos/OfficeDev/microsoft-365-agents-toolkit) | [2026-10-28](https://github.com/OfficeDev/microsoft-365-agents-toolkit/commit/edabc9b49e74ab66bc4b3b2f32cfe552a7187ff6) | 784 | Toolkit monorepo, including extension and CLI history. GitHub's SPDX detector returned NOASSERTION despite MIT license text. |
| [Agent 365 CLI](https://github.com/microsoft/Agent365-devTools) | Yes — MIT CLI source | [2025-11-05](https://api.github.com/repos/microsoft/Agent365-devTools) | [2026-10-21](https://github.com/microsoft/Agent365-devTools/commit/4fd85ec798a096d5b5c2c2591fd301dea27c8098) | 61 | Statistics describe the public development CLI, not the enterprise Agent 365 service. |
| [Azure CLI](https://github.com/Azure/azure-cli) | Yes — MIT | [2016-02-04](https://api.github.com/repos/Azure/azure-cli) | [2026-10-27](https://github.com/Azure/azure-cli/commit/af86318e68be54c937fa0012c3a18a99222ee3fa) | 4,657 | Statistics describe Azure CLI, not every optional extension or Azure service. |
| [Azure MCP Server](https://github.com/microsoft/mcp) | Yes — MIT | [2025-04-09](https://api.github.com/repos/microsoft/mcp) | [2026-10-27](https://github.com/microsoft/mcp/commit/3f9f1899ec4bf95460c41a0ad530ed8d76700c4d) | 3,803 | Shared Microsoft MCP monorepo. Stars and commit dates cover more than the Azure MCP component. |
| [Work IQ Developer Tools](https://github.com/microsoft/wiqd) | CLI source unavailable during preview. Public repository and distributed package declare MIT. | [2026-05-28](https://api.github.com/repos/microsoft/wiqd) | [2026-10-26](https://github.com/microsoft/wiqd/commit/1f226f5a31cca3e3c99781b031c8958c35614f7b) | 40 | Public documentation, download, and metadata repository. Its MIT license does not provide the private CLI source. |

**Important license and support distinctions**

- Work IQ: the public plugin and skill repository has an MIT license. The published CLI package uses a separate EULA, and Microsoft describes the tool as proprietary. [CLI terms and feedback section](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/cli#provide-feedback), [package metadata](https://registry.npmjs.org/@microsoft%2fworkiq/latest).

- Microsoft Learn: the public CLI implementation uses MIT, while documentation uses CC-BY-4.0. The public repository does not establish that the hosted MCP server implementation is open source. [Code license](https://github.com/MicrosoftDocs/mcp/blob/main/LICENSE-CODE), [CLI manifest](https://github.com/MicrosoftDocs/mcp/blob/main/cli/package.json).

- WIQD: the distribution and public repository declare MIT, but the CLI source remains private during preview. The README also labels the preview experimental and without official support. Microsoft ownership and product support are separate facts. [WIQD README](https://github.com/microsoft/wiqd/blob/main/README.md).

- Graph PowerShell, Agents Toolkit, and Work IQ repository metadata returned SPDX NOASSERTION. The actual license files contain MIT text. This comparison uses those files rather than treating the automated detection result as the license. [Graph PowerShell license](https://github.com/microsoftgraph/msgraph-sdk-powershell/blob/main/LICENSE.txt), [Toolkit license](https://github.com/OfficeDev/microsoft-365-agents-toolkit/blob/dev/LICENSE), [Work IQ repository license](https://github.com/microsoft/work-iq/blob/main/LICENSE).

**What the no-admin-approval route means for Teams**

Browser automation can operate through your normal Teams Web sign-in without creating a separate Graph application. That is a proposed design for this task, rather than a verified Teams connector. Your account permissions, MFA, session expiry, and tenant policies still apply. Playwright supports persistent profiles and saving authenticated state. [Playwright sessions](https://playwright.dev/agent-cli/sessions).

Microsoft Learn genuinely needs no authentication, but it reads public documentation. It cannot access Issue Task. [Microsoft Learn requirements and limits](https://learn.microsoft.com/en-us/training/support/mcp).

A durable Teams collector also needs a schedule, a record of collected messages, reply expansion, update handling, and original links. A successful summary does not establish that every channel post was collected. The Graph channel API also separates root messages, replies, and pagination. [Channel API documentation](https://learn.microsoft.com/en-us/graph/api/channel-list-messages?view=graph-rest-1.0).

For the remote Linux server, run browser automation headlessly. Complete any required human sign-in or browser-attachment approval through an appropriate local browser or remote-viewing setup. Do not assume a desktop browser session automatically exists on the server.

**Related PnP community projects**

PnP includes Microsoft employees and community contributors, but its project description explicitly distinguishes it from a Microsoft-supported product. [Project statement](https://github.com/pnp/cli-microsoft365#Microsoft-365--Power-Platform-Community).

| Tool | Purpose and target users | Access and limitations | Usefulness |
| --- | --- | --- | --- |
| [CLI for Microsoft 365](https://github.com/pnp/cli-microsoft365) | Broad Microsoft 365 administration from the m365 CLI. Microsoft 365 administrators and automation developers. | Standard setup requires your Entra app and the relevant scopes. Channel-reading consent requirements still apply. | Medium after approved API access. It is not a general way to remove admin consent. |
| [CLI for Microsoft 365 MCP Server](https://github.com/pnp/cli-microsoft365-mcp-server) | Discover and execute m365 commands through MCP. AI-assisted Microsoft 365 administrators. | Requires the m365 CLI and its existing sign-in. The MCP adapter does not authenticate or expand permissions for you. | Low to medium if you already adopt m365. |

| Tool | Open source and license | Repo created | Last code update | Stars | Publisher and scope |
| --- | --- | --- | --- | --- | --- |
| [CLI for Microsoft 365](https://github.com/pnp/cli-microsoft365) | Yes — MIT | [2017-10-12](https://api.github.com/repos/pnp/cli-microsoft365) | [2026-10-23](https://github.com/pnp/cli-microsoft365/commit/41515e723c0ce7dafb33245178681a037ee30e32) | 1,471 | PnP community project. Not a Microsoft-supported product. |
| [CLI for Microsoft 365 MCP Server](https://github.com/pnp/cli-microsoft365-mcp-server) | Yes — MIT | [2025-04-17](https://api.github.com/repos/pnp/cli-microsoft365-mcp-server) | [2026-10-12](https://github.com/pnp/cli-microsoft365-mcp-server/commit/9b6863daef5bb093c70540479092a8c160a657be) | 135 | PnP community MCP adapter. |

**Retired Microsoft tool**

| Tool | Open source and license | Repo created | Last code update | Stars | Status |
| --- | --- | --- | --- | --- | --- |
| [Microsoft Graph CLI](https://github.com/microsoftgraph/msgraph-cli) | Yes — MIT | [2021-11-26](https://api.github.com/repos/microsoftgraph/msgraph-cli) | [2025-06-11](https://github.com/microsoftgraph/msgraph-cli/commit/5233f10a6e8d85d9b0a7692f9c2d08a62ae0485a) | 132 | Archived. Announced retirement date: 2026-08-28. Avoid for new work. |

Microsoft announced retirement of Graph CLI and recommends Graph PowerShell. The announced retirement date passed before this snapshot. [Retirement notice](https://devblogs.microsoft.com/microsoft365dev/microsoft-graph-cli-retirement/).

**Primary sources for each vendor-owned tool**

- **Playwright CLI** (Microsoft): [Source 1](https://playwright.dev/agent-cli/introduction) [Source 2](https://playwright.dev/agent-cli/sessions) [Source 3](https://github.com/microsoft/playwright-cli/blob/main/LICENSE)

- **Playwright MCP** (Microsoft): [Source 1](https://github.com/microsoft/playwright-mcp) [Source 2](https://github.com/microsoft/playwright-mcp/blob/main/LICENSE)

- **GitHub CLI** (GitHub): [Source 1](https://cli.github.com/manual/) [Source 2](https://cli.github.com/manual/gh_run_view) [Source 3](https://github.com/cli/cli/blob/trunk/LICENSE)

- **GitHub MCP Server** (GitHub): [Source 1](https://github.com/github/github-mcp-server) [Source 2](https://github.com/github/github-mcp-server/blob/main/docs/host-integration.md) [Source 3](https://github.com/github/github-mcp-server/blob/main/LICENSE)

- **Microsoft Learn CLI + MCP** (Microsoft): [Source 1](https://github.com/MicrosoftDocs/mcp) [Source 2](https://learn.microsoft.com/en-us/training/support/mcp) [Source 3](https://github.com/MicrosoftDocs/mcp/blob/main/cli/package.json) [Source 4](https://github.com/MicrosoftDocs/mcp/blob/main/LICENSE-CODE) [Source 5](https://github.com/MicrosoftDocs/mcp/blob/main/LICENSE)

- **Chrome DevTools for agents** (Google / Chrome DevTools): [Source 1](https://github.com/ChromeDevTools/chrome-devtools-mcp) [Source 2](https://developer.chrome.com/docs/devtools/agents/get-started/configuration) [Source 3](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/main/LICENSE)

- **Work IQ CLI + MCP** (Microsoft): [Source 1](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/cli) [Source 2](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/api-overview) [Source 3](https://github.com/microsoft/work-iq/blob/main/LICENSE)

- **Microsoft Graph PowerShell** (Microsoft Graph): [Source 1](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview) [Source 2](https://learn.microsoft.com/en-us/graph/api/channel-list-messages?view=graph-rest-1.0) [Source 3](https://github.com/microsoftgraph/msgraph-sdk-powershell/blob/main/LICENSE.txt)

- **Teams Developer CLI** (Microsoft Teams): [Source 1](https://github.com/microsoft/teams-sdk/tree/main/packages/cli) [Source 2](https://github.com/microsoft/teams-sdk/blob/main/packages/cli/package.json) [Source 3](https://devblogs.microsoft.com/microsoft365dev/from-prompt-to-production-teams-agent-setup-simplified/)

- **Microsoft 365 Agents Toolkit CLI** (Microsoft / OfficeDev): [Source 1](https://learn.microsoft.com/en-us/microsoftteams/platform/toolkit/teams-toolkit-cli) [Source 2](https://github.com/OfficeDev/microsoft-365-agents-toolkit/blob/dev/LICENSE)

- **Agent 365 CLI** (Microsoft): [Source 1](https://learn.microsoft.com/en-us/microsoft-agent-365/developer/agent-365-cli) [Source 2](https://learn.microsoft.com/en-us/microsoft-agent-365/developer/get-started) [Source 3](https://github.com/microsoft/Agent365-devTools/blob/main/LICENSE)

- **Azure CLI** (Microsoft Azure): [Source 1](https://learn.microsoft.com/en-us/cli/azure/what-is-azure-cli) [Source 2](https://github.com/Azure/azure-cli/blob/dev/LICENSE)

- **Azure MCP Server** (Microsoft Azure): [Source 1](https://github.com/microsoft/mcp) [Source 2](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/overview) [Source 3](https://github.com/microsoft/mcp/blob/main/LICENSE)

- **Work IQ Developer Tools** (Microsoft): [Source 1](https://github.com/microsoft/wiqd) [Source 2](https://github.com/microsoft/wiqd/blob/main/README.md) [Source 3](https://devblogs.microsoft.com/microsoft365dev/announcing-the-preview-of-the-work-iq-developer-tools/)

**Saved research and verification**

- [Machine-readable tool catalog](./2026-10-28-official-cli-mcp-comparison-assets/official-tool-catalog.json).

- [GitHub repository snapshot](./2026-10-28-official-cli-mcp-comparison-assets/github-repository-snapshot-2026-10-28.json).

- [Microsoft 365 source evidence and discrepancies](./2026-10-28-official-cli-mcp-comparison-assets/m365-tool-evidence.md).

The research used public source repositories, official product documentation, package metadata, and GitHub REST responses. No tools were installed, no accounts were signed in, and no external messages or cloud resources were changed.

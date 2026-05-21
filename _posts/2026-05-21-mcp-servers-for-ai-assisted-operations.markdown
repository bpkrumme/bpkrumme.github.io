---
layout: single
title:  "MCP Servers: Giving AI Assistants Real Context"
date:   2026-05-21
header:
  og_image: /assets/images/bdt_square.jpeg
---

## Let's set the stage

If you've been following along, you already know my homelab is built to represent an enterprise environment as closely as I can manage.  That means Red Hat Openshift Container Platform for the platform layer and Ansible Automation Platform for orchestration.  I've already covered what the hardware in my homelab looks like, how I enable AAP on Openshift via the operator, and quite a bit of automation content along the way.

Over the last several months I've also been spending a lot of time in [Cursor](https://cursor.com/) doing AI-assisted development.  At some point it clicked that I could connect those two worlds using ___Model Context Protocol (MCP)___ servers, so an assistant in the IDE could work with my real infrastructure instead of guessing.  That's what this post is about.

## What is MCP?

The [Model Context Protocol](https://modelcontextprotocol.io/) is an open standard that lets large language models call external tools through a consistent interface.  I like to think of it as a structured API between your AI assistant and the systems you already operate.

Instead of copying and pasting the output from `oc get pods` into a chat window, the assistant can ask an MCP server to list pods, pull logs, or check the status of an automation job.  For the kind of work I do, that matters because demos, troubleshooting, and solution design all depend on accurate, current state.  MCP doesn't replace your expertise or your change-management process, but it can shorten the loop between "something is wrong" and "here is what the system actually shows right now."

Red Hat has been shipping MCP integrations across the portfolio.  At the time of writing you'll find developer-preview or technology-preview offerings for [RHEL](https://www.redhat.com/en/blog/smarter-troubleshooting-new-mcp-server-red-hat-enterprise-linux-now-developer-preview), [Openshift](https://github.com/openshift/openshift-mcp-server), [Ansible Automation Platform](https://www.redhat.com/en/blog/it-automation-agentic-ai-introducing-mcp-server-red-hat-ansible-automation-platform), and [Red Hat Lightspeed](https://developers.redhat.com/articles/2025/12/01/how-set-red-hat-lightspeed-model-context-protocol).  I focused on the three that map directly to how I already run my lab.

## The three MCP servers in my environment

I run all three servers against the same Openshift cluster that hosts my AAP instance, `ocp.bk.lab`.  Each server is exposed as an HTTPS endpoint on the cluster, and Cursor connects to them over HTTP transport rather than the `stdio` + Podman pattern you'll see in a lot of the upstream getting-started guides.  That choice keeps the MCP processes close to the APIs they call and matches how I deploy other lab services.

Here's how they break down:

* ___aap-mcp___ — Query and drive Ansible Automation Platform.  Roughly 100 tools covering job templates, jobs, inventories, credentials, organizations, settings, and more.
* ___openshift-mcp___ — Kubernetes and Openshift cluster operations and observability.  Roughly 35 tools for pods, nodes, namespaces, resources, metrics, events, and OpenShift Virtualization lifecycle.
* ___rhel-mcp___ (linux-mcp-server) — Read-only Linux host diagnostics.  Roughly 20 tools for system info, services, processes, network, storage, and logs.

Together they cover the three layers I already think about every day: the ___platform___ (Openshift), ___automation___ (AAP), and ___hosts___ (RHEL).

### Ansible Automation Platform MCP

The [AAP MCP server](https://github.com/ansible/aap-mcp-server) exposes the platform's REST API as MCP tools.  In practice that means an assistant in Cursor can list job templates, launch jobs, retrieve job stdout, inspect inventories, and do many of the same things I would run through the UI or the controller CLI.

This is the server I reach for most often when working on ___Configuration as Code___ and CASC-style workflows.  A concrete example from recent work: after adding a new job template definition for the Server organization subtree, I used the MCP tools to search for matching templates, launch the CASC dispatch job with the right `extra_vars`, and pull job events and stdout when the first run failed because the SCM project had not synced yet.  I didn't have to copy-paste API responses into the chat.  The assistant called `job_templates_list`, `job_templates_launch_create`, and `jobs_stdout_retrieve` through the protocol.

Red Hat documents the server as a technology preview in AAP 2.6, including guidance for [containerized deployment](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/deploying-ansible-mcp-server) alongside the platform.  Security is layered.  The server respects AAP RBAC, and operators can constrain read-only versus read-write behavior depending on how much autonomy you want an agent to have.

### Openshift MCP

The [openshift-mcp-server](https://github.com/openshift/openshift-mcp-server) project provides native Go implementations of cluster operations rather than wrapping `oc` or `kubectl`.  My deployment exposes tools for the everyday cluster work I actually do: listing and inspecting pods, pulling logs, exec into containers, managing generic resources, querying Prometheus-style metrics, working with Openshift projects, and even OpenShift Virtualization helpers for VM create, clone, and lifecycle.

The server also ships prompts such as `cluster-health-check` and `vm-troubleshoot` that bundle multiple tool calls into guided workflows.  Those are handy when you want the assistant to follow a structured diagnostic path instead of improvising one tool at a time.

Because this server authenticates against the cluster API, I issue a dedicated bearer token with scoped permissions rather than handing it cluster-admin privileges.

### RHEL MCP (linux-mcp-server)

The [linux-mcp-server](https://github.com/rhel-lightspeed/linux-mcp-server) is intentionally read-only.  It wraps standard Linux utilities and returns formatted results for system information, systemd services, processes, network state, block devices, and journal or file logs.  Log access can be restricted with configurable allowlists via `LINUX_MCP_ALLOWED_LOG_PATHS`.

Every tool accepts an optional `host` parameter, so the same server can look at my laptop, my lab core NUC, or a VM over SSH without installing agents on each machine.  That remote execution model is what Red Hat describes in [root-cause analysis article for VS Code and Cursor](https://developers.redhat.com/articles/2026/02/11/leverage-ai-root-cause-analysis-mcp-servers-vs-code-and-cursor).  The assistant gets eyes on real journal entries and service state instead of guessing at a likely cause.

## Connecting Cursor to the servers

Cursor reads MCP configuration from `~/.cursor/mcp.json`.  Mine defines three HTTP endpoints on the lab cluster.  The structure looks like this.  The URLs and tokens below are examples; substitute your own.

```json
{
  "mcpServers": {
    "aap-mcp": {
      "type": "http",
      "url": "https://production-mcp-aap.apps.ocp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer <your-aap-mcp-token>"
      }
    },
    "rhel-mcp": {
      "type": "http",
      "url": "https://linux-mcp-server-rhel-mcp.apps.ocp.example.com/mcp"
    },
    "openshift-mcp": {
      "type": "http",
      "url": "https://openshift-mcp-server-openshift-mcp.apps.ocp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer <your-openshift-token>"
      }
    }
  }
}
```

You can also add servers through Cursor Settings → Tools & MCP, which is how I turned them on initially before settling on the JSON file for reproducibility.

A few things I learned while getting this running:

* Treat tokens like passwords.  The AAP and Openshift servers expect bearer credentials.  Store them in Cursor's configuration or a secrets mechanism, not in blog posts or public git repositories.
* Match transport to deployment.  Upstream docs often show `stdio` with Podman on your workstation.  I chose in-cluster HTTP because my assistants and my platform already live in the same lab environment.
* Enable only what you need per project.  Cursor lets you turn individual MCP servers on or off.  When I'm editing Jekyll content for blog posts, I don't need AAP.  When I'm working on CASC tree automation I enable `aap-mcp` and often `rhel-mcp` together.

## How I use them day to day

These aren't theoretical integrations.  They've changed how I work in three recurring patterns.

### Automation engineering with AAP MCP

When building or extending Configuration as Code content, I stay in the IDE and ask the assistant to inspect live controller state.  Which job templates exist for an organization?  What did a recent job return?  Do inventory sources need a refresh?  That keeps the feedback loop in the same session where I'm editing YAML, instead of context-switching to the AAP UI for every verification step.

The workflow is still governed by normal software practices.  The assistant proposes API calls, I review what it launched, and failed jobs still require human judgment about whether the fix is SCM sync, credential scope, or playbook logic.

### Cluster operations with Openshift MCP

For Openshift troubleshooting I use pod and event tools for quick health checks, metrics queries when latency or resource pressure is suspected, and the bundled health-check prompt when I want a wider sweep across namespaces.  This pairs naturally with the AAP operator content I've written about previously.  The same cluster that runs AAP also hosts the MCP bridge.

### Host diagnostics with RHEL MCP

For RHEL VMs and bare-metal systems in the lab, I ask questions in plain language and let the assistant pull service status and journal entries via MCP.  Because the server is read-only, I'm comfortable letting it gather data aggressively while I retain control over any remediation commands.

## Security and governance

MCP doesn't bypass your security model.  It exposes it to a new consumer, the assistant.

* ___AAP___ inherits RBAC from the platform.  Limit tokens to the organizations and job templates an automation persona should touch.
* ___Openshift___ should use a dedicated ServiceAccount or user with the narrowest viable cluster role.
* ___RHEL___ stays read-only by design, with SSH keys mounted or forwarded according to the [linux-mcp-server documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/using-the-rhel-mcp-server-to-enable-ai-assistants-to-run-discover-and-troubleshoot-complex-issues).

I also keep MCP configuration out of source control.  My `mcp.json` lives only on the workstation where Cursor runs.

### Final Thoughts

MCP sits at the intersection of two things I care about professionally: enterprise automation and AI-assisted operations.  Ansible Automation Platform already centralizes how organizations run approved automation.  MCP gives assistants a governed path into that same control plane.  Openshift and RHEL MCP servers extend that idea to the platform and operating system layers.

I'm not suggesting you turn production changes over to an assistant on a whim.  I am saying that if you already operate AAP, Openshift, and RHEL together, MCP servers are a practical way to make AI tooling aware of that stack, with RBAC and read-only boundaries intact.

I plan to write follow-up posts that go deeper on individual servers: deploying the AAP MCP pod alongside the operator-managed platform, hardening Openshift MCP tokens, and tuning RHEL MCP log allowlists for homelab hosts.  If you're experimenting with MCP in your own environment, start with one server and one narrow use case, then expand only after you're comfortable with the security posture.

For more detail from Red Hat, see the posts on [the AAP MCP server](https://www.redhat.com/en/blog/it-automation-agentic-ai-introducing-mcp-server-red-hat-ansible-automation-platform) and [MCP for root-cause analysis in VS Code and Cursor](https://developers.redhat.com/articles/2026/02/11/leverage-ai-root-cause-analysis-mcp-servers-vs-code-and-cursor).  The upstream projects live on GitHub at [openshift/openshift-mcp-server](https://github.com/openshift/openshift-mcp-server) and [rhel-lightspeed/linux-mcp-server](https://github.com/rhel-lightspeed/linux-mcp-server).

As with everything I write about my lab, this is how ___I___ run things.  Your requirements, risk tolerance, and supported product versions may differ.  Use the previews accordingly, and keep secret data out of git.

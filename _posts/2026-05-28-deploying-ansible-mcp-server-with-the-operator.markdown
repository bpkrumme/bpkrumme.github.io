---
layout: single
title:  "Deploying the Ansible MCP Server with the Operator-Managed Platform"
date:   2026-05-28
header:
  og_image: /assets/images/bdt_square.jpeg
---

## Let's set the stage

Last week I published a post about ___Model Context Protocol (MCP)___ servers in my homelab and how I've been using them with Cursor to give AI assistants real context for Openshift, RHEL, and Ansible Automation Platform.  That write-up was deliberately broad.  I explained what MCP is, which three servers I run, and how they fit into my day-to-day work.  I also said I'd follow up with more detail on deploying each one.

This is the first of those follow-ups.

If you've already read about my MCP setup, you know my AAP instance runs on Openshift under the operator.  What I didn't walk through last week is how I enabled the ___Ansible MCP server___ on that same `production` instance in the `aap` namespace.  That's what we're covering today.  The good news is that if you already have the Ansible Automation Platform operator installed and an ___AnsibleAutomationPlatform___ custom resource in place, adding MCP is a small edit to that CR.  The operator handles the rest.

I'm only going to cover the command-line path here.  For instructions regarding the UI, refer to the official [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_on_openshift_container_platform/deploy-ansible-mcp-server-operator-install).

## What you're deploying

The Ansible MCP server is a ___technology preview___ feature in Ansible Automation Platform 2.6.  It acts as a secure bridge between an external AI client (Cursor, Claude, VS Code, and others) and the Ansible Automation Platform API.  When you ask an assistant to list job templates or launch a job, the MCP server validates the request and authenticates using ___your___ API token.  What the assistant can do is bounded by both server-level settings and the RBAC permissions on that token.

Red Hat documents two permission layers:

* ___Server-level___ — Set on the MCP component when you deploy it.  `allow_write_operations: false` is the default and enforces a read-only posture at the MCP layer even if a user's token could write.
* ___User-level___ — Inherited from the OAuth token you create in the platform.  A read-scoped token cannot launch jobs, regardless of what you ask the assistant to do.

Long story short, the MCP server does not bypass Ansible Automation Platform security.  It exposes it to a new kind of client.

## Prerequisites

Before you enable MCP on an existing operator deployment, you should have:

* Red Hat Openshift Container Platform with the Ansible Automation Platform operator installed and reporting a successful CSV
* A running ___AnsibleAutomationPlatform___ instance in your target namespace (mine is `production` in the `aap` namespace on `ocp.bk.lab`)
* Ansible Automation Platform 2.6 with a valid subscription attached to the instance
* Cluster-admin or sufficient privileges to edit the ___AnsibleAutomationPlatform___ CR and view routes/secrets in that namespace

The MCP feature is not something I'd run in production without understanding the technology preview terms.  Red Hat is explicit that these previews are for testing and feedback, not production SLAs.  My lab is the right place for it.

## Enable MCP on the AnsibleAutomationPlatform custom resource

When I wrote about the AAP operator earlier this year, my example instance was named `example` and lived in a tutorial-style namespace.  My lab has moved on since then.  Today I run an instance named `production` in the `aap` namespace, but the process for enabling MCP is the same regardless of what you named things.  You add an `mcp` block under `spec`.

1. Pull up your existing ___AnsibleAutomationPlatform___ manifest, or export the live resource.

    ```shell
    oc get ansibleautomationplatform production -n aap -o yaml > production-aap.yaml
    ```

2. Under `spec`, add the MCP component.  You can name the file something like _production-aap-mcp.yaml_ if you prefer a separate manifest for the change.

    ```yaml
    apiVersion: aap.ansible.com/v1alpha1
    kind: AnsibleAutomationPlatform
    metadata:
      name: production
      namespace: aap
    spec:
      image_pull_policy: IfNotPresent
      controller:
        disabled: false
      eda:
        disabled: false
      hub:
        disabled: false
        file_storage_size: 500Gi
        file_storage_storage_class: <your-read-write-many-storage-class>
      lightspeed:
        disabled: true
      mcp:
        disabled: false
        allow_write_operations: false
    ```

    Setting `mcp.disabled` to `false` tells the operator to deploy the MCP server alongside the platform components you already have enabled.  I'm starting with `allow_write_operations: false` so the server enforces read-only access at the MCP layer.  I'll flip that only when I have a specific reason to let an assistant launch jobs through MCP.

    If you want to see every field the operator supports for MCP, `oc explain` is your friend:

    ```shell
    oc explain ansibleautomationplatform.spec.mcp --recursive
    ```

3. Apply the updated custom resource.

    ```shell
    oc apply -f production-aap-mcp.yaml
    ```

4. Watch the namespace while the operator reconciles.  You should see MCP-related resources appear alongside your existing controller, hub, EDA, and gateway pods.

    ```shell
    watch "oc get pods -n aap | grep -E 'production|mcp'"
    ```

5. Confirm the MCP deployment exists.

    ```shell
    oc get deployment -n aap | grep mcp
    ```

    In my lab the route is named `production-mcp` and points at a deployment the operator created in the same `aap` namespace as the rest of the platform.  Your naming may differ slightly depending on operator version, but you should see a clear MCP deployment alongside `production-controller`, `production-hub`, and the other platform pods.

6. Check the pod logs if anything stays in a crash loop.  A clean startup should show the MCP server listening without certificate or API connection errors.

    ```shell
    oc logs -n aap deployment/<your-mcp-deployment-name> --tail=50
    ```

### Changing MCP permissions after the fact

If you change `allow_write_operations` on an instance that already has MCP running, Red Hat's documentation calls out an extra step.  You may need to delete the ___AnsibleMCPServer___ custom resource and let the operator recreate it.  In the UI that's under Resources, searching for ___AnsibleMCPServer___, selecting the instance with the `-mcp` suffix, and deleting it so reconciliation can build a fresh one.

I haven't had to do that yet because I left write access disabled from the start.  It's worth knowing before you toggle permissions on a live lab instance.

## Get the MCP route and verify connectivity

The operator creates an Openshift route for the MCP server the same way it does for the platform gateway and controller.  You need that URL to configure Cursor or any other MCP client.

1. List routes in the AAP namespace.

    ```shell
    oc get route -n aap
    ```

2. Look for the route whose name or host corresponds to your instance and MCP.  In my lab that's ___production-mcp-aap.apps.ocp.bk.lab___.  The platform gateway at ___production-aap.apps.ocp.bk.lab___ is a different route.  You want the MCP one, not the login page.

3. Verify the MCP endpoint responds.  The path your client needs is `/mcp` on that host.  A simple check from your workstation:

    ```shell
    curl -k -I "https://production-mcp-aap.apps.ocp.bk.lab/mcp"
    ```

    You should get an HTTP response from the route even before you've configured a token in your client.  An unauthorized response is fine at this stage.  It means the route and service are wired up.

## Create an API token for your AI client

The MCP server authenticates to Ansible Automation Platform using an OAuth token you create in the platform.  The assistant inherits whatever that user is allowed to do.

1. Log in to the Ansible Automation Platform UI with a user account that has the permissions you want the assistant to have.  For lab work I use a dedicated account rather than my admin user.

2. Navigate to ___Access Management → OAuth Applications___ (or create a personal access token if your workflow allows leaving the application field blank).

3. Create a token with an appropriate ___scope___:

    * ___Read___ if you only want the assistant to query job status, inventories, and logs
    * ___Write___ if you need the assistant to launch jobs or make changes, and your server-level `allow_write_operations` is set to allow it

4. Copy the token when it is displayed.  You will not see it again.

Store the token somewhere sensible.  I keep mine in Cursor's MCP configuration on my laptop, not in a git repository.

## Connect Cursor to the operator-managed MCP server

Once you have the route and the token, configuring Cursor is straightforward.  Add an entry to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "aap-mcp": {
      "type": "http",
      "url": "https://production-mcp-aap.apps.ocp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer <your-oauth-token>"
      }
    }
  }
}
```

Replace the host with your MCP route and paste the token you created in the previous section.

Red Hat's documentation also shows separate MCP URLs per toolset (job management, inventory management, system monitoring, and so on).  My operator deployment exposes a single `/mcp` endpoint on the instance route, which is simpler for Cursor.  If your environment presents multiple toolset URLs, add one `mcpServers` entry per toolset following the same pattern.  Keep each server name short.  The docs mention a 64-character limit when the client combines server name and tool name.

Restart or refresh MCP in Cursor, then ask something low-risk to confirm connectivity:

```text
What job templates are available in my Ansible Automation Platform?
```

If the server and token are correct, the assistant should call MCP tools and return real data from your controller.

## How this fits with the rest of my MCP setup

The Ansible MCP server is only one of three I run on the same Openshift cluster.  The AAP MCP server talks to the automation API.  ___openshift-mcp___ talks to the cluster API.  ___rhel-mcp___ talks to Linux hosts over SSH.  Deploying AAP MCP through the operator keeps it in the same namespace and lifecycle as the platform itself.  I'm not maintaining a separate manifest stack or a one-off Deployment just for MCP.

That co-location matters when I'm doing Configuration as Code work.  I edit job template YAML in the IDE, apply CASC changes through the platform, and ask the assistant to verify controller state through MCP without switching contexts.  I described that workflow in last week's post.  This one is the installation story behind it.

## Things that tripped me up

A few practical notes from getting this working in the lab:

* ___MCP is not a replacement for the UI or the CLI.___  The API is the same, but an LLM interprets responses before showing them to you.  If something looks different between the UI and the assistant, compare the underlying API result.
* ___Start read-only.___  Server-level `allow_write_operations: false` plus a read-scoped token is a good default.  Expand only when you're comfortable with what the assistant might launch.
* ___Point Cursor at the MCP route.___  My early mistake was putting the gateway URL in `mcp.json` instead of `production-mcp-aap.apps.ocp.bk.lab`.  The names are similar enough that it's an easy mix-up.
* ___Telemetry is on.___  Red Hat collects anonymized MCP telemetry on supported versions and it cannot be disabled.  No usernames or passwords are included, but you should be aware of it in regulated environments.
* ___Technology preview means change.___  Tool names, route patterns, and CR fields may shift between releases.  Pin your operator channel and read the release notes when you upgrade.

## Troubleshooting

When MCP does not connect, I work through this list:

1. ___Pod health___ — Is the MCP deployment running and are logs clean?
2. ___Route___ — Does `oc get route -n aap` show the MCP host you configured in `mcp.json`?
3. ___Token scope___ — Does the OAuth token have read or write scope matching what you're asking the assistant to do?
4. ___Server-level write flag___ — Even with a write token, `allow_write_operations: false` blocks mutating operations at the MCP layer.
5. ___RBAC___ — Can the token's user see the organization, inventory, or job template you're asking about?

Most of my early issues were stale tokens or pointing Cursor at the gateway route instead of the MCP route.

### Final Thoughts

Deploying the Ansible MCP server with the operator-managed platform is one of the smaller changes I've made in the homelab, but it has had an outsized impact on how I work with Configuration as Code and day-to-day automation tasks.  The operator already knows how to run AAP.  Adding `spec.mcp` just tells it to run the MCP bridge too.

If you read last week's MCP overview, consider this the AAP-specific follow-up I promised at the end of that post.  I'll try to keep the series moving with something on Openshift MCP tokens next, assuming life doesn't get in the way again.

For the authoritative procedure and troubleshooting steps, see [Deploying Ansible MCP Server on OpenShift Container Platform](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_on_openshift_container_platform/deploy-ansible-mcp-server-operator-install) and the broader [deploying Ansible MCP server](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/extend-assembly_deploying_ansible_mcp_server/index) guide.  Red Hat also has a good overview in [Introducing the MCP server for Ansible Automation Platform](https://www.redhat.com/en/blog/it-automation-agentic-ai-introducing-mcp-server-red-hat-ansible-automation-platform).

As with everything I write about my lab, this is how ___I___ run things.  Your instance names, namespaces, routes, and risk tolerance may differ.  Use the technology preview accordingly, and keep your tokens out of git.

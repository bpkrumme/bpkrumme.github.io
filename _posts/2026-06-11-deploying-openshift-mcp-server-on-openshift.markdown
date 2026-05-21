---
layout: single
title:  "Deploying the OpenShift MCP Server on Openshift"
date:   2026-06-11
header:
  og_image: /assets/images/bdt_square.jpeg
---

## Let's set the stage

This is the last installment of my short MCP series for the homelab.  I already covered the overview, deploying ___aap-mcp___ on my operator-managed platform, and deploying ___rhel-mcp___ for Linux host diagnostics.  This post is about ___openshift-mcp___: the bridge that lets an assistant in Cursor talk to my Openshift cluster API.

In the overview I said I would write up token hardening for Openshift MCP.  What I am really doing here is walking through the full deployment: manifests, auth, and how I connect Cursor.  The auth piece is the important part.

## What you're deploying

The [OpenShift MCP Server](https://github.com/openshift/openshift-mcp-server) is based on the upstream Kubernetes MCP server work.  It is a tech-preview style project that exposes cluster tools over HTTP so clients like Cursor can list pods, pull logs, query metrics, work with routes, and do a lot of other API-backed tasks without you pasting `oc` output into chat.

In my lab the server runs as a pod in the `openshift-mcp` namespace on `ocp.bk.lab`.  Cursor hits a route like ___openshift-mcp-server-openshift-mcp.apps.ocp.bk.lab/mcp___.  I do not run it with `npx` and a local kubeconfig on my laptop, though that is a valid option if you prefer.

There are two auth ideas to keep straight:

* ___Who can call the MCP URL___ — the HTTP endpoint on the route
* ___What the tools can do on the cluster___ — which Kubernetes/OpenShift APIs get used under the hood

I use ___token passthrough___ for the second one.  That means when I put my Openshift bearer token in Cursor, the MCP server uses ___my___ RBAC for API calls, not some shared super-user built into the pod.

## Prerequisites

Before you apply manifests, you should have:

* An Openshift cluster and `oc` logged in with permission to create a namespace, deployment, route, and a ___ClusterRoleBinding___
* A cluster admin or someone who can grant `view` to the MCP service account (needed for parts of in-cluster setup)
* A token for your user when you configure Cursor (`oc whoami -t` works for lab use)

## The manifest layout

I keep a directory of YAML files that create the `openshift-mcp` namespace and roll out the server.  Here is what each file is for.

### namespace.yaml

Creates the `openshift-mcp` project, separate from `aap` and `rhel-mcp`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-mcp
```

### serviceaccount.yaml

The identity the pod runs as.  Even with passthrough auth, the deployment still mounts a service account token for in-cluster provider detection.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openshift-mcp-server
  namespace: openshift-mcp
```

### clusterrolebinding.yaml

Grants the built-in `view` ___ClusterRole___ to the MCP service account.  This matters when `cluster_auth_mode` is `kubeconfig` (the default).  I still apply it in my lab even though passthrough is what I actually use day to day.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-mcp-server-view
roleRef:
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: openshift-mcp-server
    namespace: openshift-mcp
```

### configmap.yaml

The main `config.toml` for the server.  Mine enables several toolsets and is ***not*** read-only.  That was a deliberate lab choice so the assistant can do more than look around.

```toml
port = "8080"
cluster_provider_strategy = "in-cluster"
read_only = false
disable_destructive = false
toolsets = ["core", "config", "openshift", "kubevirt", "metrics", "traces"]
trust_proxy_headers = true
```

Setting `read_only = false` means write-style tools can be exposed.  What actually happens still depends on your user token's RBAC.  If your account cannot delete a namespace, the assistant cannot either.  `disable_destructive = false` is another knob on the server side; combine that with passthrough and you need to be honest about risk.

For a safer starting point, set `read_only = true` and leave destructive tools disabled until you know what you need.

### auth-configmap.yaml

A small overlay mounted at `/etc/kubernetes-mcp-server/conf.d` that switches API auth to passthrough.

```toml
cluster_auth_mode = "passthrough"
```

With this in place, the MCP server forwards the agent's `Authorization: Bearer` header to the Openshift API.  Tools run as ___you___, not as the pod service account.

I did not turn on `require_oauth` in the lab.  That would force JWT validation on every MCP HTTP call.  Openshift tokens from `oc whoami -t` are opaque, not JWTs, so that path is a better fit when you wire up a full OIDC flow.  For my homelab I rely on network isolation plus sending a user token from Cursor.

### deployment.yaml

Runs the container image `quay.io/redhat-user-workloads/crt-nshift-lightspeed-tenant/openshift-mcp-server:latest`, mounts the main config and auth overlay, and exposes port ___8080___.

```yaml
containers:
  - name: openshift-mcp-server
    image: quay.io/redhat-user-workloads/crt-nshift-lightspeed-tenant/openshift-mcp-server:latest
    args:
      - "--config"
      - "/etc/kubernetes-mcp-server/config.toml"
    volumeMounts:
      - name: config
        mountPath: /etc/kubernetes-mcp-server
        readOnly: true
      - name: auth-config
        mountPath: /etc/kubernetes-mcp-server/conf.d
        readOnly: true
```

### service.yaml

Exposes the deployment inside the cluster on port ___8080___.  The route and health probes target this service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: openshift-mcp-server
  namespace: openshift-mcp
spec:
  selector:
    app: openshift-mcp-server
  ports:
    - name: http
      port: 8080
      targetPort: http
      protocol: TCP
```

### route.yaml

Creates an Openshift ___Route___ so MCP clients outside the cluster can connect over HTTPS.  TLS terminates at the router with edge termination.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: openshift-mcp-server
  namespace: openshift-mcp
  annotations:
    haproxy.router.openshift.io/timeout: 5m
spec:
  to:
    kind: Service
    name: openshift-mcp-server
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

In my lab the MCP URL is ___https://openshift-mcp-server-openshift-mcp.apps.ocp.bk.lab/mcp___.

## Deploy on Openshift

1. Change into the directory with your manifests and apply them.

    ```shell
    oc apply -f .
    ```

2. Confirm the pod is running.

    ```shell
    oc get pods -n openshift-mcp
    ```

3. Grab the route host.

    ```shell
    oc get route openshift-mcp-server -n openshift-mcp -o jsonpath='https://{.spec.host}/mcp{"\n"}'
    ```

4. Optional health check.

    ```shell
    curl -sk "$(oc get route openshift-mcp-server -n openshift-mcp -o jsonpath='https://{.spec.host}')/healthz"
    ```

## Agent token and Cursor

For passthrough mode the assistant needs a real Openshift token in every MCP request.

1. Get a token (lab shortcut):

    ```shell
    oc whoami -t
    ```

    Tokens expire.  When tools start failing with auth errors, run it again and update Cursor.

2. Add the server to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "openshift-mcp": {
      "type": "http",
      "url": "https://openshift-mcp-server-openshift-mcp.apps.ocp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer <token-from-oc-whoami-t>"
      }
    }
  }
}
```

3. Test with something simple:

```text
List pods in the aap namespace
```

If the token and route are good, you get real data back.  If you get forbidden errors, check whether your user actually has permission for that namespace or verb.

### What passthrough buys you

When I use my normal admin-capable lab account, the assistant can do anything I could do in the API within the tools exposed by the server.  That is powerful and a little scary.

When I use a tighter account, the assistant is tightened too.  Same idea as the AAP MCP post: the server is not a bypass around RBAC.

### Security implications (plain version)

* ___Anyone who can hit the route without a token___ — In my lab I did not enable `require_oauth` on the HTTP endpoint.  If someone on my network could reach the URL, they might talk to MCP without your user token depending on server behavior.  I count on lab network boundaries.  For anything stricter, look at OAuth-protected MCP in the upstream docs.
* ___Your token in Cursor___ — Treat it like a password.  Do not commit `mcp.json` with a real bearer token in git.
* ___Write tools enabled___ — `read_only = false` plus passthrough means a confused or overly eager assistant could try mutating things you are allowed to change.  I accept that in the lab because it helps me move faster.  I would tighten both server config and RBAC for production.
* ___Token lifetime___ — Refresh tokens when they expire.  Stale tokens look like mysterious MCP failures.

The upstream repo has a detailed auth guide ([AGENT-AUTH.md](https://github.com/openshift/openshift-mcp-server) in the project) covering lab mode, passthrough, and full OAuth.  I leaned on passthrough because it maps cleanly to how I already use `oc`.

## How this fits with the other MCP servers

On `ocp.bk.lab` I now have three HTTP MCP endpoints:

* ___openshift-mcp___ — cluster API, pods, metrics, resources
* ___aap-mcp___ — automation controller
* ___rhel-mcp___ — SSH diagnostics on individual hosts

I turn on whichever ones match the problem.  Cluster question?  Openshift MCP.  Job template or dispatch?  AAP MCP.  Service failing on `idm01.bk.lab`?  RHEL MCP.  Often two at once, occasionally all three when OpenShift Virtualization is involved.

## Things that tripped me up

* ___Forgot the Bearer header___ — Without `Authorization` in Cursor, passthrough mode has nothing to forward and tools fail in confusing ways.
* ___Expired token___ — Run `oc whoami -t` again and update the config.
* ___Wrong URL___ — The MCP path is `/mcp` on the openshift-mcp route, not the main console route.
* ___RBAC surprise___ — The assistant can only do what your user can do.  If you use a limited account, expect limited tools results.
* ___Read-only vs write in config.toml___ — Server-level `read_only` gates which tools exist; your token still decides what API calls succeed.

## Troubleshooting

1. Pod running?  `oc get pods -n openshift-mcp`
2. Route matches `mcp.json`?  `oc get route -n openshift-mcp`
3. Fresh token?  `oc whoami -t`
4. Can you do it with oc?  `oc auth can-i <verb> <resource> -n <namespace>`
5. Auth overlay mounted?  Check the deployment has the `auth-config` volume at `conf.d`

### Final Thoughts

Deploying openshift-mcp closed the loop on the MCP series for my homelab.  AAP covers automation, RHEL covers hosts, and Openshift MCP covers the cluster itself.  The setup is not hard, but the auth choices matter more than the YAML.  Passthrough with a bearer token in Cursor was the right trade for me: the assistant acts as me, and I can narrow that identity whenever I want.

If you have been following along since the overview, this is the Openshift piece I kept promising.  All three posts together are how I actually run MCP today on `ocp.bk.lab`.

For more detail see the [OpenShift MCP Server on GitHub](https://github.com/openshift/openshift-mcp-server) and Red Hat's article on [MCP for root-cause analysis in VS Code and Cursor](https://developers.redhat.com/articles/2026/02/11/leverage-ai-root-cause-analysis-mcp-servers-vs-code-and-cursor).

As with everything I write about my lab, this is how ___I___ run things.  Your auth model, toolsets, and appetite for write access may differ.  Keep secret data out of git.

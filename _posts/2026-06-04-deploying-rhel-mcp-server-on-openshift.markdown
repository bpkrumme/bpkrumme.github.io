---
layout: single
title:  "Deploying the RHEL MCP Server on Openshift"
date:   2026-06-04
header:
  og_image: /assets/images/bdt_square.jpeg
---

## Let's set the stage

Two weeks ago I started a short series on ___Model Context Protocol (MCP)___ servers in my homelab.  That first post was a broad overview of how I use MCP with Cursor across Openshift, RHEL, and Ansible Automation Platform.  Last week I walked through enabling the ___Ansible MCP server___ on my operator-managed AAP instance.

This post is the RHEL piece of that series.

In the overview I described ___rhel-mcp___ as the read-only Linux diagnostics bridge: journal entries, service status, process lists, and the rest, without installing an agent on every host.  What I did not cover then is how I actually run that server.  I deploy the upstream [linux-mcp-server](https://github.com/rhel-lightspeed/linux-mcp-server) image on my Openshift cluster, expose it over HTTPS on the ingress, and let it reach my RHEL systems over SSH from inside the cluster.

I'm going to walk through the manifests I use for that deployment and how I wire the server into Cursor.  I'm only going to cover the command-line path here.  For the upstream server behavior and SSH prerequisites on managed hosts, refer to the [Red Hat documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/using-the-rhel-mcp-server-to-enable-ai-assistants-to-run-discover-and-troubleshoot-complex-issues).

## What you're deploying

The ___MCP server for RHEL___ (linux-mcp-server) is a developer-preview offering from Red Hat.  It wraps standard Linux utilities and returns formatted results to MCP clients.  The tools are read-only by design, which is the main reason I'm comfortable pointing an AI assistant at my lab hosts.

In my homelab the server does not run on my laptop via `stdio` and Podman, even though that is the pattern in a lot of the getting-started guides.  Instead it runs as a pod on Openshift in the `rhel-mcp` namespace.  Cursor and other clients connect to it over HTTP at a route like ___linux-mcp-server-rhel-mcp.apps.ocp.bk.lab/mcp___.  When I ask the assistant to check a service on `idm01.bk.lab`, the MCP server SSHes to that host from the cluster using a dedicated `mcp` user and the key I mounted into the deployment.

Long story short, the cluster hosts the MCP endpoint; the RHEL VMs stay agents-free aside from SSH and sudo for the `mcp` account.

## Prerequisites

Before you apply the manifests, you should have:

* Red Hat Openshift Container Platform with permission to create a namespace, deployment, route, and a custom ___SecurityContextConstraints___ object
* Network connectivity from the Openshift cluster to your target RHEL hosts on SSH (port 22)
* An SSH key pair and a dedicated user on each host you want the server to manage, per the [RHEL MCP server docs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/using-the-rhel-mcp-server-to-enable-ai-assistants-to-run-discover-and-troubleshoot-complex-issues)
* `oc` logged in to the cluster where you want the workload to run

## The manifest layout

I keep a small directory of YAML files that together create the `rhel-mcp` namespace and roll out the server, service, route, storage, SSH configuration, and authorization policy.  Here is what each manifest does.

### namespace.yaml

Creates the `rhel-mcp` project so the MCP workload is isolated from `aap`, `openshift-mcp`, and everything else running on the cluster.  The annotations give it a readable name in the Openshift console.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rhel-mcp
```

### serviceaccount.yaml

Defines the ___linux-mcp-server___ service account the pod runs under.  That account is what you bind the custom SCC to during deploy.  Keeping a dedicated service account means you are not reusing `default` and you can scope permissions narrowly.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: linux-mcp-server
  namespace: rhel-mcp
```

### scc.yaml

Defines a ___SecurityContextConstraints (SCC)___ object named `linux-mcp-server` so the pod may run as the fixed UID/GID the upstream image requires.

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: linux-mcp-server
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
fsGroup:
  type: MustRunAs
  ranges:
    - min: 1001
      max: 1001
runAsUser:
  type: MustRunAs
  uid: 1001
requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
```

If you're new to Openshift, SCCs are worth understanding before you apply this file.

Kubernetes lets you declare a ___security context___ on a pod: run as a particular user, drop capabilities, forbid privilege escalation, and so on.  Openshift adds SCCs on top of that model.  An SCC is a cluster-scoped policy object that defines which security contexts pods are ___allowed___ to use.  Every pod is admitted only if its security context matches an SCC that has been granted to its service account.

The default SCCs on a typical cluster are fairly restrictive.  For example, the common `restricted` policy assigns pods an arbitrary UID from a high numeric range so containers do not run as root.  That is good for security, but it breaks images that were built to run as a fixed non-root user.  The upstream ___linux-mcp-server___ image expects UID and GID ___1001___.  Without a matching SCC, Openshift will assign a different UID, the container entrypoint will not match what the image author intended, and the pod will sit in `CreateContainerConfigError` or crash immediately.

The ___scc.yaml___ manifest creates a dedicated policy so this deployment may run as UID/GID ___1001___, mount the volume types the pod needs, and still operate under a locked-down rule set: no privileged containers, no host namespaces, privilege escalation disabled, and capabilities dropped.  Applying the manifest only creates the SCC object.  During deploy you still run `oc adm policy add-scc-to-user` to bind that SCC to the `linux-mcp-server` service account in the `rhel-mcp` namespace.  That grant is namespace-scoped through the service account; it does not open the policy cluster-wide.

From a security standpoint, SCCs are how Openshift enforces guardrails.  The MCP server gets only the permissions its image actually needs, and everything else stays denied by default.

### configmap.yaml

Supplies environment variables the linux-mcp-server process reads at startup.  This is where I set HTTP transport, the listen address and port, the MCP path (`/mcp`), the SSH key path inside the container, and the location of the authorization policy file.  Optional variables such as `LINUX_MCP_ALLOWED_LOG_PATHS` can be uncommented when you want to allow `read_log_file` against specific paths on managed hosts.

```yaml
data:
  LINUX_MCP_USER: "mcp"
  LINUX_MCP_TRANSPORT: "http"
  LINUX_MCP_HOST: "0.0.0.0"
  LINUX_MCP_PORT: "8000"
  LINUX_MCP_PATH: "/mcp"
  LINUX_MCP_SSH_KEY_PATH: "/var/lib/mcp/.ssh/id_ed25519"
  LINUX_MCP_POLICY_PATH: "/etc/linux-mcp/policy.yaml"
```

### auth-policy-configmap.yaml

Mounts as `/etc/linux-mcp/policy.yaml` inside the pod.  It tells the MCP server which tools may run against which remote hosts when clients connect over HTTP.  In my lab the policy is permissive: any tool against any `*.bk.lab` host.  I cover the security implications in a later section.

```yaml
data:
  policy.yaml: |
    rules:
      - host: "*.bk.lab"
        tools: ["*"]
        action: ssh_default
        all_users: true
```

### ssh-config-configmap.yaml

Provides an SSH client configuration the pod uses when connecting to lab RHEL systems.  Each `Host` stanza names a target, sets the `mcp` user, points at the mounted private key, and relaxes host key checking for lab convenience.  If a host is missing from this file, the MCP server cannot reach it even if the key and network are otherwise fine.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: linux-mcp-ssh-config
  namespace: rhel-mcp
data:
  config: |
    Host idm01.bk.lab
      HostName idm01.bk.lab
      User mcp
      IdentityFile /var/lib/mcp/.ssh/id_ed25519
      StrictHostKeyChecking accept-new
      UserKnownHostsFile /dev/null
    Host satellite.bk.lab
      HostName satellite.bk.lab
      User mcp
      IdentityFile /var/lib/mcp/.ssh/id_ed25519
      StrictHostKeyChecking accept-new
      UserKnownHostsFile /dev/null
    # ... additional lab hosts ...
```

### pvc.yaml

Requests a small ReadWriteOnce volume for linux-mcp-server log files so they survive pod restarts.  One gibibyte is plenty for my lab.

```yaml
kind: PersistentVolumeClaim
metadata:
  name: linux-mcp-server-logs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### deployment.yaml

The core workload.  It runs the `quay.io/redhat-services-prod/rhel-lightspeed-tenant/linux-mcp-server:latest` image, wires in the ConfigMaps and optional SSH secret as volumes, sets resource requests and limits, and configures liveness and readiness probes against the HTTP port.  The pod spec uses `serviceAccountName: linux-mcp-server` and a pod-level security context matching UID ___1001___.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linux-mcp-server
  namespace: rhel-mcp
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: linux-mcp-server
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: linux-mcp-server
          image: quay.io/redhat-services-prod/rhel-lightspeed-tenant/linux-mcp-server:latest
          ports:
            - name: http
              containerPort: 8000
          envFrom:
            - configMapRef:
                name: linux-mcp-server
          env:
            - name: LINUX_MCP_USER
              valueFrom:
                secretKeyRef:
                  name: rhel-mcp-ssh
                  key: username
                  optional: true
          volumeMounts:
            - name: ssh-dir
              mountPath: /var/lib/mcp/.ssh
              readOnly: true
            - name: auth-policy
              mountPath: /etc/linux-mcp
              readOnly: true
            - name: logs
              mountPath: /var/lib/mcp/.local/share/linux-mcp-server/logs
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
      volumes:
        - name: ssh-dir
          projected:
            sources:
              - secret:
                  name: rhel-mcp-ssh
                  optional: true
              - configMap:
                  name: linux-mcp-ssh-config
        - name: auth-policy
          configMap:
            name: linux-mcp-auth-policy
        - name: logs
          persistentVolumeClaim:
            claimName: linux-mcp-server-logs
```

The important volume mounts are:

* ___ssh-dir___ — projected SSH private key (from the optional `rhel-mcp-ssh` secret) plus the SSH config ConfigMap
* ___auth-policy___ — the HTTP authorization policy
* ___logs___ — the persistent volume for server logs

### service.yaml

Exposes the deployment inside the cluster on port ___8000___.  The route and probes target this service name.

```yaml
spec:
  selector:
    app: linux-mcp-server
  ports:
    - name: http
      port: 8000
      targetPort: http
```

### route.yaml

Creates an Openshift ___Route___ so external MCP clients (Cursor on my laptop, for example) reach the server over HTTPS.  TLS terminates at the router with edge termination, and a five-minute timeout annotation keeps long-running tool calls from being cut off too aggressively.

```yaml
spec:
  to:
    kind: Service
    name: linux-mcp-server
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

In my lab the resulting URL is ___https://linux-mcp-server-rhel-mcp.apps.ocp.bk.lab/mcp___.

## Deploy on Openshift

1. From a machine with `oc` access, change into the directory that holds your RHEL MCP manifests.

2. Apply the manifests.

    ```shell
    oc apply -f .
    ```

3. Grant the custom SCC to the workload service account, as described under ___scc.yaml___ above.  This step is required once per cluster.

    ```shell
    oc adm policy add-scc-to-user linux-mcp-server -z linux-mcp-server -n rhel-mcp
    ```

4. Watch the rollout.

    ```shell
    oc get pods -n rhel-mcp -w
    ```

5. Confirm the route and probe the MCP endpoint.

    ```shell
    oc get route linux-mcp-server -n rhel-mcp -o jsonpath='https://{.spec.host}{.spec.path}{"\n"}'
    ```

    ```shell
    curl -sk "$(oc get route linux-mcp-server -n rhel-mcp -o jsonpath='https://{.spec.host}')/mcp" -o /dev/null -w '%{http_code}\n'
    ```

    An HTTP response (even unauthorized) means the route and service are wired.

## SSH credentials and target hosts

The deployment mounts SSH configuration from a ConfigMap and an optional Secret.  I create the secret with the private key I use for the `mcp` user on managed hosts:

```shell
oc create secret generic rhel-mcp-ssh -n rhel-mcp \
  --from-file=id_ed25519_mcp=$HOME/.ssh/id_ed25519_mcp \
  --from-literal=username=mcp \
  --from-literal=key-passphrase='' \
  --dry-run=client -o yaml | oc apply -f -

oc rollout restart deployment/linux-mcp-server -n rhel-mcp
```

The ___ssh-config-configmap.yaml___ file lists the lab hosts I want reachable from the pod.  Each stanza points at a `*.bk.lab` system, uses the `mcp` user, and references the key mounted at `/var/lib/mcp/.ssh/id_ed25519`.  I use the combination of ___rhel-mcp___ and ___aap-mcp___ in Cursor to keep that list current: I query AAP for inventory and managed hosts, compare what the automation platform knows about, and update the SSH ConfigMap so the same systems I run jobs against are the same systems I can ask the assistant to inspect.

When I add a new host, I still edit the ConfigMap, re-apply, and restart the deployment if needed.  With both MCP servers enabled, I often let the assistant handle that update: it can read the current inventory from AAP, draft the new `Host` stanza for `ssh-config-configmap.yaml`, and even apply the change and trigger a deployment rollout so I am not maintaining two inventories by hand.

## HTTP authorization policy

The upstream linux-mcp-server supports an authorization policy file for HTTP transport.  My lab policy in ___auth-policy-configmap.yaml___ is intentionally permissive:

```yaml
rules:
  - host: "*.bk.lab"
    tools: ["*"]
    action: ssh_default
    all_users: true
```

That allows every exposed tool against any host matching `*.bk.lab` without per-user OAuth, which is fine for an isolated lab network.  The upstream server documentation is explicit about this: HTTP transport does not include authentication by default.  TLS terminates at the Openshift route, but anyone who can reach the URL could invoke tools unless you tighten the policy or put an API gateway in front.

For production you would replace `all_users: true` with JWT or OAuth claim rules and narrow the host and tool lists.

## Configuration worth knowing

Most environment variables live in ___configmap.yaml___.  The ones I touch or think about most often:

| Variable | Purpose |
|----------|---------|
| `LINUX_MCP_USER` | Default SSH user for targets (overridden by the `rhel-mcp-ssh` secret when present) |
| `LINUX_MCP_ALLOWED_LOG_PATHS` | Comma-separated paths allowed for `read_log_file` on managed hosts |
| `LINUX_MCP_TOOLSET` | `fixed`, `run_script`, or `both` — I leave the safer defaults unless I have a reason not to |
| `LINUX_MCP_VERIFY_HOST_KEYS` | I set `false` in the lab to reduce friction with rebuilt VMs; production should verify keys |

Review [guarded command execution](https://rhel-lightspeed.github.io/linux-mcp-server/guarded-command-execution/) in the upstream docs before enabling script or write-oriented toolsets.

## Connect Cursor

Add an HTTP entry to `~/.cursor/mcp.json`.  My lab configuration does not use a bearer token on this server because of the permissive policy above.  Your posture may differ.

```json
{
  "mcpServers": {
    "rhel-mcp": {
      "type": "http",
      "url": "https://linux-mcp-server-rhel-mcp.apps.ocp.example.com/mcp"
    }
  }
}
```

Restart or refresh MCP in Cursor, then try a low-risk prompt that names a host you configured:

```text
What is the status of the sshd service on idm01.bk.lab?
```

If SSH from the pod to that host works, the assistant should return real `systemctl` or journal-backed output instead of a guess.

## How this fits with my other MCP servers

On the same Openshift cluster I also run ___aap-mcp___ in the `aap` namespace and ___openshift-mcp___ in `openshift-mcp`.  The division of labor is simple:

* ___openshift-mcp___ — cluster and platform objects
* ___aap-mcp___ — automation controller API
* ___rhel-mcp___ — operating system diagnostics on individual hosts

When I'm debugging something that spans layers, I might enable all three in Cursor.  When I'm editing playbooks for a specific host, I often enable only `rhel-mcp` and `aap-mcp`.

## Things that tripped me up

* ___SCC first.___  Forgetting `oc adm policy add-scc-to-user` produces a pod that never becomes ready.  The events will mention UID constraints.
* ___SSH from the pod, not from your laptop.___  A key that works when you `ssh` from your own workstation still has to be in the `rhel-mcp-ssh` secret and listed in the SSH ConfigMap host entries.
* ___Host key verification.___  I disabled strict checking in the lab ConfigMap (`StrictHostKeyChecking accept-new`).  That is a convenience trade-off, not a recommendation for production.
* ___HTTP is not auth.___  Do not expose the route to untrusted networks without tightening `auth-policy-configmap.yaml` or adding a proxy.

## Troubleshooting

When MCP calls fail for a specific host, I check in this order:

1. ___Pod health___ — `oc get pods -n rhel-mcp` and deployment logs
2. ___Route___ — does the URL in `mcp.json` match `oc get route`?
3. ___SSH from inside the pod___ — can the workload reach port 22 on the target?
4. ___mcp user and key___ — is the public key in `authorized_keys` for `mcp` on that host?
5. ___ConfigMap host entry___ — is the hostname spelled the same way in `linux-mcp-ssh-config`?

Most of my early failures were missing SCC grants or a host I had automated in AAP but had not yet added to the SSH ConfigMap.

### Final Thoughts

Deploying the RHEL MCP server on Openshift turned out to be more moving parts than adding `spec.mcp` to an existing AAP custom resource, but the model is the same: run the bridge close to the infrastructure it talks to, expose one HTTPS endpoint, and keep the assistants read-only on hosts until you have a reason to loosen that.

If you've been following the MCP series, this is the homelab deployment story behind the RHEL bullet in the overview post.  I still owe a write-up on hardening ___openshift-mcp___ tokens unless life gets in the way again.

For authoritative background, see [Using the MCP server for RHEL](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interacting_with_the_command-line_assistant_powered_by_rhel_lightspeed/using-the-rhel-mcp-server-to-enable-ai-assistants-to-run-discover-and-troubleshoot-complex-issues) and [Leverage AI for root-cause analysis with MCP servers in VS Code and Cursor](https://developers.redhat.com/articles/2026/02/11/leverage-ai-root-cause-analysis-mcp-servers-vs-code-and-cursor).  The upstream project is at [rhel-lightspeed/linux-mcp-server](https://github.com/rhel-lightspeed/linux-mcp-server).

As with everything I write about my lab, this is how ___I___ run things.  Your namespaces, hostnames, and security posture may differ.  Use developer-preview features accordingly, and keep SSH keys out of git.

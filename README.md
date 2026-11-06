# kagent v1.0.0-rc3 on kind + DGX Spark

A deployment guide that covers:

1. A local **kind** cluster with LoadBalancer support from `sudo cloud-provider-kind`
2. **kagent v1.0.0-rc3** (with Agent Substrate v1.0.0 and the bundled dev PostgreSQL)
3. A **DGX Spark** running vLLM (`qwen3.8-flash-next`) as the default model provider
4. **Agent #1: `agentgateway-tracker`**: a daily summary of changes on the agentgateway GitHub repos and agentgateway.dev, published to this repo
5. **Agent #2: `fortinet-cve-watch`**: a daily check of the latest Fortinet CVEs against your FortiGate's firmware, published to this repo

> Written against the `v1.0.0-rc3` charts, CRDs (`api.kagent.dev/v1alpha3`), and CLI (2026-11-06).
> The kagent CRDs are unchanged from rc2. rc3 moves Substrate to **v1.0.0** and installs
> atelet as its own `substrate-atelet` release. Upgrading an rc2 cluster? See [step 12](#12-upgrading-from-rc2).
> kagent 1.0 differs a lot from 0.x. Agents now run as **Substrate actors** inside gVisor
> sandboxes, they reach the network only through egress policies, and every Agent needs a
> **Harness**. Examples written for 0.x (`kagent.dev/v1alpha2`, `spec.declarative`, `MCPServer`) do not work here.

---

## Architecture

```
 macOS (Docker Desktop)                                   LAN
┌──────────────────────────────────────────────────┐
│ kind cluster "kagent" (k8s 1.35)                 │      ┌───────────────────────────┐
│                                                  │      │ DGX Spark 172.16.10.173   │
│  ns kagent                                       │      │ vLLM :8000  /v1           │
│   ├─ kagent-controller  :8083 (API/A2A/MCP)      │      │ model qwen3.8-flash-next  │
│   ├─ kagent-ui  (LoadBalancer via cpk)           │      └─────────────▲─────────────┘
│   ├─ postgres (bundled, dev only)                │                    │
│   ├─ Service dgx-spark ──EndpointSlice───────────┼────────────────────┘
│   ├─ Harness kagent ─► WorkerPool kagent-default │
│   ├─ Agent agentgateway-tracker ─► GitHub MCP ───┼──► api.githubcopilot.com/mcp
│   ├─ Agent fortinet-cve-watch   ─► GitHub MCP    │
│   └─ CronJobs (daily) ─► kagent CLI ─► A2A       │──► FortiGate API, NVD, CISA KEV
│  ns ate-system  (Substrate control plane/router) │
└──────────────────────────────────────────────────┘
        ▲ sudo cloud-provider-kind (host process)
```

Why the `dgx-spark` Service exists: Substrate egress rules accept **DNS names only**, not IP
addresses, and runtime-facing URLs must be full cluster DNS names. So the guide puts the
DGX Spark's IP behind a selectorless Service and points the ModelConfig at
`http://dgx-spark.kagent.svc.cluster.local:8000/v1`.

---

## 0. Prerequisites

| Tool | Version | Install |
|---|---|---|
| Docker Desktop | 8+ CPUs, 16 GB+ RAM assigned | docker.com |
| kind | **≥ v0.31** (needs `kindest/node:v1.35.0`) | `brew upgrade kind` |
| kubectl, helm, jq | recent | `brew install kubectl helm jq` |
| cloud-provider-kind | recent | `brew install cloud-provider-kind` |
| kagent CLI | **v1.0.0-rc3** | see step 3 |

> `kind version` currently reports **v0.30.0** on this machine. Upgrade it first. kagent
> rc3 uses Kubernetes 1.35 feature gates (`PodCertificateRequest`, `ClusterTrustBundle*`)
> that Substrate needs.

### 0.1 Check the DGX Spark

```bash
export DGX_IP=172.16.10.173
export DGX_MODEL=qwen3.8-flash-next

# Models served
curl -s http://$DGX_IP:8000/v1/models | jq '.data[].id'

# Chat (thinking disabled so you see the answer, not just reasoning tokens)
curl -s http://$DGX_IP:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"'$DGX_MODEL'",
       "messages":[{"role":"user","content":"Say hello in five words."}],
       "max_tokens":64,
       "chat_template_kwargs":{"enable_thinking":false}}' | jq -r '.choices[0].message.content'
```

**Agents need tool calling.** Check that vLLM returns structured `tool_calls`:

```bash
curl -s http://$DGX_IP:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{
  "model":"'$DGX_MODEL'",
  "messages":[{"role":"user","content":"What is the weather in Toronto?"}],
  "tools":[{"type":"function","function":{"name":"get_weather","description":"Get weather",
    "parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}}],
  "chat_template_kwargs":{"enable_thinking":false}}' | jq '.choices[0].message.tool_calls'
```

If that returns `null`, restart vLLM on the Spark with tool parsing enabled. kagent's
ModelConfig can't send `chat_template_kwargs`, so if you also want thinking off for
agents, set it server-side:

```bash
vllm serve <model-path> --served-model-name qwen3.8-flash-next \
  --enable-auto-tool-choice --tool-call-parser hermes \
  --default-chat-template-kwargs '{"enable_thinking": false}' \
  --host 0.0.0.0 --port 8000
```

(Pick the tool-call parser that matches your Qwen variant, for example `qwen3_coder` for Qwen3-Coder models.)

---

## 1. Create the kind cluster

```bash
mkdir -p cluster && cat > cluster/kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kagent
networking:
  apiServerAddress: "127.0.0.1"
# Substrate WorkerPools project pod identities and trust bundles into the sandbox.
featureGates:
  ClusterTrustBundle: true
  ClusterTrustBundleProjection: true
  PodCertificateRequest: true
runtimeConfig:
  "api/alpha": "false"
  "certificates.k8s.io/v1beta1": "true"
nodes:
  - role: control-plane
EOF

kind create cluster --config cluster/kind-config.yaml --image kindest/node:v1.35.0 --wait 120s
kubectl cluster-info --context kind-kagent
```

## 2. Start cloud-provider-kind (separate terminal)

cloud-provider-kind watches the cluster and gives every `type: LoadBalancer` Service an
external address. It must run **as root** and keep running:

```bash
# Terminal 2. Leave it running.
sudo cloud-provider-kind --enable-lb-port-mapping
```

On macOS the Docker network isn't routable from the host. `--enable-lb-port-mapping` publishes
each LB port on `localhost` through a `kindccm-*` container (see step 5).

## 3. Install the kagent CLI (v1.0.0-rc3)

```bash
curl -fsSL https://raw.githubusercontent.com/kagent-dev/kagent/v1.0.0-rc3/scripts/get-kagent \
  | DESIRED_VERSION=v1.0.0-rc3 bash
kagent version     # → "kagent_version":"1.0.0-rc3"
```

## 4. Point kagent at the DGX Spark and install

### 4.1 Give the DGX Spark a cluster DNS name

```bash
kubectl create namespace kagent

kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: dgx-spark
  namespace: kagent
spec:
  ports:
    - name: http
      port: 8000
      targetPort: 8000
      protocol: TCP
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: dgx-spark-1
  namespace: kagent
  labels:
    kubernetes.io/service-name: dgx-spark
addressType: IPv4
ports:
  - name: http
    port: 8000
    protocol: TCP
endpoints:
  - addresses: ["${DGX_IP}"]
EOF

# Check it from inside the cluster
kubectl -n kagent run dgx-test --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s http://dgx-spark.kagent.svc.cluster.local:8000/v1/models
```

### 4.2 Install config and values

`kagent install` is the supported path for a dev cluster. It installs, in order:
`kagent-crds` → `substrate-crds` → `substrate-podcert` → `substrate-atelet` → `substrate` → `kagent`.
It also deploys a dev PostgreSQL, creates the Substrate CAs and identities, creates the
`kagent-rustfs-credentials` Secret that atelet reads, enables the Substrate preview gates
kagent needs (`PerTemplateSnapshotLocation`, `GoldenEgressPolicy`), and labels the namespace
`ate.dev/api-access=true`.

```bash
mkdir -p install

cat > install/kagent-values.yaml <<'EOF'
providers:
  default: openAI
  openAI:
    provider: OpenAI            # vLLM speaks the OpenAI API
    model: qwen3.8-flash-next
    apiKeySecretRef: kagent-openai
    apiKeySecretKey: OPENAI_API_KEY
    config:                     # rendered into ModelConfig.spec.openAI
      baseUrl: http://dgx-spark.kagent.svc.cluster.local:8000/v1
      timeout: 600
ui:
  service:
    type: LoadBalancer          # served by cloud-provider-kind
EOF

cat > install/config.yaml <<EOF
install:
  defaultModelProvider: OpenAI
  kagent:
    version: 1.0.0-rc3
    valuesFiles:
      - $PWD/install/kagent-values.yaml
EOF

# vLLM ignores the key unless you started it with --api-key. A placeholder is fine.
export OPENAI_API_KEY=sk-dgx-spark-local
```

Optional: preview exactly what will be applied:

```bash
kagent --config install/config.yaml install --dry-run --output-dir ./kagent-install
cat ./kagent-install/install.sh
```

### 4.3 Install

```bash
kagent --config install/config.yaml install
```

### 4.4 Verify

```bash
kubectl get pods -n ate-system
kubectl get pods -n kagent
kubectl get modelconfigs,remotemcpservers,workerpools.ate.dev -n kagent
kubectl get modelconfig default-model-config -n kagent -o yaml | yq '.spec'
```

You should see `provider: OpenAI`, `model: qwen3.8-flash-next`, and
`openAI.baseUrl: http://dgx-spark.kagent.svc.cluster.local:8000/v1`.

## 5. Open the UI

```bash
kubectl get svc kagent-ui -n kagent          # EXTERNAL-IP assigned by cloud-provider-kind
docker ps --filter name=kindccm --format '{{.Names}}\t{{.Ports}}'   # host port on macOS
```

Open `http://localhost:<mapped-port>`. If you'd rather not depend on cpk:
`kubectl -n kagent port-forward svc/kagent-ui 8080:8080` → http://localhost:8080

For CLI and A2A access from your Mac, forward the controller:

```bash
kubectl -n kagent port-forward svc/kagent-controller 8083:8083
```

## 6. Harness and a smoke-test agent

In 1.0 every Agent runs on a **Harness**. A Harness picks the runtime (`kagent` = Go ADK) and
the Substrate WorkerPool. The install created the `kagent-default` WorkerPool.

```bash
kubectl apply -f - <<'EOF'
apiVersion: api.kagent.dev/v1alpha3
kind: Harness
metadata:
  name: kagent
  namespace: kagent
spec:
  kagent: {}
  substrate:
    workerPoolRef:
      name: kagent-default
    snapshotPolicy:
      location: s3://ate-snapshots/kagent
---
apiVersion: api.kagent.dev/v1alpha3
kind: Agent
metadata:
  name: hello
  namespace: kagent
spec:
  harnessRef:
    name: kagent
  template:
    inline:
      description: Smoke test agent on the DGX Spark.
      modelConfig:
        name: default-model-config
      systemPrompt: You are a concise assistant running on kagent with a local Qwen model.
EOF

# Ready = Substrate booted and snapshotted the golden actor (about 1–2 min)
kubectl wait agent/hello -n kagent --for=condition=Ready --timeout=10m

SID=$(kagent agent session create --agent hello -o json | jq -r '.session.id // .id')
kagent agent invoke --session "$SID" --task "Say hello in five words."
```

---

## 7. Agent #1: `agentgateway-tracker`

Every day it looks at what changed in the last 24h in
[`agentgateway/agentgateway`](https://github.com/agentgateway/agentgateway) (code, PRs, releases)
and [`agentgateway/website`](https://github.com/agentgateway/website), which is the source of
**agentgateway.dev**. It writes a summary to `reports/agentgateway/YYYY-MM-DD.md` in
**this repo** (`sebbycorp/kagent-getting-started`).

> The GitHub org is `agentgateway`. `agentgateway-dev` doesn't exist.

### 7.1 GitHub token and GitHub MCP server

Create a fine-grained PAT with: public repo read, plus **Contents: Read and write** on
`sebbycorp/kagent-getting-started`.

```bash
export GITHUB_PAT=github_pat_xxx

# Secret-backed headers are injected by the Substrate egress gateway. The agent never sees the token.
kubectl -n kagent create secret generic github-mcp \
  --from-literal=authorization="Bearer ${GITHUB_PAT}"

kubectl apply -f - <<'EOF'
apiVersion: api.kagent.dev/v1alpha3
kind: RemoteMCPServer
metadata:
  name: github
  namespace: kagent
spec:
  description: GitHub hosted MCP server
  url: https://api.githubcopilot.com/mcp/
  protocol: STREAMABLE_HTTP
  timeout: 60s
  headersFrom:
    - name: Authorization
      valueFrom:
        type: Secret
        name: github-mcp
        key: authorization
    - name: X-MCP-Toolsets
      value: repos,pull_requests
EOF
```

kagent infers egress to `api.githubcopilot.com` from the RemoteMCPServer, so no
EgressPolicy is needed.

### 7.2 The agent

```bash
kubectl apply -f - <<'EOF'
apiVersion: api.kagent.dev/v1alpha3
kind: Agent
metadata:
  name: agentgateway-tracker
  namespace: kagent
spec:
  harnessRef:
    name: kagent
  template:
    inline:
      description: Daily change digest for agentgateway (code, releases, docs site).
      modelConfig:
        name: default-model-config
      systemPrompt: |
        You track changes to the agentgateway project and publish a daily digest.

        Sources (GitHub org "agentgateway"):
          - agentgateway/agentgateway  : the gateway (Rust) - commits, merged PRs, releases/tags
          - agentgateway/website       : source of https://agentgateway.dev (docs/blog changes)

        For the time window given in the task:
          1. list_commits on the default branch of each repo (since/until = window).
          2. search_pull_requests for PRs merged in the window ("repo:<r> is:pr is:merged merged:<from>..<to>").
          3. list_releases / list_tags for agentgateway/agentgateway; note any new release.
        Then write a Markdown report:
          # agentgateway daily digest - <date>
          ## TL;DR            (3-5 bullets, most important first)
          ## Releases
          ## Notable changes  (group: features, fixes, breaking/deprecations, security, CI/chore)
          ## Docs / agentgateway.dev
          ## Links            (PR/commit URLs)
        Say "No changes" for empty sections. Never invent PRs, numbers, or versions; only use tool results.

        Publish it with create_or_update_file:
          owner=sebbycorp repo=kagent-getting-started branch=main
          path=reports/agentgateway/<YYYY-MM-DD>.md
          message="agentgateway digest <YYYY-MM-DD>"
        (if the file exists, fetch its sha with get_file_contents first.)
        Finish by replying with the TL;DR and the file URL.
      tools:
        - mcp:
            server:
              apiGroup: api.kagent.dev
              kind: RemoteMCPServer
              name: github
            tools:
              - list_commits
              - get_commit
              - search_pull_requests
              - list_pull_requests
              - pull_request_read
              - list_releases
              - get_latest_release
              - list_tags
              - get_file_contents
              - create_or_update_file
EOF

kubectl wait agent/agentgateway-tracker -n kagent --for=condition=Ready --timeout=10m
```

Run it once by hand:

```bash
SID=$(kagent agent session create --agent agentgateway-tracker -o json | jq -r '.session.id // .id')
kagent agent invoke --session "$SID" --timeout 30m \
  --task "Window: $(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ) to $(date -u +%Y-%m-%dT%H:%M:%SZ). Date: $(date -u +%F)."
```

### 7.3 Run it daily (CronJob)

kagent has no built-in scheduler, so a CronJob opens a session and invokes the agent
through the controller's A2A gateway.

```bash
kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: agentgateway-tracker-daily
  namespace: kagent
spec:
  schedule: "0 12 * * *"          # 12:00 UTC (08:00 ET)
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: invoke
              image: alpine:3.22
              env:
                - name: KAGENT_URL
                  value: http://kagent-controller.kagent.svc.cluster.local:8083
              command: ["/bin/sh", "-ec"]
              args:
                - |
                  apk add --no-cache curl jq >/dev/null
                  ARCH=$(uname -m); [ "$ARCH" = aarch64 ] && ARCH=arm64; [ "$ARCH" = x86_64 ] && ARCH=amd64
                  curl -fsSL -o /usr/local/bin/kagent \
                    https://github.com/kagent-dev/kagent/releases/download/v1.0.0-rc3/kagent-linux-$ARCH
                  chmod +x /usr/local/bin/kagent
                  K="kagent --api-url $KAGENT_URL --gateway-url $KAGENT_URL -n kagent"
                  FROM=$(date -u -d @$(( $(date +%s) - 86400 )) +%Y-%m-%dT%H:%M:%SZ)
                  TO=$(date -u +%Y-%m-%dT%H:%M:%SZ)
                  SID=$($K agent session create --agent agentgateway-tracker -o json | jq -r '.session.id // .id')
                  $K agent invoke --session "$SID" --timeout 45m \
                    --task "Window: $FROM to $TO. Date: $(date -u +%F)."
EOF

# Trigger a run now
kubectl -n kagent create job --from=cronjob/agentgateway-tracker-daily agw-manual-1
kubectl -n kagent logs -f job/agw-manual-1
```

---

## 8. Agent #2: `fortinet-cve-watch`

Every day:

1. The CronJob (a normal pod, not sandboxed) reads your **FortiGate firmware version** over
   the FortiOS REST API, then pulls **Fortinet CVEs from the last 30 days** from the NVD API
   and the **CISA KEV** catalog.
2. It passes that data to the agent. The agent works out which CVEs affect your exact
   FortiOS version and writes `reports/fortinet-cve/YYYY-MM-DD.md` to this repo.

The CronJob gathers the data and the agent does only the reasoning and publishing. That way
the FortiGate token never reaches the model, and the agent doesn't need egress to a
LAN IP, which Substrate wouldn't allow anyway.

### 8.1 FortiGate API access

On the FortiGate, create a REST API admin (System → Administrators → REST API Admin) with a
**read-only** profile. Set **Trusted Hosts** to your Mac's LAN IP: kind traffic leaves through
Docker Desktop NAT.

```bash
kubectl -n kagent create secret generic fortigate-api \
  --from-literal=host=192.168.1.99 \
  --from-literal=token='<fortigate-api-token>'
# Optional, for higher NVD rate limits: https://nvd.nist.gov/developers/request-an-api-key
kubectl -n kagent create secret generic nvd-api --from-literal=key='<nvd-api-key-or-empty>'
```

Test from your Mac:

```bash
curl -sk -H "Authorization: Bearer <token>" https://192.168.1.99/api/v2/monitor/system/status \
  | jq '{version, build, model: .results.model_number, hostname: .results.hostname}'
```

### 8.2 The agent

```bash
kubectl apply -f - <<'EOF'
apiVersion: api.kagent.dev/v1alpha3
kind: Agent
metadata:
  name: fortinet-cve-watch
  namespace: kagent
spec:
  harnessRef:
    name: kagent
  template:
    inline:
      description: Matches recent Fortinet CVEs against the home FortiGate and publishes a report.
      modelConfig:
        name: default-model-config
      systemPrompt: |
        You are a vulnerability analyst. The task contains:
          - FORTIGATE: model, hostname, FortiOS version and build (from the device API)
          - NVD: recent Fortinet CVEs (id, published, CVSS, description, affected-version text)
          - KEV: Fortinet entries from CISA's Known Exploited Vulnerabilities catalog

        For each CVE, decide whether it applies to FortiOS (ignore FortiWeb, FortiManager, FortiClient,
        etc. unless the description says FortiOS/FortiProxy-on-FortiOS), then compare the device's
        version with the affected ranges in the description (e.g. "7.4.0 through 7.4.5"):
          AFFECTED  - device version is inside an affected range
          NOT AFFECTED - device version is outside every range (state the fixed version)
          UNCLEAR   - ranges missing or ambiguous; say what to check
        Never invent CVE IDs, scores, or versions. Use only the data provided.

        Report format:
          # FortiGate CVE report - <date>
          Device: <model> / FortiOS <version> build <build>
          ## Action required   (AFFECTED, sorted by CVSS desc; flag anything in KEV as ⚠️ ACTIVELY EXPLOITED)
          ## Unclear
          ## Not affected      (compact table: CVE | CVSS | fixed in)
          ## Recommendation    (target upgrade version if any)
          ## Sources           (NVD links https://nvd.nist.gov/vuln/detail/<id>, FortiGuard PSIRT)

        Publish with create_or_update_file:
          owner=sebbycorp repo=kagent-getting-started branch=main
          path=reports/fortinet-cve/<YYYY-MM-DD>.md
          message="FortiGate CVE report <YYYY-MM-DD>"
        (if the file exists, fetch its sha with get_file_contents first.)
        Reply with the "Action required" section and the file URL.
      tools:
        - mcp:
            server:
              apiGroup: api.kagent.dev
              kind: RemoteMCPServer
              name: github
            tools:
              - get_file_contents
              - create_or_update_file
EOF

kubectl wait agent/fortinet-cve-watch -n kagent --for=condition=Ready --timeout=10m
```

### 8.3 Run it daily (CronJob)

```bash
kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: fortinet-cve-watch-daily
  namespace: kagent
spec:
  schedule: "30 12 * * *"         # 12:30 UTC, after the agentgateway digest
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: invoke
              image: alpine:3.22
              env:
                - name: KAGENT_URL
                  value: http://kagent-controller.kagent.svc.cluster.local:8083
                - name: FGT_HOST
                  valueFrom: {secretKeyRef: {name: fortigate-api, key: host}}
                - name: FGT_TOKEN
                  valueFrom: {secretKeyRef: {name: fortigate-api, key: token}}
                - name: NVD_KEY
                  valueFrom: {secretKeyRef: {name: nvd-api, key: key, optional: true}}
              command: ["/bin/sh", "-ec"]
              args:
                - |
                  apk add --no-cache curl jq >/dev/null
                  ARCH=$(uname -m); [ "$ARCH" = aarch64 ] && ARCH=arm64; [ "$ARCH" = x86_64 ] && ARCH=amd64
                  curl -fsSL -o /usr/local/bin/kagent \
                    https://github.com/kagent-dev/kagent/releases/download/v1.0.0-rc3/kagent-linux-$ARCH
                  chmod +x /usr/local/bin/kagent

                  # 1) FortiGate firmware (self-signed cert -> -k)
                  FGT=$(curl -sfk -H "Authorization: Bearer $FGT_TOKEN" \
                        "https://$FGT_HOST/api/v2/monitor/system/status" \
                        | jq -c '{version, build, model: (.results.model_name+" "+.results.model_number), hostname: .results.hostname}')

                  # 2) NVD: Fortinet / FortiOS CVEs published in the last 30 days
                  START=$(date -u -d @$(( $(date +%s) - 30*86400 )) +%Y-%m-%dT00:00:00.000)
                  END=$(date -u +%Y-%m-%dT23:59:59.999)
                  H=""; [ -n "$NVD_KEY" ] && H="apiKey: $NVD_KEY"
                  NVD=$(curl -sf -H "$H" \
                    "https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=FortiOS&pubStartDate=$START&pubEndDate=$END" \
                    | jq -c '[.vulnerabilities[].cve | {id, published,
                        cvss: ((.metrics.cvssMetricV31 // .metrics.cvssMetricV40 // [])[0].cvssData.baseScore),
                        description: ((.descriptions[] | select(.lang=="en") | .value)[0:900])}] | .[0:60]')

                  # 3) CISA KEV: Fortinet entries
                  KEV=$(curl -sf https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json \
                    | jq -c '[.vulnerabilities[] | select(.vendorProject=="Fortinet") | {cveID, product, dateAdded}]')

                  K="kagent --api-url $KAGENT_URL --gateway-url $KAGENT_URL -n kagent"
                  SID=$($K agent session create --agent fortinet-cve-watch -o json | jq -r '.session.id // .id')
                  printf 'Date: %s\nFORTIGATE: %s\nNVD: %s\nKEV: %s\n' \
                    "$(date -u +%F)" "$FGT" "$NVD" "$KEV" \
                    | $K agent invoke --session "$SID" --timeout 45m --file -
EOF

kubectl -n kagent create job --from=cronjob/fortinet-cve-watch-daily cve-manual-1
kubectl -n kagent logs -f job/cve-manual-1
```

> The LLM's version-range matching is a triage aid, not an authoritative scanner. Confirm every
> **AFFECTED** finding against the FortiGuard PSIRT advisory (https://www.fortiguard.com/psirt)
> before you change anything.

---

## 9. Repo layout after a few days

```
.
├── README.md
├── cluster/kind-config.yaml
├── install/{config.yaml,kagent-values.yaml}
└── reports/
    ├── agentgateway/2026-11-07.md
    └── fortinet-cve/2026-11-07.md
```

Run `git pull` to see what the agents published. Don't commit `install/` if you put real
secrets in it. The guide keeps all secrets in Kubernetes Secrets only.

---

## 10. Troubleshooting

| Symptom | Check |
|---|---|
| Agent never becomes `Ready` | `kubectl describe agent <name> -n kagent`; `kubectl get pods -n ate-system`; `kubectl rollout status daemonset/atelet -n ate-system`; `helm list -n ate-system` (expect `substrate-atelet`); `kubectl logs -n kagent -l app.kubernetes.io/component=controller` |
| Model calls fail / time out | `kubectl -n kagent run t --rm -i --restart=Never --image=curlimages/curl -- curl -s http://dgx-spark.kagent.svc.cluster.local:8000/v1/models`; confirm the EndpointSlice IP; raise `providers.openAI.config.timeout` |
| Agent replies but never calls tools | vLLM is missing `--enable-auto-tool-choice --tool-call-parser …` (see 0.1) |
| GitHub tool errors (401/403) | PAT scope or expiry; secret value must be `Bearer <token>`; `kubectl describe remotemcpserver github -n kagent` |
| `EgressPoliciesApplied=False` | `kubectl get agent <name> -n kagent -o jsonpath='{.status.conditions}'` |
| CronJob can't reach the FortiGate | Trusted Hosts on the API admin must include the Mac's LAN IP |
| `kind create` fails on node image | Upgrade kind to ≥ v0.31 for `kindest/node:v1.35.0` |
| No EXTERNAL-IP on `kagent-ui` | Is `sudo cloud-provider-kind` still running in terminal 2? |
| Python-runtime agents crash with `Illegal instruction` | Use the `kagent` (Go ADK) harness as shown; rc3's built-in harness image is Go |

Useful views:

```bash
kubectl -n kagent get agents,harnesses,modelconfigs,remotemcpservers,egresspolicies
kubectl -n kagent get cronjobs,jobs
kagent agent list
kagent agent session list
```

## 11. Cleanup

```bash
kagent uninstall                     # removes kagent + Substrate releases
kind delete cluster --name kagent
# stop cloud-provider-kind with Ctrl-C in terminal 2
```

## 12. Upgrading from rc2

For a throwaway kind cluster the simplest path is **reinstall**: run step 11, then
steps 1–8 again. Your reports stay in Git.

To upgrade in place, follow the
[rc3 release notes](https://github.com/kagent-dev/kagent/releases/tag/v1.0.0-rc3#upgrading-from-v100-rc2).
Re-running `kagent install` alone is **not** enough. The short version:

1. Upgrade Substrate **before** kagent (Helm ≥ 3.17): create the `kagent-rustfs-credentials`
   Secret in `ate-system` from the substrate release's `rustfs.accessKey`/`secretKey` →
   `substrate-podcert` → adopt atelet into
   a new `substrate-atelet` release (annotate its objects `helm.sh/resource-policy=keep` first,
   or the next step deletes the DaemonSet) → worker image `ateworker-gvisor:v1.0.0` via the
   rc2 kagent chart → `substrate-crds` and `substrate` 1.0.0 with `atelet.enabled=false`.
2. Before the `substrate` upgrade in step 1, add `GoldenEgressPolicy` to Substrate's `preview` list. Without it, ate-api-server
   rejects the golden egress policies that rc3 sets.
3. Upgrade `kagent-crds` and `kagent` to `1.0.0-rc3`.
4. Update the CLI (step 3) and the `kagent-linux-$ARCH` download URL in both CronJobs
   (re-apply 7.3 and 8.3).

## References

- kagent v1.0.0-rc3 release: https://github.com/kagent-dev/kagent/releases/tag/v1.0.0-rc3
- Substrate v1.0.0 (atelet chart values): https://github.com/kagent-dev/substrate/blob/v1.0.0/charts/substrate-atelet/README.md#values
- Helm install / Substrate notes: https://github.com/kagent-dev/kagent/blob/v1.0.0-rc3/helm/README.md
- Egress policies: https://github.com/kagent-dev/kagent/blob/v1.0.0-rc3/docs/architecture/egress-policies.md
- A2A gateway (`/agents/{namespace}/{name}` on :8083): https://github.com/kagent-dev/kagent/blob/v1.0.0-rc3/docs/architecture/a2a-gateway.md
- cloud-provider-kind: https://github.com/kubernetes-sigs/cloud-provider-kind
- GitHub MCP server: https://github.com/github/github-mcp-server
- NVD CVE API 2.0: https://nvd.nist.gov/developers/vulnerabilities
- CISA KEV: https://www.cisa.gov/known-exploited-vulnerabilities-catalog

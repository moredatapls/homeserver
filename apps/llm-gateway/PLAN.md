# LLM Gateway — Local-first (LM Studio) with Scaleway Cloud Fallback

Status: **PLAN / not yet implemented** — hand this to an implementation agent.
Scope: add a model-gateway proxy to the `moredatapls/homeserver` Flux repo so every
LLM-consuming app (paperless-ngx, hermes-agent) talks to **one** OpenAI-compatible
endpoint that prefers the local LM Studio model on the Mac and transparently falls back
to Scaleway Generative APIs when the local model is down, slow, or has failed.

Conventions in this repo (read `.github/skills/hostpath-volume-permissions/SKILL.md` and
the `homeserver-flux-apps` skill first, then verify against live manifests):

- Flux GitOps on a single-node k0s cluster (`alderaan`), flat app dirs under `apps/<app>/`
  with **no per-app kustomization.yaml** — Flux applies every YAML in the path.
- Each app is wired via one `Kustomization` entry in `clusters/berlin/apps.yaml`
  (`path: ./apps/<name>`, `prune: true`, `wait: true`, `interval: 1m`,
  `decryption: {provider: sops, secretRef: {name: sops-gpg}}`).
- Secrets: `.sops.yaml` enforces `encrypted_regex: ^(data|stringData)$` with one GPG key.
  Convention: an **unencrypted `-local` secret** holds LAN/non-secret values; any cloud key
  goes in a SOPS-encrypted file. Encrypt with `sops encrypt --in-place <file>`.
- Ingress: ingress-nginx + cert-manager (letsencrypt) + external-dns; hosts
  `<app>.berlin.bruegner.com` with a `tls` block. Long AI requests need
  `proxy-read-timeout` / `proxy-send-timeout` raised.
- NetworkPolicies are common and **strict**: egress drops RFC1918 by default — pod-to-LAN
  (the Mac) must be explicitly allowed.

---

## 1. Recommended model (dual-use: 48GB M4 Pro LM Studio **and** Scaleway catalog)

Model must exist on BOTH sides. Verified intersection (Oct 2026):

| Model | Local fit (LM Studio, GGUF) | Scaleway serverless model (api.scaleway.ai/v1) | Cloud $/1M in/out |
|---|---|---|---|
| **Qwen 3.6 35B-A3B (MoE, 3B active)** | Q4_K_M ≈ 24 GB runtime → 48 GB M4 Pro, very fast decode (MoE) | `qwen3.6-35b-a3b` | €0.25 / €1.50 |
| Mistral Small 3.2 24B Instruct | Q4_K_M ≈ 17 GB, lightest, fastest, huge headroom | `mistral-small-3.2-24b-instruct-2506` (also the default route for EOL small models) | €0.15 / €0.35 |
| Gemma 4 26B-A4B (MoE, 4B active) | Q4_K_M ≈ 18 GB, fast; strong multilingual + tool-calling, optional vision | `google/gemma-4-26b-a4b-it` | €0.25 / €0.50 |

**Recommendation: Qwen 3.6 35B-A3B as the primary dual-use model.** It is the best
quality-per-watt on the M4 Pro and is an exact hit in the Scaleway catalog, so the *same
model id* can back both the local and cloud deployments inside the gateway. Use
**Mistral Small 3.2 24B** as the lightweight/secondary (cheapest cloud, smallest local
footprint). Both have official/curated GGUF on Hugging Face.

> Implementer MUST verify the exact Scaleway model strings from the live catalog before
> finalizing config:
> https://www.scaleway.com/en/docs/generative-apis/reference-content/model-catalog/
> The `qwen/`, `mistral/`, `google/` prefixes shown may differ; use the exact model id
> Scaleway returns. Hermes already pins `deepseek-v4-flash-0731` from this catalog with
> `transport: chat_completions` — Scaleway serves only `/v1/chat/completions` (no Responses
> API), so all gateway routes to Scaleway must use `openai/…` via `base_url` +
> `chat/completions`.

To use locally, download the matching **Q4_K_M GGUF** in LM Studio and serve it on the
local OpenAI-compatible server (default port `1234`).

## 2. Architecture — the abstraction tool (answer to Q2)

**Tool: LiteLLM Proxy** (self-hosted, OSS, OpenAI-compatible gateway). Verified fit:

- Exposes one OpenAI-compatible `https://…/v1` endpoint; any app pointing at it with
  `model=<logical-name>` gets routed automatically.
- **Local-first failover**: a single `model_name` can back multiple deployments. When the
  primary (LM Studio) deployment errors (429/5xx) or times out, LiteLLM fails over to the
  next healthy deployment (same model group) — exactly the local→cloud fallback pattern.
  Cooldowns (`cooldown_time`, `allowed_fails`) bench a failing local deployment so traffic
  stays on cloud until the Mac recovers.
- Stateless config-only deployment needs **no Postgres/Redis** (those only add spend
  tracking / multi-instance). The thin container runs comfortably on the homeserver.
- Docker image: `litellm/litellm:latest` (Docker Hub) — verify arm64 tag at deploy time.

Alternatives rejected: OpenRouter (hosted, not self-hosted), LocalAI / text-generation-webui
(local-only, no cloud failover), Open WebUI (chat UI, not an OpenAI-compatible gateway),
Semantic Kernel (SDK, needs code not config), LM Studio (no native local→cloud routing).

Deployment choice: **run the gateway as a pod in the cluster** (`apps/llm-gateway/`), so it
is Flux-managed like everything else. Apps reach it via ClusterIP DNS.
(Alternative for lowest latency: run the gateway as a container on the Mac next to LM
Studio — but that is outside Flux; cluster deployment is the primary plan to match repo
conventions.)

## 3. New app directory: `apps/llm-gateway/`

Stateless — no PV, no volume-permissions init container needed.

### 3.1 `namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llm-gateway
  labels:
    app: llm-gateway
```

### 3.2 `configmap.yaml` — LiteLLM `config.yaml`
Baked in via ConfigMap. Model group `qwen3.6-35b-a3b` has two deployments: local first,
Scaleway second. Adjust `api_base`/ports/model ids to final verified values.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: litellm-config
  namespace: llm-gateway
data:
  config.yaml: |
    model_list:
      # --- Local: LM Studio on the Mac (LAN) ---
      - model_name: qwen3.6-35b-a3b
        litellm_params:
          model: openai/qwen3.6-35b-a3b
          api_base: http://192.168.178.93:1234/v1
          api_key: lm-studio            # ignored by LM Studio; any value
          timeout: 20
          stream_timeout: 20
        model_info:
          mode: chat
      # --- Cloud: Scaleway Generative APIs ---
      - model_name: qwen3.6-35b-a3b
        litellm_params:
          model: openai/qwen3.6-35b-a3b
          api_base: https://api.scaleway.ai/v1
          api_key: os.environ/SCALEWAY_API_KEY
          timeout: 120
          stream_timeout: 300
        model_info:
          mode: chat
    router_settings:
      routing_strategy: simple-shuffle   # round-robin within the group
      num_retries: 2                     # retry the local hop before failing over
      request_timeout: 30                # hung local => TimeoutError => cloud fallback
      cooldown_time: 60                  # seconds a failing deployment is benched
      allowed_fails: 3                   # fails before cooldown kicks in
    general_settings:
      master_key: os.environ/LITELLM_MASTER_KEY   # optional; set a key if desired
```
Notes for the implementer:
- `request_timeout` must exceed LM Studio’s cold-load time if models lazy-load
  (raise to 30–60 s, else the first request after idle can falsely fail over).
- Add `context_window_fallbacks` / `content_policy_fallbacks` if you want explicit routing
  on context-window or refusal errors; the plain multi-deployment group already covers
  failures and timeouts.
- To relax strict "same model both sides", add a second always-cloud group (e.g.
  `deepseek-v4-flash-0731` → Scaleway only) so Hermes can keep its flash default if wanted.

### 3.3 `secret-scaleway.yaml` — SOPS-encrypted cloud credential
Create a **plain** file, then `sops encrypt --in-place secret-scaleway.yaml`.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: scaleway-credentials
  namespace: llm-gateway
type: Opaque
stringData:
  api-key: "<SCALEWAY_API_KEY>"
  # optional master key for the gateway (any value the gateway enforces)
  master-key: "<LITELLM_MASTER_KEY>"
```
`.sops.yaml` already targets `data|stringData` — after `sops encrypt --in-place` the values
become `ENC[AES256_GCM,…]` with a `sops:` block. The SOPS public key is in the repo
(`.sops.pub.asc`, FP `CA5041D30E6B1755A06AD223242A7AC1A18F40E8`); the private key lives in
the password-safe and must be present to decrypt at reconcile (Flux already has it).

### 3.4 `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: llm-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm-gateway
  template:
    metadata:
      labels:
        app: llm-gateway
    spec:
      containers:
        - name: litellm
          image: litellm/litellm:latest   # verify tag/arch at deploy time
          args: ["--config", "/app/config.yaml"]
          ports:
            - containerPort: 4000
              name: http
          env:
            - name: SCALEWAY_API_KEY
              valueFrom:
                secretKeyRef:
                  name: scaleway-credentials
                  key: api-key
            - name: LITELLM_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: scaleway-credentials
                  key: master-key
                  optional: true
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health/liveliness
              port: 4000
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /health/readiness
              port: 4000
            initialDelaySeconds: 10
          volumeMounts:
            - name: config
              mountPath: /app/config.yaml
              subPath: config.yaml
      volumes:
        - name: config
          configMap:
            name: litellm-config
```

### 3.5 `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: llm-gateway
spec:
  selector:
    app: llm-gateway
  ports:
    - port: 4000
      targetPort: 4000
      name: http
```
Internal address used by apps: `http://gateway.llm-gateway.svc.cluster.local:4000/v1`.

### 3.6 `ingress.yaml` (optional; management/diagnostics)
Expose the gateway on the LAN/public for manual calls and the LiteLLM dashboard. Skip if
you only want cluster-internal access (paperless + hermes reach it via the Service).
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway
  namespace: llm-gateway
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
    - host: llm.berlin.bruegner.com
      http:
        paths:
          - backend:
              service:
                name: gateway
                port:
                  number: 4000
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - llm.berlin.bruegner.com
      secretName: llm-gateway-tls
```

### 3.7 `networkpolicy.yaml` — strict, matching repo style
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gateway
  namespace: llm-gateway
spec:
  podSelector:
    matchLabels:
      app: llm-gateway
  policyTypes: [Ingress, Egress]
  ingress:
    # apps that consume the gateway
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: paperless-ngx
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: hermes-agent-h
      ports:
        - protocol: TCP
          port: 4000
    # if the ingress is exposed, allow ingress-nginx
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 4000
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # the local LM Studio on the Mac (LAN) — MUST be explicit, RFC1918 is dropped by default
    - to:
        - ipBlock:
            cidr: 192.168.178.93/32
      ports:
        - protocol: TCP
          port: 1234
    # public internet for Scaleway (https), port 443
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 0.0.0.0/8
              - 10.0.0.0/8
              - 100.64.0.0/10
              - 127.0.0.0/8
              - 169.254.0.0/16
              - 172.16.0.0/12
              - 192.168.0.0/16
              - 224.0.0.0/4
              - 240.0.0.0/4
        - ipBlock:
            cidr: ::/0
            except:
              - ::1/128
              - fc00::/7
              - fe80::/10
              - ff00::/8
      ports:
        - protocol: TCP
          port: 443
```
> If the gateway must also reach a LAN HTTPS endpoint you will have nothing to add; port
> 1234 is the only LAN port needed. Keep egress tight.

## 4. Wire the app into Flux — `clusters/berlin/apps.yaml`

Append a Kustomization (copy an existing entry’s shape):
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps-llm-gateway
  namespace: flux-system
spec:
  dependsOn:
    - name: infra-controllers-ingress-nginx
    - name: infra-controllers-cert-manager
    - name: infra-controllers-external-dns
  interval: 1m
  retryInterval: 2m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/llm-gateway
  prune: true
  wait: true
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

## 5. Migrate consumers to the gateway

### 5.1 Paperless-ngx (single-endpoint, no native fallback)
Point its one LLM endpoint at the gateway so fallback happens at the gateway, not in
paperless. In `apps/paperless-ngx/deployment.yaml` the three AI env vars come from
`ai-credentials-local` (keyRef `optional: true`). Update that Secret
(`apps/paperless-ngx/ai-credentials-local.yaml`, unencrypted — contains only non-secret
LAN/endpoint values):
```yaml
stringData:
  api-key: "any"                                  # gateway ignores local key
  endpoint: "http://gateway.llm-gateway.svc.cluster.local:4000/v1"
  model: "qwen3.6-35b-a3b"                        # gateway logical model group
```
No change to `PAPERLESS_AI_LLM_BACKEND: openai-like` or other config. Keep
`ai-credentials.yaml` (SOPS cloud) as-is for now; optionally point the fallback endpoint at
the gateway later.

### 5.2 Hermes-agent (HelmRelease, currently `custom:scaleway`)
In `apps/hermes-agent-h/hermes-agent.yaml` change the provider to the gateway, keeping
`transport: chat_completions` (mandatory for Scaleway-backed routes):
```yaml
config:
  values:
    providers:
      gateway:
        api: ${OPENAI_BASE_URL}            # now the gateway URL
        key_env: OPENAI_API_KEY
        transport: chat_completions
    model:
      default: qwen3.6-35b-a3b             # or deepseek-v4-flash-0731 (always-cloud group)
      provider: custom:gateway
      base_url: null
```
Update the SOPS secret `hermes-h-secrets`:
`OPENAI_BASE_URL = http://gateway.llm-gateway.svc.cluster.local:4000/v1`,
`OPENAI_API_KEY = <gateway master key or any>`. Re-encrypt with `sops encrypt --in-place`.
Also add the `llm-gateway` namespace to Hermes’ egress NetworkPolicy
(`apps/hermes-agent-h/netpol-workspace.yaml` and the egress of `hermes-agent.yaml`), and
ensure the gateway NetPolicy above allows ingress from `hermes-agent-h`.
> If Hermes should keep its fast cloud-only flash model, add a second model group
> `deepseek-v4-flash-0731` (Scaleway-only) to the gateway config and set
> `model.default: deepseek-v4-flash-0731` — then it never uses the Mac.

## 6. Execution checklist (for the implementing agent)

1. Clone and branch: `git checkout -b feat/llm-gateway`.
2. Create `apps/llm-gateway/` with the manifests in §3 (config, secret-placeholder,
   deployment, service, ingress, networkpolicy, namespace).
3. Create placeholder `secret-scaleway.yaml`, then `sops encrypt --in-place` it
   (requires the repo’s SOPS private key; otherwise commit the file and encrypt in CI/pwd-safe).
4. Append the `apps-llm-gateway` Kustomization to `clusters/berlin/apps.yaml`.
5. Update paperless `ai-credentials-local.yaml` endpoint/model (§5.1).
6. Update hermes provider + secret + netpol egress (§5.2).
7. Validate YAML: `kubectl apply --dry-run=client -f apps/llm-gateway/` (or `flux build` if
   available); confirm no per-app kustomization.yaml was added.
8. Push branch, open PR, let Flux reconcile, then verify:
   ```sh
   kubectl -n llm-gateway get pods
   kubectl -n llm-gateway rollout status deployment/gateway
   kubectl -n llm-gateway logs deploy/gateway -c litellm
   # end-to-end, local up:
   curl -s http://<gateway-svc>:4000/v1/chat/completions \
     -H 'content-type: application/json' -d '{"model":"qwen3.6-35b-a3b","messages":[{"role":"user","content":"hi"}]}'
   ```
9. Failover test: stop LM Studio (or block `192.168.178.93:1234`) and confirm the same call
   returns a cloud answer after the retry/timeout; verify a cooldown logs for the local
   deployment. Restart LM Studio and confirm traffic returns local-first.
10. Update README (optional) and this doc’s Status line to implemented once validated.

## 7. Open items / must-verify before finalizing

- Exact Scaleway model ids (`qwen3.6-35b-a3b` etc.) from the live model catalog.
- Latest LiteLLM image (Docker Hub `litellm/litellm`) and its `--config` flag syntax.
- Whether LM Studio on the Mac lazy-loads; set `request_timeout` 30–60 s if so.
- The gateway port (`4000`) vs. any existing in-cluster service on that port (none seen).
- Confirm Mac’s LAN IP (`192.168.178.93`) is current (from `ai-credentials-local`).

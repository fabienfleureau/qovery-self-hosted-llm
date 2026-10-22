# LLM operator runbook: prepare a Qovery environment

Use this runbook when a user authorizes preparing a self-hosted LLM environment
with this repository. Use **Qovery CLI for all Qovery service creation,
configuration and deployment**. `qovery api` is an authenticated CLI command;
there is no need to extract its token or call the Qovery API with curl.
Use kubectl for cluster discovery, readiness checks and the explicitly listed
namespace prerequisite. Helm is for local dependency resolution and rendering.

The expected result is:

```text
Agent Task → public LiteLLM → private Envoy Gateway → Knative → KServe GPU model
```

For the complete seven-service deployment described here, every Helm service
must reference this chart, not the upstream charts directly. This chart
references the pinned official dependencies itself.
Keep LiteLLM, networking and controllers on CPU. Only the model requests a GPU.
Do not change ai-runner Rust code or publish a Service Catalog entry.

## 1. Inputs and operating rules

Obtain the intended organization, project, cluster and environment IDs from
the user's URL or CLI discovery. If the environment does not exist, obtain a
name and create it in the specified project and cluster. A URL is not evidence
of permission to alter unrelated environments or shared cluster controllers.
Discover account context before asking the user for information already there.

Required tools: authenticated `qovery`, `kubectl`, Helm 3 or 4, `git`, Python 3,
PyYAML and `jq`. Check availability rather than changing the user's global
Python installation. A private venv can supply PyYAML if needed.

Use bash for the shell blocks below, from a clone of this repository:

```bash
set -euo pipefail
umask 077
export ORG_ID='<organization-id>'
export PROJECT_ID='<project-id>'
export CLUSTER_ID='<cluster-id>'
# Set ENV_ID when reusing an environment; otherwise set it after creating one.
export ENV_ID='<environment-id>'
export REPO_ROOT="$PWD"
export RUN_DIR="$(mktemp -d "${TMPDIR:-/tmp}/qovery-llm.XXXXXX")"
export CHART="$REPO_ROOT/charts/qovery-kserve-config"
mkdir -p "$RUN_DIR/values"
qovery version
qovery api spec > "$RUN_DIR/qovery-openapi.yaml"
git rev-parse HEAD > "$RUN_DIR/repository-commit.txt"
qovery api organization
qovery api "organization/$ORG_ID/project"
qovery api "organization/$ORG_ID/cluster"
qovery api "project/$PROJECT_ID/environment"
```

Commands and request shapes below were checked with CLI 1.174.0 and the live
API on 2026-10-22. Inspect the downloaded specification if a request is rejected;
do not invent an endpoint, force a source change or loop over failed mutations.
Do not print credentials, turn on shell tracing, commit generated working
files, or bake credentials into an image. Read secrets from private files and
send them through subprocess stdin. Keep an inventory of IDs and execution IDs.

For an existing environment, check its project and cluster before proceeding:

```bash
qovery api "environment/$ENV_ID" > "$RUN_DIR/environment.json"
qovery api "environment/$ENV_ID/helm" > "$RUN_DIR/helm-before.json"
qovery api "environment/$ENV_ID/agenticWorkflow" > "$RUN_DIR/workflows-before.json"
qovery api "environment/$ENV_ID/deploymentStage" > "$RUN_DIR/stages-before.json"
```

If creating an environment instead, leave existing environments alone:

```bash
export ENV_NAME='<new-environment-name>'
python3 - <<'PY' | qovery api "project/$PROJECT_ID/environment" --method POST --input - > "$RUN_DIR/environment.json"
import json, os
print(json.dumps({"name": os.environ["ENV_NAME"], "cluster": os.environ["CLUSTER_ID"],
                  "mode": "DEVELOPMENT"}))
PY
export ENV_ID="$(jq -r '.id' "$RUN_DIR/environment.json")"
```

## 2. Inspect the real cluster before deploying

Download credentials into the private working directory, not the repository:

```bash
(cd "$RUN_DIR" && qovery cluster kubeconfig --cluster-id "$CLUSTER_ID")
# Read the CLI output and set this to the actual file it created.
export KUBECONFIG="$RUN_DIR/<generated-kubeconfig-file>"
kubectl version -o yaml > "$RUN_DIR/kubernetes-version.yaml"
kubectl get nodes -L kubernetes.io/arch,karpenter.sh/nodepool
kubectl get gatewayclass -o yaml > "$RUN_DIR/gatewayclasses.yaml"
kubectl get gateway -A -o yaml > "$RUN_DIR/gateways-before.yaml"
kubectl get httproute -A -o json > "$RUN_DIR/routes-before.json"
kubectl get deployments -A -o json > "$RUN_DIR/controllers-before.json"
kubectl get crd > "$RUN_DIR/crds-before.txt"
kubectl get nodepools,nodeclaims -o yaml > "$RUN_DIR/karpenter-before.yaml"
```

Check Kubernetes, Gateway API CRD versions, Envoy Gateway and Envoy images,
cert-manager, existing KServe/Knative installations and Helm ownership.
Compare them with the pinned chart versions and official upstream compatibility
documentation. The tested cluster used Kubernetes 1.35.8, Gateway API 1.6.1,
Envoy Gateway 1.9.1, KServe 0.16.0 and Knative/net-gateway-api 1.23.0. This is
observed compatibility for that cluster, not a general support matrix.

The profiles assume `qovery-private-gateway` uses controller
`qovery.com/gateway-controller`. Verify the actual class is Accepted and
compatible with the adapter. Create a dedicated private Gateway in the target
environment; do not repurpose Qovery's shared public Gateway. net-gateway-api
1.23 is beta. Do not install Istio, Kourier or KEDA.

Verify the GPU pool permits amd64 NVIDIA instances, its taints match the model
tolerations, the NVIDIA device plugin works and quotas permit one GPU. Inspect
the permitted CPU instance sizes: a 2-CPU Agent Task could not schedule on the
tested `large` pool once daemon and sidecar requests were included. Use 1000m
CPU and 2048 MiB for the initial task. Empty GPU nodes must be eligible for
Karpenter consolidation. Never modify pool policy or other workloads implicitly.

If shared KServe/Knative controllers already exist, stop the fresh-controller
path and inspect their versions, configuration, owner releases and users.
Do not install a second writer or claim another release's resources. Reuse
compatible controllers only when the intended scope permits it. The source
migration case is described in section 9; it is not a fresh install.
If reusing shared controllers, record which prerequisite stages were skipped
and their existing owner releases; do not claim the seven-service path was run.

## 3. Prepare profiles locally

```bash
cp "$REPO_ROOT"/examples/*.yaml "$RUN_DIR/values/"
helm dependency build "$CHART"
for profile in "$RUN_DIR"/values/*.yaml; do
  helm lint "$CHART" -f "$profile"
done
```

The examples target ARM64 CPU nodes and an amd64 `gpu` node pool. Adapt all
CPU selectors/affinities together if the target cluster uses amd64 CPUs.
Preserve the pinned Qwen3 0.6B weights and llama.cpp CUDA image during initial
validation. Keep the model context at 32768: the real SDK sent 13174 prompt
tokens, exceeding the previous 8192-token context. Keep min/max replicas 0/1,
GPU request and limit 1, the node selectors/tolerations and PodSpec feature flags.

There are two deferred substitutions:

- `MODEL_NAMESPACE`: discover it after the first Qovery Helm deployment.
- `ENVOY_SERVICE`: discover it after the private Gateway is programmed.

Do not guess either value from a Qovery service ID. Do not deploy a profile
containing unresolved placeholders. The first two prerequisite profiles do
not need these substitutions.

For Qovery-managed credentials, adapt the copied LiteLLM profile to the same
secret mechanism used in the live POC. The checked-in profile alternatively
supports a pre-existing Kubernetes Secret; choose one mechanism, not both.

```bash
python3 - <<'PY'
import os, pathlib, yaml
p = pathlib.Path(os.environ['RUN_DIR']) / 'values/litellm-qwen.yaml'
v = yaml.safe_load(p.read_text())
v['litellm'].pop('masterkeySecretName', None)
v['litellm'].pop('masterkeySecretKey', None)
v['litellm']['masterkey'] = 'qovery.env.LITELLM_MASTER_KEY'
p.write_text(yaml.safe_dump(v, sort_keys=False))
PY
```

## 4. Create/configure the seven services through Qovery CLI

Write this helper to `$RUN_DIR/manage.py`. It creates or updates configuration,
but deploys only when explicitly called with `deploy`. It rejects an existing
service that uses a different chart. Its inventory contains IDs, not secrets.

```python
import json, os, pathlib, subprocess, sys, time

run = pathlib.Path(os.environ['RUN_DIR'])
env_id = os.environ['ENV_ID']
repo = 'https://github.com/fabienfleureau/qovery-self-hosted-llm.git'
root = '/charts/qovery-kserve-config'

def api(path, method='GET', body=None):
    cmd = ['qovery', 'api', path, '--method', method]
    if body is not None:
        cmd += ['--input', '-']
    r = subprocess.run(cmd, input=json.dumps(body) if body is not None else None,
                       capture_output=True, text=True, check=True)
    return json.loads(r.stdout) if r.stdout.strip() else None

action, name = sys.argv[1:3]
inventory_file = run / 'services.json'
inventory = json.loads(inventory_file.read_text()) if inventory_file.exists() else {}
if action == 'apply':
    profile, cluster_wide = sys.argv[3:5]
    content = (run / 'values' / profile).read_text()
    assert 'MODEL_NAMESPACE' not in content and 'ENVOY_SERVICE' not in content
    services = api('environment/' + env_id + '/helm')['results']
    existing = next((s for s in services if s['name'] == name), None)
    if existing:
        actual = existing['source'].get('git', {}).get('git_repository', {})
        assert actual.get('url') == repo and actual.get('root_path') == root, \
            'STOP: existing service uses another chart; ownership migration required'
    body = {'name': name, 'description': 'Self-hosted LLM parent chart',
            'auto_deploy': False, 'timeout_sec': 1500,
            'allow_cluster_wide_resources': cluster_wide == 'true',
            'source': {'git_repository': {'url': repo, 'branch': 'main', 'root_path': root}},
            'arguments': [], 'ports': [],
            'values_override': {'file': {'raw': {'values': [
                {'name': 'poc-overrides.yaml', 'content': content}]}}}}
    if name == 'llm-gateway':
        body['ports'] = [{'service_name': 'poc-litellm', 'internal_port': 4000,
                          'external_port': 443, 'protocol': 'HTTP',
                          'name': 'http', 'is_default': True}]
    if existing:
        body['ports'] = existing['ports']
    result = api('helm/' + existing['id'], 'PUT', body) if existing else \
        api('environment/' + env_id + '/helm', 'POST', body)
    inventory[name] = result['id']
    inventory_file.write_text(json.dumps(inventory, indent=2) + '\n')
    print(name, result['id'])
elif action == 'deploy':
    path = 'helm/' + inventory[name]
    previous = api(path + '/status').get('execution_id')
    api(path + '/deploy', 'POST', {})
    deadline = time.monotonic() + 1800
    while time.monotonic() < deadline:
        status = api(path + '/status')
        if status.get('execution_id') != previous:
            if status['state'] == 'DEPLOYED':
                print(name, status['state'], status.get('execution_id'))
                break
            if status['state'].endswith('ERROR') or status['state'] == 'CANCELED':
                raise RuntimeError(status)
        time.sleep(5)
    else:
        raise TimeoutError(name)
else:
    raise ValueError(action)
```

Use the override filename `poc-overrides.yaml`. Calling it `values.yaml` can
overwrite the chart's default values during Qovery's preparation. Avoid YAML
anchors/aliases in raw values; the supplied Operator profile uses JSON syntax,
which is valid YAML. Helm dependency conditions do not provide readiness gates.

### Stage 1: CRDs and Operator

Only proceed here on the fresh-controller path described in section 2.

```bash
python3 "$RUN_DIR/manage.py" apply llm-crds kserve-crds.yaml true
python3 "$RUN_DIR/manage.py" apply llm-operator knative-operator.yaml true
python3 "$RUN_DIR/manage.py" deploy llm-crds
python3 "$RUN_DIR/manage.py" deploy llm-operator
export MODEL_NAMESPACE="$(kubectl get ns -l "qovery.com/environment-id=$ENV_ID" -o jsonpath='{.items[0].metadata.name}')"
test -n "$MODEL_NAMESPACE"
kubectl wait --for=condition=Established crd/inferenceservices.serving.kserve.io --timeout=180s
kubectl wait --for=condition=Established crd/knativeservings.operator.knative.dev --timeout=180s
kubectl -n "$MODEL_NAMESPACE" rollout status deploy/knative-operator --timeout=300s
kubectl -n "$MODEL_NAMESPACE" rollout status deploy/operator-webhook --timeout=300s
```

Check the Operator webhook Service has ready endpoints before stage 3.
If deployment names differ, discover them from the rendered chart and live
objects rather than weakening the gate. Qovery DEPLOYED alone is insufficient.

Substitute the discovered namespace in the working copies:

```bash
python3 - <<'PY'
import os, pathlib
for p in (pathlib.Path(os.environ['RUN_DIR']) / 'values').glob('*.yaml'):
    p.write_text(p.read_text().replace('MODEL_NAMESPACE', os.environ['MODEL_NAMESPACE']))
PY
```

### Stage 2: private Gateway

```bash
python3 "$RUN_DIR/manage.py" apply llm-private-gateway private-gateway.yaml true
python3 "$RUN_DIR/manage.py" deploy llm-private-gateway
kubectl -n "$MODEL_NAMESPACE" wait --for=condition=Programmed gateway/poc-knative-internal --timeout=300s
export ENVOY_SERVICE="$(kubectl -n qovery get svc -l "gateway.envoyproxy.io/owning-gateway-name=poc-knative-internal,gateway.envoyproxy.io/owning-gateway-namespace=$MODEL_NAMESPACE" -o jsonpath='{.items[0].metadata.name}')"
test -n "$ENVOY_SERVICE"
kubectl -n qovery get svc "$ENVOY_SERVICE"
python3 - <<'PY'
import os, pathlib
for p in (pathlib.Path(os.environ['RUN_DIR']) / 'values').glob('*.yaml'):
    p.write_text(p.read_text().replace('ENVOY_SERVICE', os.environ['ENVOY_SERVICE']))
PY
```

The generated Envoy Service must be ClusterIP. If Envoy runs in a namespace
other than `qovery`, adjust both the Serving and KServe controller profiles.
Both adapter gateway entries must point to this private Gateway and Service.

### Stage 3: Knative Serving

The profile writes to namespace `knative-serving`. If absent, create that
namespace as a prerequisite; do not recreate or relabel an existing namespace:

```bash
# Run only after discovery has confirmed this namespace does not exist.
kubectl create namespace knative-serving
```

```bash
python3 "$RUN_DIR/manage.py" apply llm-serving knative-serving.yaml true
python3 "$RUN_DIR/manage.py" deploy llm-serving
kubectl -n knative-serving wait --for=condition=Ready knativeserving/knative-serving --timeout=900s
kubectl -n knative-serving get deployments,pods
kubectl -n knative-serving get cm config-gateway config-features config-autoscaler -o yaml
```

Verify Serving version 1.23.0, net-gateway-api controllers, the required PodSpec
feature flags, private gateway entries, CPU scheduling and scale-to-zero.
Do not continue with a failed webhook or a partially reconciled installation.

### Stages 4–5: KServe and private GPU model

```bash
python3 "$RUN_DIR/manage.py" apply llm-controller kserve-controller.yaml true
python3 "$RUN_DIR/manage.py" deploy llm-controller
kubectl -n "$MODEL_NAMESPACE" rollout status deploy/kserve-controller-manager --timeout=600s
python3 "$RUN_DIR/manage.py" apply llm-model qwen-knative-gpu.yaml false
python3 "$RUN_DIR/manage.py" deploy llm-model
kubectl -n "$MODEL_NAMESPACE" wait --for=condition=Ready inferenceservice/poc-qwen-knative --timeout=1200s
kubectl -n "$MODEL_NAMESPACE" get ksvc,revision,inferenceservice
```

Check the KServe webhook's endpoints before submitting the model. Verify the
Knative Service is cluster-local and its URL matches LiteLLM's backend URL.
Do not point LiteLLM at the revision-private Service or a pod IP; those bypass
Knative activation at zero replicas. The GPU rollout can temporarily overlap
revisions: maxReplicas=1 is per revision, not a global GPU node limit.

### Stage 6: LiteLLM and its secret

```bash
python3 "$RUN_DIR/manage.py" apply llm-gateway litellm-qwen.yaml false
export GATEWAY_ID="$(jq -r '."llm-gateway"' "$RUN_DIR/services.json")"
# Point at an existing private key file, or generate one for a new installation.
export KEY_FILE="$RUN_DIR/gateway-key"
python3 - <<'PY'
import os, pathlib, secrets
p = pathlib.Path(os.environ['KEY_FILE'])
if not p.exists():
    p.write_text('sk-' + secrets.token_hex(24) + '\n')
p.chmod(0o600)
PY
```

Before creating the variable, list only its metadata. If it exists, preserve
the key and use the existing private credential source; do not silently rotate
it or overwrite it with a newly generated value.

```bash
qovery api "variable?scope=HELM&parent_id=$GATEWAY_ID&is_secret=true" \
  | jq '.results[] | {id,key,scope,is_secret}'
# Only when LITELLM_MASTER_KEY is absent:
python3 - <<'PY' | qovery api variable --method POST --input - > /dev/null
import json, os, pathlib
print(json.dumps({'key': 'LITELLM_MASTER_KEY',
 'value': pathlib.Path(os.environ['KEY_FILE']).read_text().strip(),
 'is_secret': True, 'variable_scope': 'HELM',
 'variable_parent_id': os.environ['GATEWAY_ID']}))
PY
qovery api "helm/$GATEWAY_ID/advancedSettings" > "$RUN_DIR/gateway-settings.json"
python3 - <<'PY' | qovery api "helm/$GATEWAY_ID/advancedSettings" --method PUT --input - > /dev/null
import json, os, pathlib
v = json.loads((pathlib.Path(os.environ['RUN_DIR']) / 'gateway-settings.json').read_text())
v['network.gateway_api.http_request_timeout_seconds'] = 1200
print(json.dumps(v))
PY
python3 "$RUN_DIR/manage.py" deploy llm-gateway
kubectl -n "$MODEL_NAMESPACE" rollout status deploy/poc-litellm --timeout=600s
qovery api "helm/$GATEWAY_ID/link" > "$RUN_DIR/gateway-links.json"
export GATEWAY_URL="$(jq -r '.results[] | select(.is_default) | .url' "$RUN_DIR/gateway-links.json")"
test -n "$GATEWAY_URL"
```

Discover the URL from Qovery; never construct its generated hostname. Keep
LiteLLM on CPU. The profiles use a 900-second LiteLLM/Knative request timeout
and 1200-second public gateway timeout. No database, Redis or persistent usage
accounting is configured. An invalid key can return HTTP 400 `no_db_connection`.

## 5. Configure a real Agent Task

Create a dedicated manual test workflow through
`POST environment/{environmentId}/agenticWorkflow`, or inspect a user-selected
workflow first. Preserve its prompt, connectors and credential references
unless changing them is part of the request. Do not attach personal connectors
or clone the whole environment for this smoke test.

Use the **Anthropic-compatible** LLM provider (`ANTHROPIC_COMPATIBLE` in the
API). Its `base_url` is the gateway API root, and its write-only `credential`
is the LiteLLM key. Configure these provider fields instead of workflow auth
variables or a Docker URL override.

First inspect available providers and reuse a matching one the caller may
use. Do not overwrite an existing provider's URL, credential or scope merely
because its name matches:

```bash
qovery api "organization/$ORG_ID/llmProvider" > "$RUN_DIR/providers.json"
```

If a new provider is needed, choose its name and scope. `USER` is personal to
the caller; use `ORGANIZATION` only when a shared credential is intended and
the caller has the required permission. The scope cannot be changed later.

```bash
export PROVIDER_NAME='self-hosted-llm'
export PROVIDER_SCOPE='USER'
# Run only if a suitable accessible provider does not already exist.
python3 - <<'PY' | qovery api "organization/$ORG_ID/llmProvider" --method POST --input - > "$RUN_DIR/provider.json"
import json, os, pathlib
print(json.dumps({
    'name': os.environ['PROVIDER_NAME'],
    'type': 'ANTHROPIC_COMPATIBLE',
    'scope': os.environ['PROVIDER_SCOPE'],
    'base_url': os.environ['GATEWAY_URL'],
    'credential': pathlib.Path(os.environ['KEY_FILE']).read_text().strip(),
}))
PY
export LLM_PROVIDER_ID="$(jq -r '.id' "$RUN_DIR/provider.json")"
qovery api "llmProvider/$LLM_PROVIDER_ID/models"
```

For an existing provider, set `LLM_PROVIDER_ID` to its discovered ID instead
of running the create block. Its credential is never returned by the API;
inspect `has_credential` and validate model discovery. A blank credential on
edit preserves the stored value; a nonblank value rotates it. An
`ANTHROPIC_COMPATIBLE` provider edit must resend `base_url`: omitting it or
sending null is rejected. Use the API root without `/v1/messages`; an optional
path prefix is allowed, such as OpenRouter's `https://openrouter.ai/api`.

Create the test workflow with the following request shape. Import `json`,
`os` and `urllib.parse` in the caller and read `gateway_url` from `GATEWAY_URL`:

```python
body = {
    'name': 'llm-agent-smoke', 'enabled': True,
    'execution_mode': 'IN_PLACE', 'schedule': None,
    'description': 'Manual self-hosted Agent SDK validation',
    'docker_fragment': '',
    'model': {'type': 'CLAUDE', 'llm_provider_id': os.environ['LLM_PROVIDER_ID'],
              'settings': json.dumps({'model': 'poc-qwen', 'effort': 'low'})},
    'agent_prompt': 'Reply with exactly: POC_AGENT_OK. Do not use any tools.',
    'mcp_servers': [], 'context_service_ids': [], 'outputs': [],
    'project_repositories': [],
    'governance': {'host_allowlist': [urllib.parse.urlparse(gateway_url).hostname]},
    'resources': {'cpu_milli': 1000, 'ram_mib': 2048, 'gpu': 0, 'storage_gib': 0},
}
```

The provider type is `ANTHROPIC_COMPATIBLE`; the workflow protocol type remains
`CLAUDE`. Do not put `ANTHROPIC_COMPATIBLE` in `model.type`, attach a Bedrock
provider, or send `model.api_key` together with `llm_provider_id`. Qovery uses
the provider URL and sends its credential as a bearer token, not an API key.
No manual auth variables or Docker URL override are required. For an existing
workflow, remove obsolete `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN` and
`ANTHROPIC_API_KEY` variables and only the corresponding old Docker ENV lines;
preserve unrelated variables, Docker instructions, prompts and connectors.

Set workflow-scoped tuning variables through `POST variable` with
`variable_scope=AGENTIC_WORKFLOW` and `variable_parent_id=<workflow-id>`:

| Key | Value |
| --- | --- |
| `MAX_TURNS` | `3` |
| `API_TIMEOUT_MS` | `900000` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `poc-qwen` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `poc-qwen` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `poc-qwen` |

The earlier POC needed a Docker URL override; that workaround predates the
provider feature and is not the recommended path. These provider fields were
checked against the current API specification. Run the runtime validation
below before claiming the new configuration works in the target environment.

Run with `POST agenticWorkflow/{workflowId}/deploy`, body `{}`. Inspect
`agenticWorkflow/{workflowId}/deploymentHistoryV2`, the actual Job env and logs.
Verify the generated URL matches the provider base URL, the auth token is
present, and `ANTHROPIC_API_KEY` is explicitly empty. Check the effective
process environment while the task is running; report credential presence
only, never the value or the complete environment.
Record an SDK `result` with `is_error=false`/successful logs and exit code 0.
Model self-identification is not authoritative evidence of its model version.
The tiny Qwen model may ignore exact-output instructions even on a successful
SDK run. Treat that as a quality limitation, not an authentication failure.

## 6. Set dependency stages and redeploy

Use these seven stages in order; CRDs and Operator share the first stage:

```text
01 - CRDs and Operator: llm-crds, llm-operator
02 - Private Gateway: llm-private-gateway
03 - Knative Serving: llm-serving
04 - KServe controller: llm-controller
05 - Model: llm-model
06 - LiteLLM: llm-gateway
07 - Agent validation: the test workflow
```

List/create stages with `environment/{environmentId}/deploymentStage`.
Move each stage with
`PUT deploymentStage/{laterStageId}/moveAfter/{previousStageId}` and assign
services with `PUT deploymentStage/{stageId}/service/{serviceId}`.
Reuse matching stages. Do not delete or reorder unrelated stages/services.
Before a full environment deploy, inspect its complete service list: it will
also run unrelated jobs if this is not a dedicated environment.

Deploy through `POST environment/{environmentId}/deploy`, body `{}`, or
`qovery environment deploy` with explicit organization/project/environment
names. Poll its status to a terminal result. On an existing cluster, the
tested full redeploy passed; a fresh cluster bootstrap remains untested.
Recheck runtime readiness, not just the queued response or a Helm service's
DEPLOYED state. Do not claim a full deploy passed while a later stage is running.

## 7. Validation and completion evidence

Run each check and save results without keys. Use the gateway key from its
private file; HTTP requests can be made with Python urllib to avoid putting
credentials in process arguments. For a manual curl example:

```bash
python3 - <<'PY' | curl --config - --fail-with-body --max-time 1100 \
  "$GATEWAY_URL/v1/messages" -H 'Content-Type: application/json' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{"model":"poc-qwen","max_tokens":128,"messages":[{"role":"user","content":"Reply with hello"}]}'
import os, pathlib
key = pathlib.Path(os.environ['KEY_FILE']).read_text().strip()
print('header = "Authorization: Bearer ' + key + '"')
PY
```

Required evidence:

1. All seven Helm services point to this public repository and chart path;
   record the actual deployed commits, chart/dependency versions and IDs.
2. Invalid credential rejected, model discovery, OpenAI completion, Anthropic
   messages, streaming, a forced tool call, streamed JSON tool arguments and
   a tool-result round trip. Verify content and stop reasons, not just HTTP 200.
3. An actual Agent Task succeeds against this endpoint, including Job exit and
   SDK result. Distinguish a protocol pass from model task-quality validation.
4. `nvidia-smi` in the model container shows the allocated GPU and model memory.
   Knative reports min0/max1 and traffic reaches the Activator at zero replicas.
5. After traffic stops, observe zero model pods. Then separately observe whether
   Karpenter removes the empty GPU node and NodeClaim. Don't delete the node to
   manufacture a passing observation. Other workloads may prevent removal.
6. Send a request from the observed idle state; measure it with a monotonic
   timer. Say whether that state had zero pods only or also zero GPU nodes.
7. A full redeploy succeeds in dependency order. Compare unrelated HTTPRoute
   specs with the preflight snapshot; ignore status/resourceVersion changes.

Useful observation commands:

```bash
kubectl -n "$MODEL_NAMESPACE" get pods -l serving.knative.dev/service=poc-qwen-knative-predictor
kubectl get nodes,nodeclaims -l karpenter.sh/nodepool=gpu
kubectl -n "$MODEL_NAMESPACE" get ksvc poc-qwen-knative-predictor -o yaml
kubectl -n "$MODEL_NAMESPACE" get endpointslices -o yaml
kubectl -n knative-serving get pods -l app=activator -o wide
kubectl get httproute -A -o json > "$RUN_DIR/routes-after.json"
```

Identify the model pod/container from Kubernetes before running `nvidia-smi`.
Report only observed timing. Previous reference measurements were 4.464 s for
a warm SDK task and 252.843 s for a cold task starting at zero model pods;
the previous GPU node was still terminating. These are not guarantees.
Qovery returned a negative total-duration metric for one deployment; use a
monotonic timer instead of interpreting that value as elapsed time.

If a test was skipped or blocked, report it explicitly. Leave the model able
to scale to zero and stop sending test traffic when finished. Record whether
GPU node removal completed or remains in progress. Hand over the environment
URL, current endpoint/model alias, IDs, profiles, versions, evidence and limits.

## 8. Diagnose failures before changing configuration

| Symptom | Check and next action |
| --- | --- |
| Helm DEPLOYED but workload unavailable | Inspect pods, events, webhook endpoints and resource conditions; wait for actual readiness. |
| GPU pod Pending | Inspect node selectors, taints/tolerations, pool instance requirements, quotas, NodeClaims and NVIDIA plugin. |
| Agent pod Pending | Compare request plus daemon/sidecar overhead with permitted CPU instance sizes. |
| Invalid API key from Claude | Verify the workflow references the intended ANTHROPIC_COMPATIBLE provider, its base URL/credential and the generated Job auth variables. Remove obsolete inline credentials and endpoint overrides. |
| OpenRouter 403 key limit | Fix provider budget; retries or chart changes do not fix it. |
| Context overflow | Check actual llama.cpp context and prompt token count; keep the initial model at 32K. |
| Timeout from zero | Trace LiteLLM → private Gateway → Activator → revision; inspect cold-start events and all timeout layers. |
| Model stays up | Stop probes/test inference traffic, inspect autoscaler/retention settings and in-flight requests. |
| GPU node remains | Inspect non-daemon workloads, disruption policy, deletion timestamp and provider termination; don't assume zero pods means zero billed nodes. |
| CLI token refresh fails briefly | Retry read-only commands; don't rewrite the user's CLI credentials or repeat a possibly queued mutation without checking state. |
| Helm OCI download fails with a macOS Keychain error | The POC used a local Helm 3 binary with private Docker/registry configuration and a temporary PATH excluding the credential helper. Do not rewrite the user's global Docker config. |

## 9. Existing installations, rollback and recovery

Qovery rejected changing a Helm service's chart source, including switching
from a direct upstream chart to this parent chart. Do not retry with arbitrary
source fields. A replacement Helm service requires a planned ownership transfer.

For a migration, privately back up original Qovery configs, Helm values,
manifests, resource UIDs and release ownership. Manifests and values can contain
secrets. Configure resource retention in the **stored Helm manifest**, verify
it, then empty the original release before adopting resources into a replacement.
Patching only live resource annotations is insufficient for safe uninstall.
Test UID and old ownership before replacing `meta.helm.sh/release-name`.
Never let two active releases write the same resources. Verify UIDs after
adoption before removing the old, empty Qovery service.

The tested migration preserved 41 KServe CRD/controller UIDs and the model UID.
LiteLLM's Deployment had to be recreated because its immutable selector contains
the release name. Its generated public URL changed with the Qovery service;
the gateway credential and model alias were retained. Plan for this rather
than promising URL preservation. This section describes the migration contract,
not an unattended migration script.

For a failed upgrade, preserve evidence and restore the last known working
profile/commit through the same Qovery service. Inspect actual release state
before another deploy; a cancelled request may have changed some resources.
For a stuck workflow, prefer the workflow cancel endpoint; `force_cancel=true`
abandons the job immediately and should be limited to the intended run.

Do not delete KnativeServing or KServe CRDs as rollback shortcuts: finalizers
and CRD deletion can remove shared resources. Returning an InferenceService
from Knative to Standard mode requires a separate retained resource; KServe
does not permit changing its deployment mode in place. Stop one model before
starting the other, switch LiteLLM's backend, and replay the tests. A fresh
installation has no previous Standard resource to restore. Do not destroy
shared controllers or unrelated routes during cleanup.

## Boundaries

This runbook documents a tested existing-cluster POC and supplies a fresh-install
procedure that still needs its own execution evidence. net-gateway-api 1.23 is
beta; its limitations are separate from Qwen model quality, emptyDir downloads,
single-replica availability and missing LiteLLM database/accounting. No Kimi,
Deepgram, arbitrary model compatibility or production readiness is implied.

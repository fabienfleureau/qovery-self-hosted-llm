# Qovery self-hosted LLM

A parent Helm chart referencing official upstream charts, with templates for
KnativeServing, the private Envoy Gateway and KServe InferenceServices.

Chart path: `charts/qovery-kserve-config`. Chart version: **0.2.0**.

| Component | Official Helm source | Pinned chart version | Enable flag |
| --- | --- | --- | --- |
| KServe CRDs | `oci://ghcr.io/kserve/charts/kserve-crd` | v0.16.0 | `components.kserveCRDs.enabled` |
| Knative Operator | `https://knative.github.io/operator` | v1.23.0 | `components.knativeOperator.enabled` |
| KServe controller | `oci://ghcr.io/kserve/charts/kserve` | v0.16.0 | `components.kserveController.enabled` |
| LiteLLM | `oci://ghcr.io/berriai/litellm-helm` | 0.1.837 | `components.litellm.enabled` |

Knative Operator installs Serving and net-gateway-api when configured through
a KnativeServing resource. No third-party generic chart is needed.

## Packaged Helm repository

Chart 0.2.0 is also available as a GitHub release with the official dependencies
bundled. Existing Qovery Helm services can switch to this repository without
changing from repository source to Git source:

```bash
helm repo add qovery-self-hosted-llm https://raw.githubusercontent.com/fabienfleureau/qovery-self-hosted-llm/main/helm/
helm show chart qovery-self-hosted-llm/qovery-kserve-config --version 0.2.0
```

For Qovery, register that HTTPS repository, select chart `qovery-kserve-config`
and version `0.2.0`. Dependency conditions and staged installation still apply.

## Download and render

```bash
helm dependency build charts/qovery-kserve-config
helm lint charts/qovery-kserve-config
helm template poc charts/qovery-kserve-config --namespace <environment-namespace>
```

All dependencies and configuration resources are disabled by default.
Enable the required components and pass upstream values under their aliases:
`kserveCRDs`, `knativeOperator`, `kserveController` and `litellm`.
For example, the KServe mode is configured at
`kserveController.kserve.controller.deploymentMode`.

```yaml
components:
  kserveCRDs:
    enabled: true
  knativeOperator:
    enabled: true
```

This example installs prerequisites only. A later release can use the same
parent chart with other dependencies or configuration templates enabled.

```yaml
components:
  litellm:
    enabled: true
litellm:
  fullnameOverride: poc-litellm
  environmentSecrets:
    - litellm-credentials
  proxy_config:
    model_list:
      - model_name: poc-qwen
        litellm_params:
          model: openai/poc-qwen
          api_base: http://<knative-service>.<namespace>.svc.cluster.local/v1
          api_key: unused
    general_settings:
      master_key: os.environ/LITELLM_MASTER_KEY
```

Create `litellm-credentials` with `LITELLM_MASTER_KEY` before deploying this
example. LiteLLM database, Redis and schema-migration jobs are disabled by
default in this parent chart; its other upstream defaults remain in effect.

## Deployment order

Use separate Helm releases or Qovery deployment stages:

1. KServe CRDs and Knative Operator; wait for the Operator webhook.
2. Private Gateway/EnvoyProxy; wait for the Gateway to be programmed.
3. KnativeServing configuration; wait for Ready and networking controllers.
4. KServe controller; wait for its webhook.
5. InferenceServices; verify the GPU scheduling and private route.
6. LiteLLM.

Helm dependency ordering does not wait for CRDs, controllers or webhooks to
become ready. Enabling everything in a fresh install is not a supported
bootstrap path. A KnativeServing resource requires the Operator CRD from a
previous stage; an InferenceService requires KServe prerequisites.

The `serving`, `gateway` and `models` configuration templates remain available
and accept the same values as chart 0.1.0. They are disabled by default.
Keep shared controllers in one release per cluster; do not enable their
dependencies again in every model release. Reuse existing Envoy Gateway and
cert-manager installations. The chart does not install either component.

## Qovery source

Use `https://github.com/fabienfleureau/qovery-self-hosted-llm`, branch `main`,
root path `/charts/qovery-kserve-config`. Qovery must resolve the chart's
pinned dependencies before rendering it.

This repository contains no customer credentials. The `examples/` profiles
use namespace placeholders rather than a deployed environment. Existing live Helm releases need an ownership migration;
changing this repository does not migrate the running POC.


## Qwen GPU example and Agent Tasks

After the prerequisites above are ready, deploy this chart with
`examples/qwen-knative-gpu.yaml` for the model release. It runs Qwen3 0.6B
Q8_0 through the pinned llama.cpp CUDA image, with a 32,768-token context.
The Claude Agent SDK sends its system prompt and tools with each request;
the previous 8K context rejected a 13,174-token request before generation.
This small model is for protocol validation, not a production agent benchmark.

The example targets a Karpenter `gpu` node pool, NVIDIA GPUs and amd64 nodes.
Adjust the node selector and tolerations to your cluster. The model is private
and scales from zero to one replica. Fresh nodes download the CUDA image and
model weights. The GPU node can be removed once it is empty.

For the gateway release, replace `MODEL_NAMESPACE` in
`examples/litellm-qwen.yaml` with the model release namespace. Create a Secret
named `litellm-credentials` in the gateway release namespace containing
`PROXY_MASTER_KEY`, then deploy this same chart with that profile. The profile
turns on the official LiteLLM dependency. It does not create an InferenceService.
Expose only LiteLLM port 4000 through your platform's HTTPS routing.

Configure the Agent Workflow using the Claude protocol:

```text
ANTHROPIC_BASE_URL=https://<public-litellm-host>
ANTHROPIC_AUTH_TOKEN=<LiteLLM master key; store as a secret>
ANTHROPIC_API_KEY=<empty value, without literal quote characters>
API_TIMEOUT_MS=900000
ANTHROPIC_DEFAULT_HAIKU_MODEL=poc-qwen
ANTHROPIC_DEFAULT_SONNET_MODEL=poc-qwen
ANTHROPIC_DEFAULT_OPUS_MODEL=poc-qwen
```

Use provider type `CLAUDE`, no Bedrock credential, and model settings
`{"model":"poc-qwen","effort":"low"}`. Bearer authentication avoids sending a
real Anthropic API key. The model alias is a routing name, not a model identity
assertion. Give the first request several minutes to create a GPU node and
load the model. Keep gateway and Knative timeouts at least as long.


### Qovery workflow URL override

The live Qovery test revealed that workflow-scoped `ANTHROPIC_BASE_URL` and
`ANTHROPIC_AUTH_TOKEN` were saved by the API but absent from the generated Job.
For this POC, set the gateway key in the workflow's write-only `model.api_key`
credential and add this non-secret workflow Docker fragment:

```dockerfile
ENV ANTHROPIC_BASE_URL=https://<public-litellm-host>
ENV CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

Keep provider `CLAUDE` and model `poc-qwen`. This uses LiteLLM's API-key auth
rather than bearer auth. Do not put the key in the Docker fragment. A real
Agent SDK run through the chart completed successfully with this workaround;
the eight HTTP smoke checks also passed. The 0.6B model added commentary
instead of obeying an exact-output instruction, so this validates transport
and SDK execution, not production agent quality.

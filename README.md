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

This repository contains no customer credentials or environment-specific
configuration. Existing live Helm releases need an ownership migration;
changing this repository does not migrate the running POC.

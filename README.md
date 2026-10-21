# Qovery self-hosted LLM configuration

Helm configuration for KServe, Knative Serving and a private Envoy Gateway.

Use the official KServe and LiteLLM charts and the official Knative Operator
chart for their controllers. This repository contains only the configuration
resources that connect them: KnativeServing, Gateway/EnvoyProxy and
InferenceService objects.

Chart path: `charts/qovery-kserve-config`.

All components are disabled by default. Enable `serving.enabled`,
`gateway.enabled` or `models.enabled` and supply the corresponding configuration.
Use a separate Helm release per component to preserve dependency ordering.

```bash
helm lint charts/qovery-kserve-config
helm template poc charts/qovery-kserve-config --namespace <environment-namespace>
```

For a Qovery Helm service, use this Git repository with branch `main` and
root path `/charts/qovery-kserve-config`. A private repository requires a
GitHub connection or Git credentials configured in Qovery.

This chart is experimental. Customer credentials and environment-specific
configuration are intentionally not included.

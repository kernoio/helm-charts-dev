# Kerno Agent Helm Chart (dev)

## Add the repository to your local helm:

```bash

helm repo add kerno-dev https://kernoio.github.io/helm-charts-dev

```

## Install the Chart

```bash

helm install kerno-agent kerno-dev/agent \
  --create-namespace \
  --set api-key="<KERNO_API_KEY>"
```

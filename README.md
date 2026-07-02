# flux-project

## Summary

This repository contains a working Flux demo for `minikube`:

- `clusters/minikube`: Flux entrypoint
- `clusters/staging` and `clusters/production`: promotion skeletons
- `infrastructure/sources`: shared Flux source objects
- `infrastructure/configs`: shared and environment-specific cluster config
- `infrastructure/minikube`: environment-specific infrastructure reconciled first
- `apps/base/nginx`: shared nginx app manifests
- `apps/minikube/nginx`: minikube-specific overlay reconciled after infrastructure

The demo deploys:

- `ingress-nginx` via `HelmRepository` and `HelmRelease`
- one `nginx` Deployment and Service in the `demo-nginx` namespace
- one Ingress for `nginx.minikube.local`

The existing Calico policy lab remains in the repository as a separate example, but it is not part of the active `minikube` reconciliation path.

## Relevant findings

- Verified from repository state: `fluxinstance.yaml` already points Flux at `clusters/minikube`.
- Verified from repository state: that path originally contained only a plain Kustomize file and no Flux `Kustomization` objects for staged reconciliation.
- Verified locally: the new manifests render successfully with `kubectl kustomize`.

## Proposed change

Use a layered structure that matches how larger Flux repositories are usually operated:

1. `clusters/minikube` declares Flux `Kustomization` resources.
2. `infrastructure/sources` stores shared source definitions such as `HelmRepository`.
3. `infrastructure/configs` stores shared cluster configuration such as namespaces.
4. `infrastructure/minikube` installs environment-specific controllers.
5. `apps/minikube` deploys environment-specific workloads.
6. `clusters/staging` and `clusters/production` are present as non-active expansion points.

This keeps ordering explicit and makes it straightforward to add `staging` and `production` later.

## Commands

Start minikube with Calico:

```bash
minikube start --cni calico
```

Apply or bootstrap Flux using your existing GitHub App configuration:

```bash
kubectl apply -f fluxinstance.yaml
kubectl get gitrepositories.source.toolkit.fluxcd.io -n flux-system
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -n flux-system
```

Validate infrastructure and app rollout:

```bash
kubectl get helmrepositories.source.toolkit.fluxcd.io -n flux-system
kubectl get helmreleases.helm.toolkit.fluxcd.io -n flux-system
kubectl get pods -n ingress-nginx
kubectl get all -n demo-nginx
kubectl get ingress -n demo-nginx
```

Test the app:

```bash
kubectl port-forward -n demo-nginx svc/nginx 8080:80
curl http://127.0.0.1:8080/
```

Optional ingress test:

```bash
minikube addons enable ingress
minikube ip
curl -H 'Host: nginx.minikube.local' http://$(minikube ip)/
```

Note: the repository deploys `ingress-nginx` by Flux. On some local minikube setups, using the built-in ingress addon may be simpler for first validation. Do not run both long-term without understanding which controller owns the `nginx` ingress class.

Current scope:

- active reconciliation target: `clusters/minikube`
- non-active skeletons: `clusters/staging`, `clusters/production`

## Validation steps

- `kubectl rollout status -n ingress-nginx deploy/ingress-nginx-controller`
- `kubectl rollout status -n demo-nginx deploy/nginx`
- `kubectl get svc -n demo-nginx nginx`
- `kubectl describe ingress -n demo-nginx nginx`
- `flux get all -A` if the Flux CLI is installed locally

## Rollback

Remove the Flux-managed resources:

```bash
kubectl delete -f clusters/minikube/apps.yaml
kubectl delete -f clusters/minikube/infrastructure.yaml
```

Or remove the references from `clusters/minikube/kustomization.yaml` and let Flux prune them.

## References

- Flux bootstrap for GitHub: https://fluxcd.io/flux/installation/bootstrap/github/
- minikube Network Policy handbook: https://minikube.sigs.k8s.io/docs/handbook/network_policy/
- ingress-nginx chart repository: https://kubernetes.github.io/ingress-nginx


# getting started with kustomize

deploy a small application in different environments (dev & prod)

## preparations
create the file structure
```
mkdir -p base overlays/dev overlays/prod
touch base/kustomization.yaml overlays/dev/kustomization.yaml overlays/prod/kustomization.yaml
```
## yaml
create some good old kubernetes yaml
```
kubectl run acndemo --image gcr.io/knative-samples/knative-route-demo:blue --expose --port 8080 --dry-run -o yaml > base/acndemo.yaml
```
## base
```
kustomize edit add resource acndemo.yaml
kustomize edit add label owner:acn
```
how does it render?
```
kustomize build .
```

## overlays (environments)
### dev
add base
```
cd overlays/dev/
kustomize edit add base ../../base
```

add namespace
```
kubectl create namespace dev --dry-run -o yaml > namespace.yaml
kustomize edit add resource namespace.yaml
kustomize edit set namespace dev
```

set some common labels
```
kustomize edit add label stage:dev
```
apply
```
kustomize build . | kubectl apply -f -
```

### prod
add base
```
cd overlays/prod/
kustomize edit add base ../../base
```
add namespace
```
kubectl create namespace prod --dry-run -o yaml > namespace.yaml
kustomize edit add resource namespace.yaml
kustomize edit set namespace prod
```
set some common labels and prefixes
```
kustomize edit add label stage:prod
```

### loadbalancer
generate an additional service loadbalancer
```
kubectl create service loadbalancer acndemo-lb --tcp 80:8080 --dry-run -o yaml | awk '/selector/{c=2} !(c&&c--)' > loadbalancer.yaml
```
the `awk` command removes `selector` and the following line. I haven't found a way to generate a service object with a different selector.

add the resource to kustomize
```
kustomize edit add resource loadbalancer.yaml
```

apply
```
kustomize build . | kubectl apply -f -
```

wait a bit and connect to the app 
```
kubectl -n prod get svc
```

### environment variable
add [env-green.yaml](overlays/prod/env-green.yaml), which consists of just the addition of an environmental variable.
```
kustomize edit add patch env-green.yaml
```
apply
```
kustomize build . | kubectl apply -f -
```

Now we should see a green version of the example app

## diff
show a diff between local environments
```bash
diff <(kustomize build overlays/dev) \
     <(kustomize build overlays/prod)
```

show diff between local and remote environment with [kubediff](https://github.com/weaveworks/kubediff)
```sh
kustomize build overlays/prod | sed '/creationTimestamp/d' > prod.yaml
kubediff --namespace=prod prod.yaml
```
This didn't work for me so well. If you want a better way to show diffs I suggest you check out [ksonnet](https://ksonnet.io/docs/cli-reference/#ks-diff).

## additions
We barely touched the surface for kustomize. It really shines when working with `configmaps` and `secrets`, so I suggest you continue there
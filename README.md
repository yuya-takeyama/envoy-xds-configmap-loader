# envoy-configmap-loader

The minimal and sufficient init/sidecar container to turn Kubernetes configmaps into a xDS server. No gRPC/REST server to maintain.

Distribute xDS data via Envoy's official local file config-source but via configmaps. 

## What's this?

`envoy-configmap-loader` is an init-container AND a sidecar for your Envoy proxy to use K8s ConfigMaps as xDS backend.

This works by loading kvs defined within specified configmap(s) and writing files assuming the key is the filename and the value is the content.

You then point your Envoy to read xDS from the directory `/srv/runtime/*.yaml`.

`envoy-configmap-loader` writes files read from configmap(s) into the directory, triggers [symlink swap](https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/runtime#updating-runtime-values-via-symbolic-link-swap)
 so that Envoy finally detects and applies changes. 
 
 ## Why not use configmap volumes?
 
You may [already know that K8s supports mounting configmaps as container volumes out of the box](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-volume).

The downside of using that feature to feed Envoy xDS files is that it takes 1 minute(default, configurable via kubelet `--sync-interval`) a change is reflected to the volume.

And more importantly, Envoy is unable to detect changes made in configmap volumes due to that it relies on `inotify` `MOVE` events to occur, where configmap volume changes only trigger the below events:

```
root@envoy-675dc8d98b-tvw9b:/# inotifywait -m /xds/rds.yaml
Setting up watches.
Watches established.
/xds/rds.yaml OPEN
/xds/rds.yaml ACCESS
/xds/rds.yaml CLOSE_NOWRITE,CLOSE
/xds/rds.yaml ATTRIB
/xds/rds.yaml DELETE_SELF
```

So in nutshell, `envoy-configmap-loader` is the minimal and sufficient companion to actually distribute xDS via configmaps, without using more advanced CRD-based solutions like Istio and VMWare Contour.

## Developing

Bring your own K8s cluster, move to the project root, and run the following commands to give it a ride:

```
sudo mkdir /srv/runtime
sudo chmod -R 777 /srv/runtime
k get secret -o json $(k get secret | grep default-token | awk '{print $1 }') | jq -r .data.token | base64 -D > mytoken
export APISERVER=$(k config view --minify -o json | jq -r .clusters[0].cluster.server)
make build && ./envoy-configmap-loader --namespace default --token-file ./mytoken --configmap incendiary-shark-envoy-xds --onetime --insecure
```

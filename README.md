# Home server

## Setup

* Install FluxCD using [GitHub bootstrapping](https://fluxcd.io/flux/installation/bootstrap/github/):
  ```sh
  flux bootstrap github \
    --owner=moredatapls \
    --repository=homeserver \
    --branch=main \
    --personal \
    --path=clusters/prod
  ```
* Complete the prerequisites of the [Tailscale K8s installation](https://tailscale.com/kb/1236/kubernetes-operator#prerequisites)
* Create the secret in the cluster for Tailscale with the client ID and the client secret of the OAuth app:
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: operator-oauth
    namespace: tailscale
  type: Opaque
  stringData:
    client_id: ...
    client_secret: ...
  ```

## Resources

The setup is based on various guides.

* [Home Assistant in Kubernetes the simple way](https://blog.quadmeup.com/2025/04/07/how-to-run-home-assistant-in-kubernetes/)
* [Pi-Hole auf Kubernetes](https://www.trion.de/news/2025/01/26/pi-hole-kubernetes.html)

## TODOs

* Reduce intervals: `1h` from `1m`
* Complete the setup documentation

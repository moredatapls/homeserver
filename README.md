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
* Create a secret in the cluster for Pi-Hole:
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: pi-hole-admin
    namespace: pi-hole
  type: Opaque
  stringData:
    password: ...
  ```

## SOPS

1. Follow [this guide](https://fluxcd.io/flux/guides/mozilla-sops/)
  1. Use the following to generate the key:
  ```sh
  export KEY_NAME="prod.berlin.bruegner.com"
  export KEY_COMMENT="flux secrets"

  gpg --batch --full-generate-key <<EOF
  %no-protection
  Key-Type: 1
  Key-Length: 4096
  Subkey-Type: 1
  Subkey-Length: 4096
  Expire-Date: 0
  Name-Comment: ${KEY_COMMENT}
  Name-Real: ${KEY_NAME}
  EOF
  ```
  2. Use the following enable secrets decryption:
  ```sh
  flux create kustomization decryt-secrets \
  --source=flux-system \
  --path=./clusters/prod \
  --prune=true \
  --interval=10m \
  --decryption-provider=sops \
  --decryption-secret=sops-gpg
  ```

New secrets can be encrypted using the public key in the clusters directory.

## Notes

* I had to add the hostnames of my ingresses to the DNS rebind protection exclusion list in my FritzBox. By default, FritzBoxes block DNS requests that point to its own network

## Resources

The setup is based on various guides.

* [Home Assistant in Kubernetes the simple way](https://blog.quadmeup.com/2025/04/07/how-to-run-home-assistant-in-kubernetes/)
* [Pi-Hole auf Kubernetes](https://www.trion.de/news/2025/01/26/pi-hole-kubernetes.html)

## TODOs

* Reduce intervals: `1h` from `1m`
* Complete the setup documentation

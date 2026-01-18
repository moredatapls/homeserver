# Home server

## Setup

* Install FluxCD using [GitHub bootstrapping](https://fluxcd.io/flux/installation/bootstrap/github/):
  ```sh
  flux bootstrap github \
    --owner=moredatapls \
    --repository=homeserver \
    --branch=main \
    --personal \
    --path=clusters/berlin
  ```
* Install the SOPS secret (from the password safe): `kubectl apply -f sops-gpg.yaml`
* Complete the prerequisites of the [Tailscale K8s installation](https://tailscale.com/kb/1236/kubernetes-operator#prerequisites)

## SOPS

Follow [this guide](https://fluxcd.io/flux/guides/mozilla-sops/) to setup SOPS locally.

The following was used to generate the key:
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

When reinstalling the cluster, the SOPS secret can be recovered from the password safe and installed: `kubectl apply -f sops-gpg.yaml`.

New secrets can be encrypted using the public key in the clusters directory:

```sh
sops encrypt --in-place path/to/secret.yaml
```

The Kustomization that installs the secrets has to reference the SOPS decryption provider:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  retryInterval: 2m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/my-app
  prune: true
  wait: true
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

## Notes

* I had to add the hostnames of my ingresses to the DNS rebind protection exclusion list in my FritzBox. By default, FritzBoxes block DNS requests that point to its own network

## Resources

The setup is based on various guides.

* [Home Assistant in Kubernetes the simple way](https://blog.quadmeup.com/2025/04/07/how-to-run-home-assistant-in-kubernetes/)
* [Pi-Hole auf Kubernetes](https://www.trion.de/news/2025/01/26/pi-hole-kubernetes.html)

## TODOs

* Reduce intervals: `1h` from `1m`
* Complete the setup documentation

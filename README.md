# Home server

## Repo

The repo contains the following directories:

- `apps/`: contains all the user-facing apps that get deployed inside the cluster
- `clusters/berlin/`: contains the core Flux setup of my server at home
- `controllers/`: contains core system components, like the ingress, DNS management, and storage classes

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

### SOPS

The cluster setup uses SOPS to encrypt keys. This way, they can safely be pushed to the GitHub repo. Flux then decrypts the secrets during reconcilation inside the cluster.

Follow [this guide](https://fluxcd.io/flux/guides/mozilla-sops/) to setup SOPS. The following command was used to generate the GPG key:

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
  # ...
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

## Apps

### Ingress-NGINX, External-DNS, Cert-Manager

I am running Ingress-NGINX in NodePort mode because I don't want to manage a load balancer - at least for now. This is sufficient for now as I am running a single-node cluster.

I had to configure a few things in the ingress values to get this to work:

```yaml
controller:
  # we want to use the host network for our ingresses
  hostNetwork: true
  # run a single controller pod per node, NodePort services won't work otherwise
  kind: DaemonSet
  service:
    # bind ports on the node (allows us to use ports 80, 443 etc.) and we don't need to run a load balancer
    type: "NodePort"
    # register external IPs of the host network, this is required for external-dns to create correct DNS records
    externalIPs:
      # IP of the node where Kubernetes is running, this needs to be a static IP
      - "192.168.178.104"
```

Furthermore, to get DNS to work, I have to add the hostnames of all ingresses to the DNS rebind protection exclusion list in my FritzBox. By default, FritzBoxes block DNS requests that point to its own network.

### Pi-Hole

The Pi-Hole setup is mostly based on the [Pi-Hole auf Kubernetes](https://www.trion.de/news/2025/01/26/pi-hole-kubernetes.html) article.

To bind the port 53 on my Ubuntu server, I had to disable `systemd-resolved`. I followed [this StackOverflow](https://askubuntu.com/a/907249) answer to disable the service.

### Home Assistant

The Home Assistant setup is mostly based on the [Home Assistant in Kubernetes the simple way](https://blog.quadmeup.com/2025/04/07/how-to-run-home-assistant-in-kubernetes/) guide.

To integrate Matter, I had to spin up a separate deployment based on the [matter-js/python-matter-server](https://github.com/matter-js/python-matter-server/). This is needed because the container-based setup of Home Assistant [does not support Add-Ons](https://www.home-assistant.io/installation/#about-installation-types). The Matter container setup was based on [this documentation](https://github.com/matter-js/python-matter-server/blob/main/docs/docker.md) by the server maintainers.

### Paperless-ngx

The Paperless-ngx setup is mostly based on the [Docker Compose](https://github.com/paperless-ngx/paperless-ngx/blob/dev/docker/compose/docker-compose.sqlite.yml) file and the [official Docker Compose setup guide](https://docs.paperless-ngx.com/setup/#docker) of the project.

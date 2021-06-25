# flux2-gitops-apps

[![license](https://img.shields.io/github/license/Fairbanks-io/flux-gitops-apps.svg)](https://github.com/Fairbanks-io/flux-gitops-apps/blob/main/LICENSE)

Flux managed k8s cluster deployed via Terraform.

## Prerequisites

You will need a Kubernetes cluster version 1.16 or newer and kubectl version 1.18.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **infrastructure** dir contains common infra tools such as NGINX ingress controller
- **cluster** dir contains the Flux configuration
- **sources** dir contains Helm repository references

## Bootstrap

The clusters dir contains the Flux configuration:

```
./cluster/
── apps.yaml
── infrastructure.yaml
```

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Bootstrap Flux by setting the context and path to your production cluster:

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=cluster
```

The bootstrap command commits the manifests for the Flux components in `cluster/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being install on staging:

```console
$ watch flux get helmreleases --all-namespaces 
NAMESPACE	NAME   	REVISION	SUSPENDED	READY	MESSAGE                          
nginx    	nginx  	5.6.14  	False    	True 	release reconciliation succeeded	
podinfo  	podinfo	5.0.3   	False    	True 	release reconciliation succeeded	
redis    	redis  	11.3.4  	False    	True 	release reconciliation succeeded
```

Watch the production reconciliation:

```console
$ watch flux get kustomizations
NAME          	REVISION                                        READY
apps          	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
flux-system   	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
infrastructure	main/797cd90cc8e81feb30cfe471a5186b86daf2758d	True
```

## Testing

Any change to the Kubernetes manifests or to the repository structure should be validated in CI before
a pull requests is merged into the main branch and synced on the cluster.

This repository contains the following GitHub CI workflows:

* the [test](./.github/workflows/test.yaml) workflow validates the Kubernetes manifests and Kustomize overlays with kubeval
* the [e2e](./.github/workflows/e2e.yaml) workflow starts a Kubernetes cluster in CI and tests the staging setup by running Flux in Kubernetes Kind


## Sealed Secrets:
In the cluster SealedSecrets will be decrypted and your original secret will be deployed. You can think of a sealed secret as a deployment and the regular secret as a pod. As such, when you delete a sealedSecret the secret associated with it will also be deleted.

### Pre-reqs
Download Kubeseal (sealed-secrets CLI)
- `wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.10.0/kubeseal-linux-amd64 -O kubeseal`
- `sudo install -m 755 kubeseal /usr/local/bin/kubeseal`

Download the public key for the sealed-secrets instance that is running in the cluster where the secret will be deplyoed
- `curl https://raw.githubusercontent.com/fairbanks-io/flux-gitops-apps/main/tls.crt > tls.crt`

### Creating a sealed Secret

Save the yaml for secret you want to deploy. 
> Note: Do not actually create on the cluster, only dry-run.

- `kubectl create secret generic mongoSecret --from-literal=mongoUri=mongodb://user:pass@dbhost.tld:27017/db --dry-run=client -o yaml > mongoSecret.yaml`

Use Kubeseal to encrypt the yaml of the secret just created with the public key in use by sealed-secrets in the cluster.

- `kubeseal --cert tls.crt < mongoSecret.yaml --format yaml > mongoSecret-sealed.yaml`

Optional:
Check it by applying the Sealed-Secret to the cluster
- `kubectl apply -f mongSecret-sealed.yaml`

Verify the original secret was created:
- `kubectl get secret mongoSecret -o yaml`

Delete the sealed Secret and check the regular secret no longer exists
- `kubectl delete SealedSecret mongoSecret`
- `kubectl get secretMongoSecret`

You can now include the contents of mongoSecret-Sealed.yaml in ./apps/your-app/sealed-secret.yaml.

## How to guide idpbuilder

- Verify that the applications below are installed on the host machine:
  - podman or docker
  - git

- Install idpbuilder (>= 0.8) as [documented](https://cnoe.io/docs/reference-implementation/installations/idpbuilder/quick-start).
- git clone the forked project to use the branch created to deploy using idpbuilder
```bash
git clone -b idpbuilder https://github.com/ch007m/fork-konflux-ci.git
```
- Create a new kind konfig file locally
```bash
cat <<EOF > my-konflux-cfg.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  labels:
    ingress-ready: "true"
  extraMounts:
    - containerPath: /var/lib/kubelet/config.json
      hostPath: $HOME/.config/containers/auth.json
  extraPortMappings:
  ## From IDP config - Begin ##
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
  ## From IDP config - End ##
  - containerPort: 8888 #30010
    hostPort: 8888
    protocol: TCP
  - containerPort: 9443 #30011
    hostPort: 9443
    protocol: TCP

## From IDP config - Begin ##
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gitea.cnoe.localtest.me:8443"]
      endpoint = ["https://gitea.cnoe.localtest.me"]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."gitea.cnoe.localtest.me".tls]
      insecure_skip_verify = true
## From IDP config - End ##
EOF
```
- Change within the cfg file the `hostPath: /home/snowdrop/temp/auth.json` to point locally to your podman `auth.json` or docker `config.json` which includes
  your registry credentials. This is needed to avoid to get the `docker rate limit` issue !!
- Create a cluster and deploy the konflux packages
```bash
alias idp=idpbuilder
export KIND_EXPERIMENTAL_PROVIDER=podman
export DOCKER_HOST="unix:///var/run/docker.sock"

idp create \
      --color \
      --build-name my-konflux \
      --host <IP_VM> \
      --kind-config my-konflux-cfg.yaml \
      -p fork-konflux-ci/idp/dependencies \
      -p fork-konflux-ci/idp/konflux \
      -p fork-konflux-ci/idp/testing \      
      --recreate
```
**Note**: 
- The `<IP_VM>` should be expressed as `IP.nip.io` to allow to access the UI outside the VM.
- If you install konflux on yoour machine, then no need to pass the parameter `--host`. 

When all the pods are up and running, then access the ui using the url: `https://konflux.cnoe.localtest.me:8443/application-pipeleine` or `https://konflux.IP.nip.io:8443` if you passed the parameter `--host`

**Warning**: If some resources are not sync (such as Bundles CR needed by the build-service and registry), then open the argocd console `https://argocd.cnoe.localtest.me:8443/` and resync the resources. You can get the passwords using the command `idp get secrets`



##  How to guide

- Install kind and follow instructions here: https://github.com/konflux-ci/konflux-ci?tab=readme-ov-file#bootstrapping-the-cluster
```bash
$ [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x kind
mv kind /usr/local/bin
kind --version
chmod +x /usr/local/bin/kind 
```
- Clone the project:
```bash 
git clone https://github.com/konflux-ci/konflux-ci.git
cd konflux-ci
```
- Rename the file `kind-config.yaml` to `my-kind-config.yaml` and add a mlountPoint to share your registry creds file as documented [here](https://kind.sigs.k8s.io/docs/user/private-registries/#mount-a-config-file-to-each-node).
```yaml
  extraMounts:
  - containerPath: /var/lib/kubelet/config.json
    hostPath: /home/snowdrop/temp/auth.json
```
- Create the cluster
```bash 
kind create cluster --name konflux --config my-kind-config.yaml
```
- Deploy the dependencies, konflux
```bash 
./deploy-deps.sh
./deploy-konflux.sh
```

To avoid such a docker issue, patch the kind config file to mount your `creds-registry` file (e.g. docker config.json or podman auth.json)
as documented [here](https://kind.sigs.k8s.io/docs/user/private-registries/#mount-a-config-file-to-each-node).

- Install the `image-controller` able to push your images on quay.io (and probably another registry !). See instructions [here](https://github.com/konflux-ci/konflux-ci/blob/main/docs/quay.md#automatically-provision-quay-repositories-for-container-images) to create a new Quay Application, got a token, etc
```bash
./deploy-image-controller.sh $(pass quay/ch007m/konfluxci/OAuthToken) <QUAY_ORG>
```

- Deploy demo users
```bash 
./deploy-test-resources.sh
```

- Start a new channel in [smee](https://smee.io/), and take a note of the webhook proxy URL (e.g https://smee.io/Wdavq8x0Oc9Jnl9M).
- Create a GitHub app following [Pipelines-as-Code documentation](https://pipelinesascode.com/docs/install/github_apps/#manual-setup).
**Note**: For Homepage URL you can insert https://localhost:9443/ (it doesn't matter) and for Webhook URL insert the smee client's webhook proxy URL from previous steps.
**Note**: Per the instructions on the link, generate and download the private key and create a secret on the cluster providing the location of the private key, the App ID, and the openssl-generated secret created during the process.

- To allow Konflux to send PRs to your application repositories, the GithubApp secret should be created inside the `build-service` and the `integration-service` namespaces. See additional details under [Configuring GitHub Application Secrets](https://github.com/konflux-ci/konflux-ci/blob/main/docs/github-secrets.md).
- 
**Note**: Store the application, id, secret and key in `password` store or equivalent !

```bash
pass github/apps/konfluxci/
github/apps/konfluxci
├── app_id
├── client_id
├── client_secret
├── private_key
└── webhook_secret

pass github/apps/konfluxci/private_key > private_key.pem

kubectl -n pipelines-as-code delete secret pipelines-as-code-secret
kubectl -n pipelines-as-code create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=$(pass github/apps/konfluxci/app_id) \
  --from-literal=webhook.secret=$(pass github/apps/konfluxci/webhook_secret) \
  --from-file=github-private-key=private_key.pem)

kubectl -n build-service delete secret pipelines-as-code-secret
kubectl -n build-service create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=$(pass github/apps/konfluxci/app_id) \
  --from-literal=webhook.secret=$(pass github/apps/konfluxci/webhook_secret) \
  --from-file=github-private-key=private_key.pem)

kubectl -n integration-service delete secret pipelines-as-code-secret
kubectl -n integration-service create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=$(pass github/apps/konfluxci/app_id) \
  --from-literal=webhook.secret=$(pass github/apps/konfluxci/webhook_secret) \
  --from-file=github-private-key=private_key.pem)
```

- Patch the `smee.yaml` config, replace `` with your smee proxy url and deploy smee
```bash
kubectl apply -f ../konflux-ci/smee/smee-client.yaml
kubectl patch deployment \
  gosmee-client \
  --namespace smee-client \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "client",
  "https://smee.io/Wdavq8x0Oc9Jnl9M",
  "http://pipelines-as-code-controller.pipelines-as-code:8080"
]}]'
```

- (optional) Create a kubernetes secret for the konflux user to push the images to quay or download images from `dockerhub`
```bash 
podman login quay.io -u <QUAY_USERNAME> -p <QUAY_PASSWORD>
podman login https://index.docker.io -u <QUAY_USERNAME>-p <QUAY_PASSWORD>
cat /run/user/1000/containers/auth.json | jq -c . > creds-registry.json
kubectl -n user-ns1 delete secret creds-registry
kubectl -n user-ns1 create secret generic creds-registry --from-file=.dockerconfigjson=creds-registry.json
```
- If Konflux was installed on a cluster hosted in a remote machine, SSH port-forwarding can be used to access. Open an additional terminal and run the following command (make sure to add the details of your remote machine and user):
```bash
ssh -i $HOME/.ssh/id_rsa -L 9443:localhost:9443 $USER@$VM_IP

Example:
ssh -i $HOME/.ssh/id_rsa_snowdrop_openstack -L 9443:localhost:9443 snowdrop@10.0.77.131
```

The UI will be available at https://localhost:9443. You can login using a test user: `user1` or `user2`.

```text
username: user1
password: password
```

## How to play

Two namespaces have been created to play with Konflux: `user-ns1` and `user-ns2`.
...

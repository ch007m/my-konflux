## How to guide using idpbuilder

- Verify that the applications below are installed on the host machine:
  - podman (>= 5.2) or docker
  - git

- Install idpbuilder (>= 0.8.1) as [documented](https://cnoe.io/docs/reference-implementation/installations/idpbuilder/quick-start).
- Git clone the forked project to use the branch created to deploy using idpbuilder
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
  image: "kindest/node:v1.30.3"
  labels:
    ingress-ready: "true"
  extraMounts:
    - containerPath: /var/lib/kubelet/config.json
      hostPath: $HOME/.config/containers/auth.json
  extraPortMappings:
  - containerPort: 443
    hostPort: 8443
    protocol: TCP

containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gitea.cnoe.localtest.me:8443"]
      endpoint = ["https://gitea.cnoe.localtest.me"]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."gitea.cnoe.localtest.me".tls]
      insecure_skip_verify = true
EOF
```
- Change within the cfg file the `hostPath: $HOME/.config/containers/auth.json` to point locally to your podman `auth.json` or docker `config.json` which includes
  your registry credentials. This is needed to avoid to get the `docker rate limit` issue !

- Create a cluster and deploy the modules which are needed part of the `foundation/basement`: Cert & Trust manager, Self-signed `root` certificate, local registry 
```bash
alias idp=idpbuilder
export KIND_EXPERIMENTAL_PROVIDER=podman

idp create \
  --color \
  --name my-konflux \
  --kind-config my-konflux-cfg.yaml \
  -c argocd:fork-konflux-ci/idp/argocd/argocd-cm.yaml
```

**Note**: If you plan to install the project on a remote machine, you can pass to `idpbuilder` the following parameter `--host <IP_VM>` where `<IP_VM>` should be expressed as `<IP>.nip.io`  or `<host.domain>` to allow to access the UI outside the VM.

- Next, install the modules needed by Konflux: Cert & Trust Manager, keycloak, Tekton & Tekton Pipeline As Code
```bash
idp create \
  --color \
  --name my-konflux \
  --kind-config my-konflux-cfg.yaml \
  -p fork-konflux-ci/idp/foundations \
  -p fork-konflux-ci/idp/dependencies
```

- When the Argocd Applications are healthy `kubectl get applications -A`, then deploy the modules konflux & Testing: user namespace, etc
```bash
idp create \
  --color \
  --name my-konflux \
  --kind-config my-konflux-cfg.yaml \
  -p fork-konflux-ci/idp/foundations \
  -p fork-konflux-ci/idp/dependencies \
  -p fork-konflux-ci/idp/konflux \
  -p fork-konflux-ci/idp/testing
```

- When all the pods are up and running, then access the ui using the url: `https://konflux.cnoe.localtest.me:8443/application-pipeline` or `https://konflux.IP.nip.io:8443` if you passed the parameter `--host`

- If you would like to use: smee, the image-controller and your GitHub application (as documented [here](https://github.com/konflux-ci/konflux-ci/tree/main?tab=readme-ov-file#enable-pipelines-triggering-via-webhooks)) to allow Tekton PaC to talk with your GitHub repositories, then create the following files:
  - File containing as k=v pairs the following parameters
    ```text
    // path: <YOUR_PATH>/secret_vars.yaml
    smee_url: https://smee.io/<SMEE_TOKEN>
  
    github_app_id: <GITHUB_AP_ID>
    github_webhook_secret: <GITHUB_WEBHOOK_SECRET>
    
    github_private_key: <GITHUB_PRIVATE_KEY> 
    
    quay_org: <QUAY_ORG>
    quay_token: <QUAY_TOKEN>
    ``` 
    **Warning**: The `GITHUB_PRIVATE_KEY` is the GitHub application private key file converted as one line string here and where the `\n` has been replaced with `#`. TODO: To be improved !
  - A kubernetes secret resource file `secrets.yaml` using the following command:
    ```bash
    kubectl create secret generic argocd-secret-vars -n argocd --from-file=secret_vars.yaml=<YOUR_PATH>/secret_vars.yaml \
       --dry-run=client -o yaml >> fork-konflux-ci/idp/secret-plugin/manifests/secrets.yaml
    ```

- Deploy then the additional packages able to install/configure: smee, image-controller, etc
```bash
idp create \
  --color \
  --name my-konflux \
  --kind-config my-konflux-cfg.yaml \
  -p fork-konflux-ci/idp/foundations \
  -p fork-konflux-ci/idp/dependencies \
  -p fork-konflux-ci/idp/konflux \
  -p fork-konflux-ci/idp/testing \
  -p fork-konflux-ci/idp/secret-plugin \
  -p fork-konflux-ci/idp/github-app-secrets \
  -p fork-konflux-ci/idp/smee \
  -p fork-konflux-ci/idp/image-controller
```
- When done, go back to the console: `https://konflux.cnoe.localtest.me:8443/application-pipeline` and enjoy :-)

##  How to guide

- Install kind and follow instructions here: https://github.com/konflux-ci/konflux-ci?tab=readme-ov-file#bootstrapping-the-cluster
```bash
$ [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-linux-amd64
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
- Rename the file `kind-config.yaml` to `my-kind-config.yaml` and add a mountPoint to share your registry creds file as documented [here](https://kind.sigs.k8s.io/docs/user/private-registries/#mount-a-config-file-to-each-node).
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
  --from-file=github-private-key=private_key.pem

kubectl -n build-service delete secret pipelines-as-code-secret
kubectl -n build-service create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=$(pass github/apps/konfluxci/app_id) \
  --from-literal=webhook.secret=$(pass github/apps/konfluxci/webhook_secret) \
  --from-file=github-private-key=private_key.pem

kubectl -n integration-service delete secret pipelines-as-code-secret
kubectl -n integration-service create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=$(pass github/apps/konfluxci/app_id) \
  --from-literal=webhook.secret=$(pass github/apps/konfluxci/webhook_secret) \
  --from-file=github-private-key=private_key.pem
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

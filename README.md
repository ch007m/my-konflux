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
- Create the cluster
```bash 
kind create cluster --name konflux --config kind-config.yaml
```
- Deploy the dependencies, konflux
```bash 
./deploy-deps.sh
./deploy-konflux.sh
```
**Note**: If you get the famous docker limit rate, then pull the image locally and push it to the kind cluster created !
```bash
docker login
docker pull postgres:15
kind load docker-image -n konflux postgres:15
docker pull registry:2
kind load docker-image -n konflux registry:2
```

- Deploy demo users
```bash 
./deploy-test-resources.sh
```

- Start a new channel in [smee](https://smee.io/), and take a note of the webhook proxy URL (e.g https://smee.io/Wdavq8x0Oc9Jnl9M).
- Create a GitHub app following [Pipelines-as-Code documentation](https://pipelinesascode.com/docs/install/github_apps/#manual-setup).
**Note**: For Homepage URL you can insert https://localhost:9443/ (it doesn't matter) and for Webhook URL insert the smee client's webhook proxy URL from previous steps.
**Note**: Per the instructions on the link, generate and download the private key and create a secret on the cluster providing the location of the private key, the App ID, and the openssl-generated secret created during the process.

- The GithubApp private key should be downloaded and saved to the file `githubapp-konfluxci.private-key.pem` under the project
```bash
git clone git@github.com:ch007m/my-konflux.git
cd my-konflux
touch githubapp-konfluxci.private-key.pem
echo "Save the pem key to the file !!"
```

- To allow Konflux to send PRs to your application repositories, the GithubApp secret should be created inside the `build-service` and the `integration-service` namespaces. See additional details under [Configuring GitHub Application Secrets](https://github.com/konflux-ci/konflux-ci/blob/main/docs/github-secrets.md).
```bash
kubectl -n pipelines-as-code create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=OTQ3MjI4 \
  --from-literal=webhook.secret=a29uZmx1eGNp \
  --from-file=github-private-key=githubapp-konfluxci.private-key.pem

kubectl -n build-service create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=OTQ3MjI4 \
  --from-literal=webhook.secret=a29uZmx1eGNp \
  --from-file=github-private-key=githubapp-konfluxci.private-key.pem

kubectl -n integration-service create secret generic pipelines-as-code-secret \
  --from-literal=github-application-id=OTQ3MjI4 \
  --from-literal=webhook.secret=a29uZmx1eGNp \
  --from-file=github-private-key=githubapp-konfluxci.private-key.pem
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
kubectl -n user-ns1 create secret generic creds-registry --from-file=.dockerconfigjson=/run/user/1000/containers/auth.json
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

kubectl get ns smee-client -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/smee-client/finalize" -f -

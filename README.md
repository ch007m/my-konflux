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
podman update --pids-limit 4096 konflux-control-plane
```
- Deploy the dependencies, konflux
```bash 
./deploy-deps.sh
./deploy-konflux.sh
```

- Deploy demo users
```bash 
./deploy-test-resources.sh
```

- Start a new channel in [smee](https://smee.io/), and take a note of the webhook proxy URL (e.g https://smee.io/Wdavq8x0Oc9Jnl9M).
- Create a GitHub app following [Pipelines-as-Code documentation](https://pipelinesascode.com/docs/install/github_apps/#manual-setup).
**Note**: For Homepage URL you can insert https://localhost:9443/ (it doesn't matter) and for Webhook URL insert the smee client's webhook proxy URL from previous steps.
**Note**: Per the instructions on the link, generate and download the private key and create a secret on the cluster providing the location of the private key, the App ID, and the openssl-generated secret created during the process.
```

- The generated key should be downloaded and saved under a file
```bash 
cat <<EOF > githubapp-konfluxci.private-key.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEAz5wEbPeh/3dAORFmistuarwxuJVn0k2WUBecz61l2gqFNT4m
iVbGvryp8MruuNyi/N5ST7tbLXJs4FuSBlXMJySIsTpGhMqW/oybyWSGe/n7U90T
g5FiUYYCupAd8XDRQJ2z5s5eOFKigFlMkyvntzRRhLyLF76F8w/ICGAPvPX6zF46
+5fCcqRcnybNg50OHx0gMophB4RGe7X9B9Ra4aOs4ufdloM5Fui6mL2PtmSuPFox
08Rje8HdruG80ky9Zyuh/8AKHTcEAgxfmvLqwq2tSUlCo9Pg192EYpVvjV0yT7hp
Wb5yRn/fY+ZxMB8Em+v4Q5kTnnhRMB/nrz3T3QIDAQABAoIBAQC/YKXfwSKveUFV
mgm5ti+x6ou3CNrszSLb7/hYpRI3vezvmLwzbC/HUekiFB+df54rldGwuBgju9BL
vX3NozePeakcHok1Df/E5N8S9jzWeilJNIkxhkpoti07x3AiygnLE9Tr1Z6bMumj
gS4KmLWAd5UR1DAwEiwuVITj25GGcFqKpTKbeQgB6nj6+abERLh3QT2sly6YsPbO
G6oe4JOr6EI+7D3/UsbNUv3qFRDX5hO7vDHtJhdJehyW//FzHF3fdroWOSg+bWAK
Q1PTqYmO3RUPC9vW4MVFDQQlYF5WNzg00muh9yZi4+IqwmWBuoamKLp5pCjAlkJZ
KJGAiL2hAoGBAP/YHYrs1N5IOXlgZoZ5cUQ463ADZpMHIrncYatNe7j3jMhI7NsL
DvhuBmsy2xe3ghaWphGMaWwC6749QYsWbeE3FqkzwVFbc61iPECSZ1hcOmAnFRyN
4IWydpP9j7UEO6a3HdxQ7B3IYap1GiPKfb++iSmqYR8F64z6lZ4RUOxVAoGBAM+8
YeMwT6RZZHShvZikPpMarT/u+wJuvowubpP7r3x1BTWRDM7KLslnQj3rAXh4a9Ec
4AYV1fq7hBjnqLWxjqgz8aiQc+/6POBXG9C2NG9tKgObtLKw+QEGE8o+JTf+KPAd
FzdYq3H9XVy5dDu3tc1k4wkINFbVvC0EhWp3LlFpAoGBAJdB1CWAY1GPbbtezP6P
6fElnbw5pnkibNtpeaznQFBYurjmtHHEFfO2SMEz7egVrClio4gYdXNQPsPYP1nV
xtyxzwn1+UL6SGenfmvGoqbQ0Aps0MRy9NzWZ9iSvlWMzR+Bf3vzs8Tf5S370Zp7
auDj6v/hJU5MF7jfpXkwT6GJAoGADODx9KLHHTpJhw2L8o2kL3yE5yTKvQDeoVQz
mMsOuxmKJCME90EDm4riSXJrWeulS4aNwPLTnELJ0r1x8Sm73WOzBK9H8MXDxmjA
GbViFNJgu26Iylc8aLrWuUAXEJyaLyCuksjVgDCj/B6nPRiLlds+VA4FKKkBjIzu
NIaFAZkCgYEAgT5iJ1H1bdrpIWrlZxMOeB/N7g/vgHjszeYJXdfGnPLIy+/NIbXU
N19HGJfF5zhra5NkRJtgqf8NFhBvYQ4DnM8vu/hRSbTJEkLG4vDQzDyxmU6sEoow
lfmt/yn/tXHrBihYCrUI/D4JB5R2VVXo2feegyOenpwiUHQNgfFEk8c=
-----END RSA PRIVATE KEY-----
EOF
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
- TODO
```bash 

```
- If Konflux was installed on a cluster hosted in a remote machine, SSH port-forwarding can be used to access. Open an additional terminal and run the following command (make sure to add the details of your remote machine and user):
```bash
ssh -L 9443:localhost:9443 $USER@$VM_IP
```


The UI will be available at https://localhost:9443. You can login using a test user: `user1` or `user2`.

```text
username: user2
password: password
```
FROM fedora:40

RUN sudo dnf install -y openssl

RUN curl -sLO https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl
RUN chmod +x kubectl && mv ./kubectl /usr/local/bin/kubectl

RUN kubectl version --client

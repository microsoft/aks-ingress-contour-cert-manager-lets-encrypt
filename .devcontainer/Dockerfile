# this image is built and updated weekly
# https://github.com/cse-labs/codespaces-images

FROM ghcr.io/cse-labs/k3d:latest

# some images require specific values
ARG USERNAME=vscode
ENV PATH=$PATH:/home/vscode/bin:/opt/mssql-tools/bin

# update welcome message; chown home dir
RUN echo "$(date +'%Y-%m-%d %H:%M:%S')    docker build start" >> "/home/${USERNAME}/status" && \
    mkdir -p "/home/${USERNAME}/bin" && \
    mkdir -p "/home/${USERNAME}/.ssh" && \
    mkdir -p "/home/${USERNAME}/go/bin" && \
    mkdir -p "/home/${USERNAME}/.oh-my-zsh/completions" && \
    mkdir -p "/home/${USERNAME}/.config/vscode-dev-containers" && \
    echo "export GOPATH=/home/${USERNAME}/go" >> /etc/zsh/zshrc && \
    wget -q -O - https://raw.githubusercontent.com/rancher/k3d/main/install.sh | TAG=v5.4.6 bash && \
    curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list > /etc/apt/sources.list.d/msprod.list && \
    apt-get update && \
    ACCEPT_EULA=y apt-get install -y mssql-tools unixodbc-dev && \
    mkdir -p /usr/local/bin && \
    cd /usr/local/bin && \
    curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash && \
    cd $OLD_PWD && \
    mkdir -p /usr/local/etc/vscode-dev-containers && \
    echo "👋 Welcome to AKS-Demo Labs!" > /usr/local/etc/vscode-dev-containers/first-run-notice.txt && \
    echo "" >> /usr/local/etc/vscode-dev-containers/first-run-notice.txt && \
    echo "🔍 To explore, open the README.md file" >> /usr/local/etc/vscode-dev-containers/first-run-notice.txt && \
    echo "" >> /usr/local/etc/vscode-dev-containers/first-run-notice.txt && \
    echo "cat /usr/local/etc/vscode-dev-containers/first-run-notice.txt" >> /etc/zsh/zshrc && \
    touch "/home/${USERNAME}/.config/vscode-dev-containers/first-run-notice-already-displayed" && \
    echo "$(date +'%Y-%m-%d %H:%M:%S')    docker build complete" >> "/home/${USERNAME}/status" && \
    chown -R ${USERNAME}:${USERNAME} "/home/${USERNAME}"

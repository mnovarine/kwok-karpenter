## Objetivo

Instalar a ferramenta Kwok para simulação de criação de nodes com o Karpenter.

Utilizado como referência o repo do [dacord](https://github.com/dacort/kinda-yunikarp?tab=readme-ov-file#manual-setup-1).

- [KWOK](https://kwok.sigs.k8s.io/) - O KWOK é um conjunto de ferramentas que permite configurar um cluster com milhares de nós em segundos. Nos bastidores, todos os nós são simulados para se comportarem como nós reais, portanto, a abordagem geral utiliza uma pegada de recursos bastante baixa, permitindo que você experimente facilmente em seu laptop.

- [Karpenter](https://karpenter.sh/docs/) - Karpenter é um projeto de gerenciamento do ciclo de vida de nós de código aberto, desenvolvido para Kubernetes. Adicionar o Karpenter a um cluster Kubernetes pode melhorar drasticamente a eficiência e reduzir o custo de execução de cargas de trabalho nesse cluster.

- Configuração

    O teste foi realizado em ambiente WSL com Ubuntu, versão "24.04.4 LTS".

## Problemas ao criar o cluster com o Kind

1. Para criar um cluster com o Kind, foi necessário habilitar o **cgroups v2** no WSL do Ubuntu.

    - Criar o arquivo C:\Users\**[USER]**\.wslconfig
    - Colocar as linhas

        ```ini
        [wsl2]
        kernelCommandLine = cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1
        ```

    - Reiniciar o WSL

        ```bat
        wsl --shutdown
        ```
    
    - Verificar se foi habilitado corretamente

        ```bash
        # executar o comando
        stat -fc %T /sys/fs/cgroup/

        # resultado deve ser
        cgroup2fs
        ```

## Problemas encontrados na instalação do Kwok/Karpenter

Ao executar o comando `make apply-with-kind` no item [3. Instalação do Karpenter](?#3-Instalação-do-Karpenter), foram encontrados vários erros, resolvidos com a instalação dos itens abaixo.<br>
Esses itens também não estavam descritos no repo do [dacort](https://github.com/dacort/kinda-yunikarp), que utilizei como referência.

1. Necessário adicionar o conteúdo da variável **GOPATH** na variável **PATH**

    ```bash
    export PATH="${KREW_ROOT:-$HOME/.krew}/bin:/home/novarin/go/bin:$PATH"
    ```

    [Referência](https://github.com/kubernetes-sigs/karpenter/issues/1444#issuecomment-2243676644```)

2. Necessário instalar binário **yq**, versão Golang

    ```bash
    VERSION=v4.52.5
    PLATFORM=linux_amd64

    # Download compressed binary
    wget https://github.com/mikefarah/yq/releases/download/${VERSION}/yq_${PLATFORM}.tar.gz -O - |\
    tar xz && sudo mv yq_${PLATFORM} /usr/local/bin/yq
    ```

    [Referência](https://github.com/mikefarah/yq?tab=readme-ov-file#wget)

3. Necessário instalar os binários em Go

    ```bash
    go install sigs.k8s.io/controller-tools/cmd/controller-gen@latest
    go install github.com/norwoodj/helm-docs/cmd/helm-docs@latest
    go install github.com/rhysd/actionlint/cmd/actionlint@latest
    go install github.com/google/ko@latest
    ```

    [helm-docs](https://github.com/norwoodj/helm-docs)
    [actionlint](https://github.com/rhysd/actionlint)

4. Necessário instalar **golangci-lint**

    ```bash
    curl -sSfL https://golangci-lint.run/install.sh | sh -s v2.11.4

    sudo mv ./bin/golangci-lint /usr/local/bin/
    ```

    [Referência](https://golangci-lint.run/docs/welcome/install/local/)

5. Setar as variáveis de ambiente e fazer o build

    ```bash
    # Essas variáveis serão utilizadas na instalação também
    export KWOK_REPO=kind.local
    export KIND_CLUSTER_NAME=kind-karpenter # nome do cluster

    # Necessário fazer o build da imagem
    make build-with-kind
    ```

## Instalação

1. Criar o cluster com o Kind

    ```bash
    ./kind.sh
    ```
2. Instalação do KWOK

    ```bash
    helm repo add kwok https://kwok.sigs.k8s.io/charts/
    helm repo update

    helm upgrade --install kwok kwok/kwok \
    --namespace kube-system \
    -f kwok-helm-karpenter-config.yaml

    helm upgrade --install kwok-stage kwok/stage-fast \
    --namespace kube-system
    ```

3. Instalação do Karpenter

    ```bash
    # Clone Karpenter repo
    git clone https://github.com/kubernetes-sigs/karpenter.git
    cd karpenter
    git switch --detach v1.8.0

    # Install Prometheus (required)
    ./hack/install-prometheus.sh

    # Build and install Karpenter
    export KWOK_REPO=kind.local
    export KIND_CLUSTER_NAME=kind-karpenter # nome do cluster
    make apply-with-kind

    cd ..
    ```

## Dashboards

### Habilitar interface Prometheus

```bash
kubectl --namespace monitoring port-forward $POD_NAME 3000
```
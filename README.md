Excelente. Para que esse teste funcione exatamente como no seu exemplo (o status `2/2`), precisamos garantir um passo crucial antes de rodar o deploy: **ativar a injeção automática de sidecar** no namespace. Sem isso, o Pod subiria apenas com `1/1` (só o Nginx).

Adicionei o **Passo 5** ao README, que cobre a criação do arquivo, a ativação do Istio no namespace e a validação final.

Aqui está o arquivo completo atualizado:

---

## 📄 README.md

```markdown
# Setup de Ambiente Kubernetes Local (K3D + Istio)

Este guia descreve o passo a passo para configurar um ambiente Kubernetes local utilizando **K3D** no Linux (foco em Ubuntu), incluindo a instalação do **Kubectl** e a configuração da malha de serviço (Service Mesh) **Istio**.

## 📋 Pré-requisitos

Antes de iniciar, certifique-se de ter o **Docker** instalado e rodando em sua máquina, pois o K3D utiliza containers Docker para simular os nós do Kubernetes.

---

## 🚀 1. Instalação das Ferramentas Básicas

### Instalar o K3D
O K3D executa o K3s (distribuição leve do Kubernetes) dentro do Docker.

```bash
curl -s [https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh](https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh) | bash

```

Valide a instalação: `k3d --version`

### Instalar o Kubectl

> **🐧 Recomendação para Ubuntu 24:**
> A maneira mais fácil e recomendada de instalar o Kubectl no Ubuntu é via **Snap**. Isso garante atualizações automáticas e segurança.

**Opção A: Via Snap (Recomendada)**

```bash
sudo snap install kubectl --classic

```

**Opção B: Via Binário (Genérico Linux)**

```bash
curl -LO "[https://dl.k8s.io/release/$(curl](https://dl.k8s.io/release/$(curl) -L -s [https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl](https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl)"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

```

Valide a instalação: `kubectl version --client`

---

## ⚙️ 2. Criação do Cluster K3D

Cria um cluster com 2 agentes e mapeamento de portas para o LoadBalancer.

```bash
k3d cluster create -p "8000:30000@loadbalancer" --agents 2

```

Valide se os nós estão `Ready`:

```bash
kubectl get nodes

```

---

## ⛵ 3. Instalação e Configuração da CLI do Istio

### 1. Baixar e Configurar

```bash
# Baixar
curl -L [https://istio.io/downloadIstio](https://istio.io/downloadIstio) | sh -

# Mover para /opt (Ajuste a versão conforme o download, ex: istio-1.28.2)
sudo mv istio-1.28.2 /opt/

```

### 2. Adicionar ao PATH

Edite seu arquivo de perfil:

```bash
nano ~/.bashrc

```

Adicione ao final:

```bash
export PATH=$PATH:/opt/istio-1.28.2/bin

```

Salve e saia (**Ctrl+O**, **Enter**, **Ctrl+X**).

Aplique:

```bash
source ~/.bashrc

```

---

## 🕸️ 4. Instalar o Istio no Cluster

Instala os componentes do Istio dentro do Kubernetes.

```bash
istioctl install -y

```

---

## 🧪 5. Teste Final: Validando a Injeção do Sidecar

Agora vamos subir uma aplicação Nginx para garantir que o Kubernetes está rodando e que o Istio está injetando o proxy automaticamente.

### 1. Ativar injeção automática de Sidecar

Para que o Istio adicione o proxy aos seus Pods, precisamos "etiquetar" o namespace `default`:

```bash
kubectl label namespace default istio-injection=enabled

```

### 2. Criar o arquivo deployment.yaml

Crie um arquivo chamado `deployment.yaml` com o seguinte conteúdo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80

```

### 3. Aplicar e Validar

Execute a sequência de comandos abaixo:

```bash
# Aplicar o deploy
kubectl apply -f ./deployment.yaml

# Aguarde alguns segundos e verifique os pods
kubectl get pods

```

**O que observar:**
Inicialmente, você pode ver o status `PodInitializing`.
Rode o comando novamente até ver o status `Running`.

**Resultado Esperado:**

```text
NAME                     READY   STATUS    RESTARTS   AGE
nginx-77565f76bc-r8xvg   2/2     Running   0          25s

```

> **Importante:** Observe a coluna **READY 2/2**.
> * **1/2:** Seria apenas o container do Nginx.
> * **2/2:** Significa que temos o container do **Nginx** + o container do **Istio Proxy (Envoy)** rodando juntos no mesmo Pod. Isso confirma que a instalação foi um sucesso!
> 
> 

### 4. Inspecionar Detalhes (Opcional)

Para ver os detalhes técnicos e confirmar a existência do container `istio-proxy`:

```bash
# Substitua pelo nome exato do seu pod listado acima
kubectl describe pod nginx-77565f76bc-r8xvg

```
---

## 📊 5. Observabilidade (Prometheus, Grafana e Kiali)

Agora vamos instalar os addons oficiais do Istio para monitoramento e visualização.

### 1. Instalar Prometheus

```bash
kubectl apply -f ./prometheus.yaml

```

### 2. Instalar Grafana

```bash
kubectl apply -f ./grafana.yaml

```

### 3. Instalar Kiali

```bash
kubectl apply -f ./kiali.yaml

```

### 4. Validar a Instalação

Verifique se todos os componentes estão rodando no namespace `istio-system`:

```bash
kubectl get pods -n istio-system

```

**Resultado esperado:** Você deve ver pods como `prometheus-...`, `grafana-...` e `kiali-...` com o status **Running**.

### 5. Acessar o Dashboard do Kiali

O Kiali permite visualizar o mapa de tráfego dos seus serviços. Para abrir o dashboard automaticamente no seu navegador, execute:

```bash
istioctl dashboard kiali

```
*O terminal exibirá uma URL como `http://localhost:20001/kiali`.*

---

## 🧹 6. Limpeza do Ambiente (Opcional)

Se você terminou seus testes e deseja remover o cluster para liberar recursos da sua máquina, utilize o comando abaixo:

```bash
# Remove o cluster criado e todos os seus recursos (nodes, pods, serviços e configurações)
k3d cluster delete

```

Para deletar somente o que foi gerado por um arquivo específico, você pode usar:

```bash
kubectl delete -f <nome-do-arquivo>.yaml
```

Para validar que o cluster foi removido:

```bash
k3d cluster list

```

---

### 💡 Dicas Adicionais para o seu README:

1. **Versão do Istio:** No passo 3, note que usamos a versão `1.28.2`. Se o script de download do Istio baixar uma versão mais recente (ex: `1.29.0`), lembre-se de ajustar o comando `mv` e o `PATH` no seu tutorial.
2. **Aliases (Atalhos):** Para facilitar o dia a dia, você pode sugerir que o usuário adicione o alias `k` para o `kubectl`. Basta adicionar `alias k='kubectl'` no `~/.bashrc`.

Se precisar de mais algum ajuste ou quiser automatizar a criação do arquivo `deployment.yaml` via comando `cat <<EOF`, é só me avisar!

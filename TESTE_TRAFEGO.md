Aqui está o seu **README.md** atualizado.

Fiz alterações importantes na **Seção 6 (Sticky Sessions)**. Substituí o exemplo antigo pela configuração que funcionou (usando Gateway) e adicionei a explicação técnica de **POR QUE** precisamos usar o Gateway e não o Service direto, além de corrigir o nome do campo para `httpCookie`.

---

# 📄 README.md

```markdown
# Guia de Implementação: Canary Deployment com Nginx e Kubernetes

Este guia detalha o processo de criação de duas versões de uma aplicação Nginx, o envio para o Docker Hub e o deploy em um cluster Kubernetes com balanceamento de carga e controle fino via Istio.

## 🛠️ 1. Preparação das Imagens Docker

Precisamos criar dois contextos diferentes para que cada imagem exiba uma mensagem única.

### Versão A (Latest)
1. Crie uma pasta chamada `app-a` e um arquivo `Dockerfile` dentro dela:
```dockerfile
FROM nginx:alpine
RUN echo "ACESSOU TESTE A" > /usr/share/nginx/html/index.html
```

2. Build da imagem:
```bash
docker build -t pedrovieira249/nginx-ab:latest ./app-a
```

### Versão B
1. Crie uma pasta chamada `app-b` e um arquivo `Dockerfile` dentro dela:
```dockerfile
FROM nginx:alpine
RUN echo "ACESSOU TESTE B" > /usr/share/nginx/html/index.html
```

2. Build da imagem:
```bash
docker build -t pedrovieira249/nginx-ab:b ./app-b
```

---

## 🧪 2. Teste Local (Opcional, mas Recomendado)

Antes de subir para o registro, valide se as imagens estão respondendo corretamente:

```bash
# Testar A
docker run -d -p 8081:80 --name t-a pedrovieira249/nginx-ab:latest
curl localhost:8081 # Deve retornar: ACESSOU TESTE A
docker rm -f t-a

# Testar B
docker run -d -p 8082:80 --name t-b pedrovieira249/nginx-ab:b
curl localhost:8082 # Deve retornar: ACESSOU TESTE B
docker rm -f t-b
```

---

## 🚀 3. Upload para o Docker Hub

Faça o login e envie as imagens para o seu repositório remoto:

```bash
docker login
docker push pedrovieira249/nginx-ab:latest
docker push pedrovieira249/nginx-ab:b
```

---

## ☸️ 4. Deploy no Kubernetes

Com as imagens disponíveis, aplique o manifesto `deployment.yaml` que contém os dois Deployments (`nginx` e `nginx-b`) e o Service compartilhado.

```bash
kubectl apply -f deployment.yaml
```

Verifique se os 4 pods (2 de cada versão) estão rodando:
```bash
kubectl get pods -l app=nginx
```

---

## 🚦 5. Controle de Tráfego com Istio (Canary Release - Pesos)

O Istio permite definir pesos específicos para cada versão.

### 1. Aplicar as Regras de Tráfego (Peso 60/40)
Rode os comandos abaixo para configurar a divisão de tráfego:

```bash
kubectl apply -f ./destination-rule.yaml
kubectl apply -f ./virtual-service.yaml
```

> **⚠️ Nota:** Ao usar pesos, o Istio faz um balanceamento probabilístico. Um mesmo usuário pode ver a versão A e depois a B.

---

## 🍪 6. Sticky Sessions (Sessões Persistentes via Cookie)

Para garantir que um usuário fique "preso" a uma versão específica (ex: Client-A sempre cai no Pod A), precisamos usar **Consistent Hash**.

### 1. Configuração Correta (`consistent-hash.yaml`)
Aplique o arquivo abaixo. Note que adicionamos um **Gateway** e removemos os pesos.

```bash
kubectl apply -f consistent-hash.yaml
```

**Conteúdo do arquivo explicado:**
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
spec:
  selector:
    istio: ingressgateway # Usa o Ingress Controller do Istio
  servers:
  - port: { number: 80, name: http, protocol: HTTP }
    hosts: ["*"]
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: nginx-virtual-service
spec:
  hosts: ["*"]
  gateways: ["nginx-gateway"] # OBRIGATÓRIO: Liga ao Gateway
  http:
  - route:
    - destination:
        host: nginx-service
        # SEM PESOS (weights) AQUI! 
        # Deixamos o DestinationRule decidir qual pod usar baseado no hash.
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: nginx-destination-rule
spec:
  host: nginx-service
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpCookie:          # Usa cookie HTTP para o hash
          name: "user-session"
          ttl: 60s
```

### 2. Por que esta configuração é necessária?

1.  **Gateway vs Service:** O Sticky Session **NÃO** funciona se você acessar via `localhost:8000` (Service do K8s). O Service do Kubernetes faz apenas balanceamento TCP (Round Robin). Para o Istio ler o cookie e decidir o destino, o tráfego **tem que passar pelo Istio Ingress Gateway**.
2.  **Sem Pesos (Weights):** No `VirtualService`, removemos a divisão de porcentagem. Se definíssemos pesos, o Istio escolheria aleatoriamente a versão ANTES de verificar o cookie. Sem pesos, ele delega a escolha totalmente para o algoritmo de Hash do `DestinationRule`.
3.  **httpCookie:** A sintaxe correta na API moderna do Istio é `httpCookie`, e não apenas `cookie`.

### 3. Como testar (Port-Forward no Gateway)

Não acesse o serviço diretamente. Acesse o **Gateway**:

1.  Faça o port-forward do Ingress Gateway em um terminal separado:
    ```bash
    # A porta do Gateway geralmente é a 80 interna, mapeamos para 8080 local
    kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
    ```

2.  Teste a persistência:
    ```bash
    # O usuário "client-A" será hashado sempre para o mesmo pod
    while true; do curl -b "user-session=client-A" http://localhost:8080; sleep 0.5; done
    
    # O usuário "client-B" será hashado para outro pod (e ficará lá)
    while true; do curl -b "user-session=client-B" http://localhost:8080; sleep 0.5; done
    ```
    OBS: Pode ser que ao rodar o primeiro comando, o Istio ainda não tenha criado o cookie. Aguarde alguns segundos para que o cookie seja criado e a persistência funcione corretamente. E mesmo rodando com o user-session=client-A o cliente pode ser mandado para o outro pod na primeira requisição, mas nas próximas ele sempre cairá no mesmo pod.
---

## 🔍 7. Monitoramento com Kiali

1. Dispare carga com o Fortio (ferramenta de teste de carga):
```bash
kubectl exec [NOME_DO_POD_FORTIO] -c fortio -- fortio load -c 2 -qps 0 -t 200s -loglevel Warning http://nginx-service:8000
```

2. Acesse o dashboard do Kiali e vá em **Graph**.
3. Selecione o namespace e ative **Traffic Distribution**. Você verá visualmente como o tráfego está sendo distribuído.

---

## 🧹 8. Limpeza do Ambiente

```bash
k3d cluster delete
```
```
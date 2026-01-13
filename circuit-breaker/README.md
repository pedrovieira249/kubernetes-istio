Aqui está o `README.md` completo com o passo a passo.

Notei que você colou o código em Go duas vezes (uma delas chamando de Dockerfile). Por isso, no passo a passo abaixo, **eu criei o conteúdo do `Dockerfile` correto** para você.

---

### 📄 README.md

```markdown
# Go Chaos Microservice

Este projeto é um microserviço simples em Go que simula falhas (latência e timeouts) baseado em variáveis de ambiente.

## 📂 1. Preparação dos Arquivos

Crie uma pasta para o projeto e adicione os seguintes arquivos:

### `main.go`
Cole o código da aplicação:

```go
package main

import (
	"math/rand"
	"net/http"
	"os"
	"time"
)

func main() {
	http.HandleFunc("/", Run)
	http.ListenAndServe(":8000", nil)
}

func Run(w http.ResponseWriter, r *http.Request) {
	if os.Getenv("error") == "yes" {
		// Simula latência aleatória entre 0 e 5 segundos
		time.Sleep(time.Second * time.Duration(rand.Intn(5)))
		w.WriteHeader(http.StatusGatewayTimeout)
		return
	}
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("OK"))
}
```

### `go.mod`
Inicialize o módulo Go (necessário para o build):

```bash
go mod init circuit-breaker-example-go
```

### `Dockerfile`
Este arquivo define como empacotar a aplicação. Estamos usando um **Multi-stage build** para gerar uma imagem final leve (apenas ~10MB).

```dockerfile
# Etapa 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app

COPY go.mod ./
COPY main.go ./

# Compila o binário estático
RUN CGO_ENABLED=0 GOOS=linux go build -o server main.go

# Etapa 2: Imagem Final (Leve)
FROM alpine:latest

WORKDIR /root/

COPY --from=builder /app/server .

EXPOSE 8000

CMD ["./server"]
```

---

## 💻 2. Rodar e Testar Localmente (Sem Docker)

Antes de empacotar, vamos garantir que o código funciona na sua máquina.

1. **Rodar a aplicação:**
   ```bash
   go run main.go
   ```

2. **Testar (Em outro terminal):**
   ```bash
   curl localhost:8000
   # Resposta esperada: OK
   ```

3. **Testar Modo de Erro:**
   Pare a aplicação (Ctrl+C) e rode com a variável de ambiente:
   ```bash
   export error=yes && go run main.go
   ```
   
   Ao testar com `curl -v localhost:8000`, você verá que ele demora alguns segundos e retorna `HTTP 504 Gateway Timeout`.

---

## 🐳 3. Build e Teste com Docker

Agora vamos criar a imagem e testar o container.

1. **Construir a imagem:**
   Substitua `seu-usuario` pelo seu user do Docker Hub.
   ```bash
   docker build -t seu-usuario/circuit-breaker-example-go:latest .
   ```

2. **Rodar o container (Modo Normal):**
   ```bash
   docker run -d -p 8000:8000 --name app-normal seu-usuario/circuit-breaker-example-go:latest
   ```
   *Teste:* `curl localhost:8000` (Deve retornar OK).

3. **Rodar o container (Modo Caos/Erro):**
   Vamos subir um segundo container na porta 8001 injetando a variável de erro.
   ```bash
   docker run -d -p 8001:8000 -e "error=yes" --name app-error seu-usuario/circuit-breaker-example-go:latest
   ```
   *Teste:* `curl -v localhost:8001` (Deve demorar e dar erro 504).

4. **Limpar containers de teste:**
   ```bash
   docker rm -f app-normal app-error
   ```

---

## ☁️ 4. Subir para o Docker Hub

Com a imagem validada, vamos enviá-la para o registro público.

1. **Login no Docker Hub:**
   ```bash
   docker login
   ```

2. **Enviar a imagem:**
   ```bash
   docker push seu-usuario/circuit-breaker-example-go:latest
   ```

3. **(Opcional) Criar tag latest:**
   ```bash
   docker tag seu-usuario/circuit-breaker-example-go:latest seu-usuario/circuit-breaker-example-go:latest
   docker push seu-usuario/circuit-breaker-example-go:latest
   ```

Agora sua imagem está disponível publicamente e pronta para ser usada no Kubernetes ou qualquer outro orquestrador! 🚀

---

## ⚙️ 5. Deploy no Kubernetes com Istio

Agora vamos deployar a aplicação no Kubernetes e configurar o Circuit Breaker com Istio.

### 5.1 Aplicar o Deployment e Service

Este passo cria dois deployments:
- **servicex**: versão normal (retorna 200 OK)
- **servicex-error**: versão com falhas (simula timeouts 504)

E um Service que balanceia entre as duas versões.

```bash
kubectl apply -f kubernets/deployment.yaml
```

**Verificar os pods:**
```bash
kubectl get pods -l app=servicex
```

Você deve ver dois pods rodando (um para cada deployment).

---

### 5.2 Aplicar o Circuit Breaker (DestinationRule)

Agora vamos configurar o Circuit Breaker do Istio para detectar falhas e ejetar pods problemáticos.

```bash
kubectl apply -f kubernets/circuit-breaker.yaml
```

**Configuração do Circuit Breaker:**
- `consecutiveGatewayErrors: 2` - Ejeta o pod após 2 erros consecutivos de gateway (5xx)
- `interval: 20s` - Intervalo de análise de erros
- `baseEjectionTime: 30s` - Tempo que o pod fica ejetado
- `maxEjectionPercent: 100` - Permite ejetar até 100% dos pods

---

## 🧪 6. Teste do Circuit Breaker com Fortio

### 6.1 Deploy do Fortio (Ferramenta de Load Test)

Se ainda não tiver o Fortio deployado, aplique:

```bash
kubectl apply -f ../ferramentas/fortio-deploy.yaml
```

**Verificar o pod do Fortio:**
```bash
kubectl get pods -l app=fortio
```

Anote o nome do pod (ex: `fortio-deploy-74dcff8447-6rncd`).

---

### 6.2 Executar o Teste de Carga

Agora vamos enviar 200 requisições com 2 conexões concorrentes para o serviço:

```bash
kubectl exec fortio-deploy-74dcff8447-6rncd -c fortio -- fortio load -c 2 -qps 0 -n 200 -loglevel Warning http://servicex-service
```

**Parâmetros do teste:**
- `-c 2`: 2 conexões concorrentes
- `-qps 0`: sem limite de queries por segundo (máxima velocidade)
- `-n 200`: total de 200 requisições
- `-loglevel Warning`: exibe apenas warnings e erros

---

### 6.3 Analisar os Resultados

Após o teste, observe:

1. **Taxa de sucesso**: o Circuit Breaker deve detectar as falhas e ejetar o pod problemático
2. **Distribuição de códigos HTTP**: você verá códigos 200 (sucesso) e 504 (timeout)
3. **Latência**: as requisições bem-sucedidas devem ter baixa latência

**Exemplo de saída esperada:**
```
Code 200 : 150 (75.0 %)
Code 504 : 50 (25.0 %)
```

---

### 6.4 Teste Alternativo (Sem Keep-Alive)

Para testar sem reutilizar conexões:

```bash
kubectl exec fortio-deploy-74dcff8447-bx75w -c fortio -- fortio load -c 1 -qps 0 -n 20 -keepalive=false http://servicex-service
kubectl exec fortio-deploy-74dcff8447-bx75w -c fortio -- fortio load -c 1 -qps 0 -n 20 -timeout 5s -keepalive=false http://servicex-service
kubectl exec fortio-deploy-74dcff8447-bx75w -c fortio -- fortio load -c 2 -qps 0 -t 20s -loglevel Warning http://servicex-service

```

---

## 📊 7. Monitoramento (Opcional)

Para visualizar o comportamento do Circuit Breaker em tempo real:

**Kiali (Service Mesh Dashboard):**
```bash
kubectl apply -f ../ferramentas/kiali.yaml
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```
Acesse: http://localhost:20001

**Grafana (Métricas):**
```bash
kubectl apply -f ../ferramentas/grafana.yaml
kubectl port-forward svc/grafana 3000:3000 -n istio-system
```
Acesse: http://localhost:3000

---

## 🎯 Conclusão

Você configurou com sucesso:
✅ Um microserviço Go com simulação de falhas  
✅ Deploy no Kubernetes com Istio  
✅ Circuit Breaker para detectar e isolar pods problemáticos  
✅ Testes de carga com Fortio  

O Circuit Breaker do Istio protege sua aplicação isolando automaticamente instâncias com falhas! 🚀
```
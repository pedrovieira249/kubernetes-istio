# Configuração do Gateway Istio

Este guia explica como configurar o Gateway do Istio para funcionar corretamente com o serviço nginx no Kubernetes.

## Pré-requisitos

- Kubernetes instalado (K3s ou outro)
- Istio instalado no cluster
- kubectl configurado

## Passo 1: Aplicar o Deployment

Aplique a configuração do deployment:

```bash
kubectl apply -f deployment.yaml
```

## Passo 2: Criar o Gateway

Aplique a configuração do Gateway do Istio:

```bash
kubectl apply -f gateway.yaml
```

Este arquivo contém:
- **Gateway**: Define o ponto de entrada do tráfego externo
- **VirtualService**: Roteia o tráfego para os serviços internos (nginx v1 e v2)
- **DestinationRule**: Define as políticas de balanceamento de carga e subsets

## Passo 3: Expor o Gateway com Port Forward

Para acessar o Gateway do Istio localmente, use o comando `port-forward`:

```bash
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
```

Este comando irá expor a porta 80 do Istio Ingress Gateway na porta 8080 local. Mantenha este terminal aberto enquanto estiver testando.

## Passo 4: Verificar a Configuração

Verifique se o Gateway foi criado corretamente:

```bash
kubectl get gateway
```

Verifique o VirtualService:

```bash
kubectl get virtualservice
```

Verifique o DestinationRule:

```bash
kubectl get destinationrule
```

## Passo 5: Testar o Gateway

### Testar o Acesso

Com o port-forward ativo, acesse o serviço via localhost na porta 8080:

```bash
curl http://localhost:8080
```

### Testar o Balanceamento de Carga

```bash
# Teste múltiplas requisições para verificar o balanceamento de carga
for i in {1..10}; do curl http://localhost:8080; done
```

## Troubleshooting

### Gateway não está acessível

1. Verifique se o pod do Istio Ingress Gateway está rodando:
   ```bash
   kubectl get pods -n istio-system
   ```

2. Verifique os logs do Ingress Gateway:
   ```bash
   kubectl logs -n istio-system -l app=istio-ingressgateway
   ```

3. Confirme se o port-forward está ativo:
   ```bash
   # Verifique se a porta 8080 está em uso
   lsof -i :8080
   ```

4. Teste a conexão diretamente:
   ```bash
   curl -v http://localhost:8080
   ```

### Tráfego não está sendo roteado

1. Verifique se os pods do nginx estão rodando:
   ```bash
   kubectl get pods -l app=nginx
   ```

2. Verifique se o serviço nginx está configurado corretamente:
   ```bash
   kubectl get svc nginx-service
   ```

3. Verifique a configuração do VirtualService:
   ```bash
   kubectl describe virtualservice nginx-virtual-service
   ```

## Estrutura da Configuração

### Gateway
- **Nome**: `ingress-gateway-k3s`
- **Porta**: 80 (HTTP)
- **Protocolo**: HTTP/2
- **Hosts**: `*` (aceita qualquer host)

### VirtualService
- **Nome**: `nginx-virtual-service`
- **Gateway**: `ingress-gateway-k3s`
- **Roteamento**: 
  - 100% do tráfego para `nginx-service` subset `v1`
  - 0% do tráfego para `nginx-service` subset `v2`

### DestinationRule
- **Nome**: `nginx-destination-rule`
- **Load Balancer**: ROUND_ROBIN
- **Subsets**:
  - `v1`: Pods com label `version: A`
  - `v2`: Pods com label `version: B`

## Modificando a Distribuição de Tráfego

Para alterar a distribuição de tráfego entre as versões, edite o arquivo `gateway.yaml` e ajuste os pesos (weights):

```yaml
http:
  - route:  
    - destination:
        host: nginx-service
        subset: v1
      weight: 50  # 50% do tráfego
    - destination:
        host: nginx-service
        subset: v2
      weight: 50  # 50% do tráfego
```

Reaplique a configuração:

```bash
kubectl apply -f gateway.yaml
```

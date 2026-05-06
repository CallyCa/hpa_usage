# Exemplos sistemáticos dos Feature Models HPA por versão da API

Este documento cobre todos os campos de cada versão, organizados segundo o feature model de cada versão da API. Para cada versão são apresentados: o exemplo mínimo com apenas os campos obrigatórios, seguido de um exemplo para cada campo ou grupo opcional adicionado progressivamente, e por fim o exemplo completo com todos os campos disponíveis.

A referência para obrigatoriedade e opcionalidade de cada campo é a especificação oficial da API:
- v1: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v1/
- v2: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/horizontal-pod-autoscaler-v2/

---

## autoscaling/v1

### Campos do feature model v1

**Obrigatórios:**
- `apiVersion`, `kind`, `metadata.name`
- `spec.scaleTargetRef.kind`, `spec.scaleTargetRef.name`
- `spec.maxReplicas`

**Opcionais:**
- `metadata.namespace`, `metadata.labels`, `metadata.annotations`, `metadata.generateName`
- `spec.scaleTargetRef.apiVersion`
- `spec.minReplicas` (padrão: 1 quando omitido)
- `spec.targetCPUUtilizationPercentage`
- `status` (preenchido automaticamente pelo controlador; nunca incluído pelo usuário)

**Alternativas para `scaleTargetRef.kind`:**
`Deployment`, `ReplicaSet`, `StatefulSet`, `ReplicationController`

---

### v1-E1: somente campos obrigatórios

O HPA mais simples possível. Sem `targetCPUUtilizationPercentage`, o objeto é válido pelo API server mas o controlador não tem critério de escalonamento e permanece inativo. Útil apenas para testar conectividade com o workload alvo.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api          # único campo obrigatório de metadata
spec:
  scaleTargetRef:
    kind: Deployment       # campo obrigatório; sem apiVersion, assume apps/v1
    name: minha-api        # deve corresponder ao nome exato do Deployment
  maxReplicas: 10          # único campo de réplicas obrigatório
```

---

### v1-E2: obrigatórios + namespace

`namespace` é opcional mas necessário quando o HPA e o workload estão num namespace diferente de `default`, ou quando o manifesto será aplicado por uma pipeline de CD que não injeta namespace automaticamente.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
  namespace: producao       # opcional; se omitido, usa o namespace do contexto atual
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  maxReplicas: 10
```

---

### v1-E3: obrigatórios + labels e annotations

Labels são usados para seleção e agrupamento lógico de recursos. Annotations carregam metadados arbitrários sem semântica de seleção, como links para runbooks, equipe responsável ou configurações de ferramentas externas.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
  labels:
    app: minha-api
    env: producao
    equipe: plataforma
  annotations:
    runbook: "https://wiki.empresa.com/runbooks/minha-api"
    owner: "equipe-plataforma@empresa.com"
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  maxReplicas: 10
```

---

### v1-E4: obrigatórios + scaleTargetRef.apiVersion

`apiVersion` dentro de `scaleTargetRef` é opcional para workloads nativos do Kubernetes. Quando omitido, o HPA usa `apps/v1` para Deployment, ReplicaSet e StatefulSet, e `v1` para ReplicationController. Incluir explicitamente é recomendado para CRDs customizados.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1    # opcional para tipos nativos; obrigatório para CRDs
    kind: Deployment
    name: minha-api
  maxReplicas: 10
```

---

### v1-E5: obrigatórios + minReplicas

Sem `minReplicas`, o padrão é 1. Isso significa que em carga zero o HPA pode reduzir para 1 réplica, criando risco de indisponibilidade se esse pod falhar. `minReplicas: 2` garante sempre uma réplica de reserva.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2       # garante mínimo de 2 réplicas mesmo com carga zero
  maxReplicas: 10
```

---

### v1-E6: obrigatórios + targetCPUUtilizationPercentage

Adiciona o critério de escalonamento. O HPA agora está funcional: quando o uso médio de CPU dos pods superar 70% de `requests.cpu`, o controlador calcula quantas réplicas são necessárias e atualiza o Deployment.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70   # percentual de requests.cpu; sem requests configurado no pod, este campo não funciona
```

---

### v1-E7: alternativa kind = ReplicaSet

Escala um ReplicaSet diretamente, sem passar pelo Deployment. É incomum em produção porque bypassa o controle de rollout do Deployment, mas é válido quando o workload não usa Deployment.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: ReplicaSet       # alternativa ao Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

---

### v1-E8: alternativa kind = StatefulSet

StatefulSets mantêm identidade estável para cada pod (nomes como `pod-0`, `pod-1`) e volumes persistentes associados. O HPA escala o número de réplicas seguindo as regras do StatefulSet; ao diminuir, sempre remove o pod com maior índice.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: banco-hpa
  namespace: dados
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet      # workloads com estado persistente
    name: banco-cache
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 80
```

---

### v1-E9: alternativa kind = ReplicationController

Tipo legado anterior ao Deployment. Ainda funcional em clusters antigos, mas não recomendado para novos workloads.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-legacy-hpa
spec:
  scaleTargetRef:
    apiVersion: v1             # ReplicationController usa apiVersion v1, não apps/v1
    kind: ReplicationController
    name: nginx-legacy
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

---

### v1-E10: alternativa kind = CRD customizado

Qualquer recurso que implemente o subrecurso `/scale` pode ser alvo do HPA. Neste caso, `IndexerCluster` é um CRD do Splunk Operator.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: splunk-indexer-hpa
  namespace: splunk
spec:
  scaleTargetRef:
    apiVersion: enterprise.splunk.com/v2   # apiVersion do CRD customizado
    kind: IndexerCluster                    # tipo definido pelo Splunk Operator
    name: idx-producao
  minReplicas: 3
  maxReplicas: 8
  targetCPUUtilizationPercentage: 75
```

---

### v1-E11: exemplo completo com todos os campos

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
  namespace: producao
  labels:
    app: minha-api
    env: producao
    equipe: plataforma
  annotations:
    runbook: "https://wiki.empresa.com/runbooks/minha-api"
    owner: "equipe-plataforma@empresa.com"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

---

---

## autoscaling/v2beta1

### Campos do feature model v2beta1

**Obrigatórios:**
- `apiVersion`, `kind`, `metadata.name`
- `spec.scaleTargetRef.kind`, `spec.scaleTargetRef.name`
- `spec.maxReplicas`

**Opcionais:**
- `metadata.namespace`, `metadata.labels`, `metadata.annotations`
- `spec.scaleTargetRef.apiVersion`
- `spec.minReplicas`
- `spec.metrics[]` (o array inteiro é opcional; sem ele o HPA é inativo)

**Dentro de cada entrada de `spec.metrics[]`:**

Para `type: Resource`:
- `resource.name` (obrigatório): `cpu` ou `memory`
- `resource.target` (obrigatório em v2beta1 via campos diretos)
- `resource.targetAverageUtilization` (alternativa de target para cpu)
- `resource.targetAverageValue` (alternativa de target para memory ou valor absoluto)

Para `type: Pods`:
- `pods.metricName` (obrigatório)
- `pods.targetAverageValue` (obrigatório)
- `pods.selector` (opcional)

Para `type: Object`:
- `object.metricName` (obrigatório)
- `object.target` (obrigatório): `apiVersion`, `kind`, `name`
- `object.targetValue` (obrigatório)
- `object.selector` (opcional)

Para `type: External`:
- `external.metricName` (obrigatório)
- `external.targetValue` (alternativa com `targetAverageValue`)
- `external.metricSelector` (opcional mas necessário para filtrar séries)

Para `type: ContainerResource`:
- `containerResource.name` (obrigatório)
- `containerResource.container` (obrigatório)
- `containerResource.targetAverageUtilization` ou `targetAverageValue` (obrigatório)

---

### v2beta1-E1: somente campos obrigatórios, sem métricas

O HPA é válido mas inativo; sem `spec.metrics[]`, não há critério de escalonamento.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  maxReplicas: 10
```

---

### v2beta1-E2: + tipo Resource com cpu e targetAverageUtilization

`targetAverageUtilization` é o threshold em percentual do `requests.cpu` configurado no container. Comportamento equivalente ao `targetCPUUtilizationPercentage` da v1, mas dentro da estrutura `spec.metrics[]`.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70   # 70% de requests.cpu em média por pod
```

---

### v2beta1-E3: + tipo Resource com memory e targetAverageValue

Memory usa `targetAverageValue` (valor absoluto em bytes) em vez de `targetAverageUtilization`, porque o comportamento do uso de memória em relação a `requests.memory` não é linear da mesma forma que CPU. Memória não tem throttling; o processo é morto quando excede o `limits.memory`.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 512Mi    # valor absoluto em mebibytes por pod
```

---

### v2beta1-E4: + tipo Resource com cpu e memory combinados

Quando múltiplas métricas estão configuradas, o HPA calcula o número ideal de réplicas para cada uma e usa o maior resultado. Assim, a aplicação escala se cpu OU memory estiver acima do threshold.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 512Mi
```

---

### v2beta1-E5: + tipo Pods com metricName e targetAverageValue

O tipo `Pods` monitora uma métrica customizada média por pod, exposta via Custom Metrics API. Requer um adapter instalado no cluster como o `prometheus-adapter`. O campo `metricName` é uma string direta, sintaxe exclusiva da v2beta1.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: processador-eventos
spec:
  scaleTargetRef:
    kind: Deployment
    name: processador-eventos
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metricName: http_requests_per_second   # nome da métrica no adapter (sintaxe v2beta1)
      targetAverageValue: 100                 # máximo 100 req/s por pod em média
```

---

### v2beta1-E6: + tipo Pods com selector opcional

O `selector` filtra quais pods participam do cálculo da métrica. Útil quando vários grupos de pods expõem a mesma métrica com valores distintos.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: worker-pool-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: worker-pool
  minReplicas: 1
  maxReplicas: 15
  metrics:
  - type: Pods
    pods:
      metricName: jobs_in_queue
      targetAverageValue: 5
      selector:                       # filtra pods com este label antes de calcular a média
        matchLabels:
          tier: worker
```

---

### v2beta1-E7: + tipo Object com metricName, target e targetValue

O tipo `Object` monitora uma métrica associada a um objeto Kubernetes específico, como um Ingress. `target` nesta versão é a referência ao objeto (renomeado para `describedObject` na v2beta2).

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Object
    object:
      metricName: requests_per_second     # métrica exposta pelo adapter sobre o Ingress
      target:                             # referência ao objeto K8s monitorado (campo v2beta1)
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: backend-ingress
      targetValue: 1000                   # valor total sem divisão por pod
```

---

### v2beta1-E8: + tipo Object com selector opcional

`selector` filtra labels da métrica quando o adapter expõe múltiplas séries para o mesmo objeto.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Object
    object:
      metricName: requests_per_second
      target:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: backend-ingress
      targetValue: 1000
      selector:                           # filtra série específica quando há múltiplas
        matchLabels:
          route: api-v2
```

---

### v2beta1-E9: + tipo External com metricName e targetValue

`External` monitora métricas de fora do cluster, como tamanho de fila ou lag de mensagens. Requer a External Metrics API implementada por um adapter. `targetValue` compara o valor total sem dividir pelo número de réplicas.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: consumidor-sqs
spec:
  scaleTargetRef:
    kind: Deployment
    name: consumidor-sqs
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metricName: sqs_messages_visible    # nome da métrica no adapter de métricas externas
      targetValue: 500                    # escala quando fila tiver mais de 500 mensagens
```

---

### v2beta1-E10: + tipo External com targetAverageValue

`targetAverageValue` divide o valor total pelo número de réplicas antes de comparar. O escalonamento se estabiliza naturalmente: mais réplicas processam mais mensagens, reduzindo a média por pod até atingir o threshold.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: consumidor-kafka
spec:
  scaleTargetRef:
    kind: Deployment
    name: consumidor-kafka
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metricName: kafka_consumer_group_lag
      targetAverageValue: "1000"          # máximo 1.000 mensagens de lag por réplica
```

---

### v2beta1-E11: + tipo External com metricSelector opcional

`metricSelector` filtra séries temporais quando múltiplas compartilham o mesmo `metricName`. Essencial quando o adapter serve métricas de múltiplas filas com o mesmo nome de métrica.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: consumidor-fila-prioridade
spec:
  scaleTargetRef:
    kind: Deployment
    name: consumidor-fila-prioridade
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metricName: rabbitmq_queue_messages
      metricSelector:                     # identifica qual fila monitorar entre várias
        matchLabels:
          rabbitmq_queue: fila-alta-prioridade
      targetValue: 200
```

---

### v2beta1-E12: + tipo ContainerResource com cpu

`ContainerResource` monitora CPU ou memória de um container específico dentro do pod, ignorando os demais containers. Resolve o problema de sidecars que distorcem a média agregada do tipo Resource.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: app-com-sidecar
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-com-sidecar
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app                      # monitora apenas o container principal
      targetAverageUtilization: 70        # ignora CPU do sidecar de proxy ou logs
```

---

### v2beta1-E13: + tipo ContainerResource com memory

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: app-com-sidecar
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-com-sidecar
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: memory
      container: app
      targetAverageValue: 256Mi           # valor absoluto de memória por réplica do container
```

---

### v2beta1-E14: combinação de todos os tipos de métrica

Quando todos os tipos são usados simultaneamente, o HPA calcula o número de réplicas necessário para cada um e usa o maior resultado, garantindo que nenhuma métrica fique acima do threshold.

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: servico-completo
  namespace: producao
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: servico-completo
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 512Mi
  - type: Pods
    pods:
      metricName: http_requests_per_second
      targetAverageValue: 100
  - type: Object
    object:
      metricName: requests_per_second
      target:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: servico-ingress
      targetValue: 5000
  - type: External
    external:
      metricName: kafka_consumer_group_lag
      metricSelector:
        matchLabels:
          consumer_group: servico-completo
      targetAverageValue: "500"
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app
      targetAverageUtilization: 60
```

---

---

## autoscaling/v2beta2

### Campos do feature model v2beta2

A estrutura de `spec.metrics[]` é a mesma da v2beta1, com renomeação dos campos de target para a estrutura unificada `target.type`/`target.averageUtilization`/`target.averageValue`/`target.value`. Os campos `metricName` e `metricSelector` passam a ser `metric.name` e `metric.selector`. O campo `target` do tipo Object é renomeado para `describedObject`.

**Novo bloco `spec.behavior`:**

**Obrigatórios dentro de cada policy:**
- `policies[].type`: `Pods` ou `Percent`
- `policies[].value`: número inteiro positivo
- `policies[].periodSeconds`: duração do período em segundos

**Opcionais:**
- `behavior.scaleUp` (bloco inteiro opcional)
- `behavior.scaleDown` (bloco inteiro opcional)
- `behavior.scaleUp.stabilizationWindowSeconds` (padrão: 0)
- `behavior.scaleDown.stabilizationWindowSeconds` (padrão: 300)
- `behavior.scaleUp.selectPolicy`: `Max` (padrão), `Min`, `Disabled`
- `behavior.scaleDown.selectPolicy`: `Max` (padrão), `Min`, `Disabled`
- `behavior.scaleUp.policies[]`
- `behavior.scaleDown.policies[]`

**Novo tipo de métrica `ContainerResource`:**
- `containerResource.name` (obrigatório): `cpu` ou `memory`
- `containerResource.container` (obrigatório): nome exato do container
- `containerResource.target` (obrigatório): estrutura unificada `MetricTarget`

---

### v2beta2-E1: somente campos obrigatórios

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  maxReplicas: 10
```

---

### v2beta2-E2: + métrica Resource cpu com nova sintaxe target.type

O campo `targetAverageUtilization` da v2beta1 é substituído pela estrutura `target:` com `type: Utilization` e `averageUtilization`. Manifestos v2beta1 não são válidos em v2beta2 sem esta conversão.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization            # substituiu targetAverageUtilization da v2beta1
        averageUtilization: 70
```

---

### v2beta2-E3: + métrica Resource memory com AverageValue

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue           # substituiu targetAverageValue da v2beta1
        averageValue: 512Mi
```

---

### v2beta2-E4: + behavior com apenas scaleDown

Adicionar somente `scaleDown` dentro de `behavior` configura o comportamento de redução de réplicas sem alterar o comportamento padrão de scale-up. Útil quando o scale-up padrão é adequado mas o scale-down precisa ser mais conservador.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:                        # bloco opcional; sem ele, usa o padrão de 5 min de espera
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25                     # campos obrigatórios dentro de cada policy
        periodSeconds: 60
```

---

### v2beta2-E5: + behavior com apenas scaleUp

Configurar somente `scaleUp` é útil quando o scale-down padrão é adequado mas o scale-up precisa ser mais rápido ou mais controlado do que o padrão (que permite dobrar as réplicas por ciclo).

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # escala imediatamente sem esperar
      policies:
      - type: Pods
        value: 4                        # adiciona no máximo 4 pods por ciclo
        periodSeconds: 30
```

---

### v2beta2-E6: + behavior scaleUp com type: Percent

`type: Percent` define o limite de escalonamento como percentual do número atual de réplicas. Com 4 réplicas e `value: 100`, o HPA pode adicionar até 4 pods num ciclo (100% de 4). Com 8 réplicas, pode adicionar até 8.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100                       # pode dobrar as réplicas por ciclo
        periodSeconds: 15
```

---

### v2beta2-E7: + behavior scaleUp com duas policies e selectPolicy: Max

Quando há duas policies, `selectPolicy: Max` usa a que permite adicionar mais réplicas. No exemplo abaixo, com 3 réplicas: `type: Percent value: 100` permite adicionar 3 pods; `type: Pods value: 4` permite adicionar 4. O `Max` escolhe 4.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max                  # usa a policy mais permissiva entre as duas
```

---

### v2beta2-E8: + behavior scaleDown com duas policies e selectPolicy: Min

`selectPolicy: Min` escolhe a policy que resulta na menor remoção. Com 10 réplicas: `type: Percent value: 10` remove 1 pod (10% de 10); `type: Pods value: 2` remove 2. O `Min` escolhe 1.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min                  # usa a policy mais conservadora entre as duas
```

---

### v2beta2-E9: + behavior scaleDown com selectPolicy: Disabled

`selectPolicy: Disabled` impede completamente o escalonamento nessa direção. O HPA continua monitorando as métricas e calcularia réplicas, mas nunca aplica a redução. Útil durante manutenções ou para workloads onde reduzir réplicas é perigoso (como clusters de banco de dados distribuídos).

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: banco-distribuido-hpa
spec:
  scaleTargetRef:
    kind: StatefulSet
    name: banco-distribuido
  minReplicas: 3
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
  behavior:
    scaleDown:
      selectPolicy: Disabled             # nunca remove réplicas automaticamente
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120               # adiciona apenas 1 réplica a cada 2 minutos
```

---

### v2beta2-E10: + tipo Pods com nova sintaxe metric.name

Na v2beta1 o campo era `metricName` (string direta). Na v2beta2 passa a ser `metric.name` dentro de um objeto `metric`.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: processador-eventos
spec:
  scaleTargetRef:
    kind: Deployment
    name: processador-eventos
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second   # nova sintaxe v2beta2: metric.name em vez de metricName
      target:
        type: AverageValue
        averageValue: 100
```

---

### v2beta2-E11: + tipo Pods com metric.selector opcional

`metric.selector` é o novo nome de `selector` (que era `metricSelector` para External em v2beta1). Filtra quais séries temporais participam do cálculo.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: processador-eventos
spec:
  scaleTargetRef:
    kind: Deployment
    name: processador-eventos
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
        selector:                        # nova sintaxe v2beta2 para filtro de labels
          matchLabels:
            rota: api-v2
      target:
        type: AverageValue
        averageValue: 100
```

---

### v2beta2-E12: + tipo Object com describedObject (renomeado de target)

`describedObject` substitui o campo `target` da v2beta1 no tipo Object, separando a referência ao objeto da configuração do threshold. O campo `target` agora é exclusivamente o threshold.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Object
    object:
      metric:
        name: requests_per_second        # nova sintaxe: metric.name
      describedObject:                   # renomeado de target da v2beta1
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: backend-ingress
      target:
        type: Value                      # nova estrutura unificada de target
        value: 1000
```

---

### v2beta2-E13: + tipo External com nova sintaxe

`metricName` e `metricSelector` da v2beta1 passam a ser `metric.name` e `metric.selector`.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: consumidor-kafka
spec:
  scaleTargetRef:
    kind: Deployment
    name: consumidor-kafka
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_group_lag   # nova sintaxe: metric.name
        selector:                        # nova sintaxe: metric.selector
          matchLabels:
            consumer_group: meu-grupo
      target:
        type: AverageValue
        averageValue: "1000"
```

---

### v2beta2-E14: + tipo ContainerResource com cpu

`ContainerResource` é um tipo novo introduzido no v2beta2. O campo `container` identifica qual container dentro do pod monitorar. O campo é obrigatório; se o nome não corresponder a um container real do pod, o HPA entra em erro.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: app-com-sidecar
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-com-sidecar
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app                     # nome exato do container a monitorar
      target:
        type: Utilization
        averageUtilization: 70
```

---

### v2beta2-E15: + tipo ContainerResource com memory

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: app-com-sidecar
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-com-sidecar
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: memory
      container: app
      target:
        type: AverageValue
        averageValue: 256Mi
```

---

### v2beta2-E16: + múltiplos ContainerResource para diferentes containers

Cada container do pod pode ter suas próprias métricas. O HPA calcula o número de réplicas para cada uma e usa o maior.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: app-multi-container
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-multi-container
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app                   # container principal da aplicação
      target:
        type: Utilization
        averageUtilization: 70
  - type: ContainerResource
    containerResource:
      name: memory
      container: app
      target:
        type: Utilization
        averageUtilization: 80
  - type: ContainerResource
    containerResource:
      name: cpu
      container: proxy                 # sidecar de proxy (ex: Envoy do Istio)
      target:
        type: Utilization
        averageUtilization: 80         # threshold mais alto para o proxy, que tolera picos
```

---

### v2beta2-E17: exemplo completo com todos os campos e tipos

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: servico-completo
  namespace: producao
  labels:
    app: servico-completo
    env: producao
  annotations:
    runbook: "https://wiki.empresa.com/runbooks/servico-completo"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: servico-completo
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app
      target:
        type: Utilization
        averageUtilization: 65
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
        selector:
          matchLabels:
            rota: api-v2
      target:
        type: AverageValue
        averageValue: 100
  - type: Object
    object:
      metric:
        name: requests_per_second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: servico-ingress
      target:
        type: Value
        value: 5000
  - type: External
    external:
      metric:
        name: kafka_consumer_group_lag
        selector:
          matchLabels:
            consumer_group: servico-completo
      target:
        type: AverageValue
        averageValue: "500"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

---

---

## autoscaling/v2 GA

### Campos do feature model v2 GA

A estrutura é idêntica ao v2beta2. A única diferença é `apiVersion: autoscaling/v2`. Qualquer manifesto v2beta2 válido passa a ser v2 GA apenas com essa troca. Os exemplos abaixo cobrem os campos específicos da estrutura `MetricTarget` unificada e `MetricIdentifier`, que na v2 GA são os tipos formais da especificação.

**`MetricTarget` (campo `target:` em todas as métricas):**
- `type: Utilization` com `averageUtilization` (inteiro): exclusivo de Resource e ContainerResource
- `type: AverageValue` com `averageValue` (quantidade): todos os tipos de métrica
- `type: Value` com `value` (quantidade): principalmente Object e External

**`MetricIdentifier` (campo `metric:` em Pods, Object e External):**
- `name` (obrigatório)
- `selector` (opcional): `matchLabels` ou `matchExpressions`

---

### v2-E1: somente campos obrigatórios

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  maxReplicas: 10
```

---

### v2-E2: + Resource cpu com target.type: Utilization e averageUtilization

`type: Utilization` é exclusivo de Resource e ContainerResource. Usa `requests` como denominador; sem `requests.cpu` configurado no container, o HPA não consegue calcular o percentual e entra em erro.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70         # percentual de requests.cpu; requer requests configurado no pod
```

---

### v2-E3: + Resource memory com target.type: AverageValue

`type: AverageValue` usa um valor absoluto por pod. Não depende de `requests` como denominador. Pode ser usado com Resource, ContainerResource, Pods e External.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi            # valor absoluto por pod; independente de requests.memory
```

---

### v2-E4: + Object com target.type: Value

`type: Value` compara o valor total observado diretamente com o threshold, sem dividir pelo número de réplicas. Adequado para métricas globais como total de requisições num Ingress, onde o valor não diminui naturalmente ao adicionar réplicas de backend.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Object
    object:
      metric:
        name: requests_per_second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: backend-ingress
      target:
        type: Value                    # valor total; 1000 req/s no Ingress, não dividido por pod
        value: 1000
```

---

### v2-E5: + External com target.type: AverageValue e metric.selector

`metric.selector` com `matchLabels` filtra séries temporais por labels. `matchExpressions` permite expressões mais complexas como `In`, `NotIn`, `Exists`.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: consumidor-kafka
spec:
  scaleTargetRef:
    kind: Deployment
    name: consumidor-kafka
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: kafka_consumer_group_lag
        selector:
          matchLabels:
            consumer_group: meu-grupo  # matchLabels para igualdade simples
      target:
        type: AverageValue
        averageValue: "1000"
```

---

### v2-E6: + metric.selector com matchExpressions

`matchExpressions` permite filtros mais complexos do que `matchLabels`. `operator: In` verifica se o valor está numa lista; `operator: Exists` verifica apenas a presença do label.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: consumidor-multiplas-filas
spec:
  scaleTargetRef:
    kind: Deployment
    name: consumidor-multiplas-filas
  minReplicas: 1
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: fila_mensagens_pendentes
        selector:
          matchExpressions:            # alternativa ao matchLabels para filtros complexos
          - key: ambiente
            operator: In
            values: ["producao", "staging"]
          - key: prioridade
            operator: Exists           # apenas verifica se o label existe
      target:
        type: AverageValue
        averageValue: "500"
```

---

### v2-E7: + ContainerResource com cpu e Utilization

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-com-sidecar
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-com-sidecar
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app
      target:
        type: Utilization
        averageUtilization: 70
```

---

### v2-E8: + ContainerResource com memory e AverageValue

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-com-sidecar
spec:
  scaleTargetRef:
    kind: Deployment
    name: app-com-sidecar
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: ContainerResource
    containerResource:
      name: memory
      container: app
      target:
        type: AverageValue
        averageValue: 256Mi
```

---

### v2-E9: + behavior apenas com stabilizationWindowSeconds

`stabilizationWindowSeconds` pode ser configurado sem `policies`. Neste caso, o HPA ainda usa as policies padrão mas respeita a janela de estabilização configurada.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600    # 10 minutos de observação antes de remover réplicas
```

---

### v2-E10: + behavior com policy type: Pods

`type: Pods` define um limite absoluto de quantos pods podem ser adicionados ou removidos por ciclo, independente do total atual de réplicas.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 3                          # remove no máximo 3 pods por ciclo
        periodSeconds: 60
```

---

### v2-E11: + behavior com policy type: Percent

`type: Percent` define o limite como percentual das réplicas atuais. Com 10 réplicas e `value: 20`, o limite é 2 pods. Com 50 réplicas, o limite é 10 pods. O comportamento escala com o tamanho do cluster.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 50                         # adiciona no máximo 50% das réplicas atuais por ciclo
        periodSeconds: 30
```

---

### v2-E12: + behavior scaleUp e scaleDown com selectPolicy: Max e Min respectivamente

Padrão de produção mais comum: scale-up agressivo com `Max` (responde rápido a picos) e scale-down conservador com `Min` (evita remover réplicas que serão necessárias em breve).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minha-api
spec:
  scaleTargetRef:
    kind: Deployment
    name: minha-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max                   # usa a policy que permite adicionar mais réplicas
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min                   # usa a policy que remove menos réplicas
```

---

### v2-E13: + behavior scaleDown com selectPolicy: Disabled

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: banco-distribuido-hpa
spec:
  scaleTargetRef:
    kind: StatefulSet
    name: banco-distribuido
  minReplicas: 3
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
  behavior:
    scaleDown:
      selectPolicy: Disabled             # nunca remove réplicas; scale-down somente manual
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
```

---

### v2-E14: + behavior scaleUp com selectPolicy: Disabled

Desabilitar o scale-up mantém o número atual de réplicas fixo. O HPA monitora as métricas mas nunca adiciona réplicas. Equivale a um HPA "read-only" para aquela direção.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: servico-congelado
spec:
  scaleTargetRef:
    kind: Deployment
    name: servico-congelado
  minReplicas: 4
  maxReplicas: 4                          # min == max; reforça o congelamento
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      selectPolicy: Disabled             # não adiciona réplicas automaticamente
    scaleDown:
      selectPolicy: Disabled             # não remove réplicas automaticamente
```

---

### v2-E15: alternativa Rollout como alvo (Argo Rollouts CRD)

O HPA v2 GA pode escalar CRDs que implementam o subrecurso `/scale`, incluindo o `Rollout` do Argo Rollouts, que adiciona estratégias de deploy como canary e blue-green sobre o HPA.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: demo-hpa
spec:
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1    # CRD do Argo Rollouts
    kind: Rollout                        # não é um tipo nativo do Kubernetes
    name: rollout-hpa-example
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 16Mi
```

---

### v2-E16: exemplo completo com todos os campos e tipos

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: servico-completo
  namespace: producao
  labels:
    app: servico-completo
    env: producao
    equipe: plataforma
  annotations:
    runbook: "https://wiki.empresa.com/runbooks/servico-completo"
    owner: "equipe-plataforma@empresa.com"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: servico-completo
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 512Mi
  - type: ContainerResource
    containerResource:
      name: cpu
      container: app
      target:
        type: Utilization
        averageUtilization: 65
  - type: ContainerResource
    containerResource:
      name: memory
      container: app
      target:
        type: AverageValue
        averageValue: 256Mi
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
        selector:
          matchLabels:
            rota: api-v2
      target:
        type: AverageValue
        averageValue: 100
  - type: Object
    object:
      metric:
        name: requests_per_second
        selector:
          matchLabels:
            ambiente: producao
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: servico-ingress
      target:
        type: Value
        value: 5000
  - type: External
    external:
      metric:
        name: kafka_consumer_group_lag
        selector:
          matchLabels:
            consumer_group: servico-completo
      target:
        type: AverageValue
        averageValue: "500"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

---

## Tabela de campos por versão

| Campo | v1 | v2beta1 | v2beta2 | v2 GA |
|---|---|---|---|---|
| `targetCPUUtilizationPercentage` | disponível | removido | removido | removido |
| `spec.metrics[]` | indisponível | introduzido | mantido | mantido |
| `resource.targetAverageUtilization` | n/a | disponível | substituído por `target.type: Utilization` | substituído |
| `resource.targetAverageValue` | n/a | disponível | substituído por `target.type: AverageValue` | substituído |
| `pods.metricName` | n/a | disponível | substituído por `metric.name` | substituído |
| `pods.targetAverageValue` | n/a | disponível | substituído por `target.type: AverageValue` | substituído |
| `object.target` (referência) | n/a | disponível | renomeado para `describedObject` | renomeado |
| `object.targetValue` | n/a | disponível | substituído por `target.type: Value` | substituído |
| `external.metricName` | n/a | disponível | substituído por `metric.name` | substituído |
| `external.metricSelector` | n/a | disponível | substituído por `metric.selector` | substituído |
| `external.targetValue` | n/a | disponível | substituído por `target.type: Value` | substituído |
| `external.targetAverageValue` | n/a | disponível | substituído por `target.type: AverageValue` | substituído |
| `spec.behavior` | indisponível | indisponível | introduzido | mantido |
| `behavior.*.stabilizationWindowSeconds` | n/a | n/a | introduzido | mantido |
| `behavior.*.selectPolicy` | n/a | n/a | introduzido | mantido |
| `behavior.*.policies[].type: Pods` | n/a | n/a | introduzido | mantido |
| `behavior.*.policies[].type: Percent` | n/a | n/a | introduzido | mantido |
| `ContainerResource` | n/a | disponível (parcial) | formalizado | mantido |
| `containerResource.container` | n/a | n/a | introduzido | mantido |
| `target.type: Utilization` | n/a | n/a | introduzido | mantido |
| `target.type: AverageValue` | n/a | n/a | introduzido | mantido |
| `target.type: Value` | n/a | n/a | introduzido | mantido |
| API estável (GA) | v1 (removida K8s 1.26) | beta (removida K8s 1.22) | beta (removida K8s 1.26) | estável desde K8s 1.23 |

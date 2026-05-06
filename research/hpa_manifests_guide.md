# Manifestos HPA do Kubernetes: guia completo por versão da API

O Horizontal Pod Autoscaler evoluiu por quatro versões da API, de `autoscaling/v1` com suporte exclusivo a CPU até `autoscaling/v2` GA com métricas múltiplas, controle comportamental granular e escalabilidade por contêiner individual. Este guia reúne exemplos reais e verificados da documentação oficial do Kubernetes e de repositórios públicos no GitHub, cobrindo cada campo e feature específica de cada versão. A versão `autoscaling/v2` (GA desde Kubernetes 1.23) é hoje o padrão recomendado; `v2beta1` e `v2beta2` foram removidas no Kubernetes 1.26.

---

## 1. autoscaling/v1: a versão original com CPU apenas

A primeira versão do HPA foi lançada no Kubernetes 1.2 (março de 2016) e suporta exclusivamente `targetCPUUtilizationPercentage` como critério de escalonamento. Não há suporte a memória, métricas customizadas ou controle de velocidade de escalonamento.

Os campos disponíveis são:

- `scaleTargetRef`: referência ao workload que será escalado. Aceita `kind: Deployment`, `kind: StatefulSet`, `kind: ReplicaSet`, `kind: ReplicationController` e qualquer recurso que implemente o subrecurso `/scale`.
- `minReplicas`: número mínimo de réplicas mantido mesmo com carga zero. O padrão é 1 quando omitido.
- `maxReplicas`: teto absoluto de réplicas. É o único campo obrigatório de `spec`; o API server rejeita manifestos que o omitem.
- `targetCPUUtilizationPercentage`: percentual alvo de uso de CPU em relação ao `requests.cpu` configurado nos containers do pod. O controlador usa `requests` como denominador, não `limits`.

Quando o controlador processa um objeto v1 internamente, ele traduz `targetCPUUtilizationPercentage` para a estrutura `spec.metrics[]` com `type: Resource`, `name: cpu` e `target.type: Utilization`. O manifesto YAML armazenado no etcd, porém, permanece com a sintaxe simplificada.

A limitação mais relevante da v1 em produção é a ausência de `spec.behavior`. Sem ele, o HPA usa os defaults assimétricos do controlador: scale-up imediato a cada ciclo de 15 segundos, e scale-down somente após 5 minutos de métricas abaixo do threshold. Quando o scale-down finalmente ocorre, pode remover um número grande de réplicas de uma vez.

### Exemplo oficial: ReplicaSet como alvo

**Fonte:** https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: ReplicaSet  # alvo é um ReplicaSet, não um Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

Este é o único exemplo completo v1 presente na documentação oficial atual. O uso de `kind: ReplicaSet` como alvo é incomum em produção, já que Deployments gerenciam seus próprios ReplicaSets e são o alvo preferido. Escalar um ReplicaSet diretamente é válido, mas bypassa o controle de rollout do Deployment.

O threshold de 50% foi escolhido para fins didáticos: num ambiente de laboratório, é mais fácil provocar o escalonamento com carga moderada. Em produção, valores entre 60% e 80% são mais comuns, conforme confirmado pelo corpus desta pesquisa.

### Exemplo real 1: Deployment com CPU 50% (FastAPI no Kubernetes)

**Repositório:** https://github.com/4OH4/kubernetes-fastapi
**Arquivo:** https://github.com/4OH4/kubernetes-fastapi/blob/main/autoscale.yaml

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: kf-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kf-api
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

Este repositório demonstra como usar o HPA com uma API construída em FastAPI (framework Python de alta performance). O threshold de 50% replica o valor do Walkthrough oficial para facilitar a demonstração. Com `minReplicas: 1`, há risco de indisponibilidade durante falhas de pod, mas é aceitável para um ambiente de demonstração.

O comando para verificar o estado do HPA após aplicar este manifesto seria:

```bash
kubectl apply -f autoscale.yaml
kubectl get hpa kf-api-hpa
# NAME        REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS
# kf-api-hpa  Deployment/kf-api   <unknown>/50%   1         10        1
```

O valor `<unknown>` aparece quando o Metrics Server ainda não coletou dados do pod. Após alguns ciclos de 15 segundos, o valor real de CPU aparece.

### Exemplo real 2: Template de produção com CPU 70% e múltiplos namespaces

**Repositório:** https://github.com/HariSekhon/Kubernetes-configs
**Arquivo:** https://github.com/HariSekhon/Kubernetes-configs/blob/master/horizontal-pod-autoscaler.yaml

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: APP-hpa
  namespace: NAMESPACE
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: APP-deployment
  minReplicas: 3
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

Este repositório é uma coleção de configurações Kubernetes de produção mantida por Hari Sekhon. Os valores em maiúsculas (`APP`, `NAMESPACE`) são placeholders para substituição via `sed` ou ferramentas de templating como Kustomize ou Helm.

O threshold de 70% coincide com o exemplo do HPA Walkthrough para v2 GA da documentação oficial, e é a moda mais comum no corpus analisado nesta pesquisa. A faixa estreita entre `minReplicas: 3` e `maxReplicas: 5` é típica de serviços que precisam de alta disponibilidade (mínimo 3 réplicas para tolerar falhas) mas têm capacidade de cluster limitada.

### Exemplo real 3: StatefulSet como alvo (Thanos Store Gateway)

**Repositório:** https://github.com/bitnami/charts
**Fonte:** https://github.com/bitnami/charts/issues/11413

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  annotations:
    meta.helm.sh/release-name: thanos
    meta.helm.sh/release-namespace: thanos
  labels:
    app.kubernetes.io/component: storegateway
    app.kubernetes.io/instance: thanos
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: thanos
    helm.sh/chart: thanos-10.5.3
  name: thanos-storegateway
  namespace: thanos
spec:
  maxReplicas: 6
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet     # escalando um StatefulSet, não um Deployment
    name: thanos-storegateway
  targetCPUUtilizationPercentage: 80
```

Este exemplo é gerado pelo chart Helm do Thanos (sistema de armazenamento de longo prazo para Prometheus) da Bitnami. Alguns pontos relevantes:

O alvo é um `StatefulSet`, o que é conceitualmente mais complexo do que escalar um `Deployment`. StatefulSets mantêm identidade estável para cada pod (nomes como `pod-0`, `pod-1`), volumes persistentes associados a cada instância, e ordem de inicialização garantida. O HPA adiciona e remove pods seguindo as regras do StatefulSet, mas não controla quais pods são removidos na escala para baixo; o Kubernetes remove sempre a instância com o maior índice.

As annotations `meta.helm.sh/release-name` e `meta.helm.sh/release-namespace` são injetadas pelo Helm durante o `helm install` e permitem que o Helm rastreie o recurso como parte do release.

### Exemplo real 4: CRD customizado como alvo (Splunk Operator)

**Repositório:** https://github.com/splunk/splunk-operator
**Fonte:** https://github.com/splunk/splunk-operator/issues/600

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: idx-scale
  namespace: splunk-cluster-dev
spec:
  scaleTargetRef:
    apiVersion: enterprise.splunk.com/v2   # CRD customizado do Splunk Operator
    kind: IndexerCluster                    # não é um tipo nativo do Kubernetes
    name: idx-dev
  minReplicas: 2
  maxReplicas: 2
  targetCPUUtilizationPercentage: 75
```

Este exemplo demonstra uma capacidade pouco conhecida do HPA v1: ele pode escalar qualquer recurso que implemente o subrecurso `/scale`, incluindo CRDs (Custom Resource Definitions) criados por operators. O `IndexerCluster` é um CRD do Splunk Operator que representa um cluster de indexadores do Splunk.

A configuração `minReplicas: 2` igual a `maxReplicas: 2` é intencional neste caso: o operador reportou que o HPA estava tentando escalar, mas o objetivo era apenas testar se o HPA conseguia ler as métricas. Com `min == max`, o HPA nunca age; o objeto existe apenas como monitor.

---

## 2. autoscaling/v2beta1: métricas múltiplas e quatro novos tipos

Lançada no Kubernetes 1.6 (março de 2017), a v2beta1 foi uma mudança estrutural significativa. O campo único `targetCPUUtilizationPercentage` foi substituído pelo array `spec.metrics[]`, que aceita múltiplas métricas de tipos diferentes. Quando múltiplas métricas estão configuradas, o HPA calcula o número ideal de réplicas para cada uma e usa o maior resultado, garantindo que nenhuma métrica fique acima do threshold.

Os quatro tipos de métrica introduzidos são:

- `Resource`: CPU ou memória agregada de todos os containers do pod. Usa o Metrics Server nativo; não requer adapter adicional. Os campos de target são `targetAverageUtilization` (percentual do request) e `targetAverageValue` (valor absoluto por pod).
- `Pods`: métrica média por pod, exposta via Custom Metrics API. Requer um adapter externo como o prometheus-adapter. Usa `metricName` (string direta) e `targetAverageValue`.
- `Object`: métrica de um objeto Kubernetes específico, como requisições por segundo num Ingress. Usa `metricName`, `target` (referência ao objeto) e `targetValue`.
- `External`: métrica de fora do cluster, como lag de fila SQS ou mensagens do Kafka. Usa `metricName`, `metricSelector` (filtro por labels) e `targetValue` ou `targetAverageValue`.

Um ponto de atenção importante: a sintaxe dos campos de target na v2beta1 é diferente da v2beta2 e v2 GA. A v2beta1 usa campos diretos como `targetAverageUtilization` dentro de `resource:`, enquanto as versões posteriores usam a estrutura unificada `target.type`/`target.averageUtilization`. Manifestos v2beta1 precisam de ajuste ao migrar para v2.

### Exemplo oficial: multi-métricas com Resource, Pods e Object

**Fonte:** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/ (versão anterior ao Kubernetes 1.26)

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50       # percentual do requests.cpu (campo exclusivo v2beta1)
  - type: Pods
    pods:
      metricName: packets-per-second     # string direta, sem estrutura metric.name
      targetAverageValue: 1k             # média por pod; o HPA divide o total pelo nº de réplicas
  - type: Object
    object:
      metricName: requests-per-second    # nome da métrica no adapter
      target:
        apiVersion: extensions/v1beta1
        kind: Ingress
        name: main-route                 # objeto K8s a ser consultado
      targetValue: 10k                   # valor total, sem divisão por pod
```

O campo `target` dentro do tipo `Object` na v2beta1 é uma referência ao objeto Kubernetes cuja métrica será lida. Na v2beta2 e v2 GA, esse campo foi renomeado para `describedObject` para evitar ambiguidade com o campo `target` do threshold.

### Exemplo real 1: Resource com CPU (Utilization) e Memory (AverageValue)

**Repositório:** https://github.com/thlearn/k8s-prom-hpa
**Organização:** Tutorial de HPA com Prometheus

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80      # percentual de requests.cpu
  - type: Resource
    resource:
      name: memory
      targetAverageValue: 200Mi         # valor absoluto em bytes; não usa percentual
```

Este exemplo demonstra a diferença prática entre `targetAverageUtilization` e `targetAverageValue` para métricas Resource:

`targetAverageUtilization` é um percentual do campo `requests` configurado no container. Se o pod tem `requests.cpu: 500m` e o threshold é 80%, o HPA escala quando o uso médio superar 400m. A vantagem é que o threshold se ajusta automaticamente se `requests` mudar no manifesto do Deployment.

`targetAverageValue` é um valor absoluto por pod. Para memória com `200Mi`, o HPA escala quando o uso médio por pod superar 200 mebibytes. Não depende de `requests` como denominador, mas precisa ser revisado manualmente se a configuração de recursos do container mudar.

### Exemplo real 2: Tipo Pods com métrica de requisições HTTP

**Repositório:** https://github.com/stefanprodan/eks-hpa-profile
**Organização:** Stefan Prodan (mantenedor do Flux)

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: http_requests_per_second   # métrica exposta via Custom Metrics API
      targetAverageValue: 10                 # target: 10 req/s por pod em média
```

Para que este HPA funcione, é necessário ter um adapter instalado no cluster que implemente a Custom Metrics API (`custom.metrics.k8s.io`). O adapter mais comum é o `prometheus-adapter`, configurado com uma regra que mapeia a métrica `http_requests_per_second` a uma query Prometheus como `rate(http_requests_total[1m])`.

O HPA consulta o adapter a cada ciclo de 15 segundos, obtém o valor total da métrica para todos os pods, divide pelo número de réplicas atual e compara com o `targetAverageValue`. Se a média superar 10 req/s por pod, calcula quantas réplicas seriam necessárias para reduzir a média abaixo de 10.

### Exemplo real 3: Tipo Object com Istio e Prometheus

**Repositório:** https://github.com/stefanprodan/istio-hpa
**Organização:** Stefan Prodan

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
  annotations:
    # annotations específicas do adapter Prometheus para configurar a query
    metric-config.object.istio-requests-total.prometheus/per-replica: "true"
    metric-config.object.istio-requests-total.prometheus/query: |
      sum(
        rate(
          istio_requests_total{
            destination_workload="podinfo",
            destination_workload_namespace="test"
          }[1m]
        )
      )
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  metrics:
  - type: Object
    object:
      metricName: istio-requests-total        # nome da métrica no adapter
      target:
        apiVersion: v1
        kind: Pod                             # objeto K8s usado como referência
        name: podinfo
      targetValue: 10                         # valor total sem divisão por pod
```

Este exemplo usa o tipo `Object` com métricas do Istio coletadas pelo Prometheus. O Istio injeta um sidecar proxy (Envoy) em cada pod e expõe métricas de tráfego como `istio_requests_total`. O adapter Prometheus lê essa métrica e a expõe via Custom Metrics API.

As annotations `metric-config.*` são específicas do adapter usado (o `kube-metrics-adapter` da Zalando neste caso) e configuram a query Prometheus que o adapter executa. Cada adapter tem sua própria convenção de annotations para configuração.

### Exemplo real 4: Tipo External com metricSelector (AWS SQS via DataDog)

**Repositório:** https://github.com/aws-samples/aws-workshop-for-kubernetes
**Arquivo:** https://github.com/aws-samples/aws-workshop-for-kubernetes/blob/master/03-path-application-development/305-app-scaling-custom-metrics/templates/hpa-example/hpa-manifest.yaml

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginxext
spec:
  minReplicas: 1
  maxReplicas: 5
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  metrics:
  - type: External
    external:
      metricName: nginx.net.request_per_s     # nome da métrica no DataDog
      metricSelector:
        matchLabels:
          kube_container_name: nginx          # filtra por label para identificar a série
      targetAverageValue: 50                  # média por pod após divisão automática
```

O tipo `External` usa a External Metrics API (`external.metrics.k8s.io`), que também requer um adapter. O campo `metricSelector` é essencial quando múltiplas séries temporais compartilham o mesmo `metricName`, diferenciadas por labels. Sem o seletor, o adapter retornaria todas as séries com aquele nome, e o HPA somaria os valores de todas elas.

`targetAverageValue` aqui divide o valor total pelo número de réplicas antes de comparar. Isso faz o escalonamento se estabilizar naturalmente: quando o HPA adiciona réplicas, o tráfego se distribui entre mais instâncias e a média por pod cai.

### Exemplo real 5: Tipo External com Kafka consumer lag

**Repositório:** https://github.com/sunnykrGupta/k8s-hpa-custom-autoscaling-kafka-metrics
**Arquivo:** https://github.com/sunnykrGupta/k8s-hpa-custom-autoscaling-kafka-metrics/blob/master/kafka-custom-metrics-hpa.yaml

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: consumer-kafka-go-client
spec:
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: External
    external:
      metricName: custom.googleapis.com|kafka-exporter|kafka_consumergroup_lag_sum
      metricSelector:
        matchLabels:
          metric.labels.consumergroup: golang-consumer   # identifica o consumer group
      targetAverageValue: "1000"     # lag máximo por réplica antes de escalar
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: consumer-kafka-go-client
```

O padrão de escalar consumidores Kafka com base no consumer lag é muito comum em arquiteturas de processamento de eventos. A lógica é direta: se o lag total é 5.000 mensagens e o target é 1.000 por réplica, o HPA calcula que são necessárias 5 réplicas. Quando as réplicas processam as mensagens e o lag cai, o HPA inicia o scale-down após a janela de estabilização de 5 minutos.

O nome da métrica `custom.googleapis.com|kafka-exporter|kafka_consumergroup_lag_sum` é específico do Stackdriver (Google Cloud Monitoring), que usa barras verticais como separador de namespace da métrica.

### Exemplo real 6: Múltiplas métricas External com mesmo metricName (RabbitMQ)

**Repositório:** https://github.com/DataDog/datadog-agent
**Fonte:** https://github.com/DataDog/datadog-agent/issues/3872

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: test-multi
spec:
  maxReplicas: 10
  minReplicas: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: test-multi
  metrics:
  - type: External
    external:
      metricName: rabbitmq.queue.messages     # mesmo metricName nas duas entradas
      metricSelector:
        matchLabels:
          rabbitmq_queue: kemcho-low-prio-queue   # diferenciadas por label de fila
      targetValue: 500                            # valor absoluto (não dividido por pod)
  - type: External
    external:
      metricName: rabbitmq.queue.messages
      metricSelector:
        matchLabels:
          rabbitmq_queue: dpc-go-http-error
      targetValue: 500
```

Este exemplo demonstra duas features: múltiplas métricas com o mesmo `metricName` diferenciadas por `metricSelector`, e o uso de `targetValue` (valor total agregado, sem divisão por pod) em vez de `targetAverageValue`.

`targetValue` compara o valor observado diretamente com o threshold. Se a fila tem 600 mensagens e o target é 500, o HPA calcula que precisa de `ceil(current_replicas * 600/500)` réplicas. Ao contrário de `targetAverageValue`, adicionar réplicas não reduz automaticamente o valor observado, já que o número de mensagens na fila não depende de quantos consumidores existem.

---

## 3. autoscaling/v2beta2: behavior, policies e ContainerResource

Lançada no Kubernetes 1.12 (setembro de 2018), a v2beta2 trouxe as mudanças mais significativas desde a criação do HPA. Dois recursos novos foram introduzidos: `spec.behavior` para controle granular de velocidade de escalonamento, e o tipo de métrica `ContainerResource` para monitorar containers individuais em pods multi-container.

Além disso, a sintaxe dos campos de target foi reestruturada. O campo `metricName` (string direta) virou `metric.name` dentro de um `MetricIdentifier`. O campo `target` no tipo Object virou `describedObject`. Os campos `targetAverageUtilization`, `targetAverageValue` e `targetValue` foram substituídos pela estrutura unificada `target.type` com os valores `Utilization`, `AverageValue` e `Value`, usando `target.averageUtilization`, `target.averageValue` e `target.value` respectivamente.

O `spec.behavior` permite configurar scaleUp e scaleDown independentemente, cada um com:
- `stabilizationWindowSeconds`: período de observação antes de agir. Para scale-down, usa o maior número de réplicas calculado na janela (conservador).
- `selectPolicy`: quando há múltiplas policies, `Max` usa a mais permissiva, `Min` usa a mais conservadora, `Disabled` desabilita o escalonamento nessa direção.
- `policies`: array de regras, cada uma com `type: Pods` (número absoluto por ciclo) ou `type: Percent` (percentual do total atual), `value` e `periodSeconds`.

### Comportamento padrão completo (kubernetes.io)

**Fonte:** https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/

```yaml
# Este bloco mostra os defaults implícitos do HPA quando spec.behavior é omitido.
# Colocá-los explicitamente não muda o comportamento, mas documenta a intenção.
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300      # espera 5 min de métricas estáveis antes de remover
    policies:
    - type: Percent
      value: 100                         # pode remover todas as réplicas num único ciclo
      periodSeconds: 15
  scaleUp:
    stabilizationWindowSeconds: 0        # escala imediatamente, sem esperar
    policies:
    - type: Percent
      value: 100                         # pode dobrar as réplicas por ciclo
      periodSeconds: 15
    - type: Pods
      value: 4                           # ou adicionar no máximo 4 pods por ciclo
      periodSeconds: 15
    selectPolicy: Max                    # usa a policy mais permissiva entre as duas acima
```

### Limitar escala para baixo a no máximo 5 pods por minuto

```yaml
behavior:
  scaleDown:
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
    - type: Pods
      value: 5
      periodSeconds: 60
    selectPolicy: Min   # escolhe a policy que resulta no menor número de remoções
```

### Desabilitar scale-down completamente

```yaml
behavior:
  scaleDown:
    selectPolicy: Disabled   # nenhuma réplica será removida, independente das métricas
```

Isso é útil durante manutenções ou deploys onde o número de réplicas não deve cair automaticamente.

### Exemplo real 1: Behavior completo com selectPolicy: Max no scaleUp

**Repositório:** https://github.com/kubernetes/kubernetes
**Fonte:** https://github.com/kubernetes/kubernetes/issues/105579

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
  namespace: myns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 24
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization            # nova sintaxe: target.type em vez de targetAverageUtilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # 5 minutos de estabilização antes de remover
      policies:
      - type: Percent
        value: 25                       # máximo 25% das réplicas por ciclo no scale-down
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0     # scale-up imediato sem janela de espera
      policies:
      - type: Pods
        value: 24                       # pode adicionar até 24 pods por ciclo no scale-up
        periodSeconds: 15
      selectPolicy: Max                 # usa a policy mais agressiva disponível
```

A combinação de scale-down conservador (máximo 25% por ciclo com 5 minutos de estabilização) e scale-up agressivo (sem estabilização, máximo 24 pods por ciclo) é um padrão comum em produção. A assimetria é intencional: escalar para cima mal feito causa degradação imediata de serviço para usuários, portanto a rapidez é prioritária. Escalar para baixo mal feito desperdiça recursos por alguns minutos, o que é tolerável.

### Exemplo real 2: StatefulSet com selectPolicy: Disabled no scaleDown (HDFS)

**Repositório:** https://github.com/kubernetes/kubernetes
**Fonte:** https://github.com/kubernetes/kubernetes/issues/99349

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: hdfs-datanode-custom
  namespace: monitoring
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: hdfs-datanode
  minReplicas: 3
  maxReplicas: 8
  metrics:
  - type: Object
    object:
      metric:
        name: whether_add_pod          # métrica customizada que retorna 100 quando deve escalar
      describedObject:                 # v2beta2: describedObject substituiu target do tipo Object
        apiVersion: v1
        kind: Service
        name: hdfs-hpa-exporter-service
      target:
        type: Value
        value: 100                     # escala quando o valor da métrica superar 100
  behavior:
    scaleDown:
      selectPolicy: Disabled           # nunca remove DataNodes automaticamente
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 1                       # adiciona apenas 1 DataNode por vez
        periodSeconds: 120             # e espera 2 minutos entre cada adição
      selectPolicy: Min                # escolhe a policy mais conservadora
```

HDFS (Hadoop Distributed File System) é um sistema de arquivos distribuído onde remover nós de dados pode causar perda de dados ou indisponibilidade de blocos armazenados. Por isso, `selectPolicy: Disabled` no scaleDown é a configuração correta aqui: o cluster escala para cima quando há pressão de dados, mas nunca remove nós automaticamente.

O `periodSeconds: 120` com `value: 1` garante que novos DataNodes são adicionados um por vez a cada 2 minutos, dando tempo para o HDFS rebalancear os blocos antes de adicionar o próximo nó.

### Exemplo real 3: ContainerResource, monitorando contêiner individual

**Repositório:** https://github.com/kubernetes/enhancements
**Arquivo:** https://github.com/kubernetes/enhancements/blob/master/keps/sig-autoscaling/1610-container-resource-autoscaling/README.md

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: mission-critical
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mission-critical
  minReplicas: 1
  maxReplicas: 10
  metrics:
  # Monitora CPU e memória do container principal separadamente do sidecar
  - type: ContainerResource
    resource:
      name: cpu
      container: application       # container alvo dentro do pod
      target:
        type: Utilization
        averageUtilization: 30
  - type: ContainerResource
    resource:
      name: memory
      container: application
      target:
        type: Utilization
        averageUtilization: 80
  # Monitora também o proxy de autenticação como contêiner separado
  - type: ContainerResource
    resource:
      name: cpu
      container: authnz-proxy      # sidecar de autenticação/autorização
      target:
        type: Utilization
        averageUtilization: 30
  - type: ContainerResource
    resource:
      name: memory
      container: authnz-proxy
      target:
        type: Utilization
        averageUtilization: 80
  # Monitoramento mais tolerante para o agente de log (sidecar secundário)
  - type: ContainerResource
    resource:
      name: cpu
      container: log-shipping      # agente de coleta de logs
      target:
        type: Utilization
        averageUtilization: 80     # threshold mais alto pois picos de log são aceitáveis
```

O KEP-1610 (Kubernetes Enhancement Proposal) é o documento de design que introduziu o tipo `ContainerResource`. Este exemplo do KEP mostra o caso de uso central: um pod com múltiplos containers onde cada um tem thresholds de escalonamento diferentes.

Sem `ContainerResource`, o tipo `Resource` agregaria o uso de todos os containers do pod. Se o container `log-shipping` tiver um pico de CPU temporário, o HPA veria uma média alta e escalaria o Deployment inteiro, mesmo que o container `application` esteja ocioso. Com `ContainerResource`, cada container é monitorado independentemente.

O campo `container` deve corresponder exatamente ao nome de um container em `spec.containers` do pod. Se o nome não corresponder, o HPA entra em estado de erro com a mensagem `unable to get metric`.

### Exemplo real 4: Tipo External com behavior completo (FluxCD)

**Repositório:** https://github.com/fluxcd/kustomize-controller
**Fonte:** https://github.com/fluxcd/kustomize-controller/issues/494

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60     # janela de 1 minuto para scale-down (mais agressivo que o padrão)
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 3               # ciclo de 3 segundos, muito agressivo para scale-up
      - type: Pods
        value: 4
        periodSeconds: 3
      selectPolicy: Max
  metrics:
  - type: External
    external:
      metric:
        name: httpproxy_requests_active   # nova sintaxe v2beta2: metric.name em vez de metricName
        selector:
          matchLabels:
            my-label: "metric-1"         # nova sintaxe: selector em vez de metricSelector
      target:
        type: AverageValue
        value: "5"                        # máximo 5 requisições ativas por réplica
```

### Exemplo real 5: Tipo Pods com behavior de scale conservador

**Repositório:** https://github.com/kubernetes/autoscaler
**Fonte:** https://github.com/kubernetes/autoscaler/issues/4707

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: http-pod-hpa
  namespace: kaas-monitoring
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prometheus-example-app
  minReplicas: 1
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second    # nova sintaxe v2beta2: metric.name
      target:
        type: AverageValue
        averageValue: 100m                # 100 miliunidades = 0,1 req/s por pod
  behavior:
    scaleDown:
      policies:
      - type: Pods
        value: 5
        periodSeconds: 60    # remove no máximo 5 pods por minuto no scale-down
      stabilizationWindowSeconds: 0
    scaleUp:
      policies:
      - type: Pods
        value: 10
        periodSeconds: 60    # adiciona no máximo 10 pods por minuto no scale-up
      selectPolicy: Max
      stabilizationWindowSeconds: 0
```

O valor `100m` para `averageValue` usa a notação de miliunidades do Kubernetes, onde `100m` equivale a 0,1. Esta notação é a mesma usada para CPU em milicores. Para métricas de requisições por segundo, `100m` significa 0,1 req/s por pod, o que parece baixo mas pode ser adequado para demonstrações de escalonamento com carga mínima.

---

## 4. autoscaling/v2 GA: a versão estável e definitiva

A versão `autoscaling/v2` foi promovida a GA (General Availability) no Kubernetes 1.23 (dezembro de 2021). Funcionalmente é idêntica ao v2beta2; a diferença é exclusivamente o status da API: de beta passou para estável, o que significa que não sofrerá mais mudanças incompatíveis. Todo manifesto v2beta2 válido se torna v2 GA simplesmente trocando a primeira linha de `apiVersion`.

A estrutura `MetricTarget` unificada, introduzida no v2beta2, é consolidada aqui com três variantes:

- `type: Utilization` com `averageUtilization`: percentual do `requests` configurado no container. Exclusivo para métricas `Resource` e `ContainerResource`.
- `type: AverageValue` com `averageValue`: valor absoluto médio por pod. Usado para todos os tipos de métrica.
- `type: Value` com `value`: valor total agregado de todos os pods, sem divisão por réplica. Usado principalmente com `Object` e `External`.

O `MetricIdentifier` também é consolidado com dois campos: `name` (nome da métrica) e `selector` (filtro opcional por labels, equivalente ao `metricSelector` da v2beta1).

### Exemplo oficial: CPU com status completo

**Fonte:** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
status:
  observedGeneration: 1
  lastScaleTime: <some-time>
  currentReplicas: 1
  desiredReplicas: 1
  currentMetrics:
  - type: Resource
    resource:
      name: cpu
      current:
        averageUtilization: 0
        averageValue: 0
```

O bloco `status` é preenchido pelo controlador automaticamente e não deve ser incluído em manifestos aplicados pelo usuário. Aparece neste exemplo da documentação oficial apenas para ilustrar o que o `kubectl get hpa -o yaml` retorna após o HPA estar ativo.

Para inspecionar o status via linha de comando:

```bash
kubectl get hpa php-apache
# NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# php-apache   Deployment/php-apache   0%/50%    1         10        1          1m

kubectl describe hpa php-apache
# Name:                                                  php-apache
# Namespace:                                             default
# Reference:                                             Deployment/php-apache
# Metrics:                                               (current / target)
#   resource cpu on pods  (as a percentage of request):  0% (0) / 50%
# Min replicas:                                          1
# Max replicas:                                          10
# Deployment pods:                                       1 current / 1 desired
```

### Exemplo oficial: multi-métricas completo (Resource + Pods + Object + External)

**Fonte:** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second         # MetricIdentifier com campo name
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:                   # v2: describedObject (renomeado de target na v2beta1)
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: 10k
```

### Exemplo oficial: snippet External com selector

**Fonte:** https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

```yaml
- type: External
  external:
    metric:
      name: queue_messages_ready
      selector:                    # equivale ao metricSelector da v2beta1
        matchLabels:
          queue: "worker_tasks"
    target:
      type: AverageValue
      averageValue: 30
```

### Exemplo oficial: ContainerResource

**Fonte:** https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/

```yaml
type: ContainerResource
containerResource:
  name: cpu
  container: application
  target:
    type: Utilization
    averageUtilization: 60
```

### Exemplo real 1: Behavior com selectPolicy: Min/Max de produção

**Repositório:** https://github.com/abdidarmawan007/k8s-horizontal-pod-autoscale-v2
**Arquivo:** https://github.com/abdidarmawan007/k8s-horizontal-pod-autoscale-v2/blob/main/hpa-v2.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: production-test
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-test
  minReplicas: 1
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
      stabilizationWindowSeconds: 300     # 5 minutos de observação
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60                 # remove no máximo 1 pod por minuto
      - type: Percent
        value: 10
        periodSeconds: 60                 # ou 10% das réplicas por minuto
      selectPolicy: Min                   # escolhe a remoção menor entre as duas policies
    scaleUp:
      stabilizationWindowSeconds: 0       # escala imediatamente
      policies:
      - type: Percent
        value: 100                        # pode dobrar as réplicas por ciclo
        periodSeconds: 10
      - type: Pods
        value: 4
        periodSeconds: 10                 # ou adiciona 4 pods por ciclo
      selectPolicy: Max                   # escolhe a adição maior entre as duas policies
```

Esta configuração implementa o padrão de produção mais recomendado: scale-up com `selectPolicy: Max` (agressivo, para responder rápido a picos) e scale-down com `selectPolicy: Min` (conservador, para evitar remover réplicas que seriam necessárias em breve). A combinação de `type: Pods value: 1` e `type: Percent value: 10` no scale-down significa que o HPA escolhe a menor entre "1 pod" e "10% das réplicas atuais", garantindo que a remoção seja sempre graduada.

### Exemplo real 2: Tipo Pods com kube-metrics-adapter (Zalando)

**Repositório:** https://github.com/zalando-incubator/kube-metrics-adapter
**Organização:** Zalando (e-commerce alemão)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  annotations:
    # Annotations configuram o adapter para ler a métrica via JSON path de um endpoint HTTP
    metric-config.pods.requests-per-second.json-path/json-key: "$.http_server.rps"
    metric-config.pods.requests-per-second.json-path/path: /metrics
    metric-config.pods.requests-per-second.json-path/port: "9090"
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: requests-per-second
      target:
        type: AverageValue
        averageValue: 1k              # 1.000 requisições por segundo por pod
```

O `kube-metrics-adapter` da Zalando é uma alternativa ao `prometheus-adapter` que suporta múltiplas fontes de métricas via annotations no HPA. As annotations `metric-config.*` dizem ao adapter para consultar o endpoint `/metrics` na porta `9090` de cada pod e extrair o valor `http_server.rps` do JSON retornado.

### Exemplo real 3: Tipo External com Prometheus via kube-metrics-adapter

**Repositório:** https://github.com/zalando-incubator/kube-metrics-adapter

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  annotations:
    metric-config.external.processed-events-per-second.prometheus/prometheus-server: http://prometheus.my-namespace.svc
    metric-config.external.processed-events-per-second.prometheus/query: |
      scalar(sum(rate(event-service_events_count{application="event-service",processed="true"}[1m])))
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: custom-metrics-consumer
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: processed-events-per-second
        selector:
          matchLabels:
            type: prometheus       # indica ao adapter qual backend usar
      target:
        type: AverageValue
        averageValue: "10"
```

O campo `metric.selector` no `MetricIdentifier` serve aqui para dois propósitos: filtrar qual série temporal usar (quando há múltiplas), e passar informações de configuração para o adapter via labels. O label `type: prometheus` diz ao `kube-metrics-adapter` que a fonte de dados é Prometheus, e as annotations fornecem a query e o endereço do servidor.

### Exemplo real 4: CPU + Memory com behavior de resposta rápida

**Repositório:** https://github.com/Erodotos/k8s-examples
**Arquivo:** https://github.com/Erodotos/k8s-examples/blob/master/horizontal-pod-autoscaler/manifests/hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 90        # threshold alto de memória (escala só em pressão real)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50        # threshold baixo de CPU (escala proativamente)
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 10  # espera apenas 10s antes de escalar para cima
      policies:
      - type: Pods
        value: 2
        periodSeconds: 5              # adiciona até 2 pods a cada 5 segundos
    scaleDown:
      stabilizationWindowSeconds: 20
      policies:
      - type: Pods
        value: 1
        periodSeconds: 5              # remove no máximo 1 pod a cada 5 segundos
```

Os thresholds diferentes para CPU (50%) e memory (90%) refletem comportamentos distintos de cada recurso: CPU é compressível (pode ser throttled sem matar o processo) e um pico transitório de CPU justifica escalar proativamente. Memória é incompressível (se o processo precisar de mais memória do que o disponível, é morto com OOMKill), então o threshold alto de 90% significa que o HPA só age quando a pressão de memória é crítica.

### Exemplo real 5: ContainerResource para pod com sidecar de coleta de métricas

**Fonte:** https://github.com/MartinHeinz/metrics-on-kind

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: resource-consumer-v2-container
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: resource-consumer
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: ContainerResource
    containerResource:
      name: cpu
      container: resource-consumer    # monitora apenas o container principal
      target:
        type: Utilization
        averageUtilization: 75        # ignora CPU do sidecar de coleta de métricas
```

### Exemplo real 6: Behavior ultra-conservador de produção (MuleSoft)

**Repositório:** https://github.com/mulesoft/docs-hosting
**Organização:** MuleSoft (Salesforce)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
  namespace: app-namespace
spec:
  behavior:
    scaleDown:
      policies:
      - periodSeconds: 15
        type: Percent
        value: 100
      selectPolicy: Max
      stabilizationWindowSeconds: 1800    # 30 minutos de observação antes de remover réplicas
    scaleUp:
      policies:
      - periodSeconds: 180               # ciclo de 3 minutos para scale-up
        type: Percent
        value: 100
      selectPolicy: Max
      stabilizationWindowSeconds: 0
  maxReplicas: 3
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 70
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
```

O `stabilizationWindowSeconds: 1800` (30 minutos) no scaleDown é um padrão ultra-conservador. O HPA precisa observar 30 minutos consecutivos de métricas abaixo do threshold antes de remover qualquer réplica. Para aplicações com padrões de carga cíclicos (picos de horário comercial seguidos de quedas), isso evita que o HPA remova réplicas no final de um pico para adicioná-las novamente no início do próximo.

O `periodSeconds: 180` no scaleUp significa que o HPA aguarda 3 minutos entre tentativas de adicionar réplicas, em vez dos 15 segundos do padrão. Isso é adequado para aplicações com tempo de inicialização longo, onde adicionar réplicas rapidamente em sequência sobrecarregaria o cluster sem benefício imediato, já que as novas réplicas ainda não estariam prontas para receber tráfego.

---

## HPAs em projetos conhecidos do ecossistema Kubernetes

A maioria dos projetos modernos migrou para `autoscaling/v2`, e muitos usam detecção dinâmica de versão em Helm templates para manter compatibilidade com clusters mais antigos.

**Istio (istiod)**
- Repositório: https://github.com/istio/istio/blob/master/manifests/charts/istio-control/istio-discovery/templates/autoscale.yaml
- Usa `autoscaling/v2` com CPU `averageUtilization: 80`, faixa de 1 a 5 réplicas para o plano de controle.

**Argo CD (repo-server)**
- Repositório: https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/templates/argocd-repo-server/hpa.yaml
- `autoscaling/v2` com suporte a CPU, memory, métricas customizadas e `spec.behavior` configuráveis via Helm values.

**Argo Rollouts**
- Repositório: https://github.com/argoproj/argo-rollouts/blob/master/docs/features/hpa-support.md
- Demonstra HPA v2 com `scaleTargetRef` apontando para `kind: Rollout` (CRD customizado do Argo), usando `type: AverageValue` para memory:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: demo-hpa
spec:
  minReplicas: 1
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout                          # CRD do Argo Rollouts, não um Deployment nativo
    name: rollout-hpa-example
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 16Mi                 # valor absoluto em mebibytes por pod
```

**ingress-nginx**
- Repositório: https://github.com/kubernetes/ingress-nginx/blob/main/charts/ingress-nginx/templates/controller-hpa.yaml
- Usa detecção dinâmica via Helm: `ternary "autoscaling/v2" "autoscaling/v2beta2" (semverCompare ">=1.23-0" .Capabilities.KubeVersion.GitVersion)` para suportar clusters com Kubernetes abaixo de 1.23.

**Grafana Loki**
- Repositório: https://github.com/grafana/loki/blob/main/production/helm/loki/templates/read/hpa.yaml
- HPAs para múltiplos componentes (read, write, backend, distributor, gateway), com fallback de `autoscaling/v2` para `autoscaling/v2beta1` em clusters antigos.

**kubernetes-sigs/lws (LeaderWorkerSet)**
- Repositório: https://github.com/kubernetes-sigs/lws/blob/main/docs/examples/sample/horizontal-pod-autoscaler.yaml
- HPA v2 escalando o CRD `LeaderWorkerSet`, usado para workloads de IA e ML que precisam de grupos de pods coordenados.

---

## Evolução dos campos entre versões: referência rápida

| Campo / Feature | v1 | v2beta1 | v2beta2 | v2 GA |
|---|---|---|---|---|
| `targetCPUUtilizationPercentage` | disponível | removido | removido | removido |
| `spec.metrics[]` | indisponível | introduzido | mantido | mantido |
| `metricName` (string direta) | n/a | disponível | substituído por `metric.name` | substituído por `metric.name` |
| `targetAverageUtilization` (campo direto) | n/a | disponível | substituído por `target.type: Utilization` | substituído por `target.type: Utilization` |
| `targetAverageValue` (campo direto) | n/a | disponível | substituído por `target.type: AverageValue` | substituído por `target.type: AverageValue` |
| `targetValue` (campo direto) | n/a | disponível | substituído por `target.type: Value` | substituído por `target.type: Value` |
| `spec.behavior` | indisponível | indisponível | introduzido | mantido |
| `stabilizationWindowSeconds` | n/a | n/a | introduzido | mantido |
| `selectPolicy` (Max / Min / Disabled) | n/a | n/a | introduzido | mantido |
| `policies` (Pods / Percent) | n/a | n/a | introduzido | mantido |
| `ContainerResource` | n/a | n/a | introduzido | mantido |
| `describedObject` (tipo Object) | n/a | campo `target` | renomeado para `describedObject` | mantido |
| `metric.selector` | n/a | campo `metricSelector` | renomeado para `metric.selector` | mantido |
| Status estável (GA) | disponível | removido no K8s 1.22 | removido no K8s 1.26 | disponível desde K8s 1.23 |

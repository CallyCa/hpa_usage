# Especificação Completa da API HorizontalPodAutoscaler autoscaling/v2

**Fonte**: Red Hat OpenShift Container Platform 4.17 - Autoscale APIs

## Estrutura Geral

### HorizontalPodAutoscaler

**Descrição**: Configuração para um horizontal pod autoscaler, que gerencia automaticamente a contagem de réplicas de qualquer recurso que implementa o subrecurso scale baseado nas métricas especificadas.

**Tipo**: `object`

**API Group**: `autoscaling`

**Versão**: `v2`

---

## Especificação (.spec)

### HorizontalPodAutoscalerSpec

**Descrição**: Descreve a funcionalidade desejada do HorizontalPodAutoscaler.

**Tipo**: `object`

**Campos obrigatórios**:

- `scaleTargetRef`
- `maxReplicas`

### Campos Principais do Spec

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **behavior** | object | Configura o comportamento de escalonamento do alvo em ambas as direções (Up e Down) |
| **maxReplicas** | integer | Limite superior para o número de réplicas. Não pode ser menor que minReplicas |
| **metrics** | array | Especificações de métricas para calcular a contagem de réplicas desejada. Se não definido, padrão é 80% de utilização média de CPU |
| **minReplicas** | integer | Limite inferior para o número de réplicas. Padrão: 1 pod. Pode ser 0 se HPAScaleToZero estiver habilitado |
| **scaleTargetRef** | object | Referência ao recurso alvo (Deployment, StatefulSet, etc.) |

---

## Comportamento de Escalonamento (.spec.behavior)

### HorizontalPodAutoscalerBehavior

**Descrição**: Configura o comportamento de escalonamento do alvo em ambas as direções (scaleUp e scaleDown).

**Tipo**: `object`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **scaleDown** | object | Regras de escalonamento para redução de réplicas |
| **scaleUp** | object | Regras de escalonamento para aumento de réplicas |

### HPAScalingRules

**Descrição**: Configura o comportamento de escalonamento para uma direção. Aplicadas após calcular DesiredReplicas das métricas.

**Tipo**: `object`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **policies** | array | Lista de políticas de escalonamento potenciais. Pelo menos uma política deve ser especificada |
| **selectPolicy** | string | Especifica qual política usar. Padrão: Max |
| **stabilizationWindowSeconds** | integer | Número de segundos para considerar recomendações passadas. Padrão: 0 para scale up, 300 para scale down. Máximo: 3600 (1 hora) |

### HPAScalingPolicy

**Descrição**: Política única que deve ser verdadeira para um intervalo passado especificado.

**Tipo**: `object`

**Campos obrigatórios**:

- `type`
- `value`
- `periodSeconds`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **periodSeconds** | integer | Janela de tempo para a qual a política deve ser verdadeira. Deve ser > 0 e ≤ 1800 (30 min) |
| **type** | string | Tipo de política de escalonamento (Percent, Pods) |
| **value** | integer | Quantidade de mudança permitida pela política. Deve ser > 0 |

---

## Métricas (.spec.metrics[])

### MetricSpec

**Descrição**: Especifica como escalar baseado em uma única métrica (apenas `type` e um outro campo correspondente devem ser definidos por vez).

**Tipo**: `object`

**Campo obrigatório**: `type`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **containerResource** | object | Métrica de recurso de container (CPU, memória) |
| **external** | object | Métrica não associada a nenhum objeto Kubernetes |
| **object** | object | Métrica descrevendo um objeto Kubernetes |
| **pods** | object | Métrica descrevendo cada pod no alvo de escala |
| **resource** | object | Métrica de recurso conhecida pelo Kubernetes (CPU, memória) |
| **type** | string | Tipo de fonte de métrica: "ContainerResource", "External", "Object", "Pods" ou "Resource" |

---

## Tipos de Métricas Detalhados

### 1. ContainerResourceMetricSource

**Descrição**: Escala baseado em métrica de recurso de container (CPU ou memória) especificada em requests e limits.

**Tipo**: `object`

**Campos obrigatórios**:

- `name`
- `target`
- `container`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **container** | string | Nome do container nos pods do alvo de escala |
| **name** | string | Nome do recurso (cpu, memory) |
| **target** | object | Alvo de métrica (valor, média, utilização) |

### 2. ResourceMetricSource

**Descrição**: Escala baseado em métrica de recurso conhecida pelo Kubernetes (CPU ou memória) descrevendo cada pod.

**Tipo**: `object`

**Campos obrigatórios**:

- `name`
- `target`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **name** | string | Nome do recurso (cpu, memory) |
| **target** | object | Alvo de métrica (valor, média, utilização) |

### 3. PodsMetricSource

**Descrição**: Escala baseado em métrica descrevendo cada pod no alvo de escala atual (ex: transações-processadas-por-segundo).

**Tipo**: `object`

**Campos obrigatórios**:

- `metric`
- `target`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **metric** | object | Identificador de métrica (nome e seletor opcional) |
| **target** | object | Alvo de métrica (valor, média) |

### 4. ObjectMetricSource

**Descrição**: Escala baseado em métrica descrevendo um objeto Kubernetes (ex: hits-per-second em um objeto Ingress).

**Tipo**: `object`

**Campos obrigatórios**:

- `describedObject`
- `target`
- `metric`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **describedObject** | object | Referência ao objeto descrito |
| **metric** | object | Identificador de métrica (nome e seletor opcional) |
| **target** | object | Alvo de métrica (valor, média) |

### 5. ExternalMetricSource

**Descrição**: Escala baseado em métrica não associada a nenhum objeto Kubernetes (ex: tamanho de fila em serviço de mensageria na nuvem, QPS de loadbalancer fora do cluster).

**Tipo**: `object`

**Campos obrigatórios**:

- `metric`
- `target`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **metric** | object | Identificador de métrica (nome e seletor opcional) |
| **target** | object | Alvo de métrica (valor, média) |

---

## MetricTarget

**Descrição**: Define o valor alvo, valor médio ou utilização média de uma métrica específica.

**Tipo**: `object`

**Campo obrigatório**: `type`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **averageUtilization** | integer | Valor alvo da média da métrica de recurso como % do valor solicitado. Válido apenas para Resource metric |
| **averageValue** | Quantity | Valor alvo da média da métrica entre todos os pods relevantes (como quantidade) |
| **type** | string | Tipo de métrica: "Utilization", "Value" ou "AverageValue" |
| **value** | Quantity | Valor alvo da métrica (como quantidade) |

---

## MetricIdentifier

**Descrição**: Define o nome e opcionalmente seletor para uma métrica.

**Tipo**: `object`

**Campo obrigatório**: `name`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **name** | string | Nome da métrica |
| **selector** | LabelSelector | Seletor de labels Kubernetes para escopo mais específico de métricas |

---

## Scale Target Reference (.spec.scaleTargetRef)

### CrossVersionObjectReference

**Descrição**: Contém informação suficiente para identificar o recurso referenciado.

**Tipo**: `object`

**Campos obrigatórios**:

- `kind`
- `name`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **apiVersion** | string | Versão da API do recurso alvo |
| **kind** | string | Tipo do recurso (Deployment, StatefulSet, etc.) |
| **name** | string | Nome do recurso |

---

## Status (.status)

### HorizontalPodAutoscalerStatus

**Descrição**: Descreve o status atual de um horizontal pod autoscaler.

**Tipo**: `object`

**Campo obrigatório**: `desiredReplicas`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **conditions** | array | Conjunto de condições necessárias para este autoscaler escalar seu alvo |
| **currentMetrics** | array | Último estado lido das métricas usadas por este autoscaler |
| **currentReplicas** | integer | Número atual de réplicas de pods gerenciados |
| **desiredReplicas** | integer | Número desejado de réplicas de pods gerenciados |
| **lastScaleTime** | Time | Última vez que o HPA escalou o número de pods |
| **observedGeneration** | integer | Geração mais recente observada por este autoscaler |

### HorizontalPodAutoscalerCondition

**Descrição**: Descreve o estado de um HorizontalPodAutoscaler em um certo ponto.

**Tipo**: `object`

**Campos obrigatórios**:

- `type`
- `status`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| **lastTransitionTime** | Time | Última vez que a condição transitou de um status para outro |
| **message** | string | Explicação legível contendo detalhes sobre a transição |
| **reason** | string | Razão para a última transição da condição |
| **status** | string | Status da condição (True, False, Unknown) |
| **type** | string | Tipo da condição |

---

## Resumo das Diferenças entre Versões

### autoscaling/v1

- **Suporte limitado**: apenas CPU utilization
- **Campo único**: `targetCPUUtilizationPercentage`
- **Sem behavior configuration**
- **Sem suporte a múltiplas métricas**

### autoscaling/v2beta1

- **Múltiplas métricas**: suporte inicial
- **Métricas customizadas**: introduzidas
- **Sem behavior field**
- **Deprecated em Kubernetes 1.19**

### autoscaling/v2beta2

- **Behavior field**: introduzido (desde Kubernetes 1.18)
- **Scaling policies**: controle de velocidade de escalonamento
- **Stabilization window**: prevenção de flapping
- **Container resource metrics**: suporte inicial
- **Deprecated em Kubernetes 1.23**

### autoscaling/v2 (GA - Stable)

- **Versão estável**: desde Kubernetes 1.23
- **Todas as features de v2beta2**: mantidas
- **API estável**: sem breaking changes esperadas
- **Recomendada para produção**

---

## Campos Disponíveis por Versão

| Campo/Feature | v1 | v2beta1 | v2beta2 | v2 |
|---------------|----|---------|---------|----|
| targetCPUUtilizationPercentage | ✓ | ✓ | ✓ | ✓ |
| metrics (array) | ✗ | ✓ | ✓ | ✓ |
| resource metrics | ✗ | ✓ | ✓ | ✓ |
| pods metrics | ✗ | ✓ | ✓ | ✓ |
| object metrics | ✗ | ✓ | ✓ | ✓ |
| external metrics | ✗ | ✗ | ✓ | ✓ |
| containerResource metrics | ✗ | ✗ | ✓ | ✓ |
| behavior field | ✗ | ✗ | ✓ | ✓ |
| scaleUp policies | ✗ | ✗ | ✓ | ✓ |
| scaleDown policies | ✗ | ✗ | ✓ | ✓ |
| stabilizationWindowSeconds | ✗ | ✗ | ✓ | ✓ |
| selectPolicy | ✗ | ✗ | ✓ | ✓ |

## Especificação por Versão

As diferentes versões da API `autoscaling` oferecem conjuntos de campos e comportamentos distintos. A seguir, detalhamos as especificações de cada versão, incluindo os campos obrigatórios, tipos de métricas suportados e particularidades da semântica de cada versão.

### autoscaling/v1

**Spec**

- **scaleTargetRef (object, obrigatório)**: referência ao objeto a ser escalado (Deployment, StatefulSet ou ReplicaSet). Contém `apiVersion` (opcional), `kind` (obrigatório) e `name` (obrigatório).
- **maxReplicas (integer, obrigatório)**: número máximo de réplicas permitidas.
- **minReplicas (integer, opcional)**: número mínimo de réplicas, padrão 1.
- **targetCPUUtilizationPercentage (integer, obrigatório)**: única métrica disponível neste formato; define a utilização média de CPU desejada em % sobre o resource request. O HPA calcula a média de CPU dos pods e ajusta as réplicas com base na razão entre utilização atual e desejada.

**Status**

O status da versão v1 inclui `currentReplicas` e `desiredReplicas` e registra o valor de `observedGeneration`, `lastScaleTime` e `currentCPUUtilizationPercentage`. A ausência de suporte a múltiplas métricas significa que `currentMetrics` e `conditions` não são retornados.

### autoscaling/v2beta1

**Spec**

- **scaleTargetRef (object, obrigatório)**: referência ao objeto de escala com campos `apiVersion` (opcional), `kind` e `name` obrigatórios.
- **maxReplicas (integer, obrigatório)**: limite superior de réplicas.
- **minReplicas (integer, opcional)**: limite inferior, padrão 1.
- **metrics (array, opcional)**: lista de `MetricSpec` definindo como escalar. O campo `metrics` pode ser omitido; se ausente, a API usa um target de 80 % de utilização de CPU.

**MetricSpec**

Cada objeto `MetricSpec` contém:

- **type (string, obrigatório)**: tipo de métrica. Os valores permitidos são `Resource`, `Pods`, `Object` e, dependendo do ambiente e dos adaptadores instalados, `External` e `ContainerResource`. A especificação oficial v2beta1 não contemplava external metrics; alguns provedores de métrica adicionaram suporte experimental.
- **resource (object, opcional)**: descreve uma métrica de recurso (CPU ou memória). Contém `name` e `target`. `name` é obrigatório e deve ser `cpu` ou `memory`; `target` possui `type` (`Utilization`, `Value`, `AverageValue`) e, conforme o tipo, campos `averageUtilization`, `value` ou `averageValue`.
- **pods (object, opcional)**: descreve uma métrica por pod. Contém `metric.name` e um seletor opcional, bem como `target` com `type` e `averageValue`.
- **object (object, opcional)**: descreve uma métrica associada a um objeto Kubernetes, incluindo `describedObject` (CrossVersionObjectReference), `metric.name` e `target` (valor ou média).
- **containerResource (object, opcional)**: embora a estrutura exista no código, o suporte a métricas de container só se tornou disponível na v2beta2 mediante feature gate. Requer `name`, `container` e `target`.
- **external (object, opcional)**: especifica métricas de serviços externos. Este tipo foi introduzido de forma estável apenas na v2beta2; em v2beta1 o campo não é processado.

**Status**

A v2beta1 adiciona os campos `currentReplicas` e `desiredReplicas`, assim como `currentMetrics` (estado atual das métricas) e `conditions` (condições do HPA). Entretanto, não há `behavior`, logo o escalonamento utiliza a mesma política que v1: scale up imediato e scale down conservativo.

### autoscaling/v2beta2

**Spec**

- **scaleTargetRef (object, obrigatório)**: igual às versões anteriores.
- **maxReplicas (integer, obrigatório)** e **minReplicas (integer, opcional)**: definem limites de escala; `minReplicas` padrão 1 e pode ser 0 se a feature **HPAScaleToZero** estiver habilitada.
- **metrics (array, opcional)**: lista de `MetricSpec` suportando todos os tipos de métrica (`Resource`, `Pods`, `Object`, `External` e `ContainerResource`).
- **behavior (object, opcional)**: introduzido nesta versão. Permite controlar como o HPA reage às recomendações de réplicas, definindo regras separadas para `scaleUp` e `scaleDown`. Cada regra (tipo `HPAScalingRules`) aceita uma lista de políticas (`HPAScalingPolicy`) com `type` (`Pods` ou `Percent`), `value` e `periodSeconds`, e parâmetros de estabilização como `selectPolicy` (Máx/Min/Disabled) e `stabilizationWindowSeconds`.

**MetricSpec**

- **Resource**: métricas de CPU ou memória por pod, com campos `name` e `target` obrigatórios.
- **Pods**: métricas customizadas por pod; contém `metric.name` e `target.averageValue`.
- **Object**: métricas relacionadas a objetos Kubernetes; requer `describedObject`, `metric.name` e `target`.
- **External**: métricas de sistemas externos; requer `metric.name` e `target` e permite seletor de rótulos.
- **ContainerResource**: métricas de recurso de um container específico; requer `name` (cpu ou memory), `container` e `target`.

**Status**

Além de `currentReplicas` e `desiredReplicas`, o status inclui:

- `currentMetrics`: lista das leituras atuais de cada métrica configurada.
- `conditions`: estados e mensagens sobre a saúde do autoscaler.
- `lastScaleTime`: marcação temporal da última escala.
- `observedGeneration`: número de geração mais recente observado.

### autoscaling/v2 (GA)

Esta é a versão estável da API e reúne todas as funcionalidades de v2beta2. As definições já apresentadas no início deste documento são válidas para v2. A semântica de cada campo é idêntica à v2beta2, porém sem as advertências de depreciação. O status do HPA passa a incluir `currentMetrics` e `conditions` de forma estável.

---

## Exemplos de Uso

### Exemplo 1: HPA v1 (CPU básico)

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: simple-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Exemplo 2: HPA v2 (Múltiplas métricas)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: advanced-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
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
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

### Exemplo 3: HPA v2 com Behavior

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: behavior-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 4
        periodSeconds: 60
      selectPolicy: Min
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
```

---

## Notas Importantes para Lembrar

1. **Métricas padrão**: Se nenhuma métrica for especificada, o padrão é 80% de utilização média de CPU

2. **Múltiplas métricas**: Quando múltiplas métricas são especificadas, o HPA calcula réplicas desejadas para cada métrica e escolhe o **maior valor**

3. **Behavior field**: Disponível apenas em v2beta2 e v2, permite controle fino sobre velocidade de escalonamento

4. **Container metrics**: Requer feature gate HPAContainerMetrics habilitado

5. **Scale to zero**: Requer feature gate HPAScaleToZero habilitado e pelo menos uma métrica Object ou External configurada

6. **Stabilization window**: Previne flapping ao considerar histórico de recomendações antes de escalar

# Documentação Oficial Kubernetes - HPA

**Fonte**: <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>

## Conceitos Fundamentais

### O que é HorizontalPodAutoscaler

O **HorizontalPodAutoscaler** (HPA) atualiza automaticamente um recurso de workload (como Deployment ou StatefulSet), com o objetivo de escalar automaticamente a carga de trabalho para atender à demanda.

- **Escalonamento horizontal**: resposta ao aumento de carga através da implantação de mais Pods
- **Diferente de escalonamento vertical**: que atribui mais recursos (memória, CPU) aos Pods já em execução

### Implementação

O HPA é implementado como:

1. **Recurso de API do Kubernetes**
2. **Controlador** que roda no control plane

O controlador ajusta periodicamente a escala desejada do alvo (ex: Deployment) para corresponder às métricas observadas como:

- Utilização média de CPU
- Utilização média de memória
- Qualquer métrica customizada especificada

### Funcionamento

O HPA funciona como um **loop de controle** que roda intermitentemente (não é um processo contínuo):

- **Intervalo padrão**: 15 segundos (configurável via `--horizontal-pod-autoscaler-sync-period`)

Em cada período, o controller manager:

1. Consulta a utilização de recursos contra as métricas especificadas
2. Encontra o recurso alvo definido por `scaleTargetRef`
3. Seleciona os pods baseado nos labels do `.spec.selector`
4. Obtém métricas de:
   - **Resource metrics API** (para métricas de recursos por pod)
   - **Custom metrics API** (para todas as outras métricas)

## Tipos de Métricas

### 1. Per‑pod Resource Metrics (CPU, Memory)

- Controller busca métricas da resource metrics API para cada Pod
- Se valor de utilização alvo é definido: calcula como % do resource request
- Se valor bruto é definido: usa valores brutos diretamente
- Calcula média de utilização/valor bruto entre todos os Pods alvo
- Produz uma razão para escalar o número de réplicas desejadas

### 2. Per‑pod Custom Metrics

- Funciona similarmente às resource metrics
- Trabalha com valores brutos, não valores de utilização

### 3. Object Metrics e External Metrics

- Uma única métrica é buscada, descrevendo o objeto em questão
- Métrica é comparada ao valor alvo para produzir uma razão
- Na API `autoscaling/v2`, este valor pode opcionalmente ser dividido pelo número de Pods antes da comparação

## Algoritmo de Escalonamento

### Fórmula Básica

```
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

**Exemplo**:

- Se valor atual = 200m e valor desejado = 100m → dobra réplicas (200m/100m = 2)
- Se valor atual = 50m e valor desejado = 100m → reduz pela metade (50m/100m = 0.5)

### Tolerância

- Control plane pula ação de escalonamento se a razão está próxima de 1.0
- **Tolerância padrão**: 0.1 (configurável)

### Tratamento de Pods

**Pods ignorados**:

- Pods com deletion timestamp (sendo removidos)
- Pods com falha

**Pods com métricas faltantes**:

- São separados para uso posterior
- Usados para ajustar o valor final de escalonamento

**Pods "not yet ready"** (ainda não prontos):

- Pods ainda inicializando ou não saudáveis
- Pods cuja métrica mais recente foi antes de ficarem ready
- Considerados "not yet ready" se transitaram para ready dentro de uma janela configurável

**Parâmetros de configuração**:

- `--horizontal-pod-autoscaler-initial-readiness-delay`: padrão 30 segundos
- `--horizontal-pod-autoscaler-cpu-initialization-period`: padrão 5 minutos

### Cálculo Conservador

Para métricas faltantes, o control plane recalcula a média de forma conservadora:

- **Scale down**: assume que pods consomem 100 % do valor desejado
- **Scale up**: assume que pods consomem 0 % do valor desejado

Isso **atenua a magnitude** de qualquer escalonamento potencial.

## Prontidão de Pods e Métricas de Autoscaling

### Comportamentos‑chave para Prontidão de Pods

Durante o escalonamento, o HPA considera o estado de prontidão dos pods para evitar oscilações:

1. **Pods em inicialização**: não são considerados para cálculo de métricas
2. **Pods não prontos**: podem causar atrasos no escalonamento
3. **Janela de estabilização**: previne escalonamento prematuro

### Boas Práticas

- Definir readiness probes apropriadas
- Configurar delays de inicialização adequados
- Considerar tempo de warm‑up da aplicação

## Objeto de API

O HorizontalPodAutoscaler é um recurso na API do Kubernetes com:

- **API Group**: `autoscaling`
- **Versões**: v1, v2beta1, v2beta2, v2
- **Subrecurso `scale`**: interface para definir dinamicamente o número de réplicas

## Estabilidade da Escala do Workload

### Autoscaling Durante Rolling Update

Durante atualizações rolling:

- HPA continua monitorando métricas
- Pode escalar durante o processo de atualização
- Importante considerar impacto no processo de rollout

## Suporte para Métricas

### Resource Metrics

- CPU e memória
- Fornecidas pelo Metrics Server
- API: `metrics.k8s.io`

### Container Resource Metrics

- Métricas específicas por container
- Permite controle mais granular

### Custom Metrics

- Métricas personalizadas da aplicação
- API: `custom.metrics.k8s.io`
- Requer adaptador de métricas customizadas

### External Metrics

- Métricas de sistemas externos
- API: `external.metrics.k8s.io`
- Exemplo: tamanho de fila em serviço de mensageria

## Escalonamento em Múltiplas Métricas

O HPA pode escalar baseado em múltiplas métricas simultaneamente:

- Calcula réplicas desejadas para cada métrica
- Escolhe o **maior valor** entre todas as métricas
- Garante que todas as métricas sejam satisfeitas

## Comportamento de Escalonamento Configurável

### Políticas de Escalonamento

Permite configurar:

- **Velocidade de scale up**: quão rápido adicionar réplicas
- **Velocidade de scale down**: quão rápido remover réplicas
- **Janela de estabilização**: período para observar antes de escalar

### Janela de Estabilização

- Previne "flapping" (oscilação rápida)
- Considera histórico de métricas
- Padrão: 5 minutos para scale down

### Tolerância

- Define quão próximo da métrica alvo deve estar antes de escalar
- Previne escalonamento desnecessário por pequenas variações
- Padrão: 10 % (0.1)

### Comportamento Padrão

Se não especificado:

- Scale up: imediato quando necessário
- Scale down: gradual com janela de estabilização

### Exemplos de Configuração

**Mudar janela de estabilização de scale down**:

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
```

**Limitar taxa de scale down**:

```yaml
behavior:
  scaleDown:
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
```

**Desabilitar scale down**:

```yaml
behavior:
  scaleDown:
    selectPolicy: Disabled
```

## Suporte para HorizontalPodAutoscaler no kubectl

### Comandos kubectl

- `kubectl autoscale`: criar HPA
- `kubectl get hpa`: listar HPAs
- `kubectl describe hpa`: detalhes do HPA
- `kubectl delete hpa`: remover HPA

### Desativação Implícita do Modo de Manutenção

- HPA pode ser temporariamente desabilitado
- Útil durante manutenções programadas

## Migração de Deployments e StatefulSets

Ao migrar para HPA:

1. Remover configurações manuais de réplicas
2. Definir resource requests apropriados
3. Configurar métricas de escalonamento
4. Testar comportamento em ambiente de staging

## Detalhes Avançados de Operação do HPA

A implementação do HorizontalPodAutoscaler envolve mais do que ler uma métrica e calcular um novo número de réplicas. O controlador roda um **loop de controle** de 15 em 15 segundos (valor padrão, mas configurável) que realiza as seguintes etapas:

1. **Descoberta do alvo de escala**: via `scaleTargetRef`, o HPA identifica o objeto com subrecurso `scale` (como um Deployment ou StatefulSet). O autoscaler consulta o objeto para saber o número de réplicas atuais e suas labels.
2. **Coleta de métricas**: o HPA se comunica com a *metrics pipeline* do cluster. Para métricas de CPU e memória, a API `metrics.k8s.io` expõe valores agregados coletados pelo *metrics server*. Métricas personalizadas são fornecidas via a API `custom.metrics.k8s.io`, implementada por adaptadores (como Prometheus Adapter); métricas externas são obtidas da API `external.metrics.k8s.io`, fornecida por adaptadores como o KEDA. Quando métricas de containers estão habilitadas (feature gate **HPAContainerMetrics**), a API retorna métricas de recursos de cada container nos pods. Esse ecossistema implica dependências: sem metrics server ou adaptador, o HPA não recebe valores.
3. **Cálculo de métricas**: para métricas de recurso, o HPA calcula a **utilização média** comparando a métrica atual com o *resource request* de cada pod. Para métricas de valor (como QPS), calcula a média ou soma conforme o tipo de métrica. O HPA utiliza o tipo e os campos do `MetricTarget` para interpretar corretamente: `Utilization` para porcentagens (apenas CPU/memória), `Value` para valores absolutos e `AverageValue` para médias sobre todos os pods. Quando múltiplas métricas são especificadas, o HPA determina o número de réplicas recomendado por cada métrica e escolhe o **maior** resultado.
4. **Aplicação de políticas de escalonamento**: a partir do Kubernetes 1.18 (v2beta2), o campo `.spec.behavior` permite ajustar a velocidade de escala. É possível definir políticas de escala para *up* e *down*, limitando a porcentagem ou o número absoluto de réplicas adicionadas/removidas por período, e especificar janelas de estabilização para evitar *flapping*. Nas versões anteriores, a lógica padrão aplicava imediatamente o aumento de réplicas e reduzia de forma conservadora com uma janela de 5 minutos.

### Delays e Janela de Brownout

Embora o HPA reavalie a escala a cada 15 segundos, o processo completo entre uma explosão de tráfego e a adição de novos pods pode levar vários minutos. Um estudo de operação real detalha um *"brownout window"*: (1) as métricas do metrics server são coletadas a cada 15–30 s; (2) o HPA executa seu loop e calcula a escala após outro intervalo; (3) novos pods são agendados e levam dezenas de segundos para iniciar; (4) o provisionamento de nós adicionais pode ser necessário se a cluster autoscaler estiver habilitada. Além disso, as métricas de CPU costumam ser médias sobre janelas de 60 s ou mais, suavizando picos e podendo atrasar a percepção de um spike. Como mitigação, equipes frequentemente mantêm **pods adicionais de buffer** (N+2) ou usam jobs de *pre‑warming* com CronJobs antes de picos previstos.

### Métricas de Container

Desde o Kubernetes 1.30, a feature **HPAContainerMetrics** tornou estável o suporte a métricas de recursos específicos de containers. Enquanto métricas de recursos tradicionais se baseiam no consumo agregado de todos os containers de cada pod, as métricas de container permitem escalar o deployment considerando somente um container crítico (por exemplo, `app` em um sidecar). O HPA exige que `container` e `name` sejam definidos no `MetricSpec` para esse tipo de métrica. Deve‑se atualizar o HPA se o nome do container mudar.

### Dependências e Limitações

O HPA funciona apenas com workloads que implementam o subrecurso `scale` (Deployments, StatefulSets, ReplicaSets); ele **não** escala DaemonSets, Jobs ou Pods isolados. Como escalonador de **pods**, ele não reserva capacidade no cluster, nem aumenta o número de nós: isso é função da **cluster autoscaler**, que deve estar configurada para evitar que a escala de pods falhe por falta de recursos. O HPA também não opera simultaneamente com o **Vertical Pod Autoscaler (VPA)** se ambos estiverem configurados para alterar o mesmo recurso; quando possível, use o VPA no modo *recommendation* ou combine HPA para CPU com VPA para memória. Além disso, o HPA assume workloads stateless; aplicações stateful podem exigir persistência e estratégias de escalonamento diferentes.

### Boas Práticas

- **Defina limites de recursos** (`requests` e `limits`) de forma realista para que a métrica de utilização de CPU/memória reflita adequadamente a carga.
- **Escolha métricas adequadas**: use métricas customizadas para métricas de negócio (como QPS, latência) quando CPU/memória não correlacionarem com carga.
- **Teste em staging**: monitore o comportamento em cenários de pico para ajustar tolerância (`--horizontal-pod-autoscaler-tolerance`) e janelas de estabilização.
- **Combine com cronjobs**: para cargas previsíveis, crie *pre‑warm* com CronJobs que ajustem a `minReplicas` antes dos picos.
- **Monitoramento**: acompanhe métricas de autoscaling (`currentReplicas`, `desiredReplicas`, `lastScaleTime`) e de condições para depurar comportamentos inesperados.
- **Evite flapping**: utilize `behavior.scaleDown.stabilizationWindowSeconds` e políticas para limitar reduções abruptas. Aumentos de réplicas geralmente não precisam de janelas de estabilização, mas podem ter limites para evitar explosões de pods.

## Diferenças Entre Versões do HorizontalPodAutoscaler

O HPA evoluiu ao longo de diversas versões da API `autoscaling`. Abaixo está um resumo das diferenças:

### autoscaling/v1

- **Métrica suportada**: somente CPU. O campo `targetCPUUtilizationPercentage` especifica a utilização média desejada. O controlador calcula a média de CPU de todos os pods via `metrics.k8s.io` e ajusta as réplicas com base na fórmula `desiredReplicas = ceil(currentReplicas × currentCPUUtil / targetCPUUtil)`. Min e max replicas são definidos em `.spec.minReplicas` e `.spec.maxReplicas`.
- **Sem métricas múltiplas**: apenas um alvo de CPU pode ser configurado. Métricas customizadas, object, external ou container não são suportadas.
- **Sem comportamento configurável**: não há campo `behavior` para personalizar a política de escalonamento. O HPA executa scale up imediatamente e scale down com janela de estabilização de 5 minutos.

### autoscaling/v2beta1

- **Suporte inicial a múltiplas métricas**: a estrutura `MetricSpec` aceita tipos `Resource`, `Pods`, `Object`, `External` (quando adaptadores estão presentes) e `ContainerResource`. Apenas o campo `type` é obrigatório; exatamente um campo correspondente deve ser preenchido. Métricas customizadas podem ser especificadas por meio de `metric.name` e um seletor.
- **Campos Spec**: `scaleTargetRef` e `maxReplicas` são obrigatórios; `minReplicas` e `metrics` são opcionais (quando omitidos, o HPA utiliza 1 réplica e 80 % de CPU).
- **Sem campo behavior**: não há suporte a políticas de escalonamento; o HPA utiliza o comportamento padrão de scale up imediato e scale down gradativo.
- **Depreciação**: v2beta1 foi descontinuada em Kubernetes 1.19.

### autoscaling/v2beta2

- **Adição de `behavior`**: novo campo `behavior` que contém `scaleUp` e `scaleDown`, ambos instâncias de `HPAScalingRules`. Cada regra permite definir uma lista de políticas (`type` `Pods` ou `Percent`, `value` e `periodSeconds`), o algoritmo de seleção (`selectPolicy` como `Max`, `Min` ou `Disabled`) e a janela de estabilização (`stabilizationWindowSeconds`).
- **Suporte a containerResource e external metrics**: as métricas de container e métricas externas passaram a ser definidas explicitamente no spec. Os tipos de `MetricSpec` incluem `ContainerResource` com campos `name`, `container` e `target` obrigatórios e `External` com `metric` e `target` obrigatórios.
- **Campos Spec**: `minReplicas` e `metrics` permanecem opcionais; `scaleTargetRef` e `maxReplicas` continuam obrigatórios.
- **Status mais detalhado**: `HorizontalPodAutoscalerStatus` contém `currentMetrics`, `conditions`, `lastScaleTime` e `observedGeneration` além de `currentReplicas` e `desiredReplicas`.
- **Depreciação**: v2beta2 tornou‑se obsoleta com a estabilização de v2 em Kubernetes 1.23.

### autoscaling/v2 (GA)

- **API estável**: consolidou todos os recursos introduzidos nas betas (métricas customizadas, external, object e container, behavior com políticas). Sua sintaxe de spec e status é compatível e não sofrerá breaking changes.
- **Min replicas opcional**: `minReplicas` pode ser omitido; o HPA assume 1 por padrão, ou 0 se o feature gate **HPAScaleToZero** estiver ativo e houver pelo menos uma métrica de objeto ou externa. `metrics` também é opcional; se ausente, a utilização de CPU a 80 % é usada como alvo padrão.
- **Comportamento configurável**: `behavior` e `HPAScalingRules` permanecem, permitindo configurar políticas e estabilização.
- **Métricas de container**: suporte estável; é possível escalar com base em recursos de containers individuais usando `containerResource`.
- **Status completo**: campos `currentMetrics`, `conditions`, `lastScaleTime`, `observedGeneration`, `currentReplicas` e `desiredReplicas` fornecem visibilidade das leituras e decisões de escalonamento.
- **Recomendações**: utilize v2 para workloads em produção e evite versões beta antigas.

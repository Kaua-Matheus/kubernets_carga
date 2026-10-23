# 1. Cenário
Computador usado:
    - OS: Ubuntu 24.04.4 LTS (64-bit)
    - Intel® Xeon® E5-2620 v3 × 12 núcleos
    - 16,0 GiB Ram
    - 560,1 GB
Recursos do cluster:
Resource           Requests     Limits
  --------           --------     ------
  cpu                1450m (12%)  600m (5%)
  memory             484Mi (3%)   348Mi (2%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)

Carga:
    - Taxa: 8
    - Paradas: 9
    - Duração: 60s + (10s aquecimento) + (10s wait): ~120

Taxa escolhida e por quê:
    Os testes foram executados com uma taxa fixada em 8. A ideia era extressar um pouco mais o hardware e agravar os resultados de carga. 

    Resultados Observados:

    Capacidade de Atendimento por Instância: Os testes com 1 Pod isolado demonstraram uma vazão estável de aproximadamente 1 req/s (sucessos_por_s), evidenciando o gargalo imposto pelas restrições de CPU e memória alocadas no cluster local.

    Ganho de Escalabilidade: A introdução incremental de réplicas demonstrou ganho de desempenho proporcional e previsível. Com 2 Pods, a vazão média subiu para ~1,85 req/s, e com 3 Pods atingiu o patamar máximo de ~2,5 req/s.

    Saturação e Degradação: Como a taxa injetada (8 req/s) superou a capacidade agregada do cluster de três instâncias (~2,5 req/s), o sistema operou propositalmente em regime de saturação. Isso se refletiu em uma taxa de falhas decrescente conforme a escalabilidade horizontal mitigava parte da pressão (de 87,7% com 1 Pod para ~68% com 3 Pods), mantendo zero iterações descartadas (dropped_iterations_medicao = 0), o que valida a confiabilidade e integridade da coleta de métricas.
Duração: 
    60s + (10s aquecimento) + (10s wait): ~120s
Recursos por Pod:
    Limits:
      cpu:     500m
      memory:  128Mi

# 2. Evidências
Mediana por quantidade de Pods:

| Pods | Sucessos/s | Falhas | p95 `latencia_ok` | Iterações descartadas |
|---|---|---|---|---|
| 1 | **0,983 req/s** | **87,73%** | 1,08 s | 0 |
| 2 | **1,816 req/s** | **77,33%** | 1,22 s | 0 |
| 3 | **2,500 req/s** | **68,75%** | 1,02 s | 0 |

Uso de CPU Observado: Com o limite de 500m configurado por Pod, as instâncias operaram sob forte estresse, registrando picos entre 300m e 400m de CPU. A oscilação abaixo do teto absoluto de 500m deve-se aos atrasos de amostragem do metrics-server, servindo como indício claro de estrangulamento computacional.

# 3. Explicação

Por que a CPU observada ficou abaixo de 500m (não cravada no limite): Os atrasos e o intervalo de amostragem do metrics-server do Kubernetes falham em capturar picos instantâneos e de curtíssima duração, além de o processamento do grafo alternar entre gargalos de CPU e momentos de espera de E/S.

Por que mais replicas ajudaram (mas pouco em termos de erradicar as falhas): A hipótese é que, embora 3 Pods tripliquem a capacidade de processamento individual (passando de ~1 req/s para ~2,5 req/s), a taxa de chegada escolhida (8 req/s) continuou sendo drasticamente superior à capacidade agregada do cluster. Por isso, o sistema continuou em severa saturação, mantendo taxas de erro altas (~68%), mas mostrando que a adição de réplicas aliviou parcialmente a pressão comparado ao cenário de 1 Pod.

Por que houve desbalanceamento de requisições entre os Pods: A hipótese é que a ausência de session affinity e o comportamento padrão do balanceamento de carga nativo resultaram em acúmulo de conexões em instâncias mais antigas ou ativas há mais tempo (como o pod t2c9s), enquanto pods efêmeros receberam menor volume relativo nas janelas de medição.

# 4. Decisão
Para atender de forma estável à taxa de 8 requisições por segundo considerando a capacidade unitária de um Pod (aprox. 1 req/s com os recursos atuais), o ideal seria escalar horizontalmente para 8 a 9 réplicas de Pods, mantendo a cota atual de 500m de CPU e 128Mi de memória por Pod.
Custo em CPU Reservada:
    Se considerarmos 9 réplicas com limits/requests de 500m (0.5 núcleo) cada:
    $9 \times 0.5\text{ núcleos} =$ 4,5 núcleos de CPU reservados (ou 4500m).

Como o cenário com 3 Pods ainda operou sob forte saturação (~68% de falhas), a próxima investigação deve seguir quatro frentes estruturadas:

1. Escala Horizontal Avançada (Horizontal Pod Autoscaling): Validar se o cluster suporta subir o cluster para a faixa de 8 a 10 réplicas simultâneas para absorver totalmente a taxa de 8 req/s sem perda de estabilidade.

2. Ajuste de Recursos Verticais (Vertical Scaling): Testar o aumento do limite de CPU por Pod (ex: de 500m para 1000m ou 2000m) para verificar se a vazão individual sobe proporcionalmente, reduzindo a necessidade de dezenas de réplicas.

3. Profiling de Código e Gargalos da Aplicação: Investigar o código do serviço de processamento de grafos para identificar se há travamentos (locks), processamento síncrono bloqueante ou gargalos no algoritmo que limitam o desempenho por núcleo.

4. Inspeção de Rede e Timeouts: Analisar o comportamento do balanceador de carga, Ingress Controllers e políticas de timeout HTTP para garantir que as falhas não sejam causadas por rejeição prematura de conexões na borda.
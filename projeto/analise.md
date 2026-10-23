# Análise – RotaExpress: 1, 2 e 3 Pods

## 1. Cenário

**Computador:** Ubuntu 24.04.4 LTS, Intel Xeon E5-2620 v3 (12 núcleos), 16 GiB de RAM.

**Cluster:** Minikube (perfil `ops-lab`, driver Docker), criado com `--cpus=3 --memory=3072`. Um único nó.
Na saída de `kubectl describe node ops-lab` (com 1 réplica no ar) os requests somavam 1450m de CPU (12%) e 484Mi de memória (3%).
Esses percentuais sugerem que o nó reporta a CPU do computador (cerca de 12 núcleos), e que o teto de 3 CPUs vem do Docker, não do scheduler. Isso é uma hipótese, não foi verificada.

**Recursos por Pod da aplicação:** requests = limits = 500m de CPU; memória 64Mi (request) e 128Mi (limit).
**Gerador de carga (k6):** request 250m e limit 500m de CPU, no mesmo nó.

**Carga (idêntica em todas as execuções):**
- Taxa: 8 pedidos/s (`constant-arrival-rate`), 9 paradas por rota.
- Aquecimento de 10 s (fora das métricas), intervalo de 10 s e medição de 60 s. O cenário completo dura cerca de 80 a 90 s.
- Cada pedido abre uma conexão nova (`noConnectionReuse`), então o Service escolhe o Pod a cada pedido.

**Por que taxa 8:** o piloto com 1 Pod e taxa 8 teve 86,9% de falhas e 63 rotas concluídas, sem iterações descartadas, ou seja, o k6 gerou toda a demanda pedida.
Decidi manter 8 para ter uma demanda bem acima da capacidade de 1 Pod e observar o quanto cada configuração aliviava a pressão.
Consequência assumida: com essa taxa nenhuma configuração poderia atingir as metas, porque a capacidade medida de 3 Pods (cerca de 2,5 rotas/s) é menos de um terço da demanda.

**Execuções:** piloto, 9 medições (3 repetições por configuração, ordem alternada A/B/C conforme a tabela 7.5 do roteiro) e 1 teste extra (`limite`, passo 8.1), todos com logs em `resultados/`.

## 2. Evidências

Mediana das 3 repetições de cada configuração:

| Pods | rotas_ok (3 repetições) | Sucessos/s (mediana) | Falhas (mediana) | p95 `latencia_ok` (mediana) | Iterações descartadas | CPU reservada |
|---|---|---|---|---|---|---|
| 1 | 59 / 59 / 64 | **0,983** | **87,73%** | **1,05 s** | 0 | 500m |
| 2 | 109 / 105 / 114 | **1,817** | **77,33%** | **1,09 s** | 0 | 1000m |
| 3 | 155 / 150 / 149 | **2,500** | **68,75%** | **1,01 s** | 0 | 1500m |

Sucessos por segundo = `rotas_ok / 60`. O p95 das medianas vem dos valores de cada rodada (1 Pod: 1,05 / 1,08 / 0,946 s; 2 Pods: 1,22 / 1,09 / 0,959 s; 3 Pods: 1,02 / 1,01 / 0,995 s).

Vazão por Pod: 0,98 (1 Pod), 0,91 (2 Pods) e 0,83 (3 Pods) rotas/s. Vazão por CPU reservada: 1,97, 1,82 e 1,67 rotas/s.

As faixas de `rotas_ok` das três configurações não se sobrepõem (59–64, 105–114, 149–155), então a diferença entre elas é maior que a variação entre repetições.

**Metas do k6 (falhas < 1%, p95 < 800 ms, 0 descartadas):** em todas as execuções o `dropped_iterations` foi 0, mas as metas de falhas e de p95 falharam. Todos os Jobs terminaram como `Failed` com código 99 do k6, o que é um resultado válido (a aplicação não cumpriu as metas), e não uma falha de execução.

**CPU observada (`kubectl top pods`):**
- 1 Pod: 297m a 344m.
- 2 Pods: 299m a 363m.
- 3 Pods: 355m a 403m.
- Duas leituras de `1m` (piloto e `2pods-b`) foram feitas com o Pod sem carga e foram descartadas.

Nenhuma leitura chegou perto de 500m. O `kubectl top` mostra médias de uma janela de alguns segundos, então ele pode esconder picos.

**Distribuição entre Pods:** não consegui obter uma distribuição confiável. Os valores registrados não são diferenças entre antes e depois do teste: por exemplo, o Pod `t2c9s` cresce de 174 para 593 em testes idênticos, e em 1 Pod o valor registrado (201) é maior que o total aceito. Por isso a análise não tira conclusões sobre desbalanceamento. A coluna do CSV guarda os valores brutos, marcados como tal.

**Teste extra (`limite`):** com 6 réplicas solicitadas, taxa 8, foram 210 rotas concluídas (3,5/s) e 56,34% de falhas, com p95 de 1,05 s. Não entra nas medianas. Não registrei quantos Pods ficaram `Ready` durante esse teste, então o resultado só mostra que subir o número de réplicas pedidas não multiplicou a vazão na mesma proporção.

## 3. Explicação

**Observado:**
- A vazão cresceu com o número de Pods (0,98 → 1,82 → 2,50 rotas/s) e as falhas caíram (87,7% → 77,3% → 68,8%). Cada Pod calcula uma rota por vez, então mais Pods permitem mais cálculos simultâneos.
- O ganho por Pod diminuiu (0,98 → 0,91 → 0,83 rotas/s). O ganho não foi proporcional.
- O p95 dos sucessos ficou em cerca de 1 s em todas as configurações, sem melhora com mais Pods. A média da latência dos sucessos também ficou em 0,90 a 0,96 s.
- Mesmo com 3 Pods, 68,75% dos pedidos receberam falha (503 da aplicação).

**Hipóteses (não verificadas):**
1. *Tempo de cálculo e p95.* O p95 não melhora com mais Pods porque ele mede o tempo de um cálculo, e esse tempo depende da CPU de cada Pod (500m) e da velocidade do processador do computador, não da quantidade de Pods. Isso sugere que a meta de p95 < 800 ms não seria atingida com 500m por Pod, em nenhuma quantidade de réplicas. [Opcional: comparar com o `calculo_ms` medido no passo 2.3, sem limite de CPU.]
2. *Falhas com 3 Pods.* A demanda (8/s) é mais de três vezes a capacidade medida (2,5/s), então a maior parte dos pedidos encontra todos os Pods ocupados e recebe 503 por desenho da aplicação (um cálculo por vez, sem fila). Além disso, o Service escolhe o Pod sem saber se ele está ocupado, então até sobre capacidade ainda haveria alguns 503.
3. *Rendimento por Pod decrescente.* A disputa por CPU no mesmo nó entre os Pods, o k6 e os componentes do cluster pode explicar a queda de 0,98 para 0,83 rotas/s por Pod.
4. *CPU abaixo de 500m.* Pode ser média do metrics-server sobre a janela, com o Pod ocioso em parte dela. Também poderia indicar que o limite não era o gargalo; isso não foi testado.

## 4. Decisão

**Entre as configurações medidas, adotaria 3 Pods:** foi a que mais concluiu rotas (2,5/s) e a que menos falhou (68,75%). O custo é 1500m de CPU reservada (requests = limits) e 384Mi de memória no limite, equivalente à metade dos 3 CPUs declarados ao Minikube.

**Mas nenhuma configuração atingiu as metas.** Para a taxa de 8 pedidos/s, 3 Pods não é suficiente: atende cerca de 31% da demanda.

**Extrapolação (não medida):** com ~0,83 a 0,98 rotas/s por Pod, atender 8 pedidos/s exigiria na ordem de 9 a 10 Pods, ou seja, 4,5 a 5 CPUs reservadas. Isso não cabe no cluster atual de 3 CPUs e ainda teria falhas, porque o balanceamento não conhece a ocupação dos Pods.

**Próximas investigações, em ordem:**
1. Repetir a série com uma taxa próxima da capacidade medida (por exemplo, 2 a 3 pedidos/s) para ver se 3 Pods passam nas metas e a partir de quantos Pods a falha cai abaixo de 1%.
2. Testar 1 Pod com limite de CPU maior (por exemplo, 1000m) para ver se o p95 cai abaixo de 800 ms. Acima de 1 núcleo não deve ajudar, porque cada Pod faz um cálculo por vez.
3. Refazer a medição da distribuição entre Pods com contadores antes e depois de cada teste.
4. Se a demanda real for alta, mudar o desenho da aplicação (fila, ou balanceador que conheça a ocupação) em vez de apenas aumentar réplicas. Só depois disso, avaliar um HPA, usando a capacidade por réplica medida aqui.

**Limite da conclusão:** 3 Pods em um único nó não protegem contra a queda do nó. O laboratório mede atendimento sob carga, não recuperação de desastre.

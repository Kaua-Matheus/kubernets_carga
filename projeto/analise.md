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

**Execuções:** piloto, 9 medições (3 repetições por configuração, ordem alternada A/B/C conforme a tabela 7.5 do roteiro) e 1 teste extra (`limite`, passo 8.1), todos com logs em `resultados/`. Depois da série, fiz ainda um teste adicional com o cluster recriado com 5 CPUs e 12 Pods (descrito na seção 2); ele não entra nas medianas.

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

**Teste adicional com 12 Pods (cluster com 5 CPUs, mesma taxa 8):** pretendia verificar se a falha persistia com capacidade de sobra.
- Falhas, `rotas_ok` e p95: 205 e 1050ms.
- CPU por Pod: 100m a 200m, menor que a de 3 ou 6 Pods (300m a 400m).
- Distribuição: a maioria dos Pods atendeu entre 20 e 70 pedidos; alguns Pods que já existiam antes do teste apareceram com 140 a 200. Esses valores são os contadores acumulados da aplicação, e não foram confirmados como diferença entre antes e depois do teste. Os Pods antigos carregam o histórico dos testes anteriores, então esse número sozinho não prova desbalanceamento.

## 3. Explicação

**Observado:**
- A vazão cresceu com o número de Pods (0,98 → 1,82 → 2,50 rotas/s) e as falhas caíram (87,7% → 77,3% → 68,8%). Cada Pod calcula uma rota por vez, então mais Pods permitem mais cálculos simultâneos.
- O ganho por Pod diminuiu (0,98 → 0,91 → 0,83 rotas/s). O ganho não foi proporcional.
- O p95 dos sucessos ficou em cerca de 1 s em todas as configurações, sem melhora com mais Pods. A média da latência dos sucessos também ficou em 0,90 a 0,96 s.
- Mesmo com 3 Pods, 68,75% dos pedidos receberam falha (503 da aplicação).

**Modelo que explica as falhas (hipótese que se ajusta aos dados, não medição direta):**

Cada Pod funciona como um guichê que atende um pedido por vez, durante cerca de 0,93 s (média da `latencia_ok` nos logs), e recusa na hora quem chega enquanto está ocupado, sem fila. Como cada pedido abre uma conexão nova, o Service (kube-proxy) sorteia o Pod sem saber se ele está ocupado. Nesse caso, a fração de pedidos recusados por Pod é ρ/(1+ρ), com ρ = (taxa que chega ao Pod) × 0,93 s. Para comparar, calculei também a falha de um balanceador ideal, que sempre manda o pedido a um Pod livre (fórmula de Erlang-B). Taxa de entrada: 8 pedidos/s.

| Pods | Falha com sorteio do Service | Falha com balanceador ideal | Falha medida (mediana) |
|---|---|---|---|
| 1 | 88% | 88% | 87,7% |
| 2 | 79% | 77% | 77,3% |
| 3 | 71% | 66% | 68,8% |
| 6 | 55% | 36% | 56,34 |
| 12 | 38% | 4% | 37,7% |

Com 1, 2 e 3 Pods, as falhas medidas ficaram a poucos pontos do modelo, entre os dois cenários e mais perto do sorteio. O modelo usa apenas a taxa, o tempo de cálculo e o fato de que o Pod aceita um pedido por vez. Se ele está certo, a falha alta não vem de defeito do Kubernetes nem de falta de CPU, e sim do desenho "um cálculo por vez, sem fila, com Pod sorteado".

**Hipóteses:**
1. *Tempo de cálculo e p95.* O p95 não melhora com mais Pods porque ele mede o tempo de um cálculo, e esse tempo depende da CPU de cada Pod (500m) e da velocidade do processador do computador, não da quantidade de Pods. Isso sugere que a meta de p95 < 800 ms não seria atingida com 500m por Pod, em nenhuma quantidade de réplicas. Não foi testado com outro limite de CPU.
2. *Ganho decrescente por Pod.* O modelo acima já explica a queda de 0,98 para 0,83 rotas/s por Pod apenas com o sorteio: Pods livres ficam ociosos enquanto outros recusam. A disputa por CPU entre Pods, k6 e componentes do cluster pode contribuir, mas não é necessária para explicar os números, e não foi isolada.
3. *CPU abaixo de 500m.* O mesmo modelo prevê o uso de CPU como (fração do tempo ocupado) × 500m: cerca de 440m com 1 Pod, 356m com 3 Pods e 190m com 12 Pods. Observei 297m a 344m (1 Pod), 355m a 403m (3 Pods) e 100m a 200m (12 Pods). Com 3 e 12 Pods os valores batem. Com 1 Pod a leitura ficou abaixo da previsão, o que pode ser efeito da média do metrics-server sobre uma janela de alguns segundos. A queda de CPU com 12 Pods é consequência de dividir a mesma carga por mais Pods, e não de um problema novo.
4. *Distribuição entre Pods no teste com 12 Pods.* Pedidos sorteados geram alguma diferença entre Pods mesmo sem defeito. Para ~560 pedidos em 12 Pods, o esperado só por sorteio seria algo como 20 a 40 atendidos por Pod; a faixa vista (20 a 70) é um pouco mais larga, o que pode vir de Pods que ficaram `Ready` em momentos diferentes. Os Pods com 140 a 200 provavelmente refletem o contador acumulado de testes anteriores. Essas explicações não foram verificadas, porque faltam as leituras de antes e depois do teste.

## 4. Decisão

**Entre as configurações medidas, adotaria 3 Pods:** foi a que mais concluiu rotas (2,5/s) e a que menos falhou (68,75%). O custo é 1500m de CPU reservada (requests = limits) e 384Mi de memória no limite, equivalente à metade dos 3 CPUs declarados ao Minikube. O retorno é decrescente: o 2º Pod acrescentou cerca de 0,83 rota/s e o 3º, cerca de 0,68 rota/s, pelo mesmo custo de 500m cada.

**Mas nenhuma configuração atingiu as metas.** Para a taxa de 8 pedidos/s, 3 Pods não é suficiente: atende cerca de 31% da demanda.

**Aumentar réplicas, sozinho, não resolve esta aplicação.** Pelo modelo da seção 3, com o sorteio de Pods do Service a falha continua alta mesmo com capacidade de sobra: cerca de 38% com 12 Pods, 43% com 10, e seriam necessárias centenas de Pods para ficar abaixo de 1%. Com um balanceador ideal, bastariam cerca de 15 Pods (7,5 CPUs reservadas), mas isso é cálculo do modelo; entre as quantidades acima, só 12 Pods foram testados. No teste com 12 Pods e 5 CPUs a CPU por Pod ficou em 100m a 200m, ou seja, sobrava CPU; a falha medida nesse teste é 37,7% e deve ser comparada com os 38% previstos.

**Próximas investigações, em ordem:**
1. Medir a falha com 12 Pods e outras quantidades (4, 6, 8) para checar o modelo da seção 3 com dados, e refazer a medição da distribuição com contadores antes e depois de cada teste.
2. Repetir a série com uma taxa próxima da capacidade medida (por exemplo, 2 a 3 pedidos/s) para ver se 3 Pods passam nas metas e a partir de quantos Pods a falha cai abaixo de 1%.
3. Testar 1 Pod com limite de CPU maior (por exemplo, 1000m) para ver se o p95 cai abaixo de 800 ms. Acima de 1 núcleo não deve ajudar, porque cada Pod faz um cálculo por vez.
4. Mudar o desenho do atendimento em vez de apenas aumentar réplicas: uma fila na aplicação, ou um balanceador que respeite a ocupação (por exemplo, menos conexões ativas). Só depois disso, avaliar um HPA, usando a capacidade por réplica medida aqui.

**Limite da conclusão:** 3 Pods em um único nó não protegem contra a queda do nó. O laboratório mede atendimento sob carga, não recuperação de desastre.

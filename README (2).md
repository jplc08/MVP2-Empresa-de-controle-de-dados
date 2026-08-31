# MVP — Risco, Custo e Tempo para Implantação de Nova Unidade de Armazenamento em Nuvem

**Aluno:** João Pedro Lima de Carvalho — **Matrícula:** 231013402
**Disciplina:** Sistemas de Suporte à Decisão
**Tipo de problema:** Regressão supervisionada (custo e tempo) + Classificação supervisionada (risco) + Simulação de Monte Carlo (reserva de contingência)

---

## Problema

Uma empresa de armazenamento de dados em nuvem decidiu implantar uma nova unidade — um data
center regional. A decisão sobre **quanto orçar, quando prometer a entrega e quanto reservar
para imprevistos** é tomada antes da primeira obra começar, e é nesse momento que o erro é
mais caro de corrigir.

Orçar pela estimativa determinística equivale a aceitar cerca de 50% de chance de estourar o
orçamento. Contratar a data otimista do cronograma equivale a prometer o que raramente se
cumpre. Este MVP constrói um sistema de apoio à decisão que responde três perguntas antes da
aprovação do capital:

1. **Custo** — quanto o custo final tende a se desviar do orçamento aprovado?
2. **Tempo** — quantos dias de atraso esperar em relação ao cronograma planejado?
3. **Risco** — qual a probabilidade de um pacote de trabalho estourar custo ou prazo?

E, a partir das três, a pergunta que a diretoria realmente faz: **quanto provisionar de
reserva de contingência?**

**Critério de sucesso, definido antes da modelagem:** os três modelos precisam superar com
folga um baseline ingênuo (média / classe majoritária). A expectativa declarada previamente
era R² entre 0,20 e 0,45 no estouro de custo — em projetos reais, R² acima de 0,90 é sinal
de vazamento, não de mérito.

---

## Resultado

| Alvo | Modelo final | Métrica em teste | Baseline |
|---|---|---|---|
| **Custo** (regressão) | Random Forest | R² **0,329** / MAE **0,329** no fator | R² −0,001 (dummy) |
| **Tempo** (regressão) | Random Forest | R² **0,270** / MAE **70,0 dias** | R² −0,001 (dummy) |
| **Risco** (classificação) | Random Forest | Acurácia **0,745** / AUC **0,807** | Acurácia 0,619 (dummy) |

*Avaliados em conjunto de teste (20%), separado por `GroupShuffleSplit` agrupado por projeto —
nenhum projeto aparece em treino e teste simultaneamente. Amostra: 1.874 fases de 1.101
projetos de capital reais, extraídas de uma base bruta de 12.136 registros.*

**Saída de decisão da simulação de Monte Carlo** (10.000 iterações sobre uma EAP de data
center de R$ 77,0 mi, calibrada com as fases de obra reais da base):

| Indicador | Valor |
|---|---|
| Orçamento base determinístico | R$ 77,0 mi |
| Custo P50 | R$ 73,4 mi |
| **Custo P80 — recomendado** | **R$ 88,1 mi** |
| Custo P90 | R$ 100,5 mi |
| **Reserva de contingência (P80)** | **R$ 11,1 mi — 14,4% sobre a base** |
| Buffer de cronograma (P80) | 673 dias |

Os três modelos superam o baseline com folga, e o R² na faixa de 0,27 a 0,33 é exatamente o
esperado para dados reais de projeto. Esse poder explicativo modesto **é em si a principal
conclusão de negócio**: o desempenho de um projeto não é previsível o bastante para justificar
uma estimativa pontual, o que torna a reserva de contingência baseada em distribuição a
saída correta — e não um número único de orçamento.

---

## Metodologia de seleção da base — trilha de auditoria

A instrução pedia uma base para o problema de risco, custo e tempo de implantação de uma nova
unidade. Duas famílias de candidatas foram avaliadas e descartadas antes da base adotada:

| Candidata | Motivo do descarte |
|---|---|
| Bases sintéticas de *project management* (Kaggle) | Alvo gerado por fórmula. Produzem R² próximo de 1 com a importância concentrada em uma única variável — o modelo recupera o gerador, não descobre um padrão. Não sustentam conclusão de negócio. |
| Bases de sensores de obra (IoT sintético) | Sem par planejado/realizado. Impossível derivar fator de estouro. |
| **SCA Capital Project Schedules and Budgets** | **Adotada.** Planejado contra realizado de obras efetivamente executadas, procedência governamental, dicionário de dados oficial. |

**Base adotada:** [SCA Capital Project Schedules and Budgets](https://www.kaggle.com/datasets/new-york-city/nyc-capital-project-schedules-and-budgets) — New York City School Construction Authority,
publicada no NYC Open Data e espelhada no Kaggle. Cada linha é uma **fase** de um projeto de
capital real (construção e reforma de escolas públicas de Nova York) executado entre 2003 e 2023.

Colunas essenciais, conforme o dicionário de dados oficial que acompanha a base:

| Coluna | Papel |
|---|---|
| `Project Budget Amount` | Orçamento base aprovado por fase → linha de base de custo |
| `Final Estimate of Actual Costs Through End of Phase Amount` | Estimativa final na conclusão → custo realizado |
| `Project Phase Planned End Date` | Data **originalmente** programada → linha de base de prazo |
| `Project Phase Actual End Date` | Data em que a fase realmente terminou → prazo realizado |
| `Total Phase Actual Spending Amount` | Despesa acumulada realizada (**excluída como preditor** — ver adiante) |
| `DSF Number(s)` | Identificador do projeto no Plano de Cinco Anos |

**Premissa de transferibilidade:** um data center é, na maior parte do orçamento e do
cronograma, obra civil e infraestrutura elétrica — terreno, licenciamento, obra, subestação,
climatização. Projetos de capital públicos são o proxy real e auditável mais próximo
disponível. A premissa é declarada e retomada nas Limitações.

---

## Tratamento dos dados

### Códigos-sentinela

A base usa marcadores textuais dentro de colunas numéricas e de data: `PNS` (fase não
iniciada, 6.634 linhas), `DIIR`, `IEH`, `DOES` e `FTK` (orçamento gerido por outro programa).
Não são erro de dados, são ausência estruturada — mas passariam como categoria numérica se
não fossem convertidos para nulo antes da modelagem.

### Funil de limpeza

| Etapa | Linhas restantes |
|---|---|
| Base bruta | 12.136 |
| Fase concluída (`Status = Complete`) | 2.924 |
| Orçamento e custo final válidos | 2.463 |
| Orçamento base > 0 | 2.180 |
| **Orçamento base ≥ US$ 10.000** | **1.874** |
| Datas de início, plano e realizado presentes | 1.874 |
| Coerência término ≥ início | 1.874 |

Aproveitamento de 15,4%, correspondendo a 1.101 projetos distintos.

O piso de US$ 10.000 é a decisão de limpeza mais consequente do trabalho: sem ele,
orçamentos de três dígitos produzem fatores de estouro de até **41×** que dominam qualquer
métrica de erro sem representar risco real de projeto.

### Alvos construídos

| Alvo | Definição | Distribuição |
|---|---|---|
| `fator_custo` | custo final ÷ orçamento base | mediana **0,820**, assimetria **3,54**, 33,6% acima de 1,0 |
| `atraso_dias` | término real − término planejado | mediana **0 dias**, assimetria **4,55**, 44,7% com atraso |
| `alto_risco` | estouro > 10% **ou** atraso > 90 dias | 38,8% da amostra |

**Resultado contraintuitivo e central:** a mediana do fator de custo fica **abaixo de 1** — a
maioria das fases fecha abaixo do orçamento base, porque a SCA orça de forma conservadora.
O risco não está no centro da distribuição, está na **cauda**: a assimetria de 3,54 revela
poucas fases que estouram várias vezes o previsto. Média e mediana escondem exatamente o que
a gestão precisa enxergar, e é por isso que a saída final é um percentil, não uma média.

### Estrutura de painel

A base não é uma linha por projeto: são 12.136 fases de 4.800+ projetos. Fases do mesmo
projeto compartilham escola, distrito, tipo de obra e equipe. Tratá-las como observações
independentes vaza informação entre treino e teste. **Toda a validação usa `GroupKFold`
agrupado por `DSF Number(s)`**, e a divisão de teste confirma zero projetos em comum.

---

## Auditoria de vazamento

O teste que decide a validade de todo o resto. A coluna `Total Phase Actual Spending Amount`
registra a despesa acumulada realizada — informação que só existe **depois** de a fase
terminar. No momento em que a diretoria precisa decidir, esse número ainda não existe.

Mesmo modelo, mesma validação, com e sem essa variável:

| Cenário | R² (log) | MAE do fator |
|---|---|---|
| **COM** `gasto_real` (vazamento) | **0,791** | 0,140 |
| **SEM** `gasto_real` (correto) | **0,139** | 0,382 |

**Inflação de R² causada pelo vazamento: +0,652.**

O cenário com vazamento produz um número muito melhor e completamente inútil: prevê o custo
final a partir do quanto já foi gasto. Todo o notebook usa exclusivamente variáveis
disponíveis no momento do planejamento.

---

## Viés de composição — o achado central

Para existir data real de término, a fase precisa estar concluída. Mas a taxa de conclusão
**não é uniforme entre as fases**:

| Fase | % concluída | % em andamento | Atraso mediano (amostra concluída) |
|---|---|---|---|
| Scope | 67,8% | 6,0% | −2 dias |
| Design | 53,8% | 9,8% | −0,5 dia |
| **Construction** | **12,7%** | 36,4% | **+50 dias** |
| **CM, F&E** | **3,7%** | 28,4% | **+106 dias** |
| Purch & Install | 2,0% | 12,8% | — |

Fases iniciais terminam rápido e entram na amostra. Fases de obra demoram anos e ainda estão
em andamento na data-corte. O resultado é **censura à direita**: as fases mais longas e mais
atrasadas são justamente as que ficam de fora.

Das fases em andamento com prazo definido, **99,9% já passaram da data planejada** na
data-corte, com atraso mínimo já acumulado de mediana superior a mil dias — contra atraso
mediano de 0 dias na amostra concluída.

**Consequência para a decisão:** as fases excluídas não são um recorte aleatório, são as mais
atrasadas. Qualquer estimativa geral derivada apenas das fases concluídas é **otimista por
construção**. Como um data center é dominado por Construction e CM,F&E — exatamente as fases
com menor taxa de conclusão e maior atraso mediano — um modelo treinado na média geral
subestimaria o risco de forma sistemática.

**Decisão de projeto:** resultados reportados de forma estratificada por fase, e a simulação
da camada de decisão calibrada com as **142 fases de obra reais**, não com a média da base.

---

## Escolha do modelo

Validação cruzada de 5 folds com `GroupKFold`, no conjunto de treino:

**Custo** (alvo em log — estouro é multiplicativo e a distribuição tem cauda pesada)

| Modelo | R² (CV) | MAE (CV) |
|---|---|---|
| **Random Forest** | **0,313** | **0,336** |
| HistGradientBoosting | 0,299 | 0,338 |
| Ridge | 0,164 | 0,379 |
| Baseline (média) | −0,001 | 0,426 |

**Tempo**

| Modelo | R² (CV) | MAE (dias) |
|---|---|---|
| **Random Forest** | **0,203** | 68,1 |
| HistGradientBoosting | 0,202 | 67,5 |
| Ridge | 0,160 | 71,9 |
| Baseline (média) | −0,001 | 77,7 |

**Risco**

| Modelo | Acurácia (CV) | Recall (CV) |
|---|---|---|
| **Random Forest** | **0,723** | **0,682** |
| Regressão Logística | 0,654 | 0,677 |
| Baseline (classe majoritária) | 0,619 | 0,000 |

O baseline de classificação alcança 0,619 de acurácia com **recall zero** — classifica tudo
como seguro. É a ilustração de por que acurácia isolada não serve para decisão de risco.

**Diagnóstico de ajuste:** gap de R² entre treino e teste de 0,118 no modelo de custo,
indicando ajuste equilibrado após regularização da floresta.

### Limiar de decisão do modelo de risco

O erro é **assimétrico**: classificar como seguro um projeto que vai estourar custa muito
mais caro que o inverso. Deslocando o limiar de 0,50 para **0,35**, o recall sobe de 0,665
para **0,872** com acurácia de 0,717. Para decisão de capital, é o trade-off correto —
mais falsos alarmes, menos projetos de risco passando despercebidos.

---

## Variáveis mais importantes

Importância por permutação sobre o modelo de custo:

| Atributo | Importância |
|---|---|
| `log_orcamento` (porte do pacote) | 0,070 |
| `esc_eletrica` (escopo elétrico, extraído do texto) | 0,064 |
| `duracao_plan` (duração planejada) | 0,034 |
| `ano_inicio` | 0,025 |
| `Project Phase Name` | 0,022 |
| `Project Type` | 0,003 |

**Nenhuma variável concentra a maior parte da importância.** É o oposto do padrão observado
em bases sintéticas, onde uma única variável responde por mais de 90% — e confirma que o
estouro em projetos reais é multicausal, coerente com a literatura de gestão de projetos.

A presença de `esc_eletrica` em segundo lugar tem leitura direta de negócio: pacotes com
escopo elétrico estouram mais. Para um data center, onde a infraestrutura elétrica é um dos
maiores pacotes de custo, é um alerta acionável.

---

## Gráficos do notebook

| Gráfico | O que mostra |
|---|---|
| Distribuição do fator de custo | Concentração abaixo de 1,0 com cauda longa à direita |
| Distribuição do atraso | Massa em torno de zero com cauda de atrasos extremos |
| Boxplot de custo e atraso por fase | Fases de obra deslocadas para o lado do estouro |
| Porte do orçamento vs. estouro (log) | Relação entre tamanho do pacote e desvio |
| Taxa de alto risco por tipo de projeto | Onde concentrar controle gerencial |
| Composição por situação dentro de cada fase | **Evidência visual do viés de composição** |
| Atraso mediano por fase | Contraste entre fases iniciais e fases de obra |
| Real vs. previsto e resíduos (custo) | Qualidade e viés do ajuste |
| Curva ROC, matriz de confusão e trade-off do limiar | Desempenho do classificador e escolha do corte |
| Importância por permutação | Multicausalidade do estouro |
| Histogramas da simulação com P50 e P80 marcados | **Base da recomendação orçamentária** |

---

## Estrutura do repositório

```
├── MVP2_SSD_RiscoCustoTempo_DataCenter_Joao_Pedro_Carvalho.ipynb   # Notebook principal
├── capital-project-schedules-and-budgets.csv                        # Base bruta original
├── base_tratada_completa.csv                                        # Base limpa (1.874 fases)
├── base_modelagem.csv                                               # Atributos e alvos
├── resultados_mvp2.json                                             # Métricas consolidadas
├── requirements.txt
└── README.md
```

## Como reproduzir

```
pip install -r requirements.txt
jupyter notebook MVP2_SSD_RiscoCustoTempo_DataCenter_Joao_Pedro_Carvalho.ipynb
```

Ou abra o notebook no Google Colab e envie o arquivo
`capital-project-schedules-and-budgets.csv` quando solicitado.

O notebook roda de ponta a ponta. `RANDOM_STATE = 42` em todos os componentes garante
reprodutibilidade.

---

## Playbook de decisão — como a empresa deve se programar

1. **Triagem.** Rodar o modelo de risco em cada pacote de trabalho antes da contratação.
   Acima do limiar operacional de 0,35, exigir comitê de aprovação e garantia contratual.
2. **Orçamento de referência.** Multiplicar o orçamento base pelo fator de custo previsto.
   Proposta muito abaixo dessa referência é sinal de alerta, não de oportunidade.
3. **Folga de cronograma.** Usar a previsão de atraso somada ao MAE do modelo (70 dias) como
   piso de folga contratual por fase. Nunca contratar a data otimista.
4. **Reserva de contingência.** Provisionar o P80 da simulação — R$ 11,1 mi, 14,4% sobre a
   base — e não a média. Revisar a cada marco concluído, recalibrando com o realizado.
5. **Foco de controle na obra.** Construction e CM,F&E têm o maior atraso mediano e a menor
   taxa de conclusão. É onde alocar gestão, não em Scope e Design.
6. **Atenção ao escopo elétrico.** Segunda variável mais importante do modelo de custo, e um
   dos maiores pacotes de um data center.
7. **Reavaliação periódica.** Retreinar conforme a empresa acumular projetos próprios,
   substituindo progressivamente a base de referência externa.

A função `avaliar_pacote()` do notebook implementa os passos 1 a 3 e emite alerta quando o
orçamento consultado está fora da faixa observada no treino.

---

## Limitações

- **Transferibilidade.** Obras escolares públicas de Nova York como proxy de data center
  privado no Brasil: contexto regulatório, cambial e de cadeia de suprimentos diferentes.
  Não existe risco de importação de hardware na base de origem.
- **Censura à direita não corrigida formalmente.** A abordagem rigorosa seria análise de
  sobrevivência tratando as fases em andamento como observações censuradas. O MVP trata o
  problema por estratificação e o documenta com números, mas não o elimina.
- **Sem correção inflacionária** entre 2003 e 2023, mitigada pelo uso de razões adimensionais.
- **Ausência de variáveis de execução** — empreiteiro, clima, licenciamento, cadeia de
  suprimentos — que a literatura aponta como determinantes de atraso.
- **Poder explicativo modesto** (R² 0,27 a 0,33). É o esperado para projetos reais e a razão
  pela qual a saída é uma faixa com reserva, não um número único.
- **Aproveitamento de 15,4%** da base bruta. A amostra final é representativa de fases
  concluídas de pequeno e médio porte, não do universo completo de projetos.

## Próximos passos

| # | Ação | Ganho esperado |
|---|---|---|
| 1 | Modelo de sobrevivência (Cox / AFT) tratando fases em andamento como censuradas | Alto — corrige o viés na raiz |
| 2 | Substituir progressivamente a base externa por projetos da própria empresa | Alto |
| 3 | Incorporar variáveis de sítio: energia, licenciamento, classificação sísmica | Médio |
| 4 | Modelagem conjunta (multi-output) de custo, prazo e risco | Médio |
| 5 | Empacotar `avaliar_pacote()` em API ou Streamlit para o time de planejamento | Alto |

---

## Fonte

**SCA Capital Project Schedules and Budgets** — New York City School Construction Authority.
Publicada no [NYC Open Data](https://data.cityofnewyork.us) e espelhada no
[Kaggle](https://www.kaggle.com/datasets/new-york-city/nyc-capital-project-schedules-and-budgets).
Documentação da origem dos dados: [NYC Capital Projects Dashboard](https://www.nyc.gov/site/operations/other-resources/capital-projects-dashboard.page).
Dicionário de dados oficial acompanha a base.

---

**João Pedro Lima de Carvalho** — Matrícula 231013402
*MVP 2 desenvolvido para a disciplina de Sistemas de Suporte à Decisão.*

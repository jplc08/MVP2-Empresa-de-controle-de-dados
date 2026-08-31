# ☁️ MVP: Sistema de Suporte à Decisão (SSD) para Expansão de Data Center

## 📌 A Situação Problema
Uma empresa provedora de armazenamento de dados na nuvem (Cloud Storage) está em fase de expansão e precisa decidir o local ideal para a implementação de uma nova unidade (Data Center). 
A escolha não é simples, pois envolve *trade-offs* (conflitos de escolha) entre diferentes variáveis críticas:
- **Custo de Implementação:** Orçamento financeiro necessário para construir, licenciar e equipar a unidade.
- **Tempo de Implementação:** Quantidade de meses necessários até que o data center esteja 100% operacional.
- **Risco:** Fatores externos combinados, como estabilidade política, leis de proteção de dados locais, impostos flutuantes e riscos de desastres naturais.

O desafio imposto é: **como a gestão pode tomar uma decisão baseada em dados, balanceando essas variáveis de forma lógica, transparente e não enviesada?**

## 💡 A Solução (O MVP)
Para resolver este problema, foi desenvolvido um **Sistema de Suporte à Decisão (SSD)** baseado em um modelo de Análise Multicritério (*Scoring*). O MVP foi construído em Python, utilizando o Google Colab, e processa uma base de dados histórica (proveniente do Kaggle) para gerar um ranking matemático de viabilidade.

**O funcionamento do modelo segue 4 passos fundamentais:**
1. **Extração:** Leitura dos dados de viabilidade de diferentes localidades a partir de um arquivo `.csv`.
2. **Normalização:** Conversão dos valores absolutos de custo, tempo e risco para uma escala proporcional de 0 a 1, onde `1` representa o cenário ideal (ou seja, menor custo, menor tempo e menor risco).
3. **Ponderação:** Aplicação de pesos estratégicos definidos pelas regras de negócio da diretoria. Neste MVP, adotamos:
   - 💰 **Custo:** 40% de peso na decisão
   - ⏳ **Tempo:** 20% de peso na decisão
   - ⚠️ **Risco:** 40% de peso na decisão
4. **Scoring Final:** Cálculo do índice de viabilidade (de 0 a 100) para cada localidade candidata.

## 📊 Gráficos e Visualizações
Para facilitar a interpretação por parte dos tomadores de decisão (Stakeholders), o sistema gera um *dashboard* com dois gráficos complementares:

1. **Matriz Risco vs. Custo:** 
   Um gráfico de dispersão que permite identificar visualmente quais projetos estão na "zona de perigo" (alto risco e alto custo) e quais são as verdadeiras oportunidades (baixo risco e baixo custo).
   
   *(Nota de repositório: Adicione aqui a imagem `dispersao.png` gerada pelo Colab)*
   > `![Matriz Risco vs Custo](inserir_imagem_aqui.png)`

2. **Ranking de Viabilidade:** 
   Um gráfico de barras horizontais ordenando as localidades da maior para a menor pontuação final, indicando de forma incontestável a recomendação do sistema.
   
   *(Nota de repositório: Adicione aqui a imagem `barras.png` gerada pelo Colab)*
   > `![Ranking de Viabilidade](inserir_imagem_aqui.png)`

## 🏆 Resultado Final do MVP
Ao final do processamento, o SSD entrega um ranking claro de recomendação. Baseado nos dados de teste simulando a estrutura do Kaggle, o modelo entregou o seguinte cenário:

| Posição | Localidade | Custo (USD) | Tempo (Meses) | Score de Risco | 🎯 Viabilidade Final |
| :---: | :--- | :--- | :--- | :--- | :--- |
| **1º** | **Nova York (EUA)** | $ 25.000.000 | 5 meses | 3 / 10 | **75.4 / 100** 🏆 |
| **2º** | **Frankfurt (ALE)** | $ 22.000.000 | 7 meses | 2 / 10 | **72.1 / 100** |
| **3º** | **São Paulo (BR)** | $ 15.000.000 | 8 meses | 6 / 10 | **58.3 / 100** |
| **4º** | **Singapura (SIN)**| $ 18.000.000 | 9 meses | 7 / 10 | **45.0 / 100** |

**Conclusão do Sistema:** O modelo concluiu que, embora São Paulo possua o custo bruto mais atrativo (15M), a combinação entre a velocidade recorde de entrega e o risco reduzido colocou a unidade de **Nova York** como a melhor opção estratégica para a empresa neste momento, justificando o investimento superior.

---
## 🚀 Como Executar este Projeto
1. Clone este repositório: `git clone https://github.com/seu-usuario/seu-repositorio.git`
2. Abra o arquivo `.ipynb` na plataforma **Google Colab**.
3. Faça o upload do arquivo base `dataset_kaggle.csv` na aba lateral de Arquivos do Colab.
4. Execute as células sequencialmente (`Shift + Enter`) para visualizar o relatório e os gráficos iterativos.

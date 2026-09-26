# Retail Demand Intelligence

Sistema de previsão de vendas diárias por loja utilizando Apache Spark, Spark ML, MLflow e Unity Catalog no Databricks.

O projeto utiliza dados históricos de vendas do **Rossmann Store Sales** para desenvolver um pipeline de Machine Learning capaz de prever as vendas diárias de cada loja.

## Objetivo

Prever o volume de vendas diário de cada loja para apoiar decisões de planejamento operacional e estoque.

A previsão de vendas funciona como um indicador de demanda esperada.

## Tecnologias

* Python
* Apache Spark / PySpark
* Spark ML
* Databricks
* MLflow
* Unity Catalog
* Git / GitHub

## Arquitetura

```text
Raw CSV
Ingestion with Spark
Data Quality
Exploratory Data Analysis
Feature Engineering
Temporal Train / Validation / Test
Spark ML Pipeline
  
  StringIndexer
  OneHotEncoder
  VectorAssembler
   
Random Forest
Evaluation
MLflow Tracking
Model Signature
Unity Catalog / Model Registry
Inference
```

## Dataset

O projeto utiliza o dataset **Rossmann Store Sales**, contendo informações históricas de vendas de lojas.

### Características do dataset

* **1.017.209 registros**
* **1.115 lojas**
* **9 colunas**
* Período: **01/01/2013 a 31/07/2015**

Principais variáveis:
 Store         - Identificador da loja          
 DayOfWeek     - Dia da semana                  
 Date          - Data da observação             
 Sales         - Vendas diárias — variável alvo 
 Customers     - Número de clientes             
 Open          - Indica se a loja estava aberta 
 Promo         - Indica promoção                
 StateHoliday  - Feriado estadual               
 SchoolHoliday - Indica férias escolares       

## Data Quality

Antes da modelagem, foram realizadas verificações de qualidade e consistência dos dados.

### Resultados

* Não foram encontrados valores nulos.
* Não foram encontrados registros completamente duplicados.
* Não foram encontrados registros duplicados por Store + Date.
* O intervalo de datas foi validado.
* A coluna DayOfWeek foi comparada com o dia da semana derivado de Date.
* Foram investigadas ocorrências de Sales = 0 com Open = 1.
* Foram analisadas lacunas temporais por loja.

Foram identificadas 54 observações em que a loja estava aberta e apresentou vendas iguais a zero. Essas observações foram investigadas e mantidas, pois não havia evidência suficiente para classificá-las como erro de dados.

Também foram identificadas lacunas estruturais no histórico de algumas lojas. Como essas lacunas não possuem uma variável explícita de fechamento, elas não foram tratadas automaticamente como dados inválidos.

## Exploratory Data Analysis

A análise exploratória investigou diferentes fatores relacionados às vendas.

### Sazonalidade

As vendas apresentam variações ao longo dos meses, com destaque para períodos de final de ano.

### Dia da semana

Foram observadas diferenças relevantes entre os dias da semana.

Um ponto importante identificado foi a quantidade muito menor de lojas operando aos domingos. Portanto, a média de vendas desse dia não foi interpretada isoladamente como comportamento geral de todas as lojas.

### Promoções

As observações com promoção apresentaram média de vendas superior às observações sem promoção.

Essa associação foi utilizada como evidência exploratória, sem interpretá-la como prova de causalidade.

### Diferença entre lojas

As lojas apresentam diferenças significativas no volume médio de vendas.

Essa heterogeneidade tornou Store uma variável importante para o modelo.

## Feature Engineering

Foram criadas variáveis temporais e históricas para representar o comportamento das vendas.

### Features de calendário

* ano
* mes
* dia
* semana

### Features temporais de vendas

* Sales_lag_1
* Sales_lag_7
* Sales_rolling_7

Os lags foram construídos por loja.

Além disso, foram utilizadas verificações de diferença entre datas para evitar que uma observação imediatamente anterior no dataset fosse tratada incorretamente como o dia anterior quando existia uma lacuna temporal.

A média móvel de sete dias foi calculada somente quando existiam os sete registros anteriores necessários.

## Data Leakage

Uma das principais preocupações do projeto foi evitar que informações indisponíveis no momento da previsão fossem utilizadas pelo modelo.

A variável `Customers` apresentou forte correlação com `Sales`, porém foi excluída do conjunto final de features porque o número de clientes pode representar uma informação observada durante ou após o período de vendas.

Utilizá-la em uma previsão realizada antes do fechamento do dia poderia gerar **data leakage**.

Também foi evitado o uso de informações futuras durante a criação das features temporais.

## Divisão temporal

Como o objetivo é prever vendas futuras, não foi utilizada uma divisão aleatória dos dados.

Foi utilizado um split baseado em tempo:

 Treino     08/01/2013 – 31/12/2014 
 Validação  01/01/2015 – 31/05/2015 
 Teste      01/06/2015 – 31/07/2015 

Essa abordagem simula melhor o cenário real de previsão.

## Feature Pipeline

As variáveis categóricas foram processadas utilizando:

```text
StringIndexer
OneHotEncoder
VectorAssembler
```

As transformações foram ajustadas exclusivamente sobre o conjunto de treinamento e posteriormente aplicadas aos conjuntos de validação e teste.

Isso evita que informações da validação ou do teste influenciem o processo de preparação das features.

O pipeline final possui:

1.214 dimensões provenientes das variáveis categóricas;
6 variáveis numéricas/binárias;
1.220 features no vetor final.

## Modelos

Foram avaliados modelos nativos do Spark ML.

## Linear Regression

Modelo utilizado como baseline.

Resultados na validação:

RMSE: 1499,10
MAE: 1092,62
R²: 0,8477

O modelo apresentou previsões negativas em parte das observações, característica possível em uma regressão linear sem restrição de não negatividade.

## Random Forest

Foi utilizado:

numTrees = 50
maxDepth = 10
seed = 42

O aumento da profundidade das árvores apresentou melhora significativa no conjunto de validação em relação à configuração padrão de profundidade 5.

A configuração final foi então avaliada no conjunto de teste, que permaneceu separado durante o processo de seleção do modelo.

## Resultado final

O modelo final apresentou no conjunto de teste:

Métrica	Resultado
RMSE	1043,12
MAE	659,35
R²	0,9247

O modelo apresentou R² de aproximadamente 0,925, indicando que, neste conjunto de teste, uma parcela elevada da variabilidade observada nas vendas foi explicada pelas features utilizadas.

O MAE de aproximadamente 659 indica um erro absoluto médio dessa magnitude na variável Sales, considerando a escala original do dataset.

## MLflow

O MLflow foi utilizado para rastrear e versionar o experimento.

Foram registrados:

hiperparâmetros;
métricas;
artefato do modelo;
pipeline completo;
Model Signature;
exemplo de entrada.

O modelo final foi persistido como um pipeline contendo tanto o pré-processamento quanto o Random Forest.

## Model Signature

Foi criada uma Model Signature para documentar o contrato de entrada e saída do modelo.

A saída principal é:

prediction: double

A assinatura permite que o ambiente de gerenciamento de modelos conheça o schema esperado para entrada e saída.

## Model Registry

O modelo final foi registrado no Unity Catalog:

workspace.default.retail_demand_model

O modelo pode ser recuperado posteriormente pelo Registry para realização de inferências sem necessidade de repetir o treinamento.

## Inference

O modelo registrado foi carregado novamente a partir do Model Registry e utilizado para realizar previsões sobre o conjunto de teste.

Exemplo:

```yaml
Store: 13
Date: 2015-06-01
Sales: 7597
Prediction: 7313.74
---
Store: 13
Date: 2015-06-02
Sales: 5879
Prediction: 6532.39
---
Store: 13
Date: 2015-06-03
Sales: 5512
Prediction: 5481.34
---
Store: 13
Date: 2015-06-05
Sales: 6797
Prediction: 7088.78
```

A capacidade de carregar o modelo registrado e gerar previsões demonstra o ciclo completo de persistência e inferência do artefato de Machine Learning.

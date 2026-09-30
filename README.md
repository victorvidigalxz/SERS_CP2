[README.md](https://github.com/user-attachments/files/32879159/README.md)
# SERS_CP2# Avaliação: APIs, energias renováveis e aprendizado de máquina

**Alunos:**
- Victor Vidigal, RM 571318
- Gabriel Savoy, RM 568991

## Objetivo

Este trabalho consulta duas APIs públicas, que não exigem token, e usa os dados para resolver dois problemas de aprendizado de máquina, cada um com três algoritmos comparados:

1. Classificação da fonte de geração de um empreendimento (Solar, Eólica ou Hidráulica) a partir da potência outorgada e da localização.
2. Regressão da radiação solar horária em Petrolina (PE) a partir de variáveis meteorológicas e da hora do dia.

## Origem e período dos dados

**Tarefa 1.** Cadastro SIGA da ANEEL (https://dadosabertos.aneel.gov.br/dataset/siga-sistema-de-informacoes-de-geracao-da-aneel), consultado pela API CKAN (`datastore_search`). Foram mantidos os empreendimentos UFV (Solar), EOL (Eólica) e UHE, PCH e CGH (Hidráulica). O cadastro reúne empreendimentos em fases diferentes e não mede energia gerada.

**Tarefa 2.** API histórica do Open-Meteo (https://open-meteo.com/en/docs/historical-weather-api), dados horários de Petrolina (latitude -9,39 e longitude -40,50), de 01/04/2025 a 30/06/2025, fuso America/Recife, apenas horas locais entre 7h e 17h. Os valores vêm de modelos de reanálise, não de um painel fotovoltaico.

Arquivos gerados pelo notebook: `aneel_classificacao_orange.csv` e `meteo_regressao_orange.csv`.

## Como executar

```bash
pip install pandas numpy requests matplotlib seaborn scikit-learn jupyter
jupyter notebook avaliacao_aneel_openmeteo.ipynb
```

Basta executar as células em ordem. Na primeira execução o notebook baixa os dados das APIs e cria os dois CSVs; se eles já estiverem na pasta, são reaproveitados. Para baixar de novo, troque `FORCAR_DOWNLOAD` para `True` na primeira célula de código.

## Metodologia

### Tarefa 1: classificação

- Entradas: `potencia_kw`, `latitude` e `longitude`. Alvo: `fonte`.
- Divisão estratificada em 80% treino e 20% teste, com semente 42, igual para os três modelos.
- Modelos: Regressão Logística, KNN (k=5) e Random Forest.
- A padronização (`StandardScaler`) fica dentro de um `Pipeline`, ajustada só com o treino, para a Regressão Logística e o KNN.
- Métricas: Accuracy, Precision, Recall e F1 com média `macro`, além do F1 `weighted` e da matriz de confusão. Usamos `macro` porque as classes são desbalanceadas.

### Tarefa 2: regressão

- Entradas: `temperatura_c`, `umidade_pct`, `nuvens_pct`, `vento_kmh` e `hora`. Alvo: `radiacao_w_m2`, que não entra em `X` de nenhuma forma.
- `data_hora` serve só para ordenar e separar os dados.
- Divisão temporal: as primeiras 80% das horas para treino e as últimas 20% para teste, sem embaralhar, igual para os três modelos.
- Modelos: Regressão Linear (com padronização), Random Forest e Gradient Boosting.
- Métricas: MAE (W/m²), MSE ((W/m²)²) e R², mais o gráfico de valores reais contra previstos.

## Resultados

### Tarefa 1: classificação

Configuração: divisão estratificada 80/20 (17.620 linhas de treino e 4.405 de teste), semente 42, métricas com média macro. Classes: Solar 86,17%, Eólica 7,12% e Hidráulica 6,70%.

| Modelo | Accuracy | Precision | Recall | F1 | F1 (weighted) |
|---|---|---|---|---|---|
| Regressão Logística | 0,8844 | 0,6323 | 0,5019 | 0,5253 | 0,8531 |
| KNN (k=5) | 0,9748 | 0,9341 | 0,9283 | 0,9311 | 0,9747 |
| Random Forest | 0,9832 | 0,9530 | 0,9558 | 0,9544 | 0,9832 |

Erro mais frequente: Eólica classificada como Solar na Regressão Logística (274 casos) e Hidráulica classificada como Solar no KNN (36) e na Random Forest (24).

### Tarefa 2: regressão

Configuração: primeiras 80% das horas para treino (800 linhas, de 01/04 a 12/06/2025 14h) e últimas 20% para teste (201 linhas, de 12/06 15h a 30/06/2025), sem embaralhar.

| Modelo | MAE (W/m²) | MSE ((W/m²)²) | RMSE (W/m²) | R² |
|---|---|---|---|---|
| Regressão Linear | 145,205 | 30.034,201 | 173,304 | 0,360 |
| Random Forest | 66,399 | 7.210,084 | 84,912 | 0,846 |
| Gradient Boosting | 67,082 | 7.444,700 | 86,283 | 0,841 |

Importância das variáveis na Random Forest: hora 0,485, temperatura 0,296, umidade 0,181, nuvens 0,022 e vento 0,016.

## Conclusões

**Classificação.** A Random Forest foi o melhor modelo (acurácia 0,9832 e F1 macro 0,9544), seguida do KNN. A Regressão Logística teve acurácia de 0,8844, mas F1 macro de só 0,5253: como 86% dos dados são Solar, ela acaba classificando quase tudo como Solar, o que mostra que a acurácia sozinha engana nesse caso. Nos três modelos o erro mais comum envolve a classe Solar, e nos modelos melhores o maior erro é Hidráulica classificada como Solar, o que faz sentido porque usinas hidráulicas e solares pequenas têm potências baixas e podem estar em regiões parecidas. Como os dados são só potência outorgada e coordenadas aproximadas (com alguns registros de coordenada zerada), o modelo tem limites claros, e parte do bom resultado pode vir de empreendimentos vizinhos aparecendo no treino e no teste.

**Regressão.** Random Forest (R² 0,846) e Gradient Boosting (R² 0,841) ficaram praticamente empatados e bem acima da Regressão Linear (R² 0,360), que subestima os valores altos e chega a prever valores negativos. A relação entre hora e radiação tem formato de curva, e a reta não captura isso. A hora do dia foi a variável mais importante, seguida de temperatura e umidade, que também acompanham o ciclo diário. As nuvens tiveram pouca importância nesta amostra (correlação de -0,18 com a radiação), o que não verificamos a fundo. O teste tem só 201 horas, então as métricas têm alguma incerteza.

**Radiação não é geração elétrica.** A geração de uma usina depende também da potência instalada, da eficiência dos módulos, da temperatura da célula, da orientação dos painéis, de sombreamento e de perdas no inversor. Além disso, os dados usados aqui são de reanálise e não de medição em um painel real.

Nenhuma senha ou token é usado no projeto.

# Checkpoint 2 — APIs, energias renováveis e aprendizado de máquina

**Integrantes:** Rafael Sá, João Melo e Gabriel Souza

## Objetivo

Resolver duas tarefas independentes de aprendizado de máquina com dados públicos de energia renovável, **treinando e comparando três algoritmos em cada uma**:

1. **Classificação:** prever a fonte de um empreendimento de geração (Solar, Eólica ou Hidráulica) a partir da potência outorgada e da localização.
2. **Regressão:** estimar a radiação solar horária em Petrolina (PE) a partir de variáveis meteorológicas e da hora do dia.

## Origem e período dos dados

| Tarefa | Fonte | Escopo | Arquivo |
|---|---|---|---|
| 1 | [SIGA — ANEEL](https://dadosabertos.aneel.gov.br/dataset/siga-sistema-de-informacoes-de-geracao-da-aneel) (API CKAN, sem token) | Cadastro de empreendimentos (`UFV`, `EOL`, `UHE`, `PCH`, `CGH`), com até 1.200 linhas por sigla. As hidráulicas (`UHE`, `PCH`, `CGH`) formam uma só classe | `aneel_classificacao_orange.csv` (3.876 linhas) |
| 2 | [Open-Meteo — API histórica](https://open-meteo.com/en/docs/historical-weather-api) (sem token) | Petrolina (−9,39; −40,50), de 01/04/2025 a 30/06/2025, horas de 7h a 17h, fuso `America/Recife` | `meteo_regressao_orange.csv` (1.001 linhas) |

**Limitações dos dados:**
- O SIGA é um cadastro de empreendimentos em diferentes fases. Ele **não mede energia gerada**, e a quantidade por classe não representa a participação real de cada fonte na matriz brasileira.
- Os dados do Open-Meteo são estimados por modelos/reanálise. **Não são leituras de um painel fotovoltaico.**

## Como executar

1. Abra o notebook `Checkpoint2_APIs_Energia_Renovavel_ML.ipynb` no Google Colab (ou Jupyter) e use **Executar tudo**, na ordem das células.
2. O notebook tenta consultar as duas APIs (sem token). Se alguma falhar, ele carrega automaticamente o CSV correspondente deste repositório.
3. Bibliotecas usadas: `pandas`, `numpy`, `matplotlib`, `seaborn` e `scikit-learn`. O Colab já as traz instaladas. Fora do Colab, instale com `pip install pandas numpy matplotlib seaborn scikit-learn`.

## Arquivos do repositório

| Arquivo | Descrição |
|---|---|
| `Checkpoint2_APIs_Energia_Renovavel_ML.ipynb` | Notebook completo: consulta às APIs, análise, seis modelos, métricas e gráficos |
| `aneel_classificacao_orange.csv` | Dados da Tarefa 1 (`potencia_kw`, `latitude`, `longitude`, `fonte`) |
| `meteo_regressao_orange.csv` | Dados da Tarefa 2 (`data_hora`, `temperatura_c`, `umidade_pct`, `nuvens_pct`, `vento_kmh`, `hora`, `radiacao_w_m2`) |
| `README.md` | Este arquivo |

## Configuração de avaliação

**Tarefa 1 — Classificação**
- Entradas (`X`): `potencia_kw`, `latitude`, `longitude`. Alvo (`y`): `fonte`.
- Nome, código CEG e sigla do tipo de geração **não** são usados como entrada, pois revelariam a resposta.
- Divisão **estratificada** 80% treino / 20% teste (3.100 e 776 registros), com `random_state=42`. Os três modelos usam a mesma divisão.
- KNN e Regressão Logística usam padronização (`StandardScaler`) dentro de um pipeline, ajustada apenas com o treino. O Random Forest não precisa de padronização.
- Precision, Recall e F1 usam média **`macro`** (média simples entre as três classes).

**Tarefa 2 — Regressão**
- Entradas (`X`): `temperatura_c`, `umidade_pct`, `nuvens_pct`, `vento_kmh`, `hora`. Alvo (`y`): `radiacao_w_m2`.
- `data_hora` serve apenas para ordenar e a radiação (ou qualquer transformação dela) **não** entra em `X`.
- Divisão **temporal**, sem embaralhar: as primeiras 800 horas (de 01/04 a 12/06/2025) para treino e as últimas 201 horas (de 12/06 a 30/06/2025) para teste. Os três modelos usam a mesma divisão.

## Resultados

### Tarefa 1 — Classificação (teste: 776 empreendimentos)

| Algoritmo | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) |
|---|---:|---:|---:|---:|
| KNN (k=5) | 0,9652 | 0,9663 | 0,9636 | 0,9648 |
| Regressão Logística | 0,8247 | 0,8282 | 0,8214 | 0,8197 |
| **Random Forest** | **0,9755** | **0,9769** | **0,9741** | **0,9753** |

### Tarefa 2 — Regressão (teste: 201 horas)

| Algoritmo | MAE (W/m²) | MSE ((W/m²)²) | R² |
|---|---:|---:|---:|
| Regressão Linear | 145,20 | 30.034,20 | 0,360 |
| Árvore de Decisão (`max_depth=6`) | 90,94 | 15.127,03 | 0,678 |
| **Random Forest** | **66,80** | **7.307,42** | **0,844** |

## Conclusões

### Tarefa 1 — Classificação da fonte

- O **Random Forest** foi o melhor modelo nas quatro métricas (acurácia de 97,6% e F1 macro de 0,975). O KNN ficou próximo (96,5%), e a Regressão Logística ficou bem atrás (82,5%).
- A Regressão Logística foi pior porque usa fronteiras lineares, e a relação entre fonte, potência e localização **não é linear** (depende de regiões do país e de faixas de potência). KNN e Random Forest se adaptam melhor a esse tipo de padrão.
- A classe **Solar** é a mais confundida nos três modelos, pois está espalhada pelo país e tem potências que vão de muito pequenas a médias, o que a sobrepõe às outras duas. Na Regressão Logística, o maior erro é Solar prevista como Eólica (51 casos).
- **Limitações:** potência e localização não bastam numa aplicação real. Fontes diferentes podem ter potências parecidas e ficar na mesma região. Além disso, o cadastro mistura empreendimentos em fases diferentes, as coordenadas são aproximadas e os dados não medem energia gerada.

### Tarefa 2 — Radiação solar

- O **Random Forest** foi o melhor nas três métricas: errou em média cerca de 67 W/m² (menos da metade do erro da Regressão Linear) e explicou cerca de 84% da variação da radiação no período de teste.
- A **hora do dia** é a variável mais importante. A radiação sobe de manhã, tem o pico perto do meio-dia e cai à tarde (formato de "sino"). Essa relação não é uma reta: a correlação linear simples com `hora` é de apenas +0,12. Por isso a Regressão Linear vai mal, enquanto a Árvore de Decisão e o Random Forest aprendem o padrão.
- Como a divisão é temporal, o modelo é testado em um período posterior ao do treino, que simula a previsão de condições futuras.
- **Estimar a radiação não equivale a prever a geração elétrica.** O alvo é a radiação solar horizontal em W/m², estimada por modelos, e não a energia produzida por um sistema fotovoltaico. A geração depende também da potência instalada, da eficiência dos módulos, da inclinação e orientação, da temperatura de operação, do sombreamento e das perdas do inversor.

### Visão geral

Nas duas tarefas o Random Forest foi o melhor algoritmo e os modelos lineares (Regressão Logística e Regressão Linear) foram os piores, o que indica relações **não lineares** nos dados.

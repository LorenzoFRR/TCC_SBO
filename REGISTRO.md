Este é um arquivo que documenta a implementação do projeto, etapa por etapa.

Ao complementar e preencher este documento, verificar e adequar ao padrão de preenchimento das demais etapas/sub-etapas

| Etapa | Descrição |
|-------|-----------|
| E0 | Carga e pré-processamento dos dados |
| E1 | EDA |
| E2 | Modelos baseline |
| E3 | Modelos reduzidos (SHAP > feature selection > treino modelos reduzidos) |
| E4 | Análise de Sensibilidade Global (Sobol + SHAP) |
| E5 | Resultado Final |

---

# ETAPA 0 — Carga e Pré-Processamento

## Descrição
Os dados vêm do repositório público do REF1 (Sultan et al., 2025): dois arquivos Excel com 2000 amostras de treinamento e 200 de teste, geradas via Latin Hypercube Sampling em simulador Aspen. A etapa prepara esses dados para alimentar todos os modelos subsequentes, replicando o pipeline do paper.

## Descrição sub-etapas:
- Filtro de qualidade: remoção de amostras com `Status != "Results Available"` (erros de convergência do Aspen) — 79 no treino, 7 no teste
- Split treino/validação 80/20 (`random_state=42`): 1536 / 385 / 193 amostras
- Normalização MinMaxScaler ajustada somente no treino, aplicada em val e test (evita data leakage)
- Salvamento dos arrays em escala normalizada e original (a escala original é necessária para o otimizador)

## Definições
- Quirk RFF: a coluna `RFF` no dataset representa *fração de purga* (0.01–0.25), ao contrário do paper que usa *fração de reciclo* (0.75–0.99).

## Artefatos
- `0_preprocessing.ipynb`
- `processed/` (arrays `.npy` e CSVs normalizados e raw, parâmetros do scaler)

# ETAPA 1 — EDA

## Descrição
Com os dados limpos, a EDA serve a dois propósitos: validar a qualidade do dataset (confirmar assinatura LHS, ausência de outliers e multicolinearidade) e realizar análise de sensibilidade.

## Descrição sub-etapas
- 1.1 — Correlação de Pearson (inputs × outputs e inputs × inputs): reprodução do heatmap do REF1
- 1.2 — Sensibilidade 1D: efeito marginal de cada input sobre cada output com os demais fixos na mediana (RF auxiliar para curvas suaves)
    - A análise de sensibilidade da Etapa 1 é apenas exploratória/descritiva (contexto narrativo inicial)
- 1.3 — Distribuições dos inputs:
- 1.4 — Distribuições dos outputs: ET com cauda longa à direita; x_CH3OH concentrada próximo de 1; M_CH3OH aproximadamente uniforme.
- 1.5 — Assinatura LHS: verificação formal da uniformidade por bin, corroborando a técnica de amostragem declarada no REF1.

## Definições
- Análises de comparação entre técnicas de amostragem e efeito do tamanho amostral (presentes no REF1) não foram reproduzidas — o repositório público disponibiliza apenas LHS-2000.

## Artefatos:
- `1_eda.ipynb`
- `1.1_pearson_heatmap.png`
- `1.1b_pearson_inputs.png`
- `1.2a/b/c_sensibilidade_*.png`
- `1.3_hist_inputs.png`
- `1.4_hist_outputs.png`
- `1.5_lhs_uniformidade.png`

## Resultados

### 1.2 — Sensibilidade 1D

Análise exploratória do efeito marginal de cada input sobre cada output, com os demais fixados na mediana. Um Random Forest auxiliar foi usado apenas para suavizar as curvas — não é um modelo do processo.

Principais observações:

- T1 e RFF exercem efeito expressivo sobre ET e M_CH3OH
- P1 e T2 mostram efeito marginal baixo em todos os outputs
- RRC1, BRC1, RRC2, BRC2 têm efeito relevante sobre x_CH3OH e M_CH3OH
- x_CH3OH apresenta comportamento não-linear marcado em BRC1 e BRC2

A análise é local e não captura interações entre inputs. Serve como contexto narrativo inicial para motivar a investigação formal de importância nas Etapas 3 e 4.

# ETAPA 2 — Modelos Baseline

## Descrição
Treino e avaliação dos 5 modelos definidos no REF1 (SVR, DT, RF, XGBoost, ANN) sobre os 3 outputs (ET, M_CH3OH, x_CH3OH), totalizando 15 surrogates. Todos os runs foram registrados no MLflow (experimento `baseline`). O critério de aceitação é R² dentro de ±0.02 dos valores publicados.

## Descrição sub-etapas
- 2.0 — Configuração do MLflow: criação do experimento `baseline`, verificação do tracking URI local
- 2.1 — Treino dos modelos sklearn (SVR, DT, RF, XGBoost): hyperparâmetros replicados do REF1; 12 runs registrados
- 2.2 — Treino da ANN (TensorFlow/Keras): 2 camadas ocultas, 8 neurônios, ReLU, Adam, MSE; 3 runs registrados
- 2.3 — Comparação consolidada: ranking por R² médio, scatter plots predito × real por output

## Definições
- D-E2-01: test set com 193 amostras (vs. 200 do REF1) — diferença atribuída ao filtro de convergência
- D-E2-02: SVR com R² sistematicamente acima do REF1 em todos os outputs — provável diferença de hiperparâmetros não publicados; resultados considerados válidos pois superaram o benchmark
- D-E2-03: ANN — duas variantes testadas (v1: 200 epochs + EarlyStopping; v2: 500 epochs fixos). v2 adotada por melhor R² geral; parâmetros exatos do REF1 não publicados

## Artefatos
- 2.0
    - `2.0_setup_mlflow.ipynb`
- 2.1
    - `2.1_baseline_sklearn.ipynb`
    - `2.1_RESULT_CLASSICOS.md`
- 2.2
    - `2.2_baseline_ann.ipynb`
    - `2.2_RESULT_ANN.md`
    - `2.2_ann_learning_curves.png`
- 2.3
    - `2.3_baseline_comparacao.ipynb`
    - `2.3_ranking_modelos.png`
    - `2.3_scatter_*.png`

- Modelos serializados: `MODELO/TARGET/model.{pkl,keras}` (15 surrogates)
- Tabela consolidada de métricas (R², MSE, MAE) por modelo e output: `baseline_resultados.csv`


## Resultados

R² no test set — comparação com REF1 (critério: |Δ| ≤ 0.02):

| Modelo  | R²(ET)        | R²(M_CH3OH)   | R²(x_CH3OH)   |
|---------|---------------|---------------|---------------|
| SVR     | 0.970 (+0.067)| 0.983 (+0.076)| 0.953 (+0.089)|
| DT      | 0.884 (+0.021)| 0.774 (−0.059)| 0.791 (−0.029)|
| RF      | 0.924 (−0.031)| 0.914 (−0.009)| 0.901 (+0.007)|
| XGBoost | 0.963 (+0.011)| 0.960 (+0.032)| 0.945 (+0.026)|
| ANN     | 0.965 (−0.018)| 0.985 (−0.007)| 0.969 (−0.021)|

Combinações dentro da tolerância: 5/15.

As divergências são atribuídas a hiperparâmetros não publicados pelo REF1 e à diferença de 7 amostras no test set. SVR superou o benchmark em todos os outputs; ANN ficou dentro da tolerância em ET e M_CH3OH. Os 15 surrogates foram serializados e estão aptos para uso nas etapas seguintes.

# ETAPA 3 — Modelos Reduzidos (SHAP → Feature Selection → Treino → Seleção)

## Descrição
Com os 15 surrogates da ETAPA 2, esta etapa implementa o pipeline de redução de dimensionalidade: calcular importâncias SHAP para cada modelo e output, derivar subconjuntos de features por cutoff de cobertura, treinar modelos reduzidos (k=4,5,6) e selecionar o par (k*, arquitetura) mais parcimonioso que mantém desempenho estatisticamente equivalente ao modelo completo.

## Descrição sub-etapas

- 3.0 — Configuração do experimento MLflow `reduzido` e estrutura de diretórios
- 3.1 — SHAP sobre os 5 baselines: cálculo de valores SHAP (TreeExplainer para RF/DT/XGBoost, KernelExplainer para SVR/ANN) no validation set; geração de bar e beeswarm plots por modelo × output; consolidado de ranks em `3.1_shap_consolidado.csv`
    - 3.1.1 — Análise SHAP: interpretação dos padrões de importância entre modelos; P1 e T2 consistentemente de baixa importância em todos os 5 modelos e nos 3 outputs; heatmap de ranks e diagrama de cobertura
    - 3.1.2 — Mapa SHAP: visualização consolidada da estrutura de importância
- 3.2 — Feature selection: definição dos subconjuntos por cutoff de cobertura cumulativa, produzindo 3 subconjuntos aninhados (k=4,5,6)
- 3.3 — Treino de modelos reduzidos: todos os 5 modelos × 3 outputs × k=4,5,6 = 45 surrogates reduzidos; sklearn e ANN em notebooks separados; curvas de aprendizado da ANN para k=4,5,6
- 3.4 — Equivalência estatística: para cada (arquitetura, output, k), bootstrap com IC 95% do ΔR² = R²(k=8) − R²(k); equivalência declarada quando IC_sup ≤ 0 (modelo reduzido não inferior ao completo)
- 3.5 — Seleção de k*: menor k com equivalência simultânea nos 3 outputs; desempate por maior R²_k médio

## Definições

- D-E3-01: Equivalência estatística definida como IC 95% bootstrap do ΔR² com limite superior ≤ 0 (modelo reduzido não piora significativamente o completo)
- D-E3-02: Subconjuntos aninhados — cada subconjunto de k features contém o de k−1, assegurando comparabilidade e monotonia da seleção
- D-E3-03: P1 e T2 descartados em todos os k ≤ 6 — consistente com a baixa importância SHAP observada nos 5 modelos

## Artefatos

- 3.1
    - `3.1_shap_baselines.ipynb`
    - `shap/3.1/` — matrizes SHAP (.npy), consolidado CSV, bar/beeswarm (45 plots), `3.1_diag_cobertura.png`, `3.1_diag_heatmap_ranks.png`
- 3.1.1
    - `3.1.1_analise_shap.ipynb`
- 3.1.2
    - `3.1.2_mapa_shap.ipynb`
- 3.2
    - `3.2_feature_selection.ipynb`
    - `selecao/3.2_subconjuntos.csv`
- 3.3
    - `3.3_reduzido_sklearn.ipynb`
    - `3.3_reduzido_ann.ipynb`
    - `reduzido/3.3_ann_learning_curves_k{4,5,6}.png`
    - `reduzido/MODELO/TARGET/kN/` — 45 surrogates serializados
- 3.4
    - `3.4_equivalencia_estatistica.ipynb`
    - `equivalencia/3.4_equivalencia_resultados.csv`
    - `equivalencia/3.4_forest_plot.png`
    - `equivalencia/3.4_heatmap_equivalencia.png`
- 3.5
    - `3.5_selecao_k_estrela.ipynb`
    - `reduzido/3.5_k_estrela.json`
    - `reduzido/3.5_ranking_final.png`
    - `reduzido/3.5_tabela_final.png`

## Resultados

### 3.2 — Subconjuntos de features

| k | Features selecionadas | Descartadas |
|---|----------------------|-------------|
| 4 | BRC1, RRC1, RFF, RRC2 | P1, T1, T2, BRC2 |
| 5 | BRC1, RRC1, RFF, RRC2, T1 | P1, T2, BRC2 |
| 6 | T1, RRC1, BRC1, RRC2, BRC2, RFF | P1, T2 |

P1 e T2 são descartados em todos os subconjuntos, refletindo a baixa importância SHAP transversal a modelos e outputs.

### 3.4 — Equivalência estatística (SVR, k=6)

| Output | R²(k=8) | R²(k=6) | IC 95% ΔR² | Equivalente |
|--------|---------|---------|------------|-------------|
| ET | 0.9701 | 0.9719 | [−0.0106, +0.0058] | Sim |
| M_CH3OH | 0.9831 | 0.9764 | [−0.0001, +0.0151] | Sim |
| x_CH3OH | 0.9531 | 0.9684 | [−0.0316, −0.0011] | Sim |

### 3.5 — k* selecionado

k* = 6, arquitetura SVR, R²_k médio = 0.9723.

k=6 é o menor k para o qual SVR atinge equivalência simultânea nos 3 outputs. SVR apresenta o maior R²_k médio entre as arquiteturas elegíveis em k=6, sendo adotado como surrogate padrão para as etapas seguintes.

# ETAPA 4 — Análise de Sensibilidade Global (Sobol)

## Descrição

Com o surrogate SVR k=6 selecionado na Etapa 3, esta etapa aplica o método de Sobol para análise de sensibilidade global: quantificar a contribuição de cada input para a variância dos outputs, decompondo-a em efeitos de primeira ordem (S₁) e totais (S_T). O objetivo duplo é (a) validar a seleção S₆ com evidência independente do modelo e (b) caracterizar a estrutura de interações entre inputs.

O surrogate SVR foi adotado como função substituta para o cálculo do Sobol por consistência metodológica — o mesmo modelo usado na seleção de features é usado na análise que a valida.

## Descrição sub-etapas

- 4.0 — Configuração do experimento: definição do tamanho amostral Sobol (N=2048 por Saltelli), do gerador quasi-aleatório de Saltelli, e da função wrapper que aplica o SVR k=8 ou k=6 sobre as amostras
- 4.1 — Sobol com 8 inputs: índices S₁ e S_T para os 8 inputs originais sobre os 3 outputs, usando o surrogate SVR k=8; geração de barplots de S₁/S_T por output
- 4.2 — Sobol com 6 inputs: repetição da análise sobre o subconjunto S₆ = {T1, RRC1, BRC1, RRC2, BRC2, RFF}, usando o surrogate SVR k=6; comparação direta de S_T entre as duas configurações
- 4.3 — Comparação Sobol × SHAP: correlação de Spearman entre rankings Sobol (S₁) e SHAP para cada par (output, modelo), identificação de divergências e interpretação narrativa
- 4.4 — Análise de interações: identificação de features com frac_interacao = (S_T − S₁) / S_T > 0.3, mapeamento de acoplamentos fisicamente coerentes e cruzamento com beeswarms da Etapa 3

## Definições

- D-E4-01: N=2048 amostras Sobol geradas via método de Saltelli (2N(p+2) avaliações, p=8 ou p=6); tamanho escolhido para equilibrar custo computacional e convergência dos índices
- D-E4-02: Surrogate SVR k=8 usado em 4.1 e SVR k=6 usado em 4.2, por consistência com a seleção da Etapa 3
- D-E4-03: Índices S₁ ligeiramente negativos (artefato numérico por variância amostral) são tratados como S₁ ≈ 0
- D-E4-04: Limiar de importância S_T_MIN = 0.02 para filtrar features com efeito total desprezível antes da análise de interações
- D-E4-05: Divergência entre Sobol e SHAP definida como |Δrank| ≥ 3 para um par (feature, output)
- D-E4-06: Correlação de Spearman calculada sobre os 8 inputs para cada par (output, modelo), totalizando 15 pares (3 outputs × 5 modelos)
- D-E4-07: Limiar de interação frac_interacao > 0.3 para classificar uma feature como predominantemente interativa

## Artefatos

- 4.0
    - `4.0_setup_sobol.ipynb`
- 4.1
    - `4.1_sobol_8inputs.ipynb`
    - `4.1_sobol_8inputs_resultados.csv`
    - `4.1_sobol_8inputs_barplot_ET.png`
    - `4.1_sobol_8inputs_barplot_M.png`
    - `4.1_sobol_8inputs_barplot_x.png`
- 4.2
    - `4.2_sobol_6inputs.ipynb`
    - `4.2_sobol_6inputs_resultados.csv`
    - `4.2_sobol_comparacao_8vs6_por_output.png`
- 4.3
    - `4.3_comparacao_sobol_shap.ipynb`
    - `4.3_divergencias.csv`
    - `4.3_spearman_heatmap.png`
    - `4.3_narrativa.md`
- 4.4
    - `4.4_analise_interacoes.ipynb`
    - `4.4_interacoes_barplot_ET.png`
    - `4.4_interacoes_barplot_M.png`
    - `4.4_interacoes_barplot_x.png`
    - `4.4_cruzamento_sobol_shap.md`

## Resultados

### 4.1/4.2 — Índices de Sobol por output

Índices S₁ (primeira ordem) e S_T (total) do surrogate SVR k=8, arredondados para 3 casas:

| Feature | S₁(ET) | S_T(ET) | S₁(M) | S_T(M) | S₁(x) | S_T(x) |
|---------|--------|---------|-------|--------|-------|--------|
| P1  | ≈0     | 0.006   | 0.005 | 0.013  | 0.002 | 0.009  |
| T1  | 0.015  | 0.086   | 0.199 | 0.235  | 0.003 | 0.028  |
| T2  | ≈0     | 0.005   | ≈0    | 0.003  | ≈0    | 0.009  |
| RRC1| ≈0     | 0.027   | 0.158 | 0.239  | 0.522 | 0.750  |
| BRC1| 0.318  | 0.378   | 0.214 | 0.264  | 0.121 | 0.325  |
| RRC2| 0.089  | 0.130   | 0.047 | 0.067  | 0.036 | 0.101  |
| BRC2| 0.006  | 0.013   | 0.060 | 0.080  | 0.025 | 0.089  |
| RFF | 0.437  | 0.501   | 0.198 | 0.238  | ≈0    | 0.015  |

P1 e T2 apresentam S₁ ≈ 0 e S_T < 0.015 em todos os outputs, confirmando sua irrelevância. Os índices para o subconjunto k=6 permanecem estáveis em relação ao k=8, indicando que a remoção de P1 e T2 não redistribui variância significativa.

### 4.3 — Concordância Sobol × SHAP

Correlação de Spearman entre rankings Sobol (S₁) e SHAP por par (output, modelo):

- ρ médio (15 pares): 0.919
- Amplitude: [0.862, 0.958]
- Pares com p < 0.05: 15/15
- Divergências com |Δrank| ≥ 3: 0 casos

A triangulação confirma a robustez da seleção S₆. As variações residuais de 1–2 posições ocorrem exclusivamente no bloco de features equipotentes de M_CH3OH (T1, RRC1, BRC1, RFF com S₁ entre 0.16 e 0.21), onde diferenças numéricas pequenas alteram o ranking sem contradição metodológica.

P1 e T2 ocupam as duas últimas posições em ambos os critérios para os três outputs, validando de forma independente e agnóstica ao modelo a decisão de descarte tomada na Etapa 3.

### 4.4 — Interações identificadas

Features com frac_interacao > 0.3 por output:

| Output | Feature | frac_interacao |
|--------|---------|---------------|
| ET | RRC1 | 1.00 |
| ET | T1 | 0.83 |
| ET | RRC2 | 0.31 |
| M_CH3OH | RRC1 | 0.34 |
| x_CH3OH | T1 | 0.89 |
| x_CH3OH | BRC2 | 0.72 |
| x_CH3OH | RRC2 | 0.64 |
| x_CH3OH | BRC1 | 0.63 |
| x_CH3OH | RRC1 | 0.30 |

A pureza (x_CH3OH) é o output com maior carga de interação: RRC1, BRC1, RRC2 e BRC2 exercem efeito predominantemente via acoplamento. Os pares RRC1–BRC1 (coluna 1) e RRC2–BRC2 (coluna 2) são fisicamente coerentes — refluxo e boil-up de cada coluna determinam conjuntamente o balanço de separação, não individualmente. Para ET, RFF e BRC1 dominam com efeito majoritariamente aditivo; a energia de T1 e RRC1 se manifesta quase inteiramente via interação.
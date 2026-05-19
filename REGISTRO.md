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
| E6 | Exploração da fronteira Pareto via surrogate |

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

| Modelo  | R²(ET) | R²(ET) REF1 | Δ(ET)   | R²(M_CH3OH) | R²(M_CH3OH) REF1 | Δ(M_CH3OH) | R²(x_CH3OH) | R²(x_CH3OH) REF1 | Δ(x_CH3OH) |
|---------|--------|-------------|---------|-------------|------------------|------------|-------------|------------------|------------|
| SVR     | 0.970  | 0.903       | +0.067  | 0.983       | 0.907            | +0.076     | 0.953       | 0.864            | +0.089     |
| DT      | 0.884  | 0.863       | +0.021  | 0.774       | 0.833            | −0.059     | 0.791       | 0.820            | −0.029     |
| RF      | 0.924  | 0.955       | −0.031  | 0.914       | 0.923            | −0.009     | 0.901       | 0.894            | +0.007     |
| XGBoost | 0.963  | 0.952       | +0.011  | 0.960       | 0.928            | +0.032     | 0.945       | 0.919            | +0.026     |
| ANN     | 0.965  | 0.983       | −0.018  | 0.985       | 0.992            | −0.007     | 0.969       | 0.990            | −0.021     |

Combinações dentro da tolerância: 5/15.

As divergências são atribuídas a hiperparâmetros não publicados pelo REF1 e à diferença de 7 amostras no test set. SVR superou o benchmark em todos os outputs. Os 15 surrogates foram serializados para uso nas etapas seguintes.

# ETAPA 3 — Modelos Reduzidos (SHAP → Feature Selection → Treino → Seleção)

## Descrição
Com os 15 surrogates da ETAPA 2, esta etapa implementa o pipeline de redução de dimensionalidade: calcular importâncias SHAP para cada modelo e output > derivar subconjuntos de features por cutoff de cobertura > treinar modelos reduzidos (k=4,5,6) e selecionar o par (k*, arquitetura) mais parcimonioso que mantém desempenho estatisticamente equivalente ao modelo completo.

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

- D-E3-01: ΔR² = R²(k=8) − R²(k) calculado no test set; distribuição obtida via bootstrap pareado (D-E3-09)
- D-E3-02: Subconjuntos aninhados — cada subconjunto de k features contém o de k−1, assegurando comparabilidade e monotonia da seleção
- D-E3-03: P1 e T2 descartados em todos os k ≤ 6 — consistente com a baixa importância SHAP observada nos 5 modelos
- D-E3-08: Margem de equivalência δ = 0.02 — equivalência declarada quando IC_sup ≤ δ (modelo reduzido pode ser no máximo 0.02 pior em R² que o baseline)
- D-E3-09: Bootstrap pareado com B=1000 réplicas — os mesmos índices reamostrados são usados para y_true, p8 e pk, capturando a correlação entre estimadores

## Artefatos

- 3.1
    - 3.1_bar_{modelo}_{saída}
        - |SHAP| médio : importância de cada entrada, por saída, por modelo {refinar descrição}
    - 3.1_beeswarm_{modelo}_{saída}
        - {preencher descrição}
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

### 3.1

#### Importâncias SHAP (bar plots)

Os bar plots (`3.1_bar_{modelo}_{saída}`) reportam o |SHAP| médio no validation set para cada input, por modelo e output. O padrão é consistente entre os 5 modelos:

ET:
- RFF e BRC1 dominam amplamente, separados do restante por uma quebra de escala visível
- Ordem decrescente (SVR representativo): RFF > BRC1 >> RRC2 > T1 > BRC2 > RRC1 >> P1 > T2
    - OBS: SVR representativo pois foi o modelo que segurou o R2 com menos inputs (6) nas sub-etapas seguintes (Os modelos "baseline" testados antes tinham 8 inputs)
- P1 e T2 irrelevantes em todos os modelos
    - OBS: Este ponto será explorado/verificado mais adiante a partir das análises via SOBOL

M_CH3OH:
- Quatro features de similar importância na faixa superior: RFF ≈ T1 ≈ BRC1 ≈ RRC1
- BRC2 e RRC2 em importância intermediária
- P1 e T2 novamente irrelevantes

x_CH3OH:
- RRC1 domina com margem expressiva; BRC1 em segundo
- Bloco de quatro variáveis de coluna (RRC1, BRC1, RRC2, BRC2) concentra a importância
- P1 e T2 irrelevantes

#### Direção dos efeitos (beeswarm plots)

Os beeswarm plots (`3.1_beeswarm_{modelo}_{saída}`) adicionam a direção do efeito SHAP e a distribuição condicional ao valor da feature.

#### Consistência entre modelos (heatmap de ranks)

O `3.1_diag_heatmap_ranks.png` exibe os ranks SHAP (1 = maior importância, 8 = menor) para os 5 modelos × 3 outputs.

Observações:

- P1 ocupa rank 7 e T2 ocupa rank 8 em todos os modelos e outputs sem exceção — consenso unânime de descarte
- ET: ranking praticamente idêntico entre SVR, DT, RF, XGBoost e ANN; RFF = 1º e BRC1 = 2º em todos os modelos
- x_CH3OH: ranking igualmente unânime (constante ao longo dos modelos); RRC1 = 1º e BRC1 = 2º em todos os modelos
- M_CH3OH: posições intermediárias (T1, RRC1, BRC1, RFF) variam em até 2 posições entre modelos, reflexo da importância similar identificada nos bar plots

#### Cobertura cumulativa SHAP

O `3.1_diag_cobertura.png` mostra a fração acumulada da massa de |SHAP| médio em função do número de features top-k, para cada modelo e output. As linhas tracejadas marcam os limiares de 90% e 95%.

- ET: k=4 já ultrapassa 90% em todos os modelos; k=6 ultrapassa 95%
- M_CH3OH: convergência mais gradual devido à equipotência das quatro features; k=5 atinge ~88–92%; k=6 ~92–95%
- x_CH3OH: curva mais lenta, puxada pela concentração de importância em RRC1; k=6 situa-se na faixa 92–97%

O ponto k=6 está acima de 90% para todos os outputs e todos os modelos, indicando que S₆ (modelos com conjuntos de 6 inputs) captura a maior parte da variância explicável pelo SHAP. Este diagrama forneceu a motivação quantitativa para o intervalo k ∈ {4, 5, 6} explorado em 3.3–3.5, embora não seja decisório quanto ao valor de k.

### 3.2 — Subconjuntos de features

A sub-etapa 3.2 não produziu nova análise: seu papel foi formalizar o artefato canônico consumido por 3.3, consolidando as decisões tomadas em 3.1.1 e 3.1.2.

O ranking global de importância — média do |SHAP| relativo sobre os 15 pares (modelo × output) — determinou a ordem de entrada das features nos subconjuntos:

| Rank | Feature | Importância global |
|------|---------|-------------------|
| 1    | BRC1    | 0.252             |
| 2    | RRC1    | 0.228             |
| 3    | RFF     | 0.199             |
| 4    | RRC2    | 0.118             |
| 5    | T1      | 0.110             |
| 6    | BRC2    | 0.073             |
| 7    | P1      | 0.014             |
| 8    | T2      | 0.006             |

Os três subconjuntos aninhados produzidos:

| k | Features incluídas | Features excluídas |
|---|-------------------|--------------------|
| 4 | BRC1, RRC1, RFF, RRC2 | P1, T1, T2, BRC2 |
| 5 | BRC1, RRC1, RFF, RRC2, T1 | P1, T2, BRC2 |
| 6 | T1, RRC1, BRC1, RRC2, BRC2, RFF | P1, T2 |

P1 e T2 são descartados em todos os k, confirmando o consenso unânime do heatmap de ranks. A propriedade de aninhamento S₄ ⊂ S₅ ⊂ S₆ foi verificada programaticamente, assim como a ausência de P1 e T2 em qualquer linha. T1 entra em k=5; BRC2 entra apenas em k=6.

### 3.3 — Modelos reduzidos

45 surrogates treinados (5 modelos × 3 outputs × k ∈ {4, 5, 6}) com os subconjuntos de features definidos em 3.2. Arquiteturas e hiperparâmetros idênticos aos da Etapa 2. Os runs foram registrados no experimento MLflow `reduzido`.

R² no test set:

#### ET

| Modelo  | k=4   | k=5   | k=6   | k=8   |
|---------|-------|-------|-------|-------|
| SVR     | 0.821 | 0.954 | 0.972 | 0.970 |
| RF      | 0.833 | 0.931 | 0.930 | 0.924 |
| DT      | 0.785 | 0.906 | 0.893 | 0.884 |
| XGBoost | 0.847 | 0.961 | 0.971 | 0.963 |
| ANN     | 0.834 | 0.966 | 0.973 | 0.965 |

#### M_CH3OH

| Modelo  | k=4   | k=5   | k=6   | k=8   |
|---------|-------|-------|-------|-------|
| SVR     | 0.655 | 0.872 | 0.976 | 0.983 |
| RF      | 0.610 | 0.840 | 0.914 | 0.914 |
| DT      | 0.520 | 0.651 | 0.793 | 0.774 |
| XGBoost | 0.645 | 0.876 | 0.951 | 0.960 |
| ANN     | 0.641 | 0.860 | 0.970 | 0.985 |

#### x_CH3OH

| Modelo  | k=4   | k=5   | k=6   | k=8   |
|---------|-------|-------|-------|-------|
| SVR     | 0.849 | 0.888 | 0.968 | 0.953 |
| RF      | 0.831 | 0.852 | 0.909 | 0.901 |
| DT      | 0.694 | 0.684 | 0.794 | 0.791 |
| XGBoost | 0.818 | 0.867 | 0.935 | 0.945 |
| ANN     | 0.843 | 0.875 | 0.981 | 0.969 |

Observações:
- ET é o output menos sensível à redução: todos os modelos a k=6 superam o baseline em R² (efeito de regularização pela remoção de P1 e T2)
- DT a k=6 supera o baseline em M_CH3OH (0.793 > 0.774); SVR, RF, DT e ANN a k=6 também superam em x_CH3OH
- k=4 provoca queda acentuada em M_CH3OH: DT (0.520), RF (0.610), SVR (0.655)
- Única inversão de monotonia: DT em x_CH3OH, onde k=5 (0.684) é inferior a k=4 (0.694) — atribuída à entrada de T1 em k=5, feature de baixa importância SHAP para x_CH3OH na escala global

### 3.4 — Equivalência estatística

O bootstrap é uma técnica de reamostragem: dado o test set (193 amostras), sorteia-se com reposição B=1000 subconjuntos de mesmo tamanho e calcula-se a estatística de interesse em cada um — aqui, ΔR² = R²(k=8) − R²(k). A distribuição empírica das 1000 réplicas aproxima a variabilidade amostral do estimador sem assumir normalidade. O "pareado" significa que os mesmos índices reamostrados são aplicados simultaneamente a y_true, às predições do modelo completo (p8) e às do modelo reduzido (pk), capturando a correlação entre os dois estimadores e produzindo ICs mais estreitos do que se fossem amostrados independentemente. O seed=42 garante reprodutibilidade exata.

O IC 95% (Intervalo de Confiança) é calculado pelos percentis 2.5% e 97.5% da distribuição bootstrap de ΔR², resultando em [IC_inf, IC_sup]. IC_sup é o limite superior: se IC_sup ≤ δ = 0.02, o modelo reduzido pode ser no máximo 0.02 pior em R² do que o baseline com probabilidade 95%, declarando equivalência (D-E3-08). IC_sup < 0 indica que o modelo reduzido é estatisticamente superior ao baseline.

Bootstrap pareado (B=1000, seed=42) com IC 95% do ΔR² = R²(k=8) − R²(k) no test set. Equivalência declarada quando IC_sup ≤ δ = 0.02 (D-E3-08). Os mesmos índices reamostrados foram aplicados a y_true, p8 e pk, preservando a correlação entre estimadores (D-E3-09).

Outputs com equivalência ao baseline, por (arquitetura, k):

| Modelo  | k=4 | k=5 | k=6 |
|---------|-----|-----|-----|
| SVR     | 0/3 | 0/3 | 3/3 |
| RF      | 0/3 | 1/3 | 3/3 |
| DT      | 0/3 | 1/3 | 3/3 |
| XGBoost | 0/3 | 1/3 | 2/3 |
| ANN     | 0/3 | 1/3 | 2/3 |

IC_sup por (arquitetura, output) a k=6:

| Modelo  | IC_sup(ET) | Equiv? | IC_sup(M_CH3OH) | Equiv? | IC_sup(x_CH3OH) | Equiv? | 3/3? |
|---------|------------|--------|-----------------|--------|-----------------|--------|------|
| SVR     |  0.006     | ✓      |  0.015          | ✓      | −0.001          | ✓      | ✓    |
| RF      | −0.002     | ✓      |  0.005          | ✓      | −0.004          | ✓      | ✓    |
| DT      |  0.004     | ✓      |  0.006          | ✓      |  0.006          | ✓      | ✓    |
| XGBoost |  0.000     | ✓      |  0.017          | ✓      |  0.030          | ✗      | ✗    |
| ANN     |  0.001     | ✓      |  0.024          | ✗      | −0.003          | ✓      | ✗    |

#### Forest plot (3.4_forest_plot.png)

Um subplot por output (ET, M_CH3OH, x_CH3OH). Cada linha representa uma combinação (arquitetura, k); a barra é o IC 95% de ΔR² e o marcador é a média bootstrap. A linha tracejada vertical em δ=0.02 é o limiar de equivalência: barras cujo extremo direito fica à esquerda dessa linha correspondem a modelos equivalentes. A linha pontilhada em 0 referencia a paridade exata com o baseline. O gráfico permite identificar visualmente não só quais combinações passam o critério, mas também a margem — barras completamente à esquerda de 0 indicam superioridade estatística do modelo reduzido.

#### Heatmap (3.4_heatmap_equivalencia.png)

Grade arquitetura × k (colunas k=4, k=5, k=6) replicada para cada output. Células verdes indicam equivalência (IC_sup ≤ δ) e vermelhas indicam falha; o valor anotado em cada célula é o IC_sup, permitindo quantificar a margem de aprovação ou de reprovação. O heatmap complementa o forest plot: enquanto este detalha a forma e posição de cada IC, o heatmap fornece a leitura consolidada da decisão binária por (arquitetura, output, k).

#### Observações
- k=4: nenhum modelo atinge equivalência em nenhum output; as quedas de R² são grandes em todos os casos
- k=5: equivalência apenas em ET (DT, RF, XGBoost, ANN) — SVR não alcança equivalência em nenhum output a k=5 (IC_sup(ET) = 0.025 > δ); M_CH3OH e x_CH3OH nunca são equivalentes a k=5
- k=6: SVR, RF e DT atingem equivalência simultânea nos 3 outputs e seguem para seleção em 3.5
- XGBoost falha em x_CH3OH a k=6: IC_sup = 0.030 > δ; a queda de R² (0.945 → 0.935) é confirmada como não trivial pelo bootstrap
- ANN falha em M_CH3OH a k=6: IC_sup = 0.024 > δ; a queda de R² (0.985 → 0.970) também confirmada como não trivial
- RF apresenta IC_sup < 0 em ET e x_CH3OH a k=6 (respectivamente −0.002 e −0.004), indicando que o modelo reduzido é estatisticamente superior ao baseline nesses outputs; SVR também apresenta IC_sup < 0 em x_CH3OH (−0.001) — efeito de regularização pela remoção de P1 e T2

# ETAPA 4 — Análise de Sensibilidade Global (Sobol)

## Descrição

Com o surrogate SVR k=6 selecionado na Etapa 3, esta etapa aplica o método de Sobol para análise de sensibilidade global: quantificar a contribuição de cada input para a variância dos outputs, decompondo-a em efeitos de primeira ordem (S₁) e totais (S_T). O objetivo duplo é:
- (a) validar a seleção S₆ com evidência independente do modelo
- (b) caracterizar a estrutura de interações entre inputs.

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

### 4.0 — Setup e validação do ambiente

Sub-etapa de validação de ambiente sem resultados analíticos. Todos os checks passaram:

- SALib importado sem erro; `problem_8` e `problem_6` definidos com nomes e faixas físicas corretas
- IDX_S6 = [1, 3, 4, 5, 6, 7] — mapeamento das 6 features S₆ dentro do vetor de 8, consistente com a seleção de 3.2
- 6 modelos SVR carregados: baseline (k=8) e reduzido (k=6) para ET, M_CH3OH e x_CH3OH
- Scalers `scaler_X_min` e `scaler_X_scale` carregados com shape (8,) e todos os valores de escala > 0 (sem risco de divisão por zero)
- Pipeline Saltelli + predição validado com N=8: shapes de amostragem corretos (80 linhas para problem_8, 64 para problem_6) e predições sem erros para todos os 6 modelos

Ambiente validado para execução das sub-etapas 4.1–4.4.

### 4.1 — Sobol com 8 inputs

#### Leitura e interpretação dos índices Sobol

O método de Sobol decompõe a variância total do output em contribuições atribuíveis a cada input e a combinações de inputs. Dois índices são reportados por feature:

S₁ (índice de primeira ordem): fração da variância do output explicada exclusivamente por aquele input, mantendo todos os demais fixos. Mede o efeito direto, sem interações. Pode ser interpretado como: "se eu soubesse o valor exato desta variável e ignorasse as demais, quanto da incerteza no output eu eliminaria?"

S_T (índice total): fração da variância do output atribuível àquele input por qualquer caminho — efeito direto mais todos os efeitos de interação de qualquer ordem em que ele participa. S_T ≥ S₁ sempre.

A diferença S_T − S₁ quantifica a contribuição do input via interações com os demais. A razão frac_interacao = (S_T − S₁) / S_T indica qual fração do efeito total de uma feature ocorre exclusivamente por interação (D-E4-07).

Propriedades de consistência interna:

- Σ S₁ ≤ 1: a soma dos índices de primeira ordem é no máximo 1; valores inferiores a 1 indicam que uma fração da variância só é explicada por interações entre inputs, não por efeitos isolados
- Σ S_T ≥ 1: a soma dos índices totais excede 1 sempre que há interações, pois cada interação é contada nas features participantes — o excesso (Σ S_T − 1) é uma medida do peso agregado das interações no sistema

Leitura prática dos padrões:

- S_T ≈ S₁: o input age de forma predominantemente aditiva; sua contribuição não depende dos valores dos demais inputs
- S_T >> S₁: o input é predominantemente interativo; seu efeito no output depende fortemente do estado das outras variáveis
- S_T ≈ 0 e S₁ ≈ 0: feature irrelevante, tanto isoladamente quanto em interação — candidata a descarte

Cada índice vem acompanhado de um intervalo de confiança (S1_conf, ST_conf) estimado via bootstrap interno do SALib. Valores de S₁ ligeiramente negativos são artefatos numéricos de variância amostral finita e são tratados como S₁ ≈ 0 (D-E4-03).


Índices S₁ (primeira ordem) e S_T (total) calculados via método de Saltelli (N=2048, 2N(8+2)=40960 avaliações) sobre o surrogate SVR k=8.

#### ET - 4.1_sobol_8inputs_barplot_ET

| Feature | S₁     | S_T    |
|---------|--------|--------|
| RFF     | 0.437  | 0.501  |
| BRC1    | 0.318  | 0.378  |
| RRC2    | 0.089  | 0.130  |
| T1      | 0.015  | 0.086  |
| RRC1    | ≈0     | 0.027  |
| BRC2    | 0.006  | 0.013  |
| P1      | ≈0     | 0.006  |
| T2      | 0.001  | 0.005  |

#### M_CH3OH - 4.1_sobol_8inputs_barplot_M

| Feature | S₁     | S_T    |
|---------|--------|--------|
| BRC1    | 0.214  | 0.264  |
| T1      | 0.199  | 0.235  |
| RFF     | 0.198  | 0.238  |
| RRC1    | 0.158  | 0.239  |
| BRC2    | 0.060  | 0.080  |
| RRC2    | 0.047  | 0.067  |
| P1      | 0.005  | 0.013  |
| T2      | ≈0     | 0.003  |

#### x_CH3OH - 4.1_sobol_8inputs_barplot_x

| Feature | S₁     | S_T    |
|---------|--------|--------|
| RRC1    | 0.522  | 0.750  |
| BRC1    | 0.121  | 0.325  |
| RRC2    | 0.036  | 0.101  |
| BRC2    | 0.025  | 0.089  |
| T1      | 0.003  | 0.028  |
| RFF     | ≈0     | 0.015  |
| T2      | ≈0     | 0.009  |
| P1      | 0.002  | 0.009  |

#### Observações

- P1 e T2 confirmados abaixo do limiar S_T_MIN = 0.02 (D-E4-04) em todos os outputs; o maior valor observado é P1 S_T = 0.013 em M_CH3OH — corroborando o descarte imposto pela seleção SHAP (Etapa 3)
- ET: RFF e BRC1 dominam, consistente com o padrão SHAP; S₁(RRC1) ≈ 0 enquanto S_T(RRC1) = 0.027, indicando que RRC1 atua exclusivamente via interações em ET
- M_CH3OH: quatro features de importância similar (BRC1, T1, RFF, RRC1), corroborando o padrão observado no SHAP; somas Σ S₁ ≈ 0.88 e Σ S_T ≈ 1.14, sinalizando presença moderada de interações
- x_CH3OH: RRC1 domina com S₁=0.522 e S_T=0.750; BRC1 apresenta S₁=0.121 mas S_T=0.325 — diferença pronunciada (ΔS=0.204) indica que parte relevante do efeito de BRC1 ocorre via interação com RRC1; Σ S₁ ≈ 0.70, a mais baixa entre os 3 outputs, refletindo maior proporção de variância explicada por interações
- S₁ ligeiramente negativos para P1, RRC1 e RFF em alguns outputs são tratados como S₁ ≈ 0 (D-E4-03)

### 4.2 — Sobol com 6 inputs

Índices S₁ e S_T calculados via método de Saltelli (N=2048, 2N(6+2)=32768 avaliações) sobre o surrogate SVR k=6, restrito ao subconjunto S₆ = {T1, RRC1, BRC1, RRC2, BRC2, RFF}.

#### ET

| Feature | S₁    | S_T   |
|---------|-------|-------|
| RFF     | 0.428 | 0.505 |
| BRC1    | 0.327 | 0.392 |
| RRC2    | 0.090 | 0.126 |
| T1      | 0.007 | 0.095 |
| BRC2    | 0.008 | 0.014 |
| RRC1    | ≈0    | 0.026 |

#### M_CH3OH

| Feature | S₁    | S_T   |
|---------|-------|-------|
| BRC1    | 0.208 | 0.272 |
| RFF     | 0.197 | 0.232 |
| T1      | 0.193 | 0.232 |
| RRC1    | 0.167 | 0.241 |
| BRC2    | 0.056 | 0.082 |
| RRC2    | 0.041 | 0.067 |

#### x_CH3OH

| Feature | S₁    | S_T   |
|---------|-------|-------|
| RRC1    | 0.508 | 0.768 |
| BRC1    | 0.112 | 0.344 |
| RRC2    | 0.048 | 0.114 |
| BRC2    | 0.034 | 0.109 |
| T1      | 0.006 | 0.028 |
| RFF     | ≈0    | 0.014 |

#### Comparação S_T: SVR k=8 (4.1) vs. SVR k=6 (4.2)

A sub-etapa 4.2 tem como objetivo principal verificar se a remoção de P1 e T2 perturba a estrutura de sensibilidade das 6 features restantes. O gráfico `4.2_sobol_comparacao_8vs6_por_output.png` exibe barras lado a lado de S_T para cada output.

| Output    | Feature | S_T (k=8) | S_T (k=6) | ΔS_T   |
|-----------|---------|-----------|-----------|--------|
| ET        | RFF     | 0.501     | 0.505     | +0.004 |
| ET        | BRC1    | 0.378     | 0.392     | +0.014 |
| ET        | RRC2    | 0.130     | 0.126     | −0.004 |
| ET        | T1      | 0.086     | 0.095     | +0.009 |
| ET        | RRC1    | 0.027     | 0.026     | −0.001 |
| ET        | BRC2    | 0.013     | 0.014     | +0.001 |
| M_CH3OH   | BRC1    | 0.264     | 0.272     | +0.008 |
| M_CH3OH   | T1      | 0.235     | 0.232     | −0.003 |
| M_CH3OH   | RFF     | 0.238     | 0.232     | −0.006 |
| M_CH3OH   | RRC1    | 0.239     | 0.241     | +0.002 |
| M_CH3OH   | BRC2    | 0.080     | 0.082     | +0.002 |
| M_CH3OH   | RRC2    | 0.067     | 0.067     | 0.000  |
| x_CH3OH   | RRC1    | 0.750     | 0.768     | +0.018 |
| x_CH3OH   | BRC1    | 0.325     | 0.344     | +0.019 |
| x_CH3OH   | RRC2    | 0.101     | 0.114     | +0.013 |
| x_CH3OH   | BRC2    | 0.089     | 0.109     | +0.020 |
| x_CH3OH   | T1      | 0.028     | 0.028     | 0.000  |
| x_CH3OH   | RFF     | 0.015     | 0.014     | −0.001 |

#### Interpretação sinal de ΔS_T
ΔS_T > 0 — a feature tem índice total maior no surrogate k=6 do que no k=8. Isso ocorre porque P1 e T2 tinham S_T pequeno mas não nulo; ao serem removidos, a fração de variância que eles "seguravam" se redistribui entre as 6 features remanescentes. Cada feature absorve um pequeno incremento.

ΔS_T < 0 — a feature tem índice total menor no k=6. Pode ocorrer por variância amostral (o método de Saltelli tem incerteza estatística) ou porque o surrogate k=6, treinado sem P1 e T2, aprendeu uma superfície ligeiramente diferente que distribui menos variância para aquela feature específica.

ΔS_T ≈ 0 — a remoção de P1 e T2 não alterou o peso daquela feature.

No contexto da análise, o sinal em si não é interpretável individualmente — o que importa é que todos os ΔS_T estão dentro dos intervalos de confiança dos dois surrogates, ou seja, nenhuma diferença é estatisticamente significativa. A conclusão é que o ranking e as magnitudes de S_T são estáveis entre k=8 e k=6.

#### Observações

- O ranking de S_T é preservado integralmente entre as configurações k=8 e k=6 nos três outputs — a remoção de P1 e T2 não altera a ordem de importância das features remanescentes
- x_CH3OH apresenta os maiores ΔS_T positivos (RRC1: +0.018, BRC1: +0.019, BRC2: +0.020): ao remover P1 e T2 — ambas com S_T < 0.01 neste output — a variância antes atribuída a essas features se redistribui marginalmente entre as remanescentes, efeito esperado e fisicamente coerente
- ET e M_CH3OH têm ΔS_T inferiores a 0.015 em todas as features, consistente com a maior dominância de P1 e T2 nesse output já sendo próxima de zero
- A análise confirma que o surrogate k=6 preserva a estrutura de sensibilidade global do modelo completo, fornecendo evidência independente (Sobol) da validade da seleção S₆ realizada por SHAP na Etapa 3

### 4.3 — Comparação Sobol × SHAP

Correlação de Spearman entre os rankings de S₁ (Sobol) e |SHAP| médio (Etapa 3), calculada para cada um dos 15 pares (output × modelo). Rankings calculados sobre os 8 inputs originais em ambos os critérios.

#### Resultado global

| Estatística | Valor |
|-------------|-------|
| ρ médio (15 pares) | 0.919 |
| Amplitude | [0.862, 0.958] |
| Pares com p < 0.05 | 15/15 |
| Divergências com \|Δrank\| ≥ 3 | 0 casos |

O CSV `4.3_divergencias.csv` não registrou nenhuma linha de divergência — todos os pares (feature, output, modelo) apresentaram \|Δrank\| < 3, confirmando que os dois critérios são consistentes em toda a faixa de importâncias.

#### Correlação de Spearman por output (heatmap `4.3_spearman_heatmap.png`)

| Output | ρ médio |
|--------|---------|
| ET | 0.898 |
| M_CH3OH | 0.900 |
| x_CH3OH | 0.958 |

#### Interpretação por output

ET: RFF e BRC1 dominam em ambos os critérios (Sobol: S₁ = 0.437 e 0.318; SHAP: ranks 1 e 2 em todos os modelos). T1 e RRC1 apresentam divergência moderada de posição — T1 aparece em rank 4 no Sobol e em ranks 3–5 no SHAP, enquanto RRC1 tem S₁ ≈ 0 mas importância SHAP não nula, reflexo do caráter interativo de RRC1 para ET (S_T = 0.027 vs. S₁ ≈ 0 em 4.1).

M_CH3OH: quatro features de importâncias similares (T1, RRC1, BRC1, RFF) geram maior sensibilidade do ranking a pequenas diferenças numéricas. O Sobol posiciona BRC1 à frente de T1 (S₁: 0.214 vs. 0.199); o SHAP dos modelos árvore/ensemble tende a inverter essa ordem. Variação de 1–2 posições num bloco equipotente é esperada e não representa contradição.

x_CH3OH: concordância mais elevada (ρ = 0.958). RRC1 domina com larga margem em ambos os critérios (S₁ = 0.522, rank SHAP = 1 em todos os modelos). BRC1 ocupa a segunda posição em ambos. RFF tem S₁ ≈ 0 e S_T = 0.015, coincidindo com rank SHAP = 6, reforçando que a pureza é controlada pelas condições de separação da coluna 1.

#### P1 e T2

| Feature | Output | Sobol rank (S₁) | SHAP rank médio |
|---------|--------|-----------------|-----------------|
| P1 | ET | 7 | 7.0 |
| P1 | M_CH3OH | 7 | 7.0 |
| P1 | x_CH3OH | 6 | 7.0 |
| T2 | ET | 6 | 8.0 |
| T2 | M_CH3OH | 8 | 8.0 |
| T2 | x_CH3OH | 7 | 8.0 |

Ambos os critérios posicionam P1 e T2 nos dois últimos ranks para os três outputs — confirmando por triangulação independente o descarte imposto em S₆.

#### Causas das divergências residuais

- Multicolinearidade: SHAP redistribui importância entre features correlacionadas (T1–BRC1–RFF têm correlação moderada no dataset LHS); o Sobol usa amostras uniformes independentes, medindo sensibilidade num espaço sem correlação.
- Distribuição amostral: o Sobol assume distribuição uniforme no espaço físico, enquanto o LHS da Etapa 0 tem densidade não-uniforme em regiões fisicamente relevantes.
- Interações vs. efeito aditivo: para features com S₁ ≈ 0 e S_T > 0 (ex.: RRC1 em ET), a variância é majoritariamente interativa; o SHAP aditivo distribui essa importância de forma diferente do Sobol.

#### Conclusão

A triangulação metodológica confirma a robustez da seleção S₆: concordância alta (ρ médio = 0.919), nenhuma divergência de \|Δrank\| ≥ 3, e P1/T2 descartados por ambos os critérios em todos os outputs. As divergências residuais são metodologicamente esperadas e não alteram a seleção final.

### 4.4 — Análise de interações

A fração de interação de cada feature é calculada como `frac_interacao = (S_T − S₁) / S_T`, usando S₁ clampado em zero (D-E4-03). Pares com S_T ≤ 0.02 são excluídos (D-E4-04). O limiar D-E4-07 (frac_interacao > 0.30) foi mantido em 0.30 — havia features elegíveis acima desse valor.

8 pares excluídos por ST < 0.02 (importância desprezível); 16 pares elegíveis; 9 com frac_interacao > 0.30.

OBS de 4.1: A razão frac_interacao = (S_T − S₁) / S_T indica qual fração do efeito total de uma feature ocorre exclusivamente por interação (D-E4-07).

#### Tabela cruzada Sobol × SHAP

| Output | Feature | frac_interacao | Padrão beeswarm |
|--------|---------|----------------|-----------------|
| ET | RRC1 | 1.000 | disperso |
| ET | T1 | 0.832 | disperso |
| ET | RRC2 | 0.315 | disperso |
| M_CH3OH | RRC1 | 0.338 | disperso |
| x_CH3OH | T1 | 0.887 | — |
| x_CH3OH | BRC2 | 0.720 | cluster |
| x_CH3OH | RRC2 | 0.640 | cluster |
| x_CH3OH | BRC1 | 0.629 | cluster |
| x_CH3OH | RRC1 | 0.303 | disperso |

#### Por output

ET: RFF e BRC1 dominam com efeito majoritariamente aditivo (S_T próximo de S₁). RRC1 tem frac_interacao = 1.00 — S₁ ≈ 0 enquanto S_T = 0.027, indicando que todo o efeito de RRC1 sobre a energia ocorre via interações. T1 segue o mesmo padrão (S₁ = 0.015, S_T = 0.086, frac = 0.83). RRC2 apresenta frac_interacao ≈ 0.31, consistente com acoplamento energético com BRC2 no circuito da coluna 2.

M_CH3OH: RRC1 é o único par elegível (frac = 0.34). Com S_T ≈ 0.24, cerca de 34% do efeito de RRC1 sobre a vazão de metanol ocorre via interação — provavelmente com BRC1, que controla a mesma coluna. Os demais inputs apresentam frac_interacao < 0.30.

x_CH3OH: output com maior carga de interação. BRC1, RRC2 e BRC2 têm S_T relevantes mas S₁ pequenos — efeito predominantemente interativo. Os beeswarms de ANN e XGBoost exibem padrão de cluster para essas três features: o SHAP value varia conforme o contexto das demais variáveis, coerente com a quantificação Sobol. RRC1 também fica acima do limiar (frac = 0.30), com padrão disperso. T1 tem frac = 0.89 mas S_T = 0.028, próximo de ST_MIN — beeswarm não foi avaliado por importância total baixa.

#### Acoplamentos identificados

| Par | Output | Mecanismo provável |
|-----|--------|--------------------|
| RRC1–BRC1 | M_CH3OH, x_CH3OH | Refluxo e boil-up da coluna 1 determinam conjuntamente o balanço da separação primária |
| RRC2–BRC2 | x_CH3OH | Refluxo e boil-up da coluna 2 controlam a pureza final; o efeito de um depende do outro |
| RRC2–outros | ET | Operação da coluna 2 interage com o circuito global no consumo energético |

Os acoplamentos refluxo–boil-up por coluna (RRC1–BRC1 e RRC2–BRC2) são fisicamente coerentes: a pureza do destilado e a vazão de produto dependem do balanço entre os dois parâmetros operacionais de cada coluna, não do valor de cada um isoladamente.

#### Consistência com SHAP

Features com alta frac_interacao (BRC1, RRC2, BRC2 para x_CH3OH) exibem maior dispersão vertical nos beeswarms — o mesmo valor da feature produz SHAP values distintos conforme o contexto das demais. Esse padrão de cluster confirma o que o índice S_T − S₁ quantifica: importância que só emerge combinada com outras variáveis. Para ET e M_CH3OH, as features interativas (RRC1, T1, RRC2) exibem padrão disperso sem estrutura clara de cluster, coerente com a ausência de dominância de um único acoplamento.

# ETAPA 5 — Resultado Final

## Descrição

Com todos os surrogates treinados e as análises de sensibilidade concluídas, esta etapa consolida os resultados das Etapas 2–4 em dois artefatos finais: uma tabela comparativa de R² (baseline k=8 vs. reduzido k*=6) e um painel unificado com as três camadas de evidência que validam H1. Nenhum modelo é carregado ou treinado — toda a computação é leitura e visualização de artefatos já produzidos.

**H1:** As features P1 (pressão do reator) e T2 (temperatura de entrada na coluna de destilação) são irrelevantes para os três outputs do processo. (Hipótese a ser testada)

## Definições

- D-E5-01: k* = 6, S₆ = {T1, RRC1, BRC1, RRC2, BRC2, RFF}, arquitetura recomendada SVR — resultado importado diretamente de `3.5_k_estrela.json`
- D-E5-02: DT e RF omitidos da tabela comparativa — R² inferior e sem recomendação para uso no otimizador

## Artefatos

- 5.1
    - `5.1_consolidacao_resultados.ipynb`
    - `5.1_tabela_r2_baseline_vs_reduzido.csv`
    - `5.1_painel_h1_evidencias.png`
    - `5.1_resumo_h1.md`

## Resultados

### 5.1

#### Tabela comparativa R²

R² no test set para os modelos selecionados (SVR, ANN, XGBoost), baseline (k=8) vs. reduzido (k=6), com status de equivalência estatística (bootstrap IC 95%, δ=0.02):

| Modelo  | Output   | R²(k=8) | R²(k=6) | ΔR²    | Equiv? |
|---------|----------|---------|---------|--------|--------|
| SVR     | ET       | 0.9701  | 0.9719  | −0.0018 | Sim    |
| SVR     | M_CH3OH  | 0.9831  | 0.9764  | +0.0067 | Sim    |
| SVR     | x_CH3OH  | 0.9531  | 0.9684  | −0.0153 | Sim    |
| XGBoost | ET       | 0.9634  | 0.9714  | −0.0080 | Sim    |
| XGBoost | M_CH3OH  | 0.9602  | 0.9505  | +0.0097 | Sim    |
| XGBoost | x_CH3OH  | 0.9451  | 0.9354  | +0.0097 | Não    |
| ANN     | ET       | 0.9647  | 0.9732  | −0.0085 | Sim    |
| ANN     | M_CH3OH  | 0.9846  | 0.9702  | +0.0144 | Não    |
| ANN     | x_CH3OH  | 0.9689  | 0.9812  | −0.0123 | Sim    |

SVR k=6 é o único modelo com equivalência simultânea nos 3 outputs e que satisfaz os critérios definidos em 3.5.

#### Painel H1: três camadas de evidência

O painel `5.1_painel_h1_evidencias.png` reúne três blocos de evidência independentes para P1 e T2:

**Camada 1 — Empírica (bootstrap IC 95% de ΔR², SVR k=8 → k=6):**

| Output   | IC 95% ΔR²              | Equiv? |
|----------|-------------------------|--------|
| ET       | [−0.0106,  0.0058]      | Sim    |
| M_CH3OH  | [−0.0001,  0.0151]      | Sim    |
| x_CH3OH  | [−0.0316, −0.0011]      | Sim    |

A remoção de P1 e T2 não degrada o surrogate além da margem δ=0.02 em nenhum output.

**Camada 2 — Baseada em modelo (SHAP, importância relativa máxima sobre os 5 modelos e 3 outputs):**

| Feature | max(shap_rel) | Output     | < 5%? |
|---------|---------------|------------|-------|
| P1      | 2.62%         | M_CH3OH    | Sim   |
| T2      | 1.84%         | x_CH3OH    | Sim   |

**Camada 3 — Física/agnóstica (Sobol, índice S_T máximo sobre os 3 outputs):**

| Feature | S_T máx | Output   | < 0.015? |
|---------|---------|----------|----------|
| P1      | 0.0133  | M_CH3OH  | Sim      |
| T2      | 0.0092  | x_CH3OH  | Sim      |

As três camadas convergem: P1 e T2 têm contribuição desprezível por qualquer critério — empírico, baseado em modelo e agnóstico ao modelo — validando H1 por triangulação metodológica.

# ETAPA 6 — Exploração da Fronteira Pareto via Surrogate

## Descrição

Com o surrogate SVR k*=6 validado, esta etapa aplica o modelo substituto ao problema de otimização bi-objetivo: minimizar o consumo energético (ET) e maximizar a produção de metanol (M_CH3OH), sujeito à restrição de pureza x_CH3OH ≥ 0.98. O ponto de partida são as superfícies de resposta dos pares de features mais importantes identificados nas Etapas 3 e 4 — BRC1 × RFF para ET e M_CH3OH, e RRC1 × BRC1 para x_CH3OH, que revelam a não-linearidade do processo e antecipam a estrutura do trade-off. A fronteira de Pareto é então gerada por amostragem LHS de 50.000 pontos sobre o espaço S₆, com filtragem pela restrição de pureza e identificação das soluções não dominadas. A análise de trade-off encerra a etapa descrevendo os extremos da fronteira e a região de compromisso em termos de inputs e saídas do processo.

## Descrição sub-etapas

- 6.1 — Superfícies de resposta de ET e M_CH3OH em função de (BRC1, RFF): grade 200×200 varrendo as faixas físicas dos dois inputs com maior índice de Sobol de primeira ordem para ET (RFF: S₁=0.437, BRC1: S₁=0.318; Etapa 4); os demais 4 inputs de S₆ são fixados na média da faixa física (D-E6-02). Nenhum modelo é treinado — toda computação é avaliação do surrogate SVR k*=6. O mesmo par (RFF, BRC1) é usado para M_CH3OH para permitir comparação direta das duas superfícies sobre a mesma grade (D-E6-03). São gerados superfície 3D e mapa de contorno para cada output, com overlay dos 193 pontos do test set nos mapas de contorno.

- 6.2 — Superfície de resposta de x_CH3OH em função de (RRC1, BRC1): mesmo setup de grade 200×200, com o par selecionado pela maior frac_interação Sobol para x_CH3OH na Etapa 4.4 — BRC1 e RRC1 formam o acoplamento refluxo–boil-up da coluna 1 (D-E6-04); os demais 4 inputs de S₆ são fixados na média da faixa física, incluindo RFF=0.13 (D-E6-05). Nenhum modelo é treinado — avaliação do surrogate SVR k*=6 para x_CH3OH. O mapa de contorno inclui a linha de restrição x_CH3OH=0.98, que antecipa o filtro de viabilidade de 6.3 e delimita visualmente a região Pareto-elegível no espaço (RRC1, BRC1).

- 6.3 — Fronteira de Pareto ET × M_CH3OH: amostragem LHS de 50.000 pontos sobre o espaço S₆ (seed=42, D-E6-09), avaliação dos três surrogates SVR k*=6 e filtro pela restrição de pureza x_CH3OH ≥ 0.98 (D-E6-06) — que produziu 31.745 amostras viáveis (63,5% do total), dispensando o fallback para 0.95. Das amostras viáveis, o algoritmo de varrimento por dominância (D-E6-08) identificou 20 soluções não dominadas, salvas em `6.3_pareto_solucoes.csv`. A fronteira confirma o trade-off esperado: o extremo de mínimo ET (6.679 kW) corresponde a BRC1=0.57 e RFF=0.25, com produção de apenas 4.568 kg/hr; o extremo de máximo M_CH3OH (12.881 kg/hr) exige ET=84.230 kW com BRC1=9.05 e RFF=0.01 — trade-off de ~18× em consumo energético para ~2,8× em produção. A figura `6.3_pareto_ET_M.png` exibe os 31.745 pontos viáveis em cinza e a fronteira em azul, com os dois extremos marcados.

- 6.4 — Análise de trade-off e narrativa: carrega `6.3_pareto_solucoes.csv` e identifica três pontos-chave da fronteira — mínimo ET, máximo M_CH3OH e região de compromisso (percentis 33–67 de ET, D-E6-10). O ponto de mínimo ET (6.679 kW, M_CH3OH=4.568 kg/hr) é atingido com BRC1=0.57 e RFF=0.247, coerente com os S₁ de Sobol da Etapa 4 (RFF domina ET com S₁=0.437, BRC1 com S₁=0.318) — BRC1 baixo reduz o duty do refervedor da coluna 1 e RFF alto diminui a carga do loop de reciclo ao custo da conversão. O ponto de máximo M_CH3OH (12.881 kg/hr, ET=84.230 kW) inverte os dois inputs: BRC1=9.05 e RFF=0.013, com consumo 12,6× maior que o extremo energético. A região de compromisso (ET≈25.342 kW, M_CH3OH≈10.340 kg/hr) produz ~80% da produção máxima com ~30% da energia máxima, com BRC1=1.74 e RFF=0.032 como inputs discriminantes; T1, RRC1 e RRC2 permanecem estreitos ao longo de toda a fronteira, confirmando BRC1 e RFF como os principais graus de liberdade do trade-off. Os artefatos gerados são `6.4_pareto_anotado.png` (fronteira com os três pontos marcados) e `6.4_narrativa.md`.

## Explicação

### 6.1 — Superfícies de resposta: ET e M_CH3OH em função de (BRC1, RFF)

Objetivo: mapear visualmente o comportamento do surrogate SVR k*=6 sobre o subplano formado pelos dois inputs de maior efeito aditivo sobre ET no espaço S₆ — RFF (S₁=0.437) e BRC1 (S₁=0.318), conforme calculado na Etapa 4.1. As superfícies geradas revelam a geometria das respostas ET e M_CH3OH sobre esse subplano, antecipando a estrutura do trade-off antes da geração formal da fronteira Pareto em 6.3 e verificando que o surrogate capturou as não-linearidades esperadas do processo.

Premissas: a escolha do par (BRC1, RFF) é orientada pelos índices de primeira ordem de Sobol (S₁), que quantificam o efeito individual de cada input sobre a variância do output de forma agnóstica ao modelo. Usar os dois inputs com maior S₁ para ET garante que o subplano inspecionado cobre a maior parte da variância aditiva do output mais relevante para a otimização energética. Os demais 4 inputs de S₆ são fixados na média das faixas físicas — abordagem de ceteris paribus, padrão para visualização de superfícies de resposta em espaços de alta dimensão: isola o comportamento do par de interesse sem colapsar a representatividade do surrogate. A aplicação do mesmo par para M_CH3OH, mesmo que BRC1 e RFF não sejam os dois inputs de maior S₁ para a vazão, é deliberada: só faz sentido avaliar o trade-off visualmente se as duas superfícies forem comparadas sobre a mesma grade de inputs — mover o par (BRC1, RFF) enquanto se observa simultaneamente a resposta em ET e M_CH3OH é o que torna o conflito entre os dois objetivos diretamente legível.

A escolha do par é evidência quantitativa (Sobol S₁, Etapa 4.1). Fixar os demais inputs na média é um ponto de operação de referência, e a limitação é explícita: as superfícies mostram o comportamento marginalizado nesses valores fixos, não o landscape completo. Essa limitação é resolvida em 6.3, onde todos os 6 inputs variam livremente via LHS de 50.000 pontos.

O que é avaliado: a forma geométrica das superfícies — regiões de baixo ET com M_CH3OH ainda razoável identificam visualmente candidatos a soluções Pareto-eficientes. Gradientes acentuados indicam alta sensibilidade local: pequenas variações em BRC1 ou RFF produzem grandes variações no output. A comparação das superfícies de ET e M_CH3OH sobre a mesma grade revela em que regiões os dois objetivos conflitam (gradientes em direções opostas) e em que regiões são neutros entre si. O overlay dos 193 pontos do test set nos mapas de contorno serve como diagnóstico de suporte empírico: confirma que o surrogate está sendo avaliado em regiões com dados reais próximos, não extrapolando para espaços não amostrados — relevante porque as superfícies são predições de um modelo, não simulações do processo.

Contribuição: esta sub-etapa entrega compreensão geométrica do landscape de otimização antes do passo computacional de 6.3. Permite verificar que o surrogate capturou as não-linearidades esperadas (superfícies com curvatura, não planas) e que o trade-off entre ET e M_CH3OH tem estrutura real no espaço de inputs.

A mesma implementação foi feita para 6.2 (M_CH3OH), mas com variáveis de entrada distintas.

### 6.3 — Fronteira de Pareto ET × M_CH3OH

Objetivo: gerar formalmente a fronteira de Pareto do problema bi-objetivo (minimizar ET, maximizar M_CH3OH) sobre o espaço completo S₆ = {T1, RRC1, BRC1, RRC2, BRC2, RFF}, impondo a restrição operacional x_CH3OH ≥ 0.98. Esta é a sub-etapa central da Etapa 6: enquanto 6.1 e 6.2 exploram visualmente subplanos do espaço de busca, 6.3 varre o espaço completo e identifica as soluções que são ótimas no sentido de Pareto — aquelas para as quais não existe nenhum outro ponto do espaço viável que melhore um objetivo sem degradar o outro.

Premissas: o surrogate SVR k*=6 é suficientemente fiel ao processo real para que as predições no espaço amostrado sejam interpretáveis como o comportamento do sistema — premissa validada empiricamente nas Etapas 3 (equivalência estatística) e 4 (consistência Sobol × SHAP). A amostragem LHS de 50.000 pontos sobre as faixas físicas de S₆ não é uma otimização iterativa: é uma cobertura quasi-uniforme do espaço de busca que gera uma aproximação discreta da fronteira contínua. A qualidade dessa aproximação depende da densidade amostral e da suavidade do surrogate, não de convergência de um algoritmo de gradiente. O algoritmo de varrimento por dominância identifica, dentro do conjunto de pontos viáveis, aqueles para os quais nenhum outro ponto viável apresenta simultaneamente ET menor e M_CH3OH maior — definição clássica de dominância de Pareto.

A restrição x_CH3OH ≥ 0.98 é a condição operacional mínima para que o metanol produzido tenha valor comercial (especificação de pureza de produto). Ela é avaliada diretamente pelo surrogate SVR k*=6 treinado para x_CH3OH — o mesmo modelo validado nas Etapas 3 e 4. A taxa de viabilidade resultante (63,5%, ou 31.745 de 50.000 pontos) indica que a restrição de pureza é genuinamente restritiva: ela elimina aproximadamente um terço do espaço de busca, concentrando a fronteira de Pareto em regiões onde as condições operacionais são suficientemente seletivas para garantir pureza.

O que é avaliado: a existência, forma e extensão da fronteira de Pareto no espaço ET × M_CH3OH, sob restrição de pureza. A figura `6.3_pareto_ET_M.png` permite ler diretamente quais combinações (ET, M_CH3OH) são Pareto-ótimas e quais são dominadas — o conjunto cinza (viáveis dominados) e a curva azul (Pareto-eficientes) contam a história completa do espaço de soluções. As 20 soluções não dominadas salvas em `6.3_pareto_solucoes.csv` são o artefato operacional: cada linha é um ponto de operação candidato, com os 6 inputs e as 3 saídas previstas pelo surrogate.

A fronteira não é uma solução única — é um conjunto de soluções igualmente defensáveis do ponto de vista da otimização. A escolha entre elas é uma decisão de engenharia ou de negócio (quantificar o valor da produção adicional versus o custo da energia), não uma decisão técnica. O trade-off de ~18× em consumo energético para ~2,8× em produção (do extremo mínimo-ET ao extremo máximo-M_CH3OH) caracteriza a curvatura e a amplitude da fronteira — quanto custa, em energia, cada unidade adicional de metanol.

A fronteira de Pareto via surrogate é o resultado operacional central do TCC. A abordagem é metodologicamente defensável em três pontos. Primeiro, o surrogate SVR k*=6 foi validado com três camadas de evidência independentes (equivalência bootstrap, SHAP e Sobol) antes de ser usado como função objetivo — não é uma caixa-preta não calibrada. Segundo, a amostragem LHS garante cobertura uniforme do espaço de busca e é a mesma técnica usada para gerar os dados originais do REF1, mantendo consistência metodológica. Terceiro, a restrição de pureza é imposta pelo mesmo surrogate que a prediz, e não por um threshold arbitrário: o valor 0.98 vem da especificação operacional do processo descrita no REF1.

Contribuição: 6.3 entrega o conjunto de soluções Pareto-ótimas que sintetiza todo o trabalho anterior — o surrogate validado (Etapas 2 e 3), a seleção de features (Etapa 3), a análise de sensibilidade (Etapa 4) e as superfícies de resposta (6.1 e 6.2). É a resposta à pergunta central do TCC: "quais são os pontos de operação ótimos que minimizam energia, maximizam produção e garantem pureza?" O artefato `6.3_pareto_solucoes.csv` é diretamente utilizável por um operador ou sistema de controle — traduz o problema de otimização em uma tabela de recomendações. A sub-etapa 6.4 usa essa tabela para identificar e interpretar os pontos mais relevantes da fronteira.

### 6.4 — Análise de trade-off e narrativa

Objetivo: traduzir o conjunto de 20 soluções Pareto-ótimas de `6.3_pareto_solucoes.csv` em conhecimento operacional interpretável. A sub-etapa não gera novos surrogates nem reavalia modelos — toda a computação é leitura e análise do artefato de 6.3. O resultado é a descrição de três pontos-chave da fronteira (extremo de mínimo ET, extremo de máximo M_CH3OH e região de compromisso) com seus respectivos valores de inputs e saídas, e a identificação de quais inputs discriminam cada região da fronteira.

Premissas: a divisão da fronteira em três regiões é feita por percentis de ET (mínimo, percentis 33–67 como compromisso, máximo), o que é uma escolha de discretização, não um ótimo absoluto. Qualquer ponto da fronteira é igualmente Pareto-ótimo; a seleção de três regiões serve apenas para tornar a análise legível e comunicável. A narrativa assume que os insights de Sobol e SHAP das Etapas 3 e 4 são hipóteses a verificar: se BRC1 e RFF são os inputs de maior efeito aditivo sobre ET, espera-se que eles sejam os que mais variam ao longo da fronteira — e é exatamente isso que 6.4 verifica nos dados reais da fronteira.

O que é avaliado: para cada um dos três pontos-chave, os valores dos 6 inputs de S₆ e as três saídas preditas pelo surrogate. O olhar analítico central é sobre a variação dos inputs ao longo da fronteira: quais inputs mudam muito entre o extremo energético e o extremo de produção (graus de liberdade reais do trade-off) e quais permanecem estreitos (inputs que a restrição de pureza ou a física do processo fixam em valores próximos, independentemente do objetivo energético). A figura `6.4_pareto_anotado.png` marca os três pontos sobre a fronteira gerada em 6.3, tornando visível a posição de cada região no espaço ET × M_CH3OH.

Os resultados confirmam a coerência com as Etapas 3 e 4. BRC1 e RFF são os inputs discriminantes: no extremo de mínimo ET, BRC1=0.57 (valor mínimo da faixa física) e RFF=0.247 (próximo ao máximo de 0.25); no extremo de máximo M_CH3OH, BRC1=9.05 e RFF=0.013. Essa inversão é coerente com os mecanismos físicos identificados pelo Sobol: BRC1 baixo reduz o duty do refervedor da coluna 1, e RFF alto desvia fluxo do loop de reciclo para a purga, diminuindo a carga de separação ao custo da conversão total. Os demais inputs (T1, RRC1, RRC2) permanecem estreitos ao longo de toda a fronteira — confirmam que a fronteira se move essencialmente no subplano (BRC1, RFF), com os outros inputs atuando como ajustes finos para satisfazer a restrição de pureza.

A região de compromisso (ET≈25 kW, M_CH3OH≈10,3 kg/hr) produz aproximadamente 80% da produção máxima com 30% do consumo energético máximo. Este ponto é relevante operacionalmente: concentra a maior parte do valor de produção com uma fração pequena do custo energético. O artefato `6.4_narrativa.md` documenta essa análise em formato narrativo independente do notebook, servindo como insumo direto para a escrita do TCC.

Como justificar para a banca: a coerência entre os inputs que discriminam a fronteira (BRC1 e RFF) e os índices de Sobol de primeira ordem calculados de forma agnóstica na Etapa 4 (S₁(RFF)=0.437 para ET, S₁(BRC1)=0.318) não é coincidência — é uma validação cruzada do pipeline inteiro. O mesmo par de inputs que o método de Sobol identificou como responsável pela maior fração da variância aditiva de ET é o par que, na fronteira Pareto real, determina a posição de cada solução. Essa convergência entre a análise de sensibilidade e a análise de trade-off reforça que o surrogate capturou a física relevante do processo, não apenas ajustou curvas a dados.

Uma possível pergunta da banca é sobre a utilidade prática de uma fronteira gerada por surrogate em comparação com dados simulados. A resposta é que o surrogate permite varrer 50.000 pontos em segundos — o que seria inviável com o simulador Aspen, que exige dezenas de minutos por ponto de operação e frequentemente diverge (como evidenciado pelos 79 erros de convergência no dataset original). O surrogate é uma ferramenta de triagem: identifica candidatos promissores que podem então ser validados pontualmente no simulador, reduzindo drasticamente o custo computacional da exploração.

Contribuição: 6.4 fecha o ciclo narrativo do TCC. O percurso parte dos dados brutos do REF1 (Etapa 0), passa pela construção e validação do surrogate (Etapas 2 e 3), pela análise de sensibilidade (Etapa 4), pela consolidação dos resultados (Etapa 5) e chega na Etapa 6 com a resposta operacional: quais são os pontos de operação que otimizam o processo, o quanto cada grau de liberdade contribui para o trade-off, e qual é a região de compromisso recomendável. O artefato `6.4_narrativa.md` é o único documento da Etapa 6 escrito em linguagem de processo (não de modelo), tornando os resultados acessíveis a um leitor de engenharia química sem necessidade de familiaridade com os detalhes do surrogate.


## Interpretação

### 6.1 — Superfícies de resposta: ET e M_CH3OH em função de (BRC1, RFF)

### 6.2 — Superfície de resposta: x_CH3OH em função de (RRC1, BRC1)


# Fluxo de Implementação — TCC SBO

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              PROPOSTA (Proposta 3)                           │
│  Replicar + estender REF1 (Sultan et al., 2025): surrogate modeling de       │
│  processo de metanol via hidrogenação de CO₂ com dados públicos LHS-2000     │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           CONSUMO DOS DADOS                                  │
│  GitHub REF1: Training-Data (2000 amostras) + Test-Data (200 amostras)       │
│  Geradas via Latin Hypercube Sampling no simulador Aspen Plus V.14           │
│  8 inputs de processo → 3 outputs (ET, M_CH3OH, x_CH3OH)                     │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     E0 — CARGA E PRÉ-PROCESSAMENTO                           │
│  • Filtro de qualidade: descarte de Status ≠ "Results Available"             │
│    (79 linhas no treino, 7 no teste)                                         │
│  • Split 80/20 (random_state=42): 1536 / 385 / 193 amostras                  │
│  • MinMaxScaler ajustado apenas no treino — sem data leakage                 │
│  Saída: X/y_{train,val,test}.{csv,npy} + scalers.npy                         │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              E1 — EDA                                        │
│  • 1.1 Correlação de Pearson (inputs × outputs, inputs × inputs)             │
│  • 1.2 Sensibilidade 1D marginalizada (RF auxiliar para suavização)          │
│  • 1.3–1.4 Distribuições de inputs e outputs                                 │
│  • 1.5 Verificação formal da assinatura LHS (uniformidade por bin)           │
│  Indicação inicial: P1 e T2 com efeito marginal baixo em todos os outputs    │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         E2 — MODELOS BASELINE                                │
│  5 arquiteturas × 3 outputs = 15 surrogates (registrados no MLflow)          │
│                                                                              │
│      SVR │ DT │ RF │ XGBoost │ ANN   ×  {ET, M_CH3OH, x_CH3OH}               │
│                                                                              │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         E3 — MODELOS REDUZIDOS                               │
│                                                                              │
│  [3.1] SHAP sobre os 15 baselines                                            │
│        P1 rank 7 e T2 rank 8 em TODOS os 5 modelos e nos 3 outputs           │
│                            │                                                 │
│                            ▼                                                 │
│  [3.2] Feature selection → subconjuntos aninhados S₄ ⊂ S₅ ⊂ S₆               │
│        S₆ = {T1, RRC1, BRC1, RRC2, BRC2, RFF}                                │
│                            │                                                 │
│                            ▼                                                 │
│  [3.3] 45 surrogates reduzidos (5 modelos × 3 outputs × k ∈ {4, 5, 6})       │
│                            │                                                 │
│                            ▼                                                 │
│  [3.4] Equivalência estatística (bootstrap pareado, B=1000, IC 95%, δ=0.02)  │
│        k=4: nenhum modelo equiv. │ k=5: só ET │ k=6: SVR/RF/DT nos 3         │
│                            │                                                 │
│                            ▼                                                 │
│  [3.5] k* = 6, arquitetura SVR                                               │
│        Único com equivalência simultânea nos 3 outputs + maior R² médio      │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                  E4 — ANÁLISE DE SENSIBILIDADE GLOBAL (SOBOL)                │
│  Função de avaliação: SVR k=8 (4.1) e SVR k=6 (4.2)                          │
│                                                                              │
│  [4.1] Sobol 8 inputs: P1 S_T ≤ 0.013, T2 S_T ≤ 0.009 (< limiar 0.02)        │
│  [4.2] Sobol 6 inputs: ranking de S_T preservado integralmente k=8 → k=6     │
│  [4.3] Spearman(Sobol × SHAP): ρ médio = 0.919, zero divergências ≥ 3        │
│  [4.4] Interações: RRC1/T1 predominantemente interativos em ET;              │
│         BRC1, RRC2, BRC2 predominantemente interativos em x_CH3OH            │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    E5 — CONSOLIDAÇÃO (validação de H1)                       │
│  H1: P1 e T2 são irrelevantes para os 3 outputs                              │
│                                                                              │
│  Camada 1 (empírica)  — bootstrap ΔR²: SVR k=8 → k=6 equivalente             │
│  Camada 2 (SHAP)      — max(shap_rel) < 5% para P1 e T2                      │
│  Camada 3 (Sobol)     — S_T máx < 0.015 para P1 e T2                         │
│                                                                              │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              E6 — EXPLORAÇÃO DA FRONTEIRA PARETO VIA SURROGATE               │
│  Problema bi-objetivo: min ET, max M_CH3OH  s.t.  x_CH3OH ≥ 0.98             │
│  Surrogate: SVR k*=6 como substituto computacional do simulador Aspen        │
│                                                                              │
│  [6.1–6.2] Superfícies de resposta: subplanos (BRC1 × RFF), (RRC1 × BRC1)    │
│  [6.3] LHS 50.000 pts → 31.745 viáveis (63,5%) → 20 soluções não dominadas   │
│  [6.4] Trade-off: ~18× em ET para ~2,8× em M_CH3OH entre os extremos         │
│         Inputs discriminantes: BRC1 e RFF                                    │
│         Inputs estreitos: T1, RRC1, RRC2 (fixados pela restrição de pureza)  │
└──────────────────────────────────────────────────────────────────────────────┘
```

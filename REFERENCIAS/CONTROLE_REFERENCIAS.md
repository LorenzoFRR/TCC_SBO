# Controle de Referências

## REF1

**Nome:** AI-based modeling and optimization of green methanol production process
**Autores:** Nabeel Sultan, Ali Almansoori, Ali Elkamel
**Journal:** Computers and Chemical Engineering 203 (2025) 109324
**DOI:** https://doi.org/10.1016/j.compchemeng.2025.109324
**Repositório GitHub:** https://github.com/Nab1027/Optimization-of-Methanol-Production-Process-Using-Machine-Learning

**Descrição:** Estudo de modelagem substituta e otimização do processo de produção de metanol verde via hidrogenação de CO2. Os autores simularam o processo no Aspen Plus V.14 usando cinética VBF (Vanden Bussche-Froment) e geraram datasets com técnicas de amostragem LHS, Monte Carlo e SOBOL (100 a 2000 amostras). Cinco modelos de ML foram comparados (SVR, Decision Tree, Random Forest, XGBoost, ANN) para predição de vazão de metanol, pureza e consumo de energia. ANN e XGBoost apresentaram melhor desempenho (R² > 0.98 e > 0.93, respectivamente). A otimização via DEA resultou em +33.59% produção, +2.06% pureza e −9.68% consumo de energia frente ao caso base.

**Relevância para o TCC:** Fonte primária de dados simulados (disponíveis no GitHub). O sistema inclui colunas de destilação como etapa de purificação. A estrutura metodológica (DoE → dados → metamodelo → otimizador → validação) é análoga ao fluxo proposto para o TCC.

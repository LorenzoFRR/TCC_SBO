# TCC SBO — Surrogate-Based Optimization para Colunas de Destilação

Trabalho de Conclusão de Curso (UFRGS) sobre modelagem substituta e otimização de um processo de produção de metanol verde a partir de dados públicos de simulação Aspen. Replica e estende o fluxo metodológico de Sultan et al. (2025).

Nenhum simulador é executado. Todos os dados vêm do repositório público do REF1.

## Como consultar

O ponto de entrada principal é [REGISTRO.md](REGISTRO.md): documento cronológico que narra a lógica, decisões e resultados de cada etapa. É o que deve ser lido primeiro para entender o projeto.

Todo o código+artefatos+tabelas+figuras está em Jupyter notebooks (`.ipynb`) dentro de `ARTEFATOS/ETAPA_N/`.

## Stack

Python 3.11+, MLflow (tracking local), scikit-learn, XGBoost, TensorFlow/Keras, SHAP, SALib, pandas, numpy, scipy.

## Referência principal

Sultan, N., Almansoori, A., Elkamel, A. "AI-based modeling and optimization of green methanol production process." Computers and Chemical Engineering 203 (2025) 109324. Dados: https://github.com/Nab1027/Optimization-of-Methanol-Production-Process-Using-Machine-Learning

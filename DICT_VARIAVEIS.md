# Dicionário de Variáveis

## Entradas (Inputs)

| Código | Coluna no dataset | Faixa | Nome |
|--------|-------------------|-------|--------------------|
| P1 | `Recomp` | 50–100 bar | Pressão do reator |
| T1 | `HX1` | 200–300 °C | Temperatura do reator |
| T2 | `HX2` | 85–95 °C | Temperatura de entrada na coluna de destilação |
| RRC1 | `C1 RR` | 1–10 | Razão de refluxo — coluna 1 |
| BRC1 | `C1 BR` | 0.5–10 | Razão de boil-up — coluna 1 |
| RRC2 | `C2 RR` | 1–10 | Razão de refluxo — coluna 2 |
| BRC2 | `C2 BR` | 0.5–10 | Razão de boil-up — coluna 2 |
| RFF | `RFF` | 0.01–0.25 | Fração de purga |

## Saídas (Outputs)

| Código | Coluna no dataset | Unidade | Nome |
|--------|-------------------|---------|---------------------|
| M_CH3OH | `Methanol Flow` | kg/hr | Vazão mássica de metanol |
| x_CH3OH | `Purity` | — | Pureza do metanol (fração mássica) |
| ET | `Energy` | kW | Consumo de energia total |

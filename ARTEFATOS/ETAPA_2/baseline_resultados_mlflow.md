# ETAPA 2 — Runs MLflow · Experimento `baseline`

## R²

| Run name        | model   | output  | variant               | train_r2 | val_r2 | test_r2 |
|-----------------|---------|---------|-----------------------|----------|--------|---------|
| SVR_ET          | SVR     | ET      | —                     | 0.9998   | 0.9873 | 0.9701  |
| SVR_M_CH3OH     | SVR     | M_CH3OH | —                     | 0.9961   | 0.9793 | 0.9831  |
| SVR_x_CH3OH     | SVR     | x_CH3OH | —                     | 0.9868   | 0.9470 | 0.9531  |
| DT_ET           | DT      | ET      | —                     | 0.9943   | 0.8577 | 0.8836  |
| DT_M_CH3OH      | DT      | M_CH3OH | —                     | 0.9654   | 0.7704 | 0.7740  |
| DT_x_CH3OH      | DT      | x_CH3OH | —                     | 0.9686   | 0.8062 | 0.7905  |
| RF_ET           | RF      | ET      | —                     | 0.9915   | 0.9445 | 0.9240  |
| RF_M_CH3OH      | RF      | M_CH3OH | —                     | 0.9874   | 0.9076 | 0.9143  |
| RF_x_CH3OH      | RF      | x_CH3OH | —                     | 0.9894   | 0.9315 | 0.9012  |
| XGBoost_ET      | XGBoost | ET      | —                     | 0.9995   | 0.9746 | 0.9634  |
| XGBoost_M_CH3OH | XGBoost | M_CH3OH | —                     | 0.9890   | 0.9549 | 0.9602  |
| XGBoost_x_CH3OH | XGBoost | x_CH3OH | —                     | 0.9995   | 0.9410 | 0.9451  |
| ANN_ET          | ANN     | ET      | —                     | 0.9678   | 0.9636 | 0.9666  |
| ANN_M_CH3OH     | ANN     | M_CH3OH | —                     | 0.9641   | 0.9587 | 0.9542  |
| ANN_x_CH3OH     | ANN     | x_CH3OH | —                     | 0.9539   | 0.9442 | 0.9406  |
| ANN_v2_ET       | ANN     | ET      | v2_no_earlystop_500ep | 0.9621   | 0.9576 | 0.9647  |
| ANN_v2_M_CH3OH  | ANN     | M_CH3OH | v2_no_earlystop_500ep | 0.9888   | 0.9842 | 0.9846  |
| ANN_v2_x_CH3OH  | ANN     | x_CH3OH | v2_no_earlystop_500ep | 0.9798   | 0.9697 | 0.9689  |

## MAE

| Run name        | model   | output  | variant               | train_mae | val_mae   | test_mae  |
|-----------------|---------|---------|-----------------------|-----------|-----------|-----------|
| SVR_ET          | SVR     | ET      | —                     | 110.60    | 1140.24   | 1390.67   |
| SVR_M_CH3OH     | SVR     | M_CH3OH | —                     | 79.32     | 281.80    | 247.03    |
| SVR_x_CH3OH     | SVR     | x_CH3OH | —                     | 0.005971  | 0.012274  | 0.011856  |
| DT_ET           | DT      | ET      | —                     | 742.45    | 4203.13   | 4539.01   |
| DT_M_CH3OH      | DT      | M_CH3OH | —                     | 369.72    | 977.23    | 937.58    |
| DT_x_CH3OH      | DT      | x_CH3OH | —                     | 0.006822  | 0.017409  | 0.018887  |
| RF_ET           | RF      | ET      | —                     | 1026.60   | 2569.11   | 3147.08   |
| RF_M_CH3OH      | RF      | M_CH3OH | —                     | 236.29    | 626.08    | 555.52    |
| RF_x_CH3OH      | RF      | x_CH3OH | —                     | 0.004360  | 0.011219  | 0.012946  |
| XGBoost_ET      | XGBoost | ET      | —                     | 281.69    | 1708.69   | 2230.81   |
| XGBoost_M_CH3OH | XGBoost | M_CH3OH | —                     | 233.35    | 453.68    | 426.40    |
| XGBoost_x_CH3OH | XGBoost | x_CH3OH | —                     | 0.001208  | 0.011090  | 0.009951  |
| ANN_ET          | ANN     | ET      | —                     | 2252.19   | 2268.34   | 2269.20   |
| ANN_M_CH3OH     | ANN     | M_CH3OH | —                     | 391.63    | 404.83    | 428.50    |
| ANN_x_CH3OH     | ANN     | x_CH3OH | —                     | 0.010725  | 0.011208  | 0.011698  |
| ANN_v2_ET       | ANN     | ET      | v2_no_earlystop_500ep | 2414.69   | 2432.40   | 2537.24   |
| ANN_v2_M_CH3OH  | ANN     | M_CH3OH | v2_no_earlystop_500ep | 231.20    | 266.16    | 262.43    |
| ANN_v2_x_CH3OH  | ANN     | x_CH3OH | v2_no_earlystop_500ep | 0.007613  | 0.008765  | 0.008564  |

## MSE

| Run name        | model   | output  | variant               | train_mse    | val_mse      | test_mse     |
|-----------------|---------|---------|-----------------------|--------------|--------------|--------------|
| SVR_ET          | SVR     | ET      | —                     | 5.01e+04     | 3.13e+06     | 1.04e+07     |
| SVR_M_CH3OH     | SVR     | M_CH3OH | —                     | 3.18e+04     | 1.67e+05     | 1.27e+05     |
| SVR_x_CH3OH     | SVR     | x_CH3OH | —                     | 8.22e-05     | 3.17e-04     | 2.72e-04     |
| DT_ET           | DT      | ET      | —                     | 1.60e+06     | 3.50e+07     | 4.04e+07     |
| DT_M_CH3OH      | DT      | M_CH3OH | —                     | 2.82e+05     | 1.86e+06     | 1.71e+06     |
| DT_x_CH3OH      | DT      | x_CH3OH | —                     | 1.96e-04     | 1.16e-03     | 1.22e-03     |
| RF_ET           | RF      | ET      | —                     | 2.38e+06     | 1.37e+07     | 2.64e+07     |
| RF_M_CH3OH      | RF      | M_CH3OH | —                     | 1.03e+05     | 7.48e+05     | 6.48e+05     |
| RF_x_CH3OH      | RF      | x_CH3OH | —                     | 6.61e-05     | 4.10e-04     | 5.73e-04     |
| XGBoost_ET      | XGBoost | ET      | —                     | 1.53e+05     | 6.24e+06     | 1.27e+07     |
| XGBoost_M_CH3OH | XGBoost | M_CH3OH | —                     | 8.98e+04     | 3.65e+05     | 3.01e+05     |
| XGBoost_x_CH3OH | XGBoost | x_CH3OH | —                     | 3.11e-06     | 3.53e-04     | 3.18e-04     |
| ANN_ET          | ANN     | ET      | —                     | 9.07e+06     | 8.94e+06     | 1.16e+07     |
| ANN_M_CH3OH     | ANN     | M_CH3OH | —                     | 2.93e+05     | 3.34e+05     | 3.46e+05     |
| ANN_x_CH3OH     | ANN     | x_CH3OH | —                     | 2.87e-04     | 3.34e-04     | 3.45e-04     |
| ANN_v2_ET       | ANN     | ET      | v2_no_earlystop_500ep | 1.07e+07     | 1.04e+07     | 1.22e+07     |
| ANN_v2_M_CH3OH  | ANN     | M_CH3OH | v2_no_earlystop_500ep | 9.15e+04     | 1.28e+05     | 1.16e+05     |
| ANN_v2_x_CH3OH  | ANN     | x_CH3OH | v2_no_earlystop_500ep | 1.26e-04     | 1.82e-04     | 1.80e-04     |

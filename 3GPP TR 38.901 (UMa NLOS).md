% Load Excel data
data = readtable('100MHz_10m_VV.csv');
d_km = data{:, 'DistanceToServer1_km_'};

valid = d_km > 0;
d_km = d_km(valid);
d_m = d_km * 1000;

EIRP = 63;
RSSI = data{:,'Server1Result_dBmW_'};
RSSI = RSSI(valid);
PL_measured = EIRP - RSSI;

% Parameters
f_GHz = 24;       % Frequency in GHz
h_BS = 10;        % Base station height (TX), meters
h_UT = 1.5;       % User terminal (RX) height

% LOS model (used inside max)
PL_LOS = 28 + 22 * log10(d_m) + 20 * log10(f_GHz);
PL_LOS_vec = PL_LOS;

% NLOS model
PL_NLOS = 13.54 + 39.08 * log10(d_m) + 20 * log10(f_GHz) - 0.6 * (h_BS - 1.5);
PL_NLOS_vec = PL_NLOS;

% Final UMa NLOS path loss: take max(LOS, NLOS)
PL_UMa_NLOS = max(PL_LOS_vec, PL_NLOS_vec);

% Residuals (shadow fading)
X_sigma = PL_measured - PL_UMa_NLOS;

% --- Performance Metrics ---
errors = PL_UMa_NLOS - PL_measured;
RMSE = sqrt(mean(errors.^2));
MAE  = mean(abs(errors));
MPE  = mean(errors ./ PL_measured) * 100;
Bias = mean(errors);
SDE  = std(errors);
SS_res = sum((PL_measured - PL_UMa_NLOS).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R_squared = 1 - SS_res / SS_tot;
X_std = std(X_sigma);

% Repeat metrics as columns
n = length(d_km);
RMSE_vec = repmat(RMSE, n, 1);
MAE_vec  = repmat(MAE, n, 1);
MPE_vec  = repmat(MPE, n, 1);
Bias_vec = repmat(Bias, n, 1);
SDE_vec  = repmat(SDE, n, 1);
R2_vec   = repmat(R_squared, n, 1);
Xstd_vec = repmat(X_std, n, 1);

% Create full output table
T = table(d_km, d_m, PL_measured, PL_LOS_vec, PL_NLOS_vec, PL_UMa_NLOS, X_sigma, ...
    RMSE_vec, MAE_vec, MPE_vec, Bias_vec, SDE_vec, R2_vec, Xstd_vec, ...
    'VariableNames', {'Distance_km', 'Distance_m', 'Measured_PL', 'PL_LOS', 'PL_NLOS', 'PL_UMa_NLOS', 'X_sigma', ...
                      'RMSE', 'MAE', 'MPE', 'Bias', 'SDE', 'R_squared', 'X_std'});

% --- Append Mean Row ---
mean_values = [mean(d_km), mean(d_m), mean(PL_measured), mean(PL_LOS_vec), ...
               mean(PL_NLOS_vec), mean(PL_UMa_NLOS), mean(X_sigma), ...
               RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];

mean_row = array2table(mean_values, ...
   'VariableNames', {'Distance_km', 'Distance_m', 'Measured_PL', 'PL_LOS', 'PL_NLOS', 'PL_UMa_NLOS', 'X_sigma', ...
                     'RMSE', 'MAE', 'MPE', 'Bias', 'SDE', 'R_squared', 'X_std'});

T = [T; mean_row];

% Display
disp(['3GPP UMa NLOS RMSE: ', num2str(RMSE)]);
disp(['3GPP UMa NLOS R²: ', num2str(R_squared)]);
disp(['Shadow Fading Std (σ): ', num2str(X_std)]);
disp(T);

% Export to Excel
writetable(T, '3GPP_UMa_NLOS_with_Metrics.xlsx');

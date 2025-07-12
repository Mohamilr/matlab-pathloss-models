% Load data
data = readtable('100MHz_10m_VV.csv');
d_km = data{:, 'DistanceToServer1_km_'};

valid = d_km > 0;
d_km = d_km(valid);
d_m = d_km * 1000;

EIRP = 63;  % EIRP in dBm
RSSI = data{:,'Server1Result_dBmW_'};
PL_measured = EIRP - RSSI;
PL_measured = PL_measured(valid);

% Constants
f_GHz = 24;
d0_m = 1;

% FSPL at d0 = 1m using 92.45-style
FSPL_d0 = 92.45 + 20*log10(d0_m / 1000) + 20*log10(f_GHz);  % d0 in km
FSPL_vec = repmat(FSPL_d0, size(d_m));

% Compute n using summation method
log_dist_ratio = log10(d_m / d0_m);
numerator = sum(PL_measured - FSPL_d0);
denominator = sum(10 * log_dist_ratio);
n = numerator / denominator;
n_vec = repmat(n, size(d_m));

% Predicted path loss (CI)
PL_CI = FSPL_vec + 10 * n * log_dist_ratio;

% Error terms
X_sigma = PL_measured - PL_CI;
X_mean = mean(X_sigma);
X_std = std(X_sigma);

% Evaluation Metrics
RMSE = sqrt(mean((PL_CI - PL_measured).^2));
MAE = mean(abs(PL_CI - PL_measured));
MPE = mean((PL_CI - PL_measured) ./ PL_measured) * 100;
Bias = mean(PL_CI - PL_measured);
SDE = std(PL_CI - PL_measured);
SS_res = sum((PL_measured - PL_CI).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R_squared = 1 - (SS_res / SS_tot);

% Repeat metrics as vectors for table
RMSE_vec = repmat(RMSE, size(d_m));
MAE_vec = repmat(MAE, size(d_m));
MPE_vec = repmat(MPE, size(d_m));
Bias_vec = repmat(Bias, size(d_m));
SDE_vec = repmat(SDE, size(d_m));
R2_vec   = repmat(R_squared, size(d_m));
Xstd_vec = repmat(X_std, size(d_m));

% Final table
T = table(d_m, FSPL_vec, n_vec, PL_CI, X_sigma, ...
          RMSE_vec, MAE_vec, MPE_vec, Bias_vec, SDE_vec, R2_vec, Xstd_vec, ...
          'VariableNames', {'Distance_m', 'FSPL_d0', 'n', 'CI_PL', 'CI_X_sigma', ...
                            'CI_RMSE', 'CI_MAE', 'CI_MPE', 'CI_Bias', 'CI_SDE', 'CI_R_squared', 'CI_X_std'});

% Append mean row
mean_values = [mean(d_m), mean(FSPL_vec), mean(n_vec), mean(PL_CI), ...
               mean(X_sigma), RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];

mean_row = array2table(mean_values, ...
     'VariableNames', {'Distance_m', 'FSPL_d0', 'n', 'CI_PL', 'CI_X_sigma', ...
                            'CI_RMSE', 'CI_MAE', 'CI_MPE', 'CI_Bias', 'CI_SDE', 'CI_R_squared', 'CI_X_std'});

T = [T; mean_row];

% Display
disp(['Estimated path loss exponent (n): ', num2str(n)]);
disp(['Shadow fading std (σ): ', num2str(X_std)]);
disp(T);

% Save to file
writetable(T, 'CI_Model_with_Errors_and_Metrics.xlsx');
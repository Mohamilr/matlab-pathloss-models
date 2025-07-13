% Load data from CSV
data = readtable('100MHz_10m_VV.csv');
d_km = data{:, 'DistanceToServer1_km_'};
valid = d_km > 0;
d_km = d_km(valid);
d_m = d_km * 1000;

EIRP = 63;  % Transmitter EIRP in dBm
RSSI = data{:,'Server1Result_dBmW_'};
RSSI = RSSI(valid);
PL_measured = EIRP - RSSI;

% Constants
f_GHz = 24;                  % Frequency in GHz
f_MHz = f_GHz * 1000;        % Frequency in MHz
hb = 10;                     % (Varies) Base station height in meters
hr = 1.5;                    % Mobile station height in meters

log_d = log10(d_km);
log_f = log10(f_MHz);

% ECC-33 calculations
A_fs = 92.4 + 20 * log10(d_km) + 20 * log10(f_GHz); % Free-space attenuation (correctly using GHz)
A_bm = 20.41 + 9.83 * log10(d_km) + 7.894 * log_f + 9.56 * (log_f).^2; % Basic median path loss
G_b = log10(hb / 200) .* (13.958 + 5.8 * (log_d).^2);                  % Base station height gain
G_r = 3.2 * (log10(11.75 * hr)).^2 - 4.97;                             % Receiver height gain (Large city)

% Final ECC-33 path loss prediction
PL_ECC33 = A_fs + A_bm - G_b - G_r;

% Shadow fading residuals
X_sigma = PL_measured - PL_ECC33;

% --- Performance Metrics ---
errors = PL_ECC33 - PL_measured;
RMSE = sqrt(mean(errors.^2));
MAE  = mean(abs(errors));
MPE  = mean(errors ./ PL_measured) * 100;
Bias = mean(errors);
SDE  = std(errors);
SS_res = sum((PL_measured - PL_ECC33).^2);
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
Gr_vec   = repmat(G_r, n, 1);

% Create output table
T = table(d_km, PL_measured, A_fs, A_bm, G_b, Gr_vec, PL_ECC33, X_sigma, ...
    RMSE_vec, MAE_vec, MPE_vec, Bias_vec, SDE_vec, R2_vec, Xstd_vec, ...
    'VariableNames', {'Distance_km', 'Measured_PL', 'ECC33_A_fs', 'ECC33_A_bm', 'ECC33_G_b', 'ECC33_G_r', 'ECC33_PL', ...
                      'ECC33_X_sigma', 'ECC33_RMSE', 'ECC33_MAE', 'ECC33_MPE', 'ECC33_Bias', ...
                      'ECC33_SDE', 'ECC33_R_squared', 'ECC33_X_std'});

% --- Mean Row ---
mean_values = [mean(d_km), mean(PL_measured), mean(A_fs), mean(A_bm), mean(G_b), G_r, ...
               mean(PL_ECC33), mean(X_sigma), RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];

mean_row = array2table(mean_values, ...
   'VariableNames', {'Distance_km', 'Measured_PL', 'ECC33_A_fs', 'ECC33_A_bm', 'ECC33_G_b', 'ECC33_G_r', 'ECC33_PL', ...
                     'ECC33_X_sigma', 'ECC33_RMSE', 'ECC33_MAE', 'ECC33_MPE', 'ECC33_Bias', ...
                     'ECC33_SDE', 'ECC33_R_squared', 'ECC33_X_std'});

% Append mean row
T = [T; mean_row];

% Display
disp(['ECC-33 Model RMSE: ', num2str(RMSE)]);
disp(['ECC-33 Model R²: ', num2str(R_squared)]);
disp(['Shadow Fading Std (σ): ', num2str(X_std)]);
disp(T);

% Export to Excel
writetable(T, 'ECC33_Model_with_Metrics.xlsx');

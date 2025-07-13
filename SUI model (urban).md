% Load data
data = readtable('100MHz_10m_VV.csv');
d_km = data{:, 'DistanceToServer1_km_'};
valid = d_km > 0;
d_km = d_km(valid);
d_m = d_km * 1000;

EIRP = 63;
RSSI = data{:,'Server1Result_dBmW_'};
PL_measured = EIRP - RSSI;
PL_measured = PL_measured(valid);

% Constants
f = 24000;  % MHz
hb = 10;    % (Varies) Tx height in meters
hr = 1.5;   % Rx height
d0 = 1;     % Reference distance in meters

% SUI Terrain A parameters
a = 4.6; b = 0.0075; c1 = 12.6;

% Compute gamma
gamma = a - b * hb + c1 / hb;
gamma_vec = repmat(gamma, size(d_m));

% Free-space path loss at reference distance d0
lambda = 3e8 / (f * 1e6); % in meters
A = 20 * log10((4 * pi * d0) / lambda);
A_vec = repmat(A, size(d_m));

% Frequency and receiver height correction factors
X_f = 6 * log10(f / 2000);
X_h = -10.8 * log10(hr / 2);
X_f_vec = repmat(X_f, size(d_m));
X_h_vec = repmat(X_h, size(d_m));

% Base SUI path loss
PL_SUI = A_vec + 10 * gamma_vec .* log10(d_m / d0) + X_f_vec + X_h_vec;

% Shadow fading (residuals)
X = PL_measured - PL_SUI;

% Estimate shadow fading term S as the residuals (to be added back)
S_vec = X;

% Final SUI prediction with shadow fading

% --- Performance Metrics ---
errors = PL_SUI - PL_measured;
RMSE  = sqrt(mean(errors .^ 2));
Bias  = mean(errors);
SDE   = std(errors);
MAE   = mean(abs(errors));
MPE   = mean(errors ./ PL_measured) * 100;
SS_res = sum((PL_measured - PL_SUI).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R2    = 1 - SS_res / SS_tot;
X_std = std(S_vec); % Std dev of shadowing

% Repeat metrics as columns
n = length(d_m);
RMSE_vec = repmat(RMSE, n, 1);
Bias_vec = repmat(Bias, n, 1);
SDE_vec  = repmat(SDE, n, 1);
MAE_vec  = repmat(MAE, n, 1);
MPE_vec  = repmat(MPE, n, 1);
R2_vec   = repmat(R2, n, 1);
Xstd_vec = repmat(X_std, n, 1);

% Output Table
T = table(d_m, A_vec, gamma_vec, X_f_vec, X_h_vec,  PL_SUI, S_vec, ...
    RMSE_vec,  MAE_vec, MPE_vec,Bias_vec,  SDE_vec, R2_vec, Xstd_vec, ...
    'VariableNames', {'Distance_m', 'SUI_A', 'SUI_Gamma', 'SUI_X_f', 'SUI_X_h', 'SUI_PL', 'SUI_X_sigma',  ...
                      'SUI_RMSE',   'SUI_MAE', 'SUI_MPE', 'SUI_Bias', 'SUI_SDE', 'SUI_R_squared', 'SUI_X_std'});

% Append mean row
mean_values = [mean(d_m), A, gamma, X_f, X_h,  mean(PL_SUI), mean(S_vec), ...
               RMSE, MAE, MPE, Bias, SDE,  R2, X_std];

mean_row = array2table(mean_values, ...
  'VariableNames', {'Distance_m', 'SUI_A', 'SUI_Gamma', 'SUI_X_f', 'SUI_X_h', 'SUI_PL', 'SUI_X_sigma',  ...
                      'SUI_RMSE',   'SUI_MAE', 'SUI_MPE', 'SUI_Bias', 'SUI_SDE', 'SUI_R_squared', 'SUI_X_std'});

T = [T; mean_row];

% Display
disp(['Shadow Fading Mean (S̄): ', num2str(mean(S_vec))]);
disp(['Shadow Fading Std (σ): ', num2str(X_std)]);
disp(T);

% Export
writetable(T, 'SUI_Model_with_Metrics.xlsx');

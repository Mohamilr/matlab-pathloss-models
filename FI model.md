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

% Convert to log10(d) for regression
x = log10(d_m);  % Predictor
y = PL_measured; % Response

% Linear regression: y = alpha + 10*beta*log10(d)
A = [ones(size(x)), 10 * x];  % [1, 10*log10(d)]
params = A \ y;               % Least squares solution

alpha = params(1);
beta = params(2);

% Compute modeled path loss
PL_FI = alpha + 10 * beta * x;

% Shadow fading term (X_sigma)
X_sigma = y - PL_FI;
X_std = std(X_sigma);

% --- Evaluation Metrics ---
RMSE = sqrt(mean((PL_FI - y).^2));
MAE  = mean(abs(PL_FI - y));
MPE  = mean((PL_FI - y) ./ y) * 100;
Bias = mean(PL_FI - y);
SDE  = std(PL_FI - y);
SS_res = sum((y - PL_FI).^2);
SS_tot = sum((y - mean(y)).^2);
R_squared = 1 - (SS_res / SS_tot);

% Repeat metrics as columns
n = length(d_m);
Alpha_vec = repmat(alpha, n, 1);
Beta_vec  = repmat(beta, n, 1);
RMSE_vec  = repmat(RMSE, n, 1);
MAE_vec   = repmat(MAE, n, 1);
MPE_vec   = repmat(MPE, n, 1);
Bias_vec  = repmat(Bias, n, 1);
SDE_vec   = repmat(SDE, n, 1);
R2_vec    = repmat(R_squared, n, 1);
Xstd_vec  = repmat(X_std, n, 1);

% Output table
T = table(d_m, Alpha_vec, Beta_vec, PL_FI, X_sigma, ...
    RMSE_vec, MAE_vec, MPE_vec, Bias_vec, SDE_vec, R2_vec, Xstd_vec, ...
    'VariableNames', {'Distance_m', 'FI_Alpha', 'FI_Beta', 'FI_PL', 'FI_X_sigma', ...
                      'FI_RMSE', 'FI_MAE', 'FI_MPE', 'FI_Bias', 'FI_SDE', 'FI_R_squared', 'FI_X_std'});

% Append mean row (match columns)
mean_values = [mean(d_m), alpha, beta, mean(PL_FI), mean(X_sigma), ...
               RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];
mean_row = array2table(mean_values, ...
    'VariableNames', {'Distance_m', 'FI_Alpha', 'FI_Beta', 'FI_PL', 'FI_X_sigma', ...
                      'FI_RMSE', 'FI_MAE', 'FI_MPE', 'FI_Bias', 'FI_SDE', 'FI_R_squared', 'FI_X_std'});
                      
T = [T; mean_row];

% Display
disp(['Estimated Alpha: ', num2str(alpha)]);
disp(['Estimated Beta: ', num2str(beta)]);
disp(['Shadow fading std (σ): ', num2str(X_std)]);
disp(T);

% Export
writetable(T, 'FI_Model_with_Errors_and_Metrics.xlsx');

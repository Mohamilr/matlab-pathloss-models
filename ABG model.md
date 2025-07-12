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
log_d = log10(d_m);
log_f = log10(f_GHz);
const_f = 10 * log_f;

% Adjusted design matrix: [10*log10(d), 1]
A = [10 * log_d, ones(size(d_m))];

% Adjust measured PL by subtracting gamma * 10*log10(f)
% We'll solve for alpha and beta first, then get gamma later
% Assume ABG: PL = alpha * log10(d) + beta + gamma * log10(f)

% Step 1: solve for alpha and beta
PL_adj = PL_measured - const_f;  % Remove frequency component for now
params = A \ PL_adj;

alpha = params(1);
beta  = params(2);
gamma = 1;  % Because we subtracted const_f = gamma * 10 * log10(f_GHz)

% Compute predicted path loss
PL_ABG = A * params + const_f;

% Shadow fading
X_sigma = PL_measured - PL_ABG;
X_std = std(X_sigma);

% Evaluation metrics
RMSE = sqrt(mean((PL_ABG - PL_measured).^2));
MAE  = mean(abs(PL_ABG - PL_measured));
MPE  = mean((PL_ABG - PL_measured) ./ PL_measured) * 100;
Bias = mean(PL_ABG - PL_measured);
SDE  = std(PL_ABG - PL_measured);
SS_res = sum((PL_measured - PL_ABG).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R_squared = 1 - (SS_res / SS_tot);

% Repeat metrics as vectors
n = length(d_m);
alpha_vec = repmat(alpha, n, 1);
beta_vec  = repmat(beta, n, 1);
gamma_vec = repmat(gamma, n, 1);
RMSE_vec  = repmat(RMSE, n, 1);
MAE_vec   = repmat(MAE, n, 1);
MPE_vec   = repmat(MPE, n, 1);
Bias_vec  = repmat(Bias, n, 1);
SDE_vec   = repmat(SDE, n, 1);
R2_vec    = repmat(R_squared, n, 1);
Xstd_vec  = repmat(X_std, n, 1);

% Final table
T = table(d_m, alpha_vec, beta_vec, gamma_vec, PL_ABG, X_sigma, ...
    RMSE_vec, MAE_vec, MPE_vec, Bias_vec, SDE_vec, R2_vec, Xstd_vec, ...
    'VariableNames', {'Distance_m', 'ABG_Alpha', 'ABG_Beta', 'ABG_Gamma', 'ABG_PL', 'ABG_X_sigma', ...
                      'ABG_RMSE', 'ABG_MAE', 'ABG_MPE', 'ABG_Bias', 'ABG_SDE', 'ABG_R_squared', 'ABG_X_std'});

% Append mean row
mean_values = [mean(d_m), alpha, beta, gamma, mean(PL_ABG), mean(X_sigma), ...
               RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];

mean_row = array2table(mean_values, ...
   'VariableNames', {'Distance_m', 'ABG_Alpha', 'ABG_Beta', 'ABG_Gamma', 'ABG_PL', 'ABG_X_sigma', ...
                     'ABG_RMSE', 'ABG_MAE', 'ABG_MPE', 'ABG_Bias', 'ABG_SDE', 'ABG_R_squared', 'ABG_X_std'});

T = [T; mean_row];

% Display
disp(['Estimated Alpha: ', num2str(alpha)]);
disp(['Estimated Beta: ', num2str(beta)]);
disp(['Estimated Gamma (assumed 1 for fixed f): ', num2str(gamma)]);
disp(['Shadow Fading Std (σ): ', num2str(X_std)]);
disp(T);

% Export
writetable(T, 'ABG_Model_with_FixedFreq_and_Metrics.xlsx');
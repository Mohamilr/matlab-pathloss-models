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
FSPL_d0 = 92.45 + 20*log10(d0_m / 1000) + 20*log10(f_GHz);  % Note: d0 in km
FSPL_vec = repmat(FSPL_d0, size(d_m));

% Compute n using d in meters
log_dist_ratio = log10(d_m / d0_m);
numerator = sum(PL_measured - FSPL_d0);
denominator = sum(10 * log_dist_ratio);
n = numerator / denominator;
n_vec = repmat(n, size(d_m));

% Predicted path loss (CI)
PL_CI = FSPL_vec + 10 * n * log_dist_ratio;

% Shadow fading term X ~ N(0, σ^2)
X_sigma = PL_measured - PL_CI;
X_mean = mean(X_sigma);
X_std = std(X_sigma);

% Table with output
T = table(d_m, FSPL_vec, n_vec, PL_CI, X_sigma, ...
    'VariableNames', {'Distance_m', 'FSPL_d0', 'n', 'PL_CI', 'X_sigma'});

% Append a row of means
mean_row = array2table([mean(d_m), mean(FSPL_vec), mean(n_vec), mean(PL_CI), mean(X_sigma)], ...
    'VariableNames', {'Distance_m', 'FSPL_d0', 'n', 'PL_CI', 'X_sigma'});
T = [T; mean_row];

% Display
disp(['Estimated path loss exponent (n): ', num2str(n)]);
disp(['Shadow fading std (σ): ', num2str(X_std)]);
disp(T);

% Optional: write to Excel
writetable(T, 'CI_Model_with_ShadowFading.xlsx');
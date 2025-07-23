% Load data from the Excel file
data = readtable('Data_to_plot.xlsx');

d_m = d_km * 1000;

d_m = data.Distance_m;
d_km = d_m / 1000;
PL_measured = data.Measured_PL; 
f_GHz = 24;                % Frequency in GHz
f_MHz = f_GHz * 1000;
hb = 10;                   % Base station height in meters
hr = 1.5;                  % Mobile station height in meters
log_f = log10(f_MHz);
log_d = log10(d_km);

% ECC-33 Model function with tunable parameters:
% p = [a1, a2, a3, a4] for basic median path loss
model_fun = @(p, d_km) ...
    (92.4 + 20 * log10(d_km) + 20 * log10(f_GHz)) + ...         % A_fs
    (p(1) + p(2)*log10(d_km) + p(3)*log_f + p(4)*(log_f).^2) - ... % A_bm
    (log10(hb / 200) .* (13.958 + 5.8 * (log10(d_km)).^2)) - ...  % G_b
    (3.2 * (log10(11.75 * hr)).^2 - 4.97);                        % G_r

% Initial guess for p
p0 = [20.41, 9.83, 7.894, 9.56];

% Optimization using lsqcurvefit (use optimset instead of optimoptions)
options = optimset('Display', 'iter', 'MaxIter', 1000, 'TolX', 1e-6, 'TolFun', 1e-6);
[p_opt, resnorm] = lsqcurvefit(model_fun, p0, d_km, PL_measured, [], [], options);

% Calculate optimized path loss
PL_optimized = model_fun(p_opt, d_km);

% Calculate error metrics
errors = PL_optimized - PL_measured;
RMSE = sqrt(mean(errors.^2));
MAE  = mean(abs(errors));
MPE  = mean(errors ./ PL_measured) * 100;
Bias = mean(errors);
SDE  = std(errors);
SS_res = sum((PL_measured - PL_optimized).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R_squared = 1 - SS_res / SS_tot;
X_sigma = PL_measured - PL_optimized;
X_std = std(X_sigma);

% Create output table
n = length(d_km);
T = table(d_km, PL_measured, PL_optimized, X_sigma, ...
    repmat(RMSE,n,1), repmat(MAE,n,1), repmat(MPE,n,1), repmat(Bias,n,1), ...
    repmat(SDE,n,1), repmat(R_squared,n,1), repmat(X_std,n,1), ...
    'VariableNames', {'Distance_km', 'Measured_PL', 'Optimized_PL', 'X_sigma', ...
                      'RMSE', 'MAE', 'MPE', 'Bias', 'SDE', 'R_squared', 'X_std'});

% Append mean row
mean_vals = [mean(d_km), mean(PL_measured), mean(PL_optimized), ...
             mean(X_sigma), RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];
T = [T; array2table(mean_vals, 'VariableNames', T.Properties.VariableNames)];

% Display results
disp(['Optimized RMSE: ', num2str(RMSE)]);
disp(['Optimized R²: ', num2str(R_squared)]);

% Save to Excel
writetable(T, 'Optimized_ECC33_Results.xlsx');
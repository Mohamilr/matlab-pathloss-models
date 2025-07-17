% Load data
data = readtable('Data_to_plot.xlsx');

% Extract distance and measured PL
d_m = data.Distance_m;
PL_measured = data.Measured_PL;

% Filter valid data
valid = d_m > 0 & ~isnan(PL_measured);
d_m = d_m(valid);
PL_measured = PL_measured(valid);

% -------------------------------
% Compute FSPL at 1 meter using 92.45 method
f_GHz = 24;
FSPL_d0 = 92.45 + 20*log10(f_GHz) + 20*log10(0.001);  % 0.001 km = 1m
fprintf('FSPL at 1m (92.45 method): %.4f dB\n', FSPL_d0);

% -------------------------------
% Estimate initial guess for n from regression
x = log10(d_m);
y = PL_measured - FSPL_d0;
n0 = (x \ y);  % Least squares estimate
x0 = n0;       % Initial guess for n

% Bounds for n
lb = 1;
ub = 6;

% -------------------------------
% CI Model function: PL = FSPL_d0 + 10 * n * log10(d)
ci_model = @(n, d) FSPL_d0 + 10 * n * log10(d);

% Run optimization
options = optimoptions('lsqcurvefit', 'Display', 'iter');
[n_opt, resnorm, residuals, exitflag, output] = ...
    lsqcurvefit(ci_model, x0, d_m, PL_measured, lb, ub, options);

% Optimized prediction
PL_CI_optimized = ci_model(n_opt, d_m);

% -------------------------------
% X_sigma and Metrics
X_sigma = PL_measured - PL_CI_optimized;
X_std = std(X_sigma);

errors = PL_CI_optimized - PL_measured;
RMSE = sqrt(mean(errors.^2));
Bias = mean(errors);
SDE = std(errors);
MAE = mean(abs(errors));
MPE = mean(errors ./ PL_measured) * 100;
SS_res = sum((PL_measured - PL_CI_optimized).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R2 = 1 - SS_res / SS_tot;

% -------------------------------
% Prepare output table
n_vec = repmat(n_opt, length(d_m), 1);
FSPL_vec = repmat(FSPL_d0, length(d_m), 1);

T = table(d_m, FSPL_vec, n_vec, PL_CI_optimized, X_sigma, ...
    repmat(RMSE, size(d_m)), repmat(MAE, size(d_m)), repmat(MPE, size(d_m)), ...
    repmat(Bias, size(d_m)), repmat(SDE, size(d_m)), repmat(R2, size(d_m)), ...
    repmat(X_std, size(d_m)), ...
    'VariableNames', {'Distance_m', 'FSPL_d0', 'n', 'CI_PL', 'CI_X_sigma', ...
                      'CI_RMSE', 'CI_MAE', 'CI_MPE', 'CI_Bias', 'CI_SDE', ...
                      'CI_R_squared', 'CI_X_std'});

% -------------------------------
% Append mean row
mean_values = [mean(d_m), FSPL_d0, n_opt, mean(PL_CI_optimized), ...
               mean(X_sigma), RMSE, MAE, MPE, Bias, SDE, R2, X_std];

mean_row = array2table(mean_values, ...
    'VariableNames', {'Distance_m', 'FSPL_d0', 'n', 'CI_PL', 'CI_X_sigma', ...
                      'CI_RMSE', 'CI_MAE', 'CI_MPE', 'CI_Bias', 'CI_SDE', ...
                      'CI_R_squared', 'CI_X_std'});

T = [T; mean_row];

% -------------------------------
% Display results
fprintf('\n--- Optimized CI Model via lsqcurvefit ---\n');
fprintf('Initial n₀: %.4f\n', x0);
fprintf('Optimized n: %.4f\n', n_opt);
fprintf('RMSE:  %.4f dB\n', RMSE);
fprintf('MAE:   %.4f dB\n', MAE);
fprintf('Bias:  %.4f dB\n', Bias);
fprintf('SDE:   %.4f dB\n', SDE);
fprintf('MPE:   %.4f %%\n', MPE);
fprintf('R²:    %.4f\n', R2);
fprintf('Residual Norm: %.4f\n', resnorm);

% -------------------------------
% Export
writetable(T, 'CI_Optimized_lsqcurvefit_with_Xsigma.xlsx');
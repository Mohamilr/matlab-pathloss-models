% Load data
data = readtable('24GHz considered Data_to_plot.xlsx');

% Extract and filter data
d_m = data.Distance_m;
PL_measured = data.Measured_PL;

valid = d_m > 0 & ~isnan(PL_measured);
d_m = d_m(valid);
PL_measured = PL_measured(valid);

% -------------------------------
% Estimate initial guess using linear regression
x = log10(d_m);
y = PL_measured;
A = [ones(size(x)), 10 * x];
params = A \ y;
alpha0 = params(1);
beta0  = params(2);
x0 = [alpha0, beta0];  % Initial guess

% -------------------------------
% Define bounds
alpha_lb = 0;
beta_lb = 0.5;  % close to free-space
alpha_ub = max(PL_measured) + 10;
beta_ub = 6;

lb = [alpha_lb, beta_lb];
ub = [alpha_ub, beta_ub];

% -------------------------------
% Define FI model: PL = alpha + 10 * beta * log10(d)
fi_model = @(params, d) params(1) + 10 * params(2) * log10(d);

% Run optimization
options = optimoptions('lsqcurvefit', 'Display', 'iter');
[opt_params, resnorm, residuals, exitflag, output] = ...
    lsqcurvefit(fi_model, x0, d_m, PL_measured, lb, ub, options);

% Extract optimized parameters
alpha_opt = opt_params(1);
beta_opt  = opt_params(2);

% Predicted path loss
PL_FI_optimized = fi_model(opt_params, d_m);

% -------------------------------
% Evaluation Metrics
errors = PL_FI_optimized - PL_measured;
RMSE = sqrt(mean(errors.^2));
Bias = mean(errors);
SDE = std(errors);
MAE = mean(abs(errors));
MPE = mean(errors ./ PL_measured) * 100;
SS_res = sum((PL_measured - PL_FI_optimized).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R2 = 1 - SS_res / SS_tot;

% -------------------------------
% Display results
fprintf('\n--- Optimized FI Model via lsqcurvefit ---\n');
fprintf('Initial guess: alpha = %.4f, beta = %.4f\n', alpha0, beta0);
fprintf('Optimized Alpha: %.4f\n', alpha_opt);
fprintf('Optimized Beta:  %.4f\n', beta_opt);
fprintf('RMSE:  %.4f dB\n', RMSE);
fprintf('MAE:   %.4f dB\n', MAE);
fprintf('Bias:  %.4f dB\n', Bias);
fprintf('SDE:   %.4f dB\n', SDE);
fprintf('MPE:   %.4f %%\n', MPE);
fprintf('R²:    %.4f\n', R2);
fprintf('Residual Norm: %.4f\n', resnorm);
fprintf('Exit Flag: %d\n', exitflag);

% -------------------------------
% Append predictions to full data
data.FI_PL_Optimized = NaN(height(data), 1);
data.FI_PL_Optimized(valid) = PL_FI_optimized;

% Export to Excel
writetable(data, 'FI_Optimized_lsqcurvefit.xlsx');
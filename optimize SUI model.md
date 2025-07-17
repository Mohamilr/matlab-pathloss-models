% --- Load Excel Data ---
data = readtable('Data_to_plot.xlsx');


d_m = data.Distance_m;
              % dBm
PL_measured = data.Measured_PL;        % Measured path loss
f = 24000;                        % MHz
hb = 25;                          % Base station height (m)
hr = 1.5;                         % User terminal height (m)
d0 = 1;                           % Reference distance (m)
lambda = 3e8 / (f * 1e6);         % Wavelength (m)
A = 20 * log10((4 * pi * d0) / lambda);  % Free-space PL at d0
X_f = 6 * log10(f / 2000);        % Frequency correction
X_h = -10.8 * log10(hr / 2);      % Rx height correction

% --- SUI Model Function for Optimization ---
sui_model = @(params, d) ...
    A + 10 * (params(1) - params(2)*hb + params(3)/hb) .* log10(d / d0) + X_f + X_h;

% Initial guess for parameters [a, b, c1]
x0 = [4.6, 0.0075, 12.6];

% Optimization using nonlinear least squares
opt_params = lsqcurvefit(sui_model, x0, d_m, PL_measured);

% Extract optimized parameters
a_opt  = opt_params(1);
b_opt  = opt_params(2);
c1_opt = opt_params(3);
gamma_opt = a_opt - b_opt * hb + c1_opt / hb;

% --- Predicted Path Loss using Optimized SUI Model ---
PL_SUI_opt = sui_model(opt_params, d_m);
X_sigma = PL_measured - PL_SUI_opt;

% --- Performance Metrics ---
errors = PL_SUI_opt - PL_measured;
RMSE  = sqrt(mean(errors.^2));
MAE   = mean(abs(errors));
Bias  = mean(errors);
SDE   = std(errors);
MPE   = mean(errors ./ PL_measured) * 100;
SS_res = sum((PL_measured - PL_SUI_opt).^2);
SS_tot = sum((PL_measured - mean(PL_measured)).^2);
R2     = 1 - SS_res / SS_tot;
X_std  = std(X_sigma);

% --- Table Assembly ---
n = length(d_m);
T = table(d_m, PL_measured, PL_SUI_opt, X_sigma, ...
    repmat(RMSE, n, 1), repmat(MAE, n, 1), repmat(Bias, n, 1), ...
    repmat(SDE, n, 1), repmat(MPE, n, 1), repmat(R2, n, 1), repmat(X_std, n, 1), ...
    'VariableNames', {'Distance_m', 'PL_measured', 'PL_SUI_opt', 'X_sigma', ...
                      'RMSE', 'MAE', 'Bias', 'SDE', 'MPE', 'R_squared', 'X_std'});

% Append optimized parameters
opt_param_row = array2table([NaN, NaN, NaN, NaN, RMSE, MAE, Bias, SDE, MPE, R2, X_std], ...
    'VariableNames', T.Properties.VariableNames);

T = [T; opt_param_row];

% --- Display Results ---
fprintf('Optimized SUI Parameters:\n');
fprintf('a  = %.4f\n', a_opt);
fprintf('b  = %.6f\n', b_opt);
fprintf('c1 = %.4f\n', c1_opt);
fprintf('gamma = %.4f\n', gamma_opt);
fprintf('RMSE  = %.4f\n', RMSE);
fprintf('R^2   = %.4f\n', R2);

% --- Export to Excel ---
writetable(T, 'Optimized_SUI_Model_Results.xlsx');
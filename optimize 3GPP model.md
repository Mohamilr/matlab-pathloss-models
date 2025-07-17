% Read data from Excel
data = readtable('Data_to_plot.xlsx');

distance = data.Distance_m;
              % dBm
measuredPL = data.Measured_PL; 
% Frequency in GHz
f = 24;

% Model: max(LOS, NLOS)
modelFunc = @(params, d) max( ...
    28 + 22 * log10(d) + 20 * log10(f), ...                 % PL_LOS
    params(1) + params(2) * log10(d) + 20 * log10(f) ...    % PL_NLOS
);

% Initial guess and bounds
params0 = [50, 30];
lb = [0, 0];
ub = [200, 100];
options = optimset('Display','off','MaxIter',1000);

% Curve fitting
[bestParams, ~] = lsqcurvefit(modelFunc, params0, distance, measuredPL, lb, ub, options);

% Predicted values and residuals
predictedPL = modelFunc(bestParams, distance);
residuals = measuredPL - predictedPL;

% Metrics
X_sigma = std(residuals);
RMSE = sqrt(mean(residuals.^2));
MAE = mean(abs(residuals));
MPE = mean((residuals ./ measuredPL) * 100);
Bias = mean(residuals);
SDE = std(residuals);
R_squared = 1 - sum(residuals.^2) / sum((measuredPL - mean(measuredPL)).^2);
X_std = std(residuals);

% Repeat metrics row-wise
n = length(distance);
X_sigma_vec = repmat(X_sigma, n, 1);
RMSE_vec = repmat(RMSE, n, 1);
MAE_vec = repmat(MAE, n, 1);
MPE_vec = repmat(MPE, n, 1);
Bias_vec = repmat(Bias, n, 1);
SDE_vec = repmat(SDE, n, 1);
R2_vec = repmat(R_squared, n, 1);
X_std_vec = repmat(X_std, n, 1);

% Build table using valid MATLAB variable names
resultsTable = table(distance, measuredPL, predictedPL, residuals, ...
    X_sigma_vec, RMSE_vec, MAE_vec, MPE_vec, Bias_vec, SDE_vec, R2_vec, X_std_vec, ...
    'VariableNames', {'Distance_m', 'Measured_PL_dB', 'Predicted_PL_dB', 'Residuals_dB', ...
                      'X_sigma_3GPP', 'RMSE_3GPP', 'MAE_3GPP', 'MPE_3GPP', 'Bias_3GPP', ...
                      'SDE_3GPP', 'R_squared_3GPP', 'X_std_3GPP'});

% Add summary row
mean_values = [ ...
    mean(distance), mean(measuredPL), mean(predictedPL), mean(residuals), ...
    X_sigma, RMSE, MAE, MPE, Bias, SDE, R_squared, X_std];

meanRow = array2table(mean_values, ...
    'VariableNames', resultsTable.Properties.VariableNames);

% Append summary
resultsTable = [resultsTable; meanRow];

% Rename column headers to match your requested format
desiredNames = {'Distance_m', 'Measured_PL_dB', 'Predicted_PL_dB', 'Residuals_dB', ...
                '3GPP_X_sigma', '3GPP_RMSE', '3GPP_MAE', '3GPP_MPE', '3GPP_Bias', ...
                '3GPP_SDE', '3GPP_R_squared', '3GPP_X_std'};

resultsTable.Properties.VariableNames = matlab.lang.makeValidName(desiredNames, 'ReplacementStyle', 'delete');

% Write to Excel
writetable(resultsTable, 'Fitted_Max_LOS_NLOS_Model.xlsx', 'Sheet', 1, 'Range', 'A1');
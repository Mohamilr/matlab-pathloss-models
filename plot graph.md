% Read data from Excel file
T = readtable('AllModels_Pathloss.xlsx');

% Extract required columns
distance_m = T.Distance_;
measured_PL = T.Measured_PL;
fspl = T.FSPL_d0;
ci = T.CI_PL;
fi = T.FI_PL;
abg = T.ABG_PL;
sui = T.SUI_PL;
ecc33 = T.ECC33_PL;
uma = T.PL_UMa_NLOS;

% Create figure
figure;
hold on;

% Scatter plots for each model
scatter(distance_m, measured_PL, 'k', 'filled', 'DisplayName', 'Measured PL');
scatter(distance_m, fspl, 'c', 'o', 'DisplayName', 'FSPL');
scatter(distance_m, ci, 'g', 'd', 'DisplayName', 'CI model');
scatter(distance_m, fi, 'b', '*', 'DisplayName', 'FI model');
scatter(distance_m, abg, 'm', '+', 'DisplayName', 'ABG model');
scatter(distance_m, sui, 'r', 's', 'DisplayName', 'SUI model');
scatter(distance_m, ecc33, [0.8500 0.3250 0.0980], '^', 'DisplayName', 'ECC-33 model'); % orange
scatter(distance_m, uma, [0.4940 0.1840 0.5560], 'v', 'DisplayName', '3GPP UMa NLOS');   % purple

% Labels and styling
xlabel('Distance (m)');
ylabel('Path Loss (dB)');
title('Scatter Plot of Path Loss vs Distance');
legend('show');
grid on;
hold off;

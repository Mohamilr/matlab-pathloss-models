% Read data from Excel file
T = readtable('Optimized_data_to_plot.xlsx');

% Extract required columns
distance_m = T.Distance_m;
measured_PL = T.Measured_PL;
ci = T.Optimized_CI_PL;
fi = T.Optimized_FI_PL;
sui = T.Optimized_SUI_PL;
ecc33 = T.Optimized_ECC33_PL;
uma = T.Optimized_PL_UMa_NLOS;

% Create figure
figure;
hold on;

% Scatter plots with color and marker properly set
scatter(distance_m, measured_PL, 25, 'k', 'filled', 'DisplayName', 'Measured PL');
scatter(distance_m, ci, 25, 'g', 'd', 'DisplayName', 'CI model');
scatter(distance_m, fi, 25, 'b', '*', 'DisplayName', 'FI model');
scatter(distance_m, sui, 25, 'r', 's', 'DisplayName', 'SUI model');
scatter(distance_m, ecc33, 25, 'Marker', '^', 'MarkerEdgeColor', [0.8500 0.3250 0.0980], 'DisplayName', 'ECC-33 model');
scatter(distance_m, uma, 25, 'Marker', 'v', 'MarkerEdgeColor', [0.4940 0.1840 0.5560], 'DisplayName', '3GPP UMa NLOS');

% Labels and legend
xlabel('Distance (m)');
ylabel('Path Loss (dB)');
title('OPTIMIZED PATH LOSS MODEL COMPARISON FOR 24GHz FR2 BAND');
legend('Location', 'best');
grid on;
hold off;
% Load data
filename = 'Compute_average_data.xlsx';
data = readtable(filename);

% Identify numeric columns
numericVars = varfun(@isnumeric, data, 'OutputFormat', 'uniform');

% Initialize a cell array for the mean row
mean_row_cell = repmat({NaN}, 1, width(data));

% Fill numeric columns with their respective mean values
for i = 1:width(data)
    if numericVars(i)
        mean_row_cell{i} = mean(data{:, i}, 'omitnan');
    end
end

% Convert mean row to table with same variable names
mean_row = cell2table(mean_row_cell, 'VariableNames', data.Properties.VariableNames);

% Append to the data table
data_with_mean = [data; mean_row];


% Write to new Excel file
writetable(data_with_mean, '24GHzCompute_average_data_with_mean.xlsx');
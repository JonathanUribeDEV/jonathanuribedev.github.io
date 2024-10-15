---
title: 'From Well Logs to Distributions: How to Use the Kolmogorov-Smirnov Test for Data Fitting - Eng'
date: 2024-10-15 00:00:00 -0500
categories: [Statistics, Geophysics]
tags: [Well Log Analysis, Kolmogorov-Smirnov, Geostatistics, Geophysics]
---

In geophysics and reservoir engineering, analyzing well log data is crucial for understanding the properties of subsurface formations. One of the most common challenges is determining whether a particular dataset, such as porosity or neutron logs, follows a theoretical distribution. This is where the **Kolmogorov-Smirnov (K-S) test** becomes invaluable.

The **K-S test** is a **non-parametric statistical test** that compares the cumulative distribution function (CDF) of an empirical dataset with that of a reference distribution. It evaluates how well the observed data fits a hypothesized distribution by measuring the maximum difference between their CDFs. This test is particularly useful for continuous distributions and provides a clear indication of whether the data deviates significantly from the expected model.

### Why Use the Kolmogorov-Smirnov Test?

In well log analysis, applying the K-S test helps us:

- **Evaluate the goodness of fit**: We can compare the actual log data against a variety of probability distributions (such as normal, log-normal, or gamma distributions).
- **Identify the best-fitting model**: By comparing several distributions, the K-S test helps determine which one best describes the data.
- **Quantify statistical significance**: The test provides a **p-value**, which tells us whether the fit is statistically significant.

For this analysis, we will focus on **well log data**, specifically the **NEUT data**, which represents neutron porosity measurements. These logs are key indicators in formation evaluation, helping to estimate the porosity of rock formations. By applying the Kolmogorov-Smirnov test and other statistical techniques, we aim to determine the best probability distribution to model this NEUT data.

In the following sections, we'll walk through the steps of preparing the data, applying the K-S test, and visualizing the results to understand the distribution that best fits the NEUT data.

### **Loading and Preparing the Data**

First, we need to load and clean our well log data, specifically focusing on the porosity logs (`DPHI` column). We’ll remove outliers and normalize the data to make it more suitable for statistical analysis.

```python
# Importing necessary libraries for the analysis
import lasio  # Library to read LAS files (Log ASCII Standard)
import lasio.examples  # Example LAS files from lasio package
import scipy  # Scientific computing and statistical tools
import scipy.stats  # Module for statistical functions
from sklearn.preprocessing import StandardScaler  # Used for scaling features
import pandas as pd  # For data manipulation and analysis
import numpy as np  # Numerical operations on arrays
import seaborn as sns  # Data visualization library for statistical graphics
import matplotlib.pyplot as plt  # Plotting library for creating static, animated, and interactive visualizations

# This line allows inline plotting in Jupyter Notebooks
%matplotlib inline

# Loading an example LAS file provided by lasio
las = lasio.examples.open("6038187_v1.2.las")

# Converting LAS data into a Pandas DataFrame for easier analysis
df = las.df()

# Checking the column names of the loaded DataFrame
df.columns  # Output: ['CALI', 'DFAR', 'DNEAR', 'GAMN', 'NEUT', 'PR', 'SP', 'COND']

# Checking the shape of the 'NEUT' column, i.e., the number of rows
df['NEUT'].shape  # Output: (2732,) -> 2732 rows

# Extracting the 'DPHI' (porosity log) data from the DataFrame and removing NaN values
porosity_logs = logs['DPHI'].dropna()

# Keeping only non-negative values (>= 0) from the porosity logs
porosity_logs = porosity_logs[porosity_logs >= 0]

# Displaying descriptive statistics for the porosity logs (mean, std, min, max, etc.)
porosity_logs.describe()

# Calculating the mean value of the porosity logs
mean = porosity_logs.mean()
```

### Plotting the distributions

```python
# Plotting the distribution of porosity logs using Seaborn with a Kernel Density Estimate (KDE)
sns.displot(porosity_logs, kde=True, color='blue', height=10)
# Adding a vertical red line on the plot to represent the mean value of porosity logs
plt.axvline(mean, 0, 1, color='red')
```

![https://link.storjshare.io/raw/juu3fb56sf7lrmassdwtjtlhpnba/images/02/dist.png](https://link.storjshare.io/raw/juu3fb56sf7lrmassdwtjtlhpnba/images/02/dist.png)

```python
# Setting the default figure size for all Seaborn plots to be 12x10 inches
sns.set(rc={'figure.figsize':(12,10)})
# Plotting the Kernel Density Estimate (KDE) for the porosity logs with a blue color
# 'fill=True' fills the area under the KDE curve
sns.kdeplot(porosity_logs, color='blue', fill=True)
# Adding a vertical red line at the mean value of the porosity logs
plt.axvline(mean, 0, 1, color='red')
```

![https://link.storjshare.io/raw/ju7jdobw5agkys6soyanbr4tgi3a/images/02/dist2.png](https://link.storjshare.io/raw/ju7jdobw5agkys6soyanbr4tgi3a/images/02/dist2.png)

### Removing Outliers and Normalizing Data

We start by cleaning the data, removing any negative values from the porosity logs (`DPHI`), and then standardizing the data using `StandardScaler` from `sklearn`.

```python
# Initializing the StandardScaler object to standardize features (mean=0, std=1)
scaler = StandardScaler()

# Converting the porosity logs to a NumPy array and reshaping it to a 2D array (required for scaling)
y = np.array(porosity_logs.values).reshape(-1, 1)

# Fitting the StandardScaler to the reshaped porosity data (calculating mean and standard deviation)
scaler.fit(y)

# Transforming (normalizing) the porosity data using the calculated mean and standard deviation
y_normalized = scaler.transform(y)

# Flattening the normalized data back to a 1D array for ease of plotting
y_normalized = y_normalized.flatten()

# Plotting the Kernel Density Estimate (KDE) of the normalized porosity data with a blue color and filled curve
sns.kdeplot(y_normalized, color='blue', fill=True)

# Adding a vertical red line at the mean of the normalized porosity data
plt.axvline(y_normalized.mean(), 0, 1, color='red')
```

The standardized porosity data now follows a mean of 0 and standard deviation of 1, making it easier to compare across different distributions.

![https://link.storjshare.io/raw/jw2cmdb2otgtxbqzqa2zf44a7glq/images/02/distNormalized.png](https://link.storjshare.io/raw/jw2cmdb2otgtxbqzqa2zf44a7glq/images/02/distNormalized.png)

### Fitting Probability Distributions

We fit the normalized data to a set of candidate distributions using `scipy.stats`. The goal is to find which distribution provides the best fit by calculating both the chi-square and Kolmogorov-Smirnov statistics.

As a litte reminder,  the **lower the chi-square value**, the better the fit of the distribution to the observed data.

A **p-value greater than 0.05** means that the adjusted distribution is **not significantly different** from the observed data's distribution.

For large datasets, it is common to have **Kolmogorov-Smirnov p-values less than 0.05**, indicating that the adjusted distribution is statistically different from the observed distribution. However, this does not necessarily mean the fit is poor.

In our analysis, we will fit **10 different distributions**, rank them by their chi-square goodness-of-fit values, and report the corresponding Kolmogorov-Smirnov (KS) p-values. Remember, we aim for **the lowest chi-square value** (indicating the best fit), and ideally, we want the **KS p-value to be greater than 0.05**, suggesting that the adjusted distribution is not significantly different from the observed data.

```python
import scipy.stats
import numpy as np

# Candidate distributions
dist_names = ['norminvgauss', 'johnsonsu', 'nct', 'levy_stable', 'cauchy', 't', 'burr12', 'loglaplace', 'foldcauchy', 'mielke', 'norm']

# Initialize lists for storing results
chi_square = []
p_values = []

# Percentile bins for chi-square test
percentile_bins = np.linspace(0, 100, 51)
percentile_cutoffs = np.percentile(y_normalized, percentile_bins)
observed_frequency, bins = np.histogram(y_normalized, bins=percentile_cutoffs)
cum_observed_frequency = np.cumsum(observed_frequency)

# Fit the distributions and calculate statistics
for distribution in dist_names:
    dist = getattr(scipy.stats, distribution)
    param = dist.fit(y_normalized)

    # Kolmogorov-Smirnov test
    p = scipy.stats.kstest(y_normalized, distribution, args=param)[1]
    p_values.append(p)

    # Calculate expected frequencies for chi-square test
    cdf_fitted = dist.cdf(percentile_cutoffs, *param[:-2], loc=param[-2], scale=param[-1])
    expected_frequency = np.diff(cdf_fitted) * len(y_normalized)
    chi_square.append(np.sum(((expected_frequency - cum_observed_frequency) ** 2) / cum_observed_frequency))

```

### 3. **Ranking the Distributions**

Once we’ve calculated the K-S and chi-square statistics, we can sort the results to see which distribution best fits our data. Here, the **chi-square** values give an overall measure of goodness of fit, while the **p-values** from the K-S test help us determine if the differences between our sample and the fitted distributions are significant.

```python
# Organize results into a DataFrame
results = pd.DataFrame({
    'Distribution': dist_names,
    'chi_square': chi_square,
    'p_value': p_values
}).sort_values('chi_square')

# Print the top distributions
print(results.head(5))
```

### 4. **Visualizing the Results**

With our top distributions ranked, let’s visualize the results. We’ll plot the observed data along with the probability density functions (PDFs) of the top three distributions. This will give us a clear picture of how well each distribution models the porosity data.

```python
y = porosity_logs.values

# Divide the observed data into 100 bins for plotting (this can be changed)
number_of_bins = 100
bin_cutoffs = np.linspace(np.percentile(y,0), np.percentile(y,99),number_of_bins)

# Create the plot
h = plt.hist(y, bins = bin_cutoffs, color='0.75')
x = np.arange(len(y))
# Get the top three distributions from the previous phase
number_distributions_to_plot = 3
dist_names = results['Distribution'].iloc[0:number_distributions_to_plot]

# Create an empty list to stroe fitted distribution parameters
parameters = []

# Loop through the distributions ot get line fit and paraemters

for dist_name in dist_names:
    # Set up distribution and store distribution paraemters
    dist = getattr(scipy.stats, dist_name)
    param = dist.fit(y)
    parameters.append(param)

    # Get line for each distribution (and scale to match observed data)
    pdf_fitted = dist.pdf(x, *param[:-2], loc=param[-2], scale=param[-1])
    scale_pdf = np.trapz (h[0], h[1][:-1]) / np.trapz (pdf_fitted, x)
    pdf_fitted *= scale_pdf

    # Add the line to the plot
    plt.plot(pdf_fitted, label=dist_name)

    # Set the plot x axis to contain 99% of the data
    # This can be removed, but sometimes outlier data makes the plot less clear
    plt.xlim(0,np.percentile(y,99))

# Add legend and display plot

plt.legend()
plt.show()

# Store distribution paraemters in a dataframe (this could also be saved)
dist_parameters = pd.DataFrame()
dist_parameters['Distribution'] = (results['Distribution'].iloc[0:number_distributions_to_plot])
dist_parameters['Distribution parameters'] = parameters

# Print parameter results
print ('\nDistribution parameters:')
print ('------------------------')

for index, row in dist_parameters.iterrows():
    print ('\nDistribution:', row[0])
    print ('Parameters:', row[1] )
```

### 5. **Quantile-Quantile (Q-Q) and Probability-Probability (P-P) Plots**

To further evaluate the fit of the distributions, we’ll use **Q-Q plots** and **P-P plots**. These plots compare the theoretical quantiles and cumulative distributions of the fitted distributions with the observed data.

```python
## qq and pp plots

data = y_normalized.copy()
data.sort()

# Loop through selected distributions (as previously selected)

for distribution in dist_names:
    # Set up distribution
    dist = getattr(scipy.stats, distribution)
    param = dist.fit(y_normalized)

    # Get random numbers from distribution
    norm = dist.rvs(*param[0:-2],loc=param[-2], scale=param[-1],size = size)
    norm.sort()

    # Create figure
    fig = plt.figure(figsize=(10,8))

    # qq plot
    ax1 = fig.add_subplot(121) # Grid of 2x2, this is suplot 1
    ax1.plot(norm,data,"o")
    min_value = np.floor(min(min(norm),min(data)))
    max_value = np.ceil(max(max(norm),max(data)))
    ax1.plot([min_value,max_value],[min_value,max_value],'r--')
    ax1.set_xlim(min_value,max_value)
    ax1.set_xlabel('Theoretical quantiles')
    ax1.set_ylabel('Observed quantiles')
    title = 'qq plot for ' + distribution +' distribution'
    ax1.set_title(title)

    # pp plot
    ax2 = fig.add_subplot(122)

    # Calculate cumulative distributions
    bins = np.percentile(norm,range(0,101))
    data_counts, bins = np.histogram(data,bins)
    norm_counts, bins = np.histogram(norm,bins)
    cum_data = np.cumsum(data_counts)
    cum_norm = np.cumsum(norm_counts)
    cum_data = cum_data / max(cum_data)
    cum_norm = cum_norm / max(cum_norm)

    # plot
    ax2.plot(cum_norm,cum_data,"o")
    min_value = np.floor(min(min(cum_norm),min(cum_data)))
    max_value = np.ceil(max(max(cum_norm),max(cum_data)))
    ax2.plot([min_value,max_value],[min_value,max_value],'r--')
    ax2.set_xlim(min_value,max_value)
    ax2.set_xlabel('Theoretical cumulative distribution')
    ax2.set_ylabel('Observed cumulative distribution')
    title = 'pp plot for ' + distribution +' distribution'
    ax2.set_title(title)

    # Display plot
    plt.tight_layout(pad=4)
    plt.show()
```

![https://link.storjshare.io/raw/jupjahoqp774ddjerlpzycvm6xdq/images/02/qqpp.png](https://link.storjshare.io/raw/jupjahoqp774ddjerlpzycvm6xdq/images/02/qqpp.png)

![https://link.storjshare.io/raw/jv2xcglipkep462buhjmav4rk5aq/images/02/qqpp2.png](https://link.storjshare.io/raw/jv2xcglipkep462buhjmav4rk5aq/images/02/qqpp2.png)

![https://link.storjshare.io/raw/jwfdtsdjyo5mit7c5ogqmbzhpmoa/images/02/qqpp3.png](https://link.storjshare.io/raw/jwfdtsdjyo5mit7c5ogqmbzhpmoa/images/02/qqpp3.png)

### Conclusion

The Kolmogorov-Smirnov test, combined with the chi-square test, provides a robust framework for evaluating how well different probability distributions model observed data. In this example, we applied these techniques to a porosity log from well data and identified the best-fitting distributions using both visual and statistical analysis. These insights are crucial in fields like reservoir evaluation, where understanding subsurface characteristics is essential for making informed decisions.

We walked through the process of visualizing well log data, cleaning and normalizing the data, fitting it to multiple probability distributions, and finally ranking those distributions based on goodness of fit. These statistical techniques are crucial for understanding subsurface characteristics and making informed decisions in reservoir evaluation.

### Best Fitting Distributions for NEUT Data

After performing the Kolmogorov-Smirnov (K-S) test and calculating the chi-square values, we identified the top three probability distributions that best fit the **NEUT data** from well logs. These distributions, ranked by goodness of fit, are:

1. **Johnson Su (johnsonsu)**
2. **Normal Inverse Gaussian (norminvgauss)**
3. **Non-Central T Distribution (nct)**

Let’s take a closer look at each of these distributions and discuss their parameters, fit quality, and what this means for our analysis.

### 1. **Johnson Su Distribution**

- **Chi-square value**: 471.87
- **p-value**: 0.0
- **Parameters**:
    - Shape parameter (γ): -2.247
    - Scale parameter (δ): 0.738
    - Location parameter (ξ): 103.79
    - Scale parameter (ω): 15.24

The **Johnson Su distribution** is a flexible distribution that can model skewed data with heavy tails. It is a good choice for data that is not symmetric, which aligns with the characteristics often seen in well log data. The chi-square value is relatively low, indicating that this distribution fits the NEUT data very well. The **p-value of 0.0** suggests that the fit is statistically significant, and the distribution captures the overall pattern of the NEUT data quite effectively.

### 2. **Normal Inverse Gaussian Distribution (Norminvgauss)**

- **Chi-square value**: 527.43
- **p-value**: 0.0
- **Parameters**:
    - Shape parameter (α): 6.110
    - Location parameter (β): 6.094
    - Scale parameter (δ): 85.79
    - Location parameter (λ): 25.82

The **Normal Inverse Gaussian** (NIG) distribution is commonly used for modeling data with skewness and heavy tails, making it suitable for financial and geophysical data like NEUT logs. The chi-square value is slightly higher than that of the Johnson Su distribution, but still quite reasonable, indicating a good fit to the data. The **p-value** is also 0.0, reinforcing the statistical significance of this fit.

### 3. **Non-Central T Distribution (Nct)**

- **Chi-square value**: 550.61
- **p-value**: 0.0
- **Parameters**:
    - Degrees of freedom (ν): 1.38
    - Non-centrality parameter (λ): 37.39
    - Location parameter (ξ): -0.27
    - Scale parameter (ω): 5.51

The **Non-Central T Distribution** (NCT) is a generalization of the Student’s T distribution and is used to model data that is heavy-tailed and has a large amount of variability. It is particularly useful when the data is not centered around a single value. Although the chi-square value is a bit higher than the Johnson Su and NIG distributions, it still represents a good fit, particularly for data with heavy tails.

### In Summary

- The **Johnson Su distribution** emerges as the best model for the NEUT data, with the lowest chi-square value and the highest goodness of fit. It effectively captures skewness and heavy tails, making it ideal for neutron porosity logs in well log analysis.
- The **Normal Inverse Gaussian** distribution is also a strong candidate, with slightly higher chi-square values but still significant. It is flexible and robust for capturing the characteristics of the NEUT data.
- The **Non-Central T distribution** is a good fit, but it lags slightly behind the other two in terms of chi-square values.

For this **well log data**, the **Johnson Su distribution** provides the best statistical fit. Its ability to capture skewness and heavy tails makes it particularly suited to model well log data, which often exhibits such characteristics. Using this distribution will improve the accuracy of any subsequent analysis or modeling that involves neutron porosity data.

These findings are critical for further modeling of well log data, where accurate probabilistic representations of subsurface properties can inform drilling and reservoir management decisions.
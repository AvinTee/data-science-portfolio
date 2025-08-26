# Forecasting British Automobile Stocks' Price-Trends with ARIMA

## Introduction
This project seeks to determine how market-volatility affects the short-term future price-trends of three British car-manufacturers' stocks, namely TATA Motors, BMW and Rolls-Royce's.

## Literature Review 
Researchers’ perspectives regarding the effectiveness of ARIMA-based share-price forecasting has shifted in recent years. Despite ARIMA’s historical popularity in time-series modelling (Zhang, 2003), current multi-method quantitative studies investigating share-price and demand-modelling have outlined that the technique’s prediction-power and forecast-accuracy were inferior to deep-learning’s (Bousqaoui, Slimani and Achchab, 2021; Panchal, Ferdouse and Sultana, 2024; Siami-Namini, Tavakoli and Namin, 2018; Adebiyi et al., 2014).

While this consensus on ARIMA’s shortcomings is not universal, with
Ahammad et al.’s (2024) investigation into Apex Food shares demonstrating the contrary, these findings still illustrate ARIMA’s challenges with modelling prices in agitated markets.

## Data Infrastructure
This project used the pandas library's inherent data-quality methods to develop its quality-validation functions, the yfinance API to collect the price-data needed for model-development, the statsmodel package to implement its ARIMA-models and statistical tests, and the matplotlib module to create visualisations
to support its analyses.

## Data Engineering 
![Figure 1: Data Pipeline](images/ARIMA_Project_Data_Pipeline.png)

*Figure 1: Data Pipeline*

This project uses yfinance-extracted historical daily close-price data from 01/01/2020 to 31/07/2025 for its price-trend forecasts.  

80% of the data was reserved for model-training, 10% for model-validation and the final 10% for model-testing. 

The train-data was validated against DAMA's six quality-dimensions (Government Data Quality Hub, 2021) using custom Python functions. The only data-quality issues the validation-functions found were nulls and inconsistencies. 

```
def NullsDecompose(tables):
  dt = datetime.datetime.strptime
  FORMAT = '%Y-%m-%d'
  COUNT = dt(END, FORMAT) - dt(START, FORMAT)
  COUNT = COUNT.days
  nulls = {'TABLES':[], 'COLUMNS':[], '% NULLS':[]}
  for table, name in tables:
    _table = table.reset_index()
    for col in [*_table]:
      base = _table[col]
      null_vals = COUNT-base[base.notnull()].count()
      nulls['% NULLS'].append(null_vals/COUNT)
      nulls['COLUMNS'].append(col)
      nulls['TABLES'].append(name)
  nulls = pandas.DataFrame.from_dict(nulls)
  return nulls.style.format({'% NULLS':'{:.0%}'})
```

*Figure 2: Function to check for nulls*

The 32%, 30% and 31% null data populating the TATA, BMW, and Rolls-Royce’s close-price datasets was patched through linear interpolation.

```
def ConsistencyDecompose(tables):
  inconsistent = {'TABLES':[], 'COLUMNS':[], '% MAX INCONSISTENT':[]}
  REFDATA = {
      "TATA Shares" : pandas.read_csv('TATAMOTORS - Reference.csv'),
      "BMW Shares" : pandas.read_csv('BMW - Reference.csv'),
      "Rolls-Royce Shares" : pandas.read_csv('RR - Reference.csv')
  }
  dt = datetime.datetime.strptime
  FORMAT = '%d/%m/%Y'
  for table, name in tables:
      _table = table.reset_index()
      _table['Date'] = _table['Date'].dt.tz_localize(None)
      refsource = REFDATA[name]
      convert_dt = lambda t: dt(t.split(" ")[0], FORMAT)
      refsource['Date'] = refsource['Date'].apply(convert_dt)
      base = _table.merge(right=refsource, how='left', on='Date')
      COUNT = len(base)
      base['MATCH'] = (base['Close_x'] - base['Close_y']).abs()/base.iloc[:,1]
      inconsistent['% MAX INCONSISTENT'].append(base['MATCH'].max())
      inconsistent['COLUMNS'].append('PRICE')
      inconsistent['TABLES'].append(name)
  inconsistent = pandas.DataFrame.from_dict(inconsistent)
  return inconsistent.style.format({'% MAX INCONSISTENT':'{:.0%}'})
```

*Figure 3: Function to check for inconsistencies*

Since the maximum difference between the yfinance datasets used for this project's data-collection and the Google Finance tables used as this project's reference data was only 3%, no consistency-fixing cleansing action was taken.

## Exploratory Data Analysis
![Figure 4: Trends diagram](images/Graphs_Trends.png)

*Figure 4: Trends diagram*

![Figure 5: Seasonality diagram](images/Graphs_Seasonality.png)

*Figure 5: Seasonality diagram*

Visualising TATA, BMW and Rolls-Royce's train-datasets' trend and seasonality charts showcased that all three stocks exhibited both periodicity and non-zero trends - two time-series properties that violate ARIMA's non-stationarity assumption (Cheng, 2015; Ryan, Haslbeck, and Waldorp, 2025). Therefore, the data was differenced before being fed to ARIMA models.

![Figure 6: Residuals diagram](images/Graphs_Residuals.png)

*Figure 6: Residuals diagram*

![Figure 7: Residuals distribution diagram](images/Graphs_Residual_Distribution.png)

*Figure 7: Residuals distribution diagram*

Plotting the residuals of the resultant differenced datasets revealed the presence of leptokurtic distributions and outliers, both of which could distort ARIMA's prediction-intervals (Ledolter, 1979; Ledolter, 1989). 

![Figure 8: Heatmap comparing transformations' kurtosis-reduction](images/Graphs_Transformations_Comparison.png)

*Figure 8: Heatmap comparing transformations' kurtosis-reduction*

Therefore, outliers were removed, the resultant nulls were replaced through linear interpolation, and the final outlier-free datasets were transformed through the kurtosis-minimising arcsinh-function to resolve the data's leptokurtic distributions. 

## Data Analysis
The null hypothesis this project posits is that market volatility does not affect TATA, BMW and Rolls-Royce's future price-trends, with the alternative stating that there is some effect.

The Akaike-Information-Criteria (AIC) for different model-orders were compared to determine which ARIMA-models should be used for TATA, BMW and Rolls-Royce's price-trend forecasts. Only model-orders with a differencing level of 0 were considered to reflect the fact that project's train-datasets have already undergone differencing.

![Figure 9: AIC table](images/AIC_Table.png)

*Figure 9: AIC table*

Since ARIMA(0,0,3), ARIMA(0,0,2) and ARIMA(3,0,3) minimise the AIC-scores for TATA, BMW, and Rolls-Royce's train-data, respectively, they were chosen as this project's ARIMA-models.

![Figure 10: Validation-set predictions](images/Graphs_Model_Validation.png)

*Figure 10: Validation-set predictions*

![Figure 11: Stock-price regimes](images/Graphs_Regimes.png)

*Figure 11: Stock-price regimes*

However, ARIMA's predicted price-trends did not seem to fit the stock-price data well. Therefore, the most recent regimes of each of TATA, BMW and Rolls-Royce's close-price data were used to re-train the ARIMA-models, and determine new model-orders.

![Figure 12: Validation-set error-metrics](images/Validation_Error_Metrics.png)

*Figure 12: Validation-set error metrics*

![Figure 13: Test-set error-metrics](images/Test_Set_Error_Metrics.png)

*Figure 13: Test-set error metrics*

Although the low square and absolute errors indicated that the models performed well on the test-data, the fact that the error-metrics decrease from the validation-set to the test-set also implies that the test-data was too easy (Sivakumar, Parthasarathy, and Padmapriya, 2014). Future work could explore Bootstrap resampling to increase the test-set's diversity and achieve a more realistic picture of the models' performance (LeBaron and Weigend, 1998). 

![Figure 14: Hypothesis test p-values](images/Hypothesis_Test_Results.png)

*Figure 14: Hypothesis test's p-values*

Since only some and not all of the p-values for this project's ARIMA model-variables indicate a failure to reject the null hypothesis, the models were not reformulated.

![Figure 15: Statistical tests' p-values](images/Statistical_Test_Results.png)

*Figure 15: Statistical tests' p-values*

Inspecting the p-values of different statistical tests show that besides ARIMA's residual normality assumption, the other model assumptions were satisfied by the cleaned and transformed datasets.

![Figure 16: 1.5 month forecast](images/Graphs_Forecast.png)

*Figure 16: 1.5 month forecast*

This project's key findings were it expects with 95% confidence that TATA, BMW and Rolls-Royce's stock-prices will flatten from 01/08/2025 to 15/09/2025. The commercial impact of these findings is that it should motivate British car-businesses to tackle the conservative growth-prospects of the auto-industry.

## Conclusion
This report concludes that market-volatility is predicted to slow down TATA, BMW and Rolls-Royce's price-growth. In the future, further work could be done to assess the reliability of these results.

## References
Adebiyi, A. A., Adewumi, A. O. & Ayo, C. K. (2014), ‘Comparison of arima and artificial neural networks models for stock price prediction’, Journal of Applied Mathematics 2014(1). doi:10.1155/2014/614342

Ahammad, I., Sarkar, W. A., Meem, F. A., Ferdus, J., Ahmed, M. K., Rahman, M. R., Sultana, R. & Islam, M. S. (2024), ‘Advancing stock market predictions with time series analysis including lstm and arima’, Cloud Computing and Data Science pp. 226–241. doi:10.37256/ccds.5220244470

Bousqaoui, H., Slimani, I. & Achchab, S. (2021), ‘Comparative analysis of short-term demand predicting models using arima and deep learning’, International Journal of Electrical and Computer Engineering 11(4), 3319. doi:10.11591/ijece.v11i4.pp3319-3328

Cheng, C., Sa-Ngasoongsong, A., Beyca, O., Le, T., Yang, H., Kong, Z. & Bukkapatnam, S. T. (2015), ‘Time series forecasting for nonlinear and non-stationary processes: a review and comparative study’, Iie Transactions 47(10), 1053–1071. doi:10.1080/0740817X.2014.999180

Government Data Quality Hub (2021), ‘Meet the data quality dimensions’, Available at: https://www.gov.uk/government/news/meet-the-data-quality-dimensions. (Accessed: 2025-06-28).

Ledolter, J. (1979), ‘Inference robustness of arima models under non-normality—special application to stock price data’, Metrika 26(1), pp. 43–56. Available at: https://pure.iiasa.ac.at/id/eprint/609/1/RM-76-075.pdf (Accessed:
2025-07-28).

Ledolter, J. (1989), ‘The effect of additive outliers on the forecasts from arima models’, International Journal of Forecasting 5(2), pp. 231–240. doi:10.1016/0169-2070(89)90090-3

Panchal, S. A., Ferdouse, L. & Sultana, A. (2024), Comparative analysis of arima and lstm models for stock price prediction, in ‘2024 IEEE/ACIS 27th International Conference on Software Engineering, Artifi￾cial Intelligence, Networking and Parallel/Distributed Computing (SNPD)’, IEEE, pp. 240–244. Available at:
https://ieeexplore.ieee.org/abstract/document/10673919 (Accessed: 2025-06-24).

Ryan, O., Haslbeck, J. M. & Waldorp, L. J. (2025), ‘Non-stationarity in time-series analysis: Modeling stochastic and deterministic trends’, Multivariate Behavioral Research 60(3), pp. 556–588. doi:10.1080/00273171.2024.2436413.

Siami-Namini, S., Tavakoli, N. & Namin, A. S. (2018), A comparison of arima and lstm in forecasting time series, in ‘2018 17th IEEE international conference on machine learning and applications (ICMLA)’, IEEE, pp. 1394–1401. Available at: https://par.nsf.gov/servlets/purl/10186768 (Accessed: 2025-06-24).

Sivakumar, M., Parthasarathy, S. & Padmapriya, T. (2024), ‘Trade-off between training and testing ratio in machine learning for medical image processing’, PeerJ Computer Science 10, e2245.

Zhang, G. P. (2003), ‘Time series forecasting using a hybrid arima and neural network model’, Neurocomputing 50, pp. 159–175. doi:10.1016/S0925-2312(01)00702-0.





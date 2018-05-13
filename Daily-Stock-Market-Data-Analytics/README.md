# Pipeline Demo: Daily Stock Market Data Analytics *(Buildout and documentation in progress)*

#### End-to-end pipeline demonstrating the implementation of the following steps:
1. **Collect data from API and store in data lake**: Python, AWS CLI, AWS S3, AWS EC2 *(In progress)*
    * Download list of <a href="https://www.nasdaq.com/screening/companies-by-industry.aspx" target="_blank">NASDAQ companies</a>.
    * Filter out companies for which to collect market data.
    * Query Daily Adjusted Stock Market Time Series <a href="https://www.alphavantage.co/documentation" target="blank">web service</a> for all specified companies.<br><br>
2. **Transform data in data lake** Python, PySpark, AWS EMR (Spark) *(Coming soon)*
3. **Import data into analytics columnar database**: AWS Redshift, SQL *(Coming soon)*
4. **Build pipeline orchestration & scheduling engine**: Python, Apache Airflow *(Coming soon)*
5. **Surface data to public using RESTful web API**: Python, Flask *(Coming soon)*


# Part 1: Collect data from API and store in data lake

### Get configurations.

Create function to get configuration value from configuration JSON file. By default, configuration file is located in the current working directory and named configuration.json.

```python
def getConfigurationValue(configurationKey):

    # Import packages and functions.
    import json, os

    # Get current working directory.
    currentWorkingDirectory = os.getcwd()

    # Load configuration file.
    with open(os.path.join(currentWorkingDirectory, 'configuration.json'), 'r') as configurationFile:
        dictConfigurations = json.load(configurationFile)

    return(dictConfigurations[configurationKey])
```

**Test:** Print sample configuration values.

```python
print('NASDAQCompaniesSourceURL:', getConfigurationValue('NASDAQCompaniesSourceURL'))
print('NASDAQCompaniesDestinationPathPart:', getConfigurationValue('NASDAQCompaniesDestinationPathPart'))
```

**Result:**

NASDAQCompaniesSourceURL: https://www.nasdaq.com/screening/companies-by-industry.aspx?exchange=NASDAQ&render=download
<br>NASDAQCompaniesDestinationPathPart: Data/NASDAQCompanies.csv

### Programmatically download list of <a href="https://www.nasdaq.com/screening/companies-by-industry.aspx" target="blank">NASDAQ companies</a>.

Create function to download companies CSV into Pandas data frame.

```python
def getNASDAQCompanies(downloadFile = False):
    # Import package(s)/function(s).
    import os, urllib
    import pandas as pd
    
    # Declare variables.

    # Get current working directory.
    currentWorkingDirectory = os.getcwd()

    # Get source URL for NASDAQ companies file.
    NASDAQCompaniesSourceURL = getConfigurationValue('NASDAQCompaniesSourceURL')
    
    # Get destination path part for NASDAQ companies file.
    NASDAQCompaniesDestinationPathPart = getConfigurationValue('NASDAQCompaniesDestinationPathPart')
    
    # Get destination path for NASDAQ companies file.
    NASDAQCompaniesDestinationPath = os.path.abspath(os.path.join(currentWorkingDirectory, NASDAQCompaniesDestinationPathPart))

    # Download data file, if so specified.
    if (downloadFile):
        # Remove destination file, it it exists.
        if(os.path.isfile(NASDAQCompaniesDestinationPath)):
            os.remove(NASDAQCompaniesDestinationPath)

        # Download file.
        downloadedFile, downloadedFileHeaders = urllib.request.urlretrieve(NASDAQCompaniesSourceURL,
                                                                           NASDAQCompaniesDestinationPathPart)

    # Create dfCompanies data frame from NASDAQ companies file.
    dfCompanies = pd.read_csv(NASDAQCompaniesDestinationPathPart)
    
    # Drop emppty last column created as a result of trailing comma.
    dfCompanies.drop(dfCompanies.columns[-1], axis=1, inplace=True)

    return(dfCompanies)
```

**Test:** Download companies CSV into Pandas data frame.

```python
# dfCompanies = getNASDAQCompanies(downloadFile=True)
dfCompanies = getNASDAQCompanies()

print('dfCompanies count:', len(dfCompanies))

dfCompanies.head(10)
```
**Result:**

![Screenshot](https://s3.amazonaws.com/public-k0l1081/dfCompanies.head(10).PNG)

Create function to filter companies by Symbol or Name.

```python
def getCompaniesByKeyword(dfCompanies, companyKeyword):
    companyKeyword = companyKeyword.lower()
    # Filter dfCompanies by Symbol or Name based on search argument.
    # Use vectorized string methods: lower() and contains()
    dfCompanies = dfCompanies[dfCompanies.Symbol.str.lower().str.contains(companyKeyword)
                              | dfCompanies.Name.str.lower().str.contains(companyKeyword)]
    
    return(dfCompanies)
```

**Test:** Filter companies by Symbol or Name.

```python
# Filter by 'technologies'.
dfCompaniesFiltered = getCompaniesByKeyword(dfCompanies, 'technologies')

print('dfCompaniesFiltered count:', len(dfCompaniesFiltered))

dfCompaniesFiltered.head(10)
```

**Result:**

dfCompaniesFiltered count: 67

![Screenshot](https://s3.amazonaws.com/public-k0l1081/dfCompaniesFiltered.head(10).PNG)

### Query Alpha Advantage's Daily Adjusted Stock Market Time Series <a href="https://www.alphavantage.co/documentation/#dailyadj" target="blank">web service</a> for all specified companies.

Create function to query web service to load daily adjusted time series for specified stock symbol into Pandas data frame. Optionally, save service response as JSON file.

```python
def getAlphaAdvantageDailyAdjustedStockMarketTimeSeries(ticker, downloadFile = False):
    # Import package(s)/function(s).
    import os, json, requests
    import pandas as pd
    
    # Declare variables.

    # Get current working directory.
    currentWorkingDirectory = os.getcwd()

    # Get API Key.
    APIKey = getConfigurationValue('AlphaVantageAPIKey')
    
    # Get service request URL template.
    serviceRequestURLTemplate = getConfigurationValue('AlphaVantageTimeSeriesDailyAdjustedServiceRequestURLTemplate')
    
    # Get service request URL.
    serviceRequestURL = serviceRequestURLTemplate.replace('$symbol$', ticker).replace('$apiKey$', APIKey)
    
    # Get destination path part.
    destinationPathPart = getConfigurationValue('AlphaAdvantageTimeSeriesDailyAdjustedDestinationPathPart')

    # Get destination path.
    destinationPath = os.path.abspath(os.path.join(currentWorkingDirectory, destinationPathPart, ticker + '.json'))

    # Issue request to service. Get response.
    response = requests.get(serviceRequestURL)
    
    # Convert response to JSON format.
    JSONResponse = response.json()

    # Get time series dictionary from JSON.
    dictResponse = JSONResponse['Time Series (Daily)']
    
    # Specify column order to use for data frame.
    columns = ['open', 'high', 'low', 'close', 'adjustedClose', 'volume', 'dividend', 'split']
    
    # Construct data frame from dictionary.
    # Rename columns.
    # Use specified column order.
    dfMarketData = (pd.DataFrame
                    .from_dict(dictResponse, orient='index')
                    .rename(columns={'1. open':'open',
                                     '2. high':'high',
                                     '3. low':'low',
                                     '4. close':'close',
                                     '5. adjusted close':'adjustedClose',
                                     '6. volume':'volume',
                                     '7. dividend amount':'dividend',
                                     '8. split coefficient':'split'})
                   [columns])
    
    # Download data file, if so specified.
    if (downloadFile):
        # Remove destination file, it it exists.
        if(os.path.isfile(destinationPath)):
            os.remove(destinationPath)
            
        with open(destinationPath, 'w') as fileDestinationPath:
            json.dump(JSONResponse, fileDestinationPath, ensure_ascii=False, indent=0, sort_keys=True)

    # Return Pandas data frame.
    return(dfMarketData)
```

**Test:** Query web service to get daily adjusted time series for **Amazon**.

```python
#getAlphaAdvantageDailyAdjustedStockMarketTimeSeries('AMZN', downloadFile=True)
getAlphaAdvantageDailyAdjustedStockMarketTimeSeries('AMZN').head(10)
```

**Result:**

![Screenshot](https://s3.amazonaws.com/public-k0l1081/getAlphaAdvantageDailyAdjustedStockMarketTimeSeries('AMZN').head(10).PNG)

**Test:** Query web service to get daily adjusted time series for **Google**.

**Result:**

```python
#getAlphaAdvantageDailyAdjustedStockMarketTimeSeries('GOOG', downloadFile=True)
getAlphaAdvantageDailyAdjustedStockMarketTimeSeries('GOOG').head(10)
```

![Screenshot](https://s3.amazonaws.com/public-k0l1081/getAlphaAdvantageDailyAdjustedStockMarketTimeSeries('GOOG').head(10).PNG)
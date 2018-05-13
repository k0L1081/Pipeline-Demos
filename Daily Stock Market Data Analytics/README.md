# Pipeline Demo: Daily Stock Market Data Analytics *(Buildout and documentation in progress)*

#### End-to-end pipeline demonstrating the implementation of the following steps:
1. **Collect data from API and store in data lake**: Python, AWS CLI, AWS S3, AWS EC2 *(In progress)*
    * Download list of <a href="https://www.nasdaq.com/screening/companies-by-industry.aspx" target="_blank">NASDAQ companies</a>.
    * Filter out companies for which to collect market data.
    * Query Daily Adjusted Stock Market Time Series <a href="https://www.alphavantage.co/documentation" target="blank">web service</a> for all specified companies.<br><br>
2. **Transform data in data lake** Python, PySpark, AWS EMR (Spark) *(Coming soon)*
3. **Import data into analytics columnar database**: AWS Redshift, SQL *(Coming soon)*
4. **Build pipeline orchestration & scheduling engine**: Python, Apache Airflow *(Coming soon)*
5. **Surface data to public using RESTful web API**: Python, Django *(Coming soon)*

### Get configurations.

Create function to get configuration value from configuration JSON file. By default, configuration file is located in the current working directory and named `configuration.json`.

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
![Screenshot](dfCompanies.head(10).PNG)
# IA 626 Final Project:
#### Austin Stegall

### Data Sources:

- Coronavirus NYC Data: https://github.com/nychealth/coronavirus-data

- NYC Traffic Data: https://data.cityofnewyork.us/Transportation/DOT-Traffic-Speeds-NBE/i4gi-tjb9

### Project Summary:
With the current state of the world, COVID-19 is a hot topic for all analysts.  I am looking to combine two data sources with the overall goal of uncovering potential insights related to COVID-19.
The first data source (link shown above) is the most up-to-date coronavirus data for NYC.  There are various files hosted on the github page like one that lists confirmed cases per zip code (in NYC).  This data set is updated daily to keep track of the coronavirus in NYC.  The second link contains real-time data from NYCDOT speed sensors on various highways in the five boroughs.  I want to combine this data to see if there is one, a decrease in traffic volume due to the travel restriction/social distancing imposed on NYC, and two, see if there is some type of relationship between number of positive cases and travel volume.  The coronavirus data is in the form of a .csv file in tabular format.  The traffic data is on data.cityofnewyork.us and can be retrieved from an API.  This data will need a bit of preprocessing before it can be combined with the coronavirus data.

After doing some preliminary exploration on the data, I do not think each row represents a car.  Unfortunately, the NYC open data website does not give specific detail as to what the representation of one row of data is (at the time of writing this).  The data from March of 2019 contains 1,040,659 rows while the data from March of 2020 contains 1,027,351.  A decrease of about 13000 rows which seems insignificant when compared to the total number of rows.  It also looks like there are about 1,000,000 rows of data taken each month.  So there appears to be no direct way to measure traffic volume from this data.  I am going to continue to assess if I can combine this data with the coronavirus data to gain any meaningful insight.  

Since the NYC open data has been recording data since 2017, it may be possible to compare the data from previous years to the data now to gauge if social distancing and travel restrictions have had any effect on the metrics derived from the travel data.  

Two important metrics that this data tracks is average speed and average travel time.  A potential hypothesis is that during times of increased traffic volume, the average travel time should increase while average speed should decrease.  This is due to increased traffic congestion causing slower speeds and in turn, slower travel times.  This data set will be explored to see if there are any significant differences in average travel time and speed.     

It should be noted that the goal of this project is to demonstrate an understanding of data processing skills such as ETL and analysis/filtering.  So significant portions of this document will be dedicated to explaining the data processing steps.

---

### Outline:
- Extract data from repository 
- Perform necessary filtering/transformation on data
- Create supporting material for hypothesis
- Document data processing steps

#### Hypothesis: Has quarantine resulting from the coronavirus affected traffic volume in NYC?

---

### Load Libraries


```python
# import libraries
import pandas as pd
from sodapy import Socrata
import time
import csv
from plotnine import *
from dfply import *
```

The above libraries were imported:
- pandas   --> preferred method for data structures
- sodapy   --> API used by NYC open data
- time     --> used to time scripts
- csv      --> used to write file to .csv
- plotnine --> used for plotting
- dfply    --> data wrangling package

---

### Create function to access NYC traffic data API
A function will be created to allow easy and repeatable access to the NYC traffic API.  This function will allow the user to specify a start date and end date and return the data from the API.  There is also a feature to write the results to a .csv file.  


```python
# specify username and password for API
username = '#####################'
password = '#######'
MyAppToken = '###################'
```

#### GetNYCdata Function:
The following function will accept the following 6 inputs: username, password MyApptoken, start_date, end_date, and create_csv.   The first three are associated with an account needed for the API access.  You just create a username and password and the app token is generated for you.  The last three variables are your start and end date, in the format of YYYY-MM-DD, and a boolean variable to determine if you want to write your results to a .csv file or not.  The file name for the .csv file is automatically named based on your date range.  The function will output your results to a pandas data frame.

The function works by taking the user inputs for the start date and building a SoQL query which is very similar to a SQL query.  This SoQL query pulls the data from the NYC opendata API.


```python
# create function to get NYC Data for a specific month and year
def GetNYCdata(start_date, end_date, create_csv=False, username=username, password=password, MyAppToken=MyAppToken):
    '''
    Function to retrieve real time traffic data for a specified date range.  
    Option to write to csv file
    
    Inputs: username, password, MyAppToken -> credentials needed for API access
            start and end date -> date range for data retrieval
                                  format: YYYY-MM-DD
    
    Outputs: pandas dataframe
    '''
    # Create Client
    client = Socrata('data.cityofnewyork.us',
                 MyAppToken,
                 username=username,
                 password=password)

    # increase client timeout, default is too short
    client.timeout = 120
       
    # dictionaries by sodapy.
    results = client.get("i4gi-tjb9",
                         where= 'DATA_AS_OF BETWEEN '+"'"+start_date+'T00:00:00'+"'"+' AND '+"'"+end_date+'T23:59:59'+"'",
                         limit= 2000000)
    
    # convert results to dataframe
    df = pd.DataFrame.from_records(results)
    
    # create .csv file if true
    if create_csv == True:
        df.to_csv('traffic_data_from_'+start_date+'_to_'+end_date+'.csv')
    
    return(df)
```

### Use GetNYCdata Function to Retrieve Data:
The GetNYCdata function will now be used to get traffic data for the month of March from years 2018, 2019, and 2020.  This is accomplished by using a for loop and looping over a list of years.  Each iteration of the loop will create the start and end date for that year to be passed into the GetNYCdata function.  The results will all be concatenated to a single dataframe. A dataframe was chosen as the preferred type of format as it is very compatable with the plotting package plotnine.  This will also allow pandas and dplyr filtering methods to be used. Note that the if else statement inside the for loop is needed to create the dataframe that will be appended to.


```python
# write loop to get data for march from 2018-2020

# specify year list
year = ['2018', '2019', '2020']
n=1

# loop through years and concatenate to single dataframe
for value in year: 
    data = (GetNYCdata(start_date=value+'-03-01', end_date=value+'-03-31'))
    if n == 1:
        data_full = data
    else:
        data_full = pd.concat([data_full, data], axis=0)
    n+=1
```

### The Data:
Now that we have our data, we can begin to assess its basic attributes such as shape, and determine how appropriate the current data types and formats are for the analysis.


```python
# view first 5 rows of data:
data_full.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>speed</th>
      <th>travel_time</th>
      <th>status</th>
      <th>data_as_of</th>
      <th>link_id</th>
      <th>link_points</th>
      <th>encoded_poly_line</th>
      <th>encoded_poly_line_lvls</th>
      <th>owner</th>
      <th>transcom_id</th>
      <th>borough</th>
      <th>link_name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>325</td>
      <td>39.76</td>
      <td>138</td>
      <td>0</td>
      <td>2018-03-01T00:02:03.000</td>
      <td>4329472</td>
      <td>40.75829,-73.997531 40.7605,-74.0032 40.762060...</td>
      <td>irwwFpssbMyLlb@wHzVmYv}@</td>
      <td>BBBB</td>
      <td>PA -Lincoln Tunnel</td>
      <td>4329472</td>
      <td>Manhattan</td>
      <td>LINCOLN TUNNEL E SOUTH TUBE - NJ - NY</td>
    </tr>
    <tr>
      <th>1</th>
      <td>440</td>
      <td>43.49</td>
      <td>176</td>
      <td>0</td>
      <td>2018-03-01T00:02:03.000</td>
      <td>4329483</td>
      <td>40.5264504,-74.27001 40.52568,-74.267851 40.52...</td>
      <td>iijvFpzhdMxCoLnAoKfAuQz@yP~@_QCiNMmHc@gLuAmk@[...</td>
      <td>BBBBBBBBBBBBBBB</td>
      <td>NYC_DOT_LIC</td>
      <td>4329483</td>
      <td>Staten Island</td>
      <td>WSE S TYRELLAN AVENUE - 440 S FRANCIS STREET</td>
    </tr>
    <tr>
      <th>2</th>
      <td>330</td>
      <td>39.14</td>
      <td>128</td>
      <td>0</td>
      <td>2018-03-01T00:02:03.000</td>
      <td>4329507</td>
      <td>40.75719,-73.99724 40.76017,-74.00382 40.76185...</td>
      <td>mkwwFvqsbMsQbh@oIlTmYp}@</td>
      <td>BBBB</td>
      <td>PA -Lincoln Tunnel</td>
      <td>4329507</td>
      <td>Manhattan</td>
      <td>LINCOLN TUNNEL W NORTH TUBE NY - NJ</td>
    </tr>
    <tr>
      <th>3</th>
      <td>318</td>
      <td>52.19</td>
      <td>75</td>
      <td>0</td>
      <td>2018-03-01T00:02:03.000</td>
      <td>4362249</td>
      <td>40.7442206,-73.771661 40.7454306,-73.76907 40....</td>
      <td>kztwFzogaMqFeOu@wC_DeQcFwXyAoF{CkIaAmDu@aEiDeT...</td>
      <td>BBBBBBBBBBBBBBBBBBBBBBBBBBBBBB</td>
      <td>NYC-DOT-Region 10</td>
      <td>4362249</td>
      <td>Queens</td>
      <td>LIE WB LITTLE NECK PKWY - NB CIP</td>
    </tr>
    <tr>
      <th>4</th>
      <td>211</td>
      <td>53.43</td>
      <td>307</td>
      <td>-101</td>
      <td>2018-03-01T00:02:03.000</td>
      <td>4362250</td>
      <td>40.78795,-73.790191 40.78647,-73.78812 40.7862...</td>
      <td>uk}wFtckaMfH}Kr@cAbAoAzAyAvCeCt@e@|EwBxBg@jR_C...</td>
      <td>BBBBBBBBBBBBBBBBBBBBBBBBBB</td>
      <td>NYC-DOT-Region 10</td>
      <td>4362250</td>
      <td>Queens</td>
      <td>CVE NB GCP - WILLETS PT BLVD</td>
    </tr>
  </tbody>
</table>
</div>




```python
# get number of rows and columns of data
data_full.shape
```




    (3261930, 13)



The data has 13 columns (variables) and 3,261,930 rows.  Thats over a million rows of data per month!  Lets examine the variables and their data types.


```python
# get variables names and data types
data_full.dtypes
```




    id                        object
    speed                     object
    travel_time               object
    status                    object
    data_as_of                object
    link_id                   object
    link_points               object
    encoded_poly_line         object
    encoded_poly_line_lvls    object
    owner                     object
    transcom_id               object
    borough                   object
    link_name                 object
    dtype: object



#### Parameter Table with Data Type and Descriptions:

Parameters|Data Type|Description
:---|:---|:---
id|object|TRANSCOM Link ID
speed|object|Average speed a vehicle traveled between end points on the link in the most recent interval
travel_time|object|Time the average vehicle took to traverse the link
status|object|Artifact – not usefu
data_as_of|object|Last time data was received from link 
link_id|object|TRANSCOM Link ID (same as ID field)
link_points|object|Sequence of Lat/ Long points, describes locations of the sensor links
encoded_poly_line|object|Google compatible polyline from http://code.google.com/apis/maps/documentation/polylineutility.html
encoded_poly_line_lvls|object|Google compatible poly levels from http://code.google.com/apis/maps/documentation/polylineutility.html
owner|object|Owner of the Detector
transcom_id|object|Artifact – not useful
borough|object|NYC borough (Brooklyn, Bronx, Manhattan, Queens, Staten Island)
link_name|object|Description of the link location and end points


To make more use out of the data, certain variables will be converted to the proper type ex: data_as_of will be converted to the proper datetime type.  Speed will be converted to a float and travel_time will be converted to an int.


```python
# convert speed to int
data_full['speed'] = data_full['speed'].astype(float)
# convert travel time to int
data_full['travel_time'] = data_full['travel_time'].astype(int)
# convert date time to proper format
data_full['data_as_of'] = pd.to_datetime(data_full['data_as_of'], format='%Y-%m-%d %H:%M:%S')
```

The following values had their data types updated:

Parameter|New Data Type
:---|:---
speed|float
travel_time|int
data_as_of|datetime

With the proper data types set for the variables we are currently concerned with, we can now explore the data.  We will begin with checking the summary statistics for each variable to determine if there are any outliers or strange values.  This is easily accomplished by using the .describe() function from pandas.  This displays the summary statistics.  The .apply(lambda...) formats the output to 2 decimal places.


```python
# using summary statistics function with lambda function to get proper formatting
data_full['speed'].describe().apply(lambda x: '%.2f' % x)
```




    count    3261930.00
    mean          37.05
    std           19.46
    min            0.00
    25%           22.36
    50%           43.49
    75%           52.81
    max          118.68
    Name: speed, dtype: object



A max speed of 118.68 mph is pretty fast!  Due to how average speeds are measured, speeds of 0 mph will be filtered out because this equates to a travel time of infinity and is not possible.


```python
data_full['travel_time'].describe().apply(lambda x: '%.2f' % x)
```




    count    3261930.00
    mean         189.40
    std          296.81
    min          -56.00
    25%           67.00
    50%          127.00
    75%          212.00
    max        13409.00
    Name: travel_time, dtype: object



It looks like some of the travel times are negative.  Unless people are effectivly time traveling on the highways, these values are most likely sometype of error.  It looks like the max travel time was 13409 minutes or around 9 days. I certainly hope nobody was stuck on a NYC road for 9 days.  Lets examine the top end of travel times a little more before deciding on a filter.


```python
# create time travel outliers dataframe with only values greater than 5 hours
travel_time_outliers = data_full >> mask(X.travel_time > 300)
travel_time_outliers.shape
```




    (487557, 13)



It looks like there are 487557 rows where travel time is greater than 5 hours. Lets look at the distribution.  First, lets examine the number of rows per borough.


```python
plot1 = (ggplot(travel_time_outliers, aes(x='borough'))+
         geom_bar(alpha=0.8)+
         geom_text(aes(label='stat(count)'),
                        position=position_stack(vjust=0.5),
                        stat='count',
                        size=10, 
                        va='top',
                        format_string='{}')+
         labs(x='Borough', y='Row Count', title='Row Count of Travel Times > 5 Hours')+
         theme_minimal()
)
plot1
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_26_0.PNG)





    <ggplot: (-9223371858074908600)>



Manhattan and Queens have the most rows, suggesting that they have the highest frequency of travel times greater than 5 hours.  Now lets examine the distribution of travel time by borough.  Note: There appear to be two Staten Island labels, one with a lower case "i" and one with an upper case "I".  The "Staten island" term will be filtered our later due to it having a insignificant row count.


```python
plot2 = (ggplot(travel_time_outliers)+
         geom_histogram(aes(x='travel_time', y="..count.."), color='black', bins=20)+
         scale_x_continuous(breaks=range(0,15000,5000))+
         #facet_wrap('~borough')+
         theme_minimal()
)
plot2
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_28_0.PNG)





    <ggplot: (-9223371858061220304)>



It looks like travel time really drops off afer ~3500.  Lets apply one more filter to eliminate travel times greater than 3500 min.


```python
travel_time_outliers2 = travel_time_outliers >> mask(X.travel_time < 3500)
travel_time_outliers2.shape
```




    (483100, 13)




```python
plot3 = (ggplot(travel_time_outliers2)+
         geom_histogram(aes(x='travel_time', y="..count.."), color='black', bins=20)+
         scale_x_continuous(breaks=range(0,3500,1000))+
         labs(x='Travel Time [min]', y='Count', title='Histogram of Travel Time > ')+
         #facet_wrap('~borough')+
         theme_minimal()
)
plot3
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_31_0.PNG)





    <ggplot: (-9223371858061241672)>



After examining the data and the distributions above, the data will be filtered down further to focus on shorter travel times.  A filter of 240 minutes or 4 hours will be chosen.  Data will be filtered by the following parameters, travel_time greater than 0 minutes but less than 240, speeds greater than 0, and boroughs not named 'Staten island'.


```python
# create time travel outliers dataframe with only values greater than 4 hours,
# no travel times and speeds of 0 or below.
data_240 = data_full >> mask(X.travel_time < 240,
                             X.travel_time > 0,
                             X.speed > 0,
                             X.borough != 'Staten island')
data_240.shape
```




    (2193932, 13)



After applying the filters, the data set has been reduced to the following: 

DataFrame|Row Count Before|Row Count After|% Reduction
:---|:---|:---|:---
data_full|3261930|2193932|~32%

Lets view the filtered data set with percentages per borough.


```python
plot4 = (ggplot(data_240, aes(x='borough'))+
         geom_bar(alpha=0.8)+
         geom_text(aes(label='stat(prop)*100', group=1),
                        position=position_stack(vjust=0.5),
                        stat='count',
                        size=10, 
                        va='top',
                        format_string='{:.1f}%')+
         labs(x='Borough', y='Row Count', title='Row Count of Travel Times < 4 Hours')+
         theme_classic()
)
plot4
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_35_0.png)





    <ggplot: (-9223371950716507412)>



Now that the proper filters have been set, the data can begin to be analyzed to determine if there is a difference in traffic characteristics for the month of March between the years 2018, 2019, and 2020.  Some would expect a decline in traffic volume for March 2020 due to social distancing caused by the coronavirus.  This is because there are many business shutdown, people furloughed, and state orders to stay inside, etc..  Let’s see if this is reflected in the traffic data.

### Distribution of Data:

In order to easily visualize the data and clearly compare data across year, we will add a column to the data frame with just the year.  This can be used as a categorical label when plotting.  To do this, the year will be extracted from the datetime column and converted to a string type.  The conversion to a string type allows the variable to be categorical.



```python
# create new column with year
data_240['year'] = data_240['data_as_of'].dt.year.astype('str')
data_240.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>speed</th>
      <th>travel_time</th>
      <th>status</th>
      <th>data_as_of</th>
      <th>link_id</th>
      <th>link_points</th>
      <th>encoded_poly_line</th>
      <th>encoded_poly_line_lvls</th>
      <th>owner</th>
      <th>transcom_id</th>
      <th>borough</th>
      <th>link_name</th>
      <th>year</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>325</td>
      <td>39.76</td>
      <td>138</td>
      <td>0</td>
      <td>2018-03-01 00:02:03</td>
      <td>4329472</td>
      <td>40.75829,-73.997531 40.7605,-74.0032 40.762060...</td>
      <td>irwwFpssbMyLlb@wHzVmYv}@</td>
      <td>BBBB</td>
      <td>PA -Lincoln Tunnel</td>
      <td>4329472</td>
      <td>Manhattan</td>
      <td>LINCOLN TUNNEL E SOUTH TUBE - NJ - NY</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>1</th>
      <td>440</td>
      <td>43.49</td>
      <td>176</td>
      <td>0</td>
      <td>2018-03-01 00:02:03</td>
      <td>4329483</td>
      <td>40.5264504,-74.27001 40.52568,-74.267851 40.52...</td>
      <td>iijvFpzhdMxCoLnAoKfAuQz@yP~@_QCiNMmHc@gLuAmk@[...</td>
      <td>BBBBBBBBBBBBBBB</td>
      <td>NYC_DOT_LIC</td>
      <td>4329483</td>
      <td>Staten Island</td>
      <td>WSE S TYRELLAN AVENUE - 440 S FRANCIS STREET</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>2</th>
      <td>330</td>
      <td>39.14</td>
      <td>128</td>
      <td>0</td>
      <td>2018-03-01 00:02:03</td>
      <td>4329507</td>
      <td>40.75719,-73.99724 40.76017,-74.00382 40.76185...</td>
      <td>mkwwFvqsbMsQbh@oIlTmYp}@</td>
      <td>BBBB</td>
      <td>PA -Lincoln Tunnel</td>
      <td>4329507</td>
      <td>Manhattan</td>
      <td>LINCOLN TUNNEL W NORTH TUBE NY - NJ</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>3</th>
      <td>318</td>
      <td>52.19</td>
      <td>75</td>
      <td>0</td>
      <td>2018-03-01 00:02:03</td>
      <td>4362249</td>
      <td>40.7442206,-73.771661 40.7454306,-73.76907 40....</td>
      <td>kztwFzogaMqFeOu@wC_DeQcFwXyAoF{CkIaAmDu@aEiDeT...</td>
      <td>BBBBBBBBBBBBBBBBBBBBBBBBBBBBBB</td>
      <td>NYC-DOT-Region 10</td>
      <td>4362249</td>
      <td>Queens</td>
      <td>LIE WB LITTLE NECK PKWY - NB CIP</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>6</th>
      <td>395</td>
      <td>44.73</td>
      <td>181</td>
      <td>0</td>
      <td>2018-03-01 00:02:03</td>
      <td>4456476</td>
      <td>40.7723,-73.91983 40.7738304,-73.92197 40.7744...</td>
      <td>{izwF|mdbMqHjLwB~BmEhDgQpLqOnKgOjK_Cv@eDNmCY_B...</td>
      <td>BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB</td>
      <td>MTA Bridges &amp; Tunnels</td>
      <td>4456476</td>
      <td>Queens</td>
      <td>TBB N QUEENS ANCHORAGE - MANHATTAN LIFT SPAN</td>
      <td>2018</td>
    </tr>
  </tbody>
</table>
</div>



With the conversion done, we can now view a color coded distribution of the data.  Ideally, we would like to see around 33.3% of data in each column.  That would indicate an even distribution of data.


```python
data_dist = (ggplot(data_240, aes(x='year', fill='year'))+
             geom_bar(stat='count', alpha=0.8)+
             geom_text(aes(label='stat(prop)*100', group=1),
                        position=position_stack(vjust=0.5),
                        size=13, stat= 'count',
                        va='top',
                        format_string='{:.1f}%')+
             labs(x='Year', y='File Count',  title='Number of Files Colored by Year')+
             theme_classic()
              
            )
data_dist
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_40_0.png)





    <ggplot: (-9223371950716565752)>



Data appears to be mostly evenly distributed amongst year.  With slight variations across the board.  To compare the distributions of speed and travel time by year, a density plot will be used instead of a histogram.  A density plot is more computationally expensive than a histogram and the current data set is too large to compute a density plot on my machine.  A sample dataframe will be created for the purpose of plotting.  This sample dataframe will be a subset of 150000 randomly selected data points.  This is done using the .sample() function from pandas.


```python
# create sample dataframe
df_sample_240 = data_240.sample(n=150000, random_state=2)
df_sample_240.shape
```

Lets recreate the distribution plot above to ensure our sample was kept random


```python
data_dist_sample = (ggplot(df_sample_240, aes(x='year', fill='year'))+
             geom_bar(stat='count', alpha=0.8)+
             geom_text(aes(label='stat(prop)*100', group=1),
                        position=position_stack(vjust=0.5),
                        size=13, stat= 'count',
                        va='top',
                        format_string='{:.1f}%')+
             labs(title='Number of Files Colored by Year (sample set)')+
             theme_classic()
              
            )
data_dist_sample
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_44_0.png)





    <ggplot: (-9223371950716658124)>



Looks like the new dataframe was sampled randomly with only slight variations in the new distribtuion.  Now lets examine the density plots.  We will create a dataframe that contains the averages for speed and travel_time grouped by year.  This will allow us to add labels to the density plot.  The density plot will feature all three years and verticle lines representing the averages for those years. 


```python
# create mean data frame to be used for plotting labels
mean = df_sample_240.groupby('year', as_index=False).mean()
mean
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>speed</th>
      <th>travel_time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2018</td>
      <td>46.151203</td>
      <td>117.402198</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2019</td>
      <td>45.402025</td>
      <td>119.937859</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2020</td>
      <td>49.385842</td>
      <td>115.076885</td>
    </tr>
  </tbody>
</table>
</div>




```python
plot_speed_density = (ggplot(df_sample_240)+
                     geom_density(aes(x='speed', color='year', fill='year'), alpha=0.1)+
                     geom_vline(mean, aes(xintercept='speed', color='year'))+
                     geom_text(mean, aes(x=90, y=[0.0225,0.025,0.0275], label='speed', color='year'),
                               format_string='Avg Speed: {:.1f} [mph]',
                               size=10,
                               show_legend=False
                              )+
                     scale_x_continuous(breaks=range(0,100,10))+
                     labs(x='Avg. Speed [mph]', y='', title='Density Plot of Speed ')+
                     theme_classic()
                     )
plot_speed_density
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_47_0.png)





    <ggplot: (-9223371950716641148)>



By examining the density plot above, along with the average values, we can see that the average speed for March 2020 is indeed higher than that of the previous years.  There is almost a 5 mph increase in speed from 2018 to 2020.  By examining the beginning of the density plot, from about 20 mph to 40 mph there is a noticeable change in speeds.  The trends for 2018 and 2019 appear to have a significantly higher amount of average speeds in 20-40 mph range than in 2020.  This suggests that there are fewer people traveling at slower speeds in 2020 than in previous years.  Perhaps this is due to less traffic jams that cause people to drive within that 20-40 mph range.  Now let's examine the density plot of average travel times.


```python
plot_trav_density = (ggplot(df_sample_240)+
                     geom_density(aes(x='travel_time', color='year', fill='year'), alpha=0.1)+
                     geom_vline(mean, aes(xintercept='travel_time', color='year'))+
                     geom_text(mean, aes(x=192, y=[0.007,0.0075,0.008], label='travel_time', color='year'),
                               format_string='Avg Time: {:.1f} [min]',
                               size=10,
                               show_legend=False
                              )+
                     scale_x_continuous(breaks=range(0,240,24))+
                     labs(x='Travel Time [min]', y='', title='Density Plot of Travel Time ')+
                     theme_classic()
                     )
plot_trav_density
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_49_0.png)





    <ggplot: (-9223371950716825368)>



By observing the figure above, we can see the distributions for travel time colored per year.  The vertical lines represent the average values for each of those years.  Travel times for year 2020 are about 2-4 minutes shorter when compared to year 2018 and 2019.  This makes sense as the average speed for year 2020 was higher than the previous two years.  There appears to be a tri-modal trend in travel times.  Times of about 48, 84, and 130 minutes were significantly more frequent than other times.  These could be popular work commute travel times. 

While not critical to this project, two additional plots will be generated to get a better understanding about how speed and travel time are different per year by borough.   They will be similar to the density plots shown above, but faceted by borough.  Due to the size of the plots, the vertical lines representing averages will not be plotted.  However, the code to create the data frame and modify the plot is included below.


```python
# optional statistics needed if vertical lines are to be added to the plots.

# create df with mean by borough
mean_borough = df_sample_240.groupby(['year', 'borough'], as_index=False).mean()
mean_borough
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>borough</th>
      <th>speed</th>
      <th>travel_time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2018</td>
      <td>Bronx</td>
      <td>46.462771</td>
      <td>111.368042</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2018</td>
      <td>Brooklyn</td>
      <td>46.230304</td>
      <td>88.765877</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2018</td>
      <td>Manhattan</td>
      <td>32.908197</td>
      <td>145.289106</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2018</td>
      <td>Queens</td>
      <td>46.704024</td>
      <td>141.488626</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2018</td>
      <td>Staten Island</td>
      <td>52.124913</td>
      <td>95.276151</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2019</td>
      <td>Bronx</td>
      <td>44.435997</td>
      <td>118.560745</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2019</td>
      <td>Brooklyn</td>
      <td>43.667571</td>
      <td>92.848034</td>
    </tr>
    <tr>
      <th>7</th>
      <td>2019</td>
      <td>Manhattan</td>
      <td>32.891938</td>
      <td>150.828159</td>
    </tr>
    <tr>
      <th>8</th>
      <td>2019</td>
      <td>Queens</td>
      <td>46.344640</td>
      <td>146.389927</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2019</td>
      <td>Staten Island</td>
      <td>52.260150</td>
      <td>90.481804</td>
    </tr>
    <tr>
      <th>10</th>
      <td>2020</td>
      <td>Bronx</td>
      <td>50.257265</td>
      <td>103.219001</td>
    </tr>
    <tr>
      <th>11</th>
      <td>2020</td>
      <td>Brooklyn</td>
      <td>48.223879</td>
      <td>94.167313</td>
    </tr>
    <tr>
      <th>12</th>
      <td>2020</td>
      <td>Manhattan</td>
      <td>37.527892</td>
      <td>137.698636</td>
    </tr>
    <tr>
      <th>13</th>
      <td>2020</td>
      <td>Queens</td>
      <td>50.361829</td>
      <td>139.312860</td>
    </tr>
    <tr>
      <th>14</th>
      <td>2020</td>
      <td>Staten Island</td>
      <td>54.805560</td>
      <td>91.785199</td>
    </tr>
  </tbody>
</table>
</div>




```python
# facet by borough
plot_speed_density_facet = (ggplot(df_sample_240)+
                             geom_density(aes(x='speed', color='year', fill='year'), alpha=0.1)+
                             #geom_vline(mean_borough, aes(xintercept='speed', color='year'))+
                             facet_wrap('~borough')+
                             scale_x_continuous(breaks=range(0,100,25))+
                             labs(x='Avg. Speed [mph]', y='', title='Density Plot of Speed ')+
                             theme_minimal()
                             )
plot_speed_density_facet
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_52_0.PNG)





    <ggplot: (-9223371950712427440)>



Note, rightward shifts in peaks represent increases in average speed.


```python
# facet by borough
plot_trav_density_facet = (ggplot(df_sample_240)+
                             geom_density(aes(x='travel_time', color='year', fill='year'), alpha=0.1)+
                             #geom_vline(mean_borough, aes(xintercept='travel_time', color='year'))+  
                             facet_wrap('~borough')+
                             scale_x_continuous(breaks=range(0,240,60))+
                             labs(x='Travel Time [min]', y='', title='Density Plot of Travel Times ')+
                             theme_minimal()
                             )
plot_trav_density_facet
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_54_0.PNG)





    <ggplot: (-9223371950709236788)>



Note, leftward shifts in peaks represent decreases in average travel times.

### How does average speed and travel time change throughout the day during Monday-Friday?

To answer this question, some significant data filtering will be required.  First, to filter out weekends, Saturdays and Sundays must be identified.  They also are different for each year, as in, a Saturday that occurs on the 4th of March in 2018 may not be a Saturday in March 2019.  The best way I could come up with for the time being was to create a list for each year that identifies which days Saturday and Sunday occur on.  Next I created a copy of the sampled data frame used to create the density plots as the original dataframe was far too large for my computer to process in a reasonable time.  This copy, weekday_data needed its index reset in order to proper work with the following for loop.  A for loop was now created to loop through each row of the dataframe.  If statements are used to check what year the rows timestamp is, and then determine if that day is a weekend day by checking it against the list of days that weekends occur.  If the day was determined to be a weekend day for that given year, the row was dropped from the dataframe based on the current index.  The indexes needed to be reset as they were all random from the subsample and needed to be in order for the for loop to properly process.   This operation took about 2.5 hours on my computer.


```python
# filter out weekend days

# create list of days saturday and sunday occur by year
weekend_2018 = [3,4,10,11,17,18,24,25,31]
weekend_2019 = [2,3,9,10,16,17,23,24,30,31]
weekend_2020 = [1,7,8,14,15,21,22,28,29]
n=0

# create weekday dataframe: (reset index to allow dynamic dropping of rows)
weekday_data = df_sample_240.reset_index()

# filter through data frame excluding weekends
for row in df_sample_240.itertuples():
    
    # filter 2018 dates
    if row[5].year == 2018 and row[5].day in weekend_2018:
        weekday_data.drop(index=n, axis=0, inplace=True)
    # filter 2019 dates
    elif row[5].year == 2019 and row[5].day in weekend_2019:
        weekday_data.drop(index=n, axis=0, inplace=True)
    # filter 2020 dates
    elif row[5].year == 2020 and row[5].day in weekend_2020:
        weekday_data.drop(index=n, axis=0, inplace=True)
           
    # print row every 10000 lines to check progress
    if n % 10000 == 0:
        print(row[5])
    
    n+=1
    
#     if n == 10000:
#         break
display(weekday_data)
```


```python
weekday_data.shape
```




    (105332, 15)



After applying the weekend filter, the data set has been reduced to the following: 

DataFrame|Row Count Before|Row Count After|% Reduction
:---|:---|:---|:---
weekday_data|150000|105332|~29%

We will now create an hour of the day column just like we did earlier with year.  This will be used to create the averages for speed and travel time by hour.


```python
# create hour of the day column:
weekday_data['hour'] = weekday_data['data_as_of'].dt.hour.astype('float')
weekday_data.head()
```




<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>index</th>
      <th>id</th>
      <th>speed</th>
      <th>travel_time</th>
      <th>status</th>
      <th>data_as_of</th>
      <th>link_id</th>
      <th>link_points</th>
      <th>encoded_poly_line</th>
      <th>encoded_poly_line_lvls</th>
      <th>owner</th>
      <th>transcom_id</th>
      <th>borough</th>
      <th>link_name</th>
      <th>year</th>
      <th>hour</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>258539</td>
      <td>378</td>
      <td>55.30</td>
      <td>48</td>
      <td>0</td>
      <td>2020-03-11 20:14:14</td>
      <td>4616197</td>
      <td>40.6210105,-74.168861 40.6207604,-74.168 40.61...</td>
      <td>ix\\\\\\\\\\\\\\\\|vFjbucMp@kD\\\\\\\\\\\\\\\\...</td>
      <td>BBBBB</td>
      <td>NYC_DOT_LIC</td>
      <td>4616197</td>
      <td>Staten Island</td>
      <td>SIE E SOUTH AVENUE - RICHMOND AVENUE</td>
      <td>2020</td>
      <td>20.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>641004</td>
      <td>213</td>
      <td>32.93</td>
      <td>72</td>
      <td>0</td>
      <td>2019-03-21 10:33:05</td>
      <td>4456450</td>
      <td>40.80069,-73.92878 40.8013005,-73.930181 40.80...</td>
      <td>i{_xFzefbMyBvGUlACt@Rj@d@f@z@@`@W\g@bA_DTk@b@i...</td>
      <td>BBBBBBBBBBBBBBBBBBBBBBB</td>
      <td>MTA Bridges &amp; Tunnels</td>
      <td>4456450</td>
      <td>Manhattan</td>
      <td>FDR N - TBB E 116TH STREET - MANHATTAN TRUSS</td>
      <td>2019</td>
      <td>10.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>757924</td>
      <td>191</td>
      <td>49.70</td>
      <td>86</td>
      <td>0</td>
      <td>2020-03-24 20:03:03</td>
      <td>4620331</td>
      <td>40.8465405,-73.932021 40.84611,-73.93075 40.84...</td>
      <td>{yhxFbzfbMtA}FfDyT\\\\\\|@iGZ_F?_Fe@oSk@i]AkUFgF</td>
      <td>BBBBBBBBBB</td>
      <td>PA-GWBridge</td>
      <td>4620331</td>
      <td>Bronx</td>
      <td>CBE W MORRIS AVE - GWB W AMSTERDAM AVE (U/LVL)</td>
      <td>2020</td>
      <td>20.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>850725</td>
      <td>364</td>
      <td>31.06</td>
      <td>172</td>
      <td>0</td>
      <td>2019-03-26 20:03:03</td>
      <td>4456511</td>
      <td>40.745726,-73.97359 40.745616,-73.97305 40.745...</td>
      <td>wcuwF|}nbMTkB?qAwAuRd@qEbBqKnMk]vB{Ir@_EtCwWLaF</td>
      <td>BBBBBBBBBBB</td>
      <td>NYC_DOT_LIC</td>
      <td>4456511</td>
      <td>Manhattan</td>
      <td>QMT E Manhattan Side - Toll Plaza</td>
      <td>2019</td>
      <td>20.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>511997</td>
      <td>145</td>
      <td>27.96</td>
      <td>93</td>
      <td>0</td>
      <td>2018-03-14 06:08:03</td>
      <td>4616342</td>
      <td>40.7081105,-73.99944 40.7084705,-73.99884 40.7...</td>
      <td>uxmwFn_tbMgAwBi@eBc@sB[qC[wE</td>
      <td>BBBBBB</td>
      <td>NYC_DOT_LIC</td>
      <td>4616342</td>
      <td>Manhattan</td>
      <td>BKN Bridge Manhattan Side - FDR N Catherine Slip</td>
      <td>2018</td>
      <td>6.0</td>
    </tr>
  </tbody>
</table>
</div>



With the hour of day column now created, we can create a new data frame with speed and travel time averaged by hour of day.  To do this, we need to group the data by year then hour, and then take the average.   


```python
# create data frame with grouped by year, hour, then average speed:
avg_by_hour = weekday_data.groupby(['year', 'hour'], as_index=False).mean()

# drop index column that was an extra
avg_by_hour.drop('index', axis=1, inplace=True)

avg_by_hour
```

 
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>year</th>
      <th>hour</th>
      <th>speed</th>
      <th>travel_time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2018</td>
      <td>0.0</td>
      <td>48.804764</td>
      <td>112.574455</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2018</td>
      <td>1.0</td>
      <td>49.584779</td>
      <td>111.913479</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2018</td>
      <td>2.0</td>
      <td>50.102431</td>
      <td>111.901227</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2018</td>
      <td>3.0</td>
      <td>51.063382</td>
      <td>110.539273</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2018</td>
      <td>4.0</td>
      <td>51.436338</td>
      <td>108.222476</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>67</th>
      <td>2020</td>
      <td>19.0</td>
      <td>48.310892</td>
      <td>116.681206</td>
    </tr>
    <tr>
      <th>68</th>
      <td>2020</td>
      <td>20.0</td>
      <td>48.798710</td>
      <td>115.363532</td>
    </tr>
    <tr>
      <th>69</th>
      <td>2020</td>
      <td>21.0</td>
      <td>48.903917</td>
      <td>114.628115</td>
    </tr>
    <tr>
      <th>70</th>
      <td>2020</td>
      <td>22.0</td>
      <td>49.728097</td>
      <td>114.423299</td>
    </tr>
    <tr>
      <th>71</th>
      <td>2020</td>
      <td>23.0</td>
      <td>50.617259</td>
      <td>113.560976</td>
    </tr>
  </tbody>
</table>
<p>72 rows × 4 columns</p>
</div>



Now with our dataframe created, we can finally plot average speed vs hour of the day colored by year.


```python
# create plot
hour_plot = (ggplot(avg_by_hour)+
                     geom_line(aes(x='hour', y='speed', color='year', fill='year'))+
                     #geom_rect(aes(xmax=7, xmin=5, ymin=40, ymax=52.5), color='purple', alpha=0, size=.005)+
                     scale_x_continuous(breaks=range(0,24,1))+
                     labs(x='Time of Day', y='Avg. Speed [mph]', title='Average Speed vs Time of Day [March]')+
                     theme_classic()
                     )
hour_plot
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_64_0.png)





    <ggplot: (-9223371950715422128)>



This plot agrees with the previous speed density plot.  Speed for March 2020 is on average, always greater than the previous two years.  We can observe that average speed decreases a lot during rush hours, but when compared across years, March 2020 appears to drop what looks like 50% less than March 2018 and 2019.  Rush hour drops are much less profound during March 2020 than the previous two years.  This is a good indicator that there is less traffic on the road, and in turn, more people social distancing.  Now let’s examine average travel time vs time of day.


```python
# create plot
travel_plot = (ggplot(avg_by_hour)+
                     geom_line(aes(x='hour', y='travel_time', color='year', fill='year'))+
                     scale_x_continuous(breaks=range(0,24,1))+
                     labs(x='Time of Day', y='Avg. Travel Time', title='Average Travel Time vs Time of Day [March]')+
                     theme_classic()
                     )
travel_plot
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_66_0.png)





    <ggplot: (-9223371950712423604)>



Similar to what we would expect, average travel times appear to be the lowest for March 2020.  To describe the trend throughout the day, March 2020 mimics the trend of previous years except the amplitude of the peaks are almost always smaller; indicating shorter travel times in general.  These shorter travel times supports the claim that traffic volume has decreased.  Extra plots have been included below which facet the previous two plots by borough.  This helps uncover trends by borough.


```python
# create data frame with grouped by year, hour, then average speed:
avg_by_hour2 = weekday_data.groupby(['year', 'borough', 'hour'], as_index=False).mean()

# drop index column that was an extra
avg_by_hour2.drop('index', axis=1, inplace=True)

```


```python
# create denisty plot
hour_plot2 = (ggplot(avg_by_hour2)+
                     geom_line(aes(x='hour', y='speed', color='year', fill='year'))+
                     #geom_rect(aes(xmax=7, xmin=5, ymin=40, ymax=52.5), color='purple', alpha=0, size=.005)+
                     facet_wrap('~borough')+
                     scale_x_continuous(breaks=range(0,24,4))+
                     labs(x='Time of Day', y='Avg. Speed [mph]', title='Average Speed vs Time of Day [March]')+
                     theme_classic()
                     )
hour_plot2
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_69_0.png)





    <ggplot: (-9223371950711965236)>




```python
# create denisty plot
travel_plot2 = (ggplot(avg_by_hour2)+
                     geom_line(aes(x='hour', y='travel_time', color='year', fill='year'))+
                     #geom_rect(aes(xmax=7, xmin=5, ymin=40, ymax=52.5), color='purple', alpha=0, size=.005)+
                     facet_wrap('~borough')+
                     scale_x_continuous(breaks=range(0,24,4))+
                     labs(x='Time of Day', y='Avg. Travel Time [min]', title='Average Travel Time vs Time of Day [March]')+
                     theme_classic()
                     )
travel_plot2
```


![png](https://github.com/stegs2/IA-626-Final-Project/blob/master/output_70_0.png)





    <ggplot: (-9223371950704972080)>



---

## Conclusions:

In summary, the overall question trying to be answered was if social distancing and general “lockdown” was having any effect on traffic volume in NYC.  After determining that the real-time traffic data on NYC’s website, https://data.cityofnewyork.us/Transportation/DOT-Traffic-Speeds-NBE/i4gi-tjb9, does not directly measure traffic volume, the following assumptions were made to gauge traffic volume.  First, when traffic volume is higher, the average speed of vehicles will be slower due to congestion and traffic jams.  This will in turn cause an increase in average travel times.  When traffic volume is lower, average speeds will be higher and average travel times will be shorter.

Data was extracted from NYC opendata base using their API.  A function was written to extract the data given a specific date range and return a single pandas dataframe.  This date range was the full month of March for 2018, 2019, and 2020.  After obtaining the data, various operations were performed to get the data into the proper format.  Some of these operations included filtering, sampling, and changing data types to their proper data type.  Density plots for average speed and travel time were generated for each year.  The density plots described the distributions for each year highlighting changes in trends.  It was quite evident that March 2020 had lower travel times and higher average speeds, suggesting that traffic volume had been reduced.  To further confirm these findings, two more plots were created.  The first plot was average speed per hour of the day for Monday through Friday.  After this significant data filtering, the plot generated confirmed the previous findings.  Average speeds were higher throughout every hour of the day for March 2020 vs the previous two years.   Rush hour drops were also observed to be significantly less extreme.  The second plot generated was average travel time per hour of the day for Monday through Friday.  Travel times were almost always shorter for March 2020 when compared to the previous years too.  This also supports a decrease in traffic volume.  Judging from the data and analysis presented above, we can conclude that social distancing and quarantining may have resulted in a decrease in traffic volume for NYC.



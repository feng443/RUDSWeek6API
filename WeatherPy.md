## Analysis

1. On South Hemisphere, there are much fewer cities at higher laptitude
2. On South Hemisphere, temperate drop slower with laptitude than North Hemisphere. This might be because the it is winter at North but summer at South right now.
3. Humidity level by average looks higher on North vs. South Hemisphere. This might also due to seasons.
4. No apparent patterns obsered from scatter plots for couldiness and wind speed

```python
import os
import requests
import random
import pandas as pd
from datetime import date
import matplotlib.pyplot as plt
import seaborn as sns
from citipy import citipy
from geopy.geocoders import Nominatim
from config import WOM_API_KEY

class WeatherPy(object):
    
    _number_of_cities = None
    _cities_set = set()
    _cities_df = pd.DataFrame()
    _log_file = None
    _csv_file = None
    _png_file = None
    _actual_png_file = None
    _geolocator = Nominatim()
    _base_url = 'http://api.openweathermap.org/data/2.5/weather?units=Imperial&APPID=' + WOM_API_KEY + '&q='
    _today = date.today().strftime('%m/%d/%y')
    
    @property
    def cities(self):
        return self._cities_df.reset_index()[[
            'City Name', 'Country Code', 'Date',
            'Latitude', 'Longitude', 'Actual Latitude', 'Actual Longitude',
            'Humidity', 'Max Temperature', 'Cloudiness', 'Wind Speed'
        ]]

    @property
    def number_of_cities(self):
        return self._number_of_cities
      
    def __init__(self, 
                 number_of_cities=None,
                 log_file=None,
                 csv_file=None,
                 png_file=None,
                 actual_png_file=None):
        self._number_of_cities = number_of_cities or 4 # Default to 4 for testing
        self._png_file = png_file
        self._actual_png_file = actual_png_file
        self._csv_file = csv_file
        if log_file:
            self._log_file = open(log_file, 'w')
        sns.set()
        sns.set_style('darkgrid', {'axes.facecolor': '0.9'})
        
    def __del__(self):
        if self._log_file:
            self._log_file.close()

    def _pick_a_city(self, i):
        while True:
            lat, lon = (random.uniform(-90, 90), random.uniform(-180, 180))
            city = citipy.nearest_city(lat, lon)
            city_name = city.city_name
            country_code = city.country_code
            city_full = city_name + ',' + city.country_code

            self.log('Processing City # {} | {}'.format(i, city_full))

            # Check if city is already picked
            if city_full in self._cities_set:
                self.log('Skip duplicate {}'.format(city_full))
                continue
            
            # Get actual location of the city.
            location = self._geolocator.geocode(city_full)
            if location:                                    
                actual_lat = location.latitude
                actual_lon = location.longitude
                                            
            #g = geocoder.google(city_full)
            #if g:
            #    (actual_lat, actual_lon) = g.latlng
            else:
                self.log('Unable to get actual location of the city {}. Skip.'.format(city_full))
                continue
                
            # Check weather
            url = self._base_url + city_full
            self.log(url)
            response = requests.get(url)
            if response.status_code != 200:
                self.log('Weather not found for ' + city_full + '. Try another city.')
                continue

            w = response.json()   
            self._cities_df = self._cities_df.append(
                pd.DataFrame([{
                        'City Name': city_name,
                        'Country Code': country_code,
                        'Date': self._today,
                        'Latitude': lat,
                        'Longitude': lon,
                        'Actual Latitude': actual_lat,
                        'Actual Longitude': actual_lon,
                        'Humidity': w['main']['humidity'],
                        'Max Temperature': w['main']['temp_max'],
                        'Wind Speed': w['wind']['speed'],
                        'Cloudiness': w['clouds']['all'],
                    }]
                )
            )

            self._cities_set.add(city_full)
            break
        
    def pick_all_cities(self):
        for i in range(self.number_of_cities):
            self._pick_a_city(i)
        
    def write_data_to_csv(self, csv_file=None):
        csv_file = csv_file or self._csv_file
        self.cities.to_csv(csv_file)
        
    def plot_lat_lon(self, use_actual=False):
        fig, ax = plt.subplots(figsize=(10, 5))
        if use_actual:
            self.cities.plot.scatter('Actual Longitude', 'Actual Latitude',  ax=ax)
        else:
            self.cities.plot.scatter('Longitude', 'Latitude', ax=ax)
            
        plt.xlim(-180, 180); plt.ylim(-90, 90)
        plt.show()
        
    def plot_weather(self, use_actual=False):
        fig = plt.figure(figsize=(15,10))
        fig.suptitle("Correlation of City Latitude vs. Weather Metrics")

        x = 'Actual Latitude' if use_actual else 'Latitude'
        for i, metric in [(1, 'Max Temperature'),
                          (2, 'Humidity'),
                          (3, 'Cloudiness'),
                          (4, 'Wind Speed')]:
            self.cities.plot.scatter(x=x, y=metric, 
                                   title='City Latitude vs. {} ({})'.format(metric, self._today),
                                   ax=plt.subplot(2, 2, i),
                                   xlim=(-90, 90))                          

        if use_actual and self._actual_png_file:
            print('save actual png')
            fig.savefig(self._actual_png_file)
        if not use_actual and self._png_file:
            fig.savefig(self._png_file)
        plt.show()

    def log(self, msg):
        if self._log_file:
            self._log_file.write(msg + '\n')
            self._log_file.flush() # Easier to see progress in tail -f
        else:
            print(msg)
```


```python
weather_py = WeatherPy(
    number_of_cities=500,
    log_file='weather_py.log',
    csv_file='weather_py.csv',
    png_file='weather_py.png',
    actual_png_file='weather_actual.png',
)
weather_py.pick_all_cities()
weather_py.write_data_to_csv()
```


```python
weather_py.cities.head()
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>City Name</th>
      <th>Country Code</th>
      <th>Date</th>
      <th>Latitude</th>
      <th>Longitude</th>
      <th>Actual Latitude</th>
      <th>Actual Longitude</th>
      <th>Humidity</th>
      <th>Max Temperature</th>
      <th>Cloudiness</th>
      <th>Wind Speed</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>carutapera</td>
      <td>br</td>
      <td>03/11/18</td>
      <td>8.251069</td>
      <td>-40.611262</td>
      <td>-1.201664</td>
      <td>-46.020562</td>
      <td>86</td>
      <td>83.35</td>
      <td>32</td>
      <td>10.11</td>
    </tr>
    <tr>
      <th>1</th>
      <td>ushuaia</td>
      <td>ar</td>
      <td>03/11/18</td>
      <td>-88.345452</td>
      <td>-52.040169</td>
      <td>-54.806933</td>
      <td>-68.307325</td>
      <td>66</td>
      <td>51.80</td>
      <td>20</td>
      <td>4.59</td>
    </tr>
    <tr>
      <th>2</th>
      <td>beira</td>
      <td>mz</td>
      <td>03/11/18</td>
      <td>-21.125569</td>
      <td>35.654364</td>
      <td>-19.828707</td>
      <td>34.841782</td>
      <td>88</td>
      <td>78.80</td>
      <td>40</td>
      <td>6.76</td>
    </tr>
    <tr>
      <th>3</th>
      <td>high level</td>
      <td>ca</td>
      <td>03/11/18</td>
      <td>60.269763</td>
      <td>-117.642164</td>
      <td>58.516667</td>
      <td>-117.133333</td>
      <td>63</td>
      <td>28.40</td>
      <td>20</td>
      <td>5.82</td>
    </tr>
    <tr>
      <th>4</th>
      <td>ancud</td>
      <td>cl</td>
      <td>03/11/18</td>
      <td>-41.174507</td>
      <td>-82.892929</td>
      <td>-41.868195</td>
      <td>-73.828751</td>
      <td>98</td>
      <td>56.89</td>
      <td>92</td>
      <td>6.38</td>
    </tr>
  </tbody>
</table>
</div>



Using online tool: https://www.latlong.net/, I did some spot checks and found out that the 'actual' location is much better. The final chart will use the 'Actual Latitude' instead of Latitude.


```python
weather_py.plot_lat_lon()
```


![png](output_5_0.png)



```python
weather_py.plot_lat_lon(use_actual=True)
```


![png](output_6_0.png)


The 'Actual' coordinates match the map of the world so analysis will based on the scatter plot using the 'Actual' latitude as they are more precise.


```python
weather_py.plot_weather(use_actual=True)
```

    save actual png
    


![png](output_8_1.png)


### Destruct object to clean up


```python
del weather_py
```

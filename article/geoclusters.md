# Geocluster Development Script


```python
import requests
import pandas as pd
import geopandas as gpd
import io
from netCDF4 import Dataset, num2date
import xarray as xr
import rioxarray
import numpy as np
import matplotlib.pyplot as plt
import math

import numpy as np
import rasterio
from rasterio.features import shapes
#from affine import Affine
from shapely.geometry import shape, MultiPolygon, MultiPoint, Point
from shapely.ops import unary_union, cascaded_union
import shapely.ops as sh_ops
from descartes import PolygonPatch
from itertools import product, combinations
import pyproj

pd.options.display.max_rows = 10
```

**Stage 1 of the THERMOSS project**: creation of geoclusters for district heating and cooling catalogue

This script corresponds to the stage 1 of the THERMOSS project which goal was the creation of geoclusters of similar characterisitcs (energy, climate and building typology related) to help stakeholders choose the most suited technologies to implement. The idea behind is that if a technology has proven being efficient within a certain geocluster, it is most likely that the solution can be reproducible in other location of this same geocluster.

**Input dataset:**<br>

The table below gives the list of features that have been used to create the geoclusters. Those are believed to be the most influencial parameters for heating and cooling loads.

*Note: Environmental conditions taken from NetCDF (Network Common Data Form) meteorology dataset. The dataset used cover the area: 25N-75N x 40W-75E (extended Europe) with a 0.5 degree regular lat-lon grid with daily mean temperature from 1950-01-01 to 2016-08-31 (ENSEMBLES project 2017).*

| Parameter | Source |
|:-----------------------------------------------------------------|:--------------------------------------------------------------|
| **Environmental conditions** |  |
| Cold season mean temperature (C) | (ENSEMBLES project 2017) |
| Cold season mean temperature (C) | (ENSEMBLES project 2017) |
| **Building typology** |  |
| Residential building energy use (KWh/m2) | (EU Buildings Observatory 2017) |
| Non-residential building energy use (kWh/m2) | (EU Buildings Observatory 2017) |
| Single family unit U-value | http://webtool.building-typology.eu/#bm |
| Multiple family unit U-value | http://webtool.building-typology.eu/#bm |
| Single family unit floor area (m2) | http://webtool.building-typology.eu/#bm |
| Multiple family unit floor area (m2) | http://webtool.building-typology.eu/#bm |
| **Building stock** |  |
| Residential building stock (%) | (EU Buildings Observatory 2017) |
| Single family stock (% of residential) | (EU Buildings Observatory 2017) |
| **Economic considerations** |  |
| Energy use (kg of oil equivalent per capita) | (World Bank Group 2017) |
| Gas prices for domestic consumers (€/kWh ex. VAT) | (European Commission 2017) |
| Gas prices for industrial consumers (€/kWh ex. VAT) | (European Commission 2017) |
| Electricity prices for domestic consumers (€/kWh ex. VAT) | (European Commission 2017) |
| Electricity prices for industrial consumers (€/kWh ex. VAT) | (European Commission 2017) |
| Share of renewable energy in gross final energy consumption (%) | (European Environment Agency 2016; European Commission 2017) |


## Inputs

## countries list


```python
countries = pd.read_csv('countries.csv')
```


```python
shpFilePath = r'european-union-countries.geojson'
eu_df = gpd.read_file(shpFilePath)
#shapeData.set_index('NAME', inplace=True)
eu_df.to_crs('EPSG:3857', inplace=True)
```


```python
eu_df = eu_df.merge(countries, left_on='admin',right_on='Name', how='left')
```


```python
eu = eu_df.geometry.values.unary_union()
```

### energy


```python
url = "https://ec.europa.eu/eurostat/api/dissemination/sdmx/2.1/data/T2020_RD330/?format=CSV"
urlData = requests.get(url).content
```


```python
renew_df = pd.read_csv(io.StringIO(urlData.decode('utf-8')))
renew_df.iloc[:,0] = renew_df.iloc[:,0].apply(lambda x: x.split(';')[-1])
renew_df.rename(columns={'freq;nrg_bal;unit;geo\TIME_PERIOD': 'CODE'},inplace=True)
display(renew_df)
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
      <th>CODE</th>
      <th>2004</th>
      <th>2005</th>
      <th>2006</th>
      <th>2007</th>
      <th>2008</th>
      <th>2009</th>
      <th>2010</th>
      <th>2011</th>
      <th>2012</th>
      <th>2013</th>
      <th>2014</th>
      <th>2015</th>
      <th>2016</th>
      <th>2017</th>
      <th>2018</th>
      <th>2019</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AL</td>
      <td>29.621</td>
      <td>31.367</td>
      <td>32.070</td>
      <td>32.657</td>
      <td>32.448</td>
      <td>31.437</td>
      <td>31.867</td>
      <td>31.187</td>
      <td>35.152</td>
      <td>33.167</td>
      <td>31.856</td>
      <td>34.896</td>
      <td>36.939</td>
      <td>35.898</td>
      <td>36.844</td>
      <td>36.667</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AT</td>
      <td>22.554</td>
      <td>24.355</td>
      <td>26.277</td>
      <td>28.145</td>
      <td>28.790</td>
      <td>31.041</td>
      <td>31.207</td>
      <td>31.553</td>
      <td>32.736</td>
      <td>32.666</td>
      <td>33.553</td>
      <td>33.502</td>
      <td>33.374</td>
      <td>33.141</td>
      <td>33.806</td>
      <td>33.626</td>
    </tr>
    <tr>
      <th>2</th>
      <td>BA</td>
      <td>20.274</td>
      <td>19.752</td>
      <td>19.213</td>
      <td>18.701</td>
      <td>16.708</td>
      <td>18.471</td>
      <td>18.710</td>
      <td>17.995</td>
      <td>18.014</td>
      <td>19.307</td>
      <td>24.873</td>
      <td>26.607</td>
      <td>25.358</td>
      <td>23.241</td>
      <td>35.972</td>
      <td>37.578</td>
    </tr>
    <tr>
      <th>3</th>
      <td>BE</td>
      <td>1.890</td>
      <td>2.332</td>
      <td>2.633</td>
      <td>3.101</td>
      <td>3.590</td>
      <td>4.715</td>
      <td>6.002</td>
      <td>6.275</td>
      <td>7.089</td>
      <td>7.650</td>
      <td>8.043</td>
      <td>8.026</td>
      <td>8.752</td>
      <td>9.113</td>
      <td>9.478</td>
      <td>9.924</td>
    </tr>
    <tr>
      <th>4</th>
      <td>BG</td>
      <td>9.231</td>
      <td>9.173</td>
      <td>9.415</td>
      <td>9.098</td>
      <td>10.345</td>
      <td>12.005</td>
      <td>13.928</td>
      <td>14.152</td>
      <td>15.837</td>
      <td>18.898</td>
      <td>18.050</td>
      <td>18.261</td>
      <td>18.760</td>
      <td>18.701</td>
      <td>20.592</td>
      <td>21.564</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>34</th>
      <td>SE</td>
      <td>38.677</td>
      <td>40.265</td>
      <td>42.040</td>
      <td>43.551</td>
      <td>44.288</td>
      <td>47.476</td>
      <td>46.595</td>
      <td>48.135</td>
      <td>50.027</td>
      <td>50.792</td>
      <td>51.817</td>
      <td>52.947</td>
      <td>53.328</td>
      <td>54.157</td>
      <td>54.651</td>
      <td>56.391</td>
    </tr>
    <tr>
      <th>35</th>
      <td>SI</td>
      <td>18.397</td>
      <td>19.809</td>
      <td>18.416</td>
      <td>19.674</td>
      <td>18.645</td>
      <td>20.764</td>
      <td>21.080</td>
      <td>20.936</td>
      <td>21.549</td>
      <td>23.161</td>
      <td>22.461</td>
      <td>22.880</td>
      <td>21.977</td>
      <td>21.658</td>
      <td>21.378</td>
      <td>21.974</td>
    </tr>
    <tr>
      <th>36</th>
      <td>SK</td>
      <td>6.391</td>
      <td>6.360</td>
      <td>6.584</td>
      <td>7.766</td>
      <td>7.723</td>
      <td>9.368</td>
      <td>9.099</td>
      <td>10.348</td>
      <td>10.453</td>
      <td>10.133</td>
      <td>11.713</td>
      <td>12.883</td>
      <td>12.029</td>
      <td>11.465</td>
      <td>11.896</td>
      <td>16.894</td>
    </tr>
    <tr>
      <th>37</th>
      <td>UK</td>
      <td>1.096</td>
      <td>1.281</td>
      <td>1.488</td>
      <td>1.735</td>
      <td>2.814</td>
      <td>3.448</td>
      <td>3.862</td>
      <td>4.392</td>
      <td>4.461</td>
      <td>5.524</td>
      <td>6.737</td>
      <td>8.385</td>
      <td>9.032</td>
      <td>9.858</td>
      <td>11.138</td>
      <td>12.336</td>
    </tr>
    <tr>
      <th>38</th>
      <td>XK</td>
      <td>20.541</td>
      <td>19.773</td>
      <td>19.512</td>
      <td>18.812</td>
      <td>18.429</td>
      <td>18.230</td>
      <td>18.230</td>
      <td>17.598</td>
      <td>18.625</td>
      <td>18.823</td>
      <td>19.544</td>
      <td>18.484</td>
      <td>24.472</td>
      <td>23.082</td>
      <td>24.616</td>
      <td>25.686</td>
    </tr>
  </tbody>
</table>
<p>39 rows × 17 columns</p>
</div>

from tabulate import tabulate
import pyperclip

pyperclip.copy(tabulate(renew_df, tablefmt="pipe", headers="keys"))
print(tabulate(renew_df, tablefmt="pipe", headers="keys"))

```python
renew_df = renew_df[['CODE','2019']].rename(columns={'2019': 'renew share in %'}).set_index('CODE')
```


```python
url = "https://ec.europa.eu/eurostat/api/dissemination/sdmx/2.1/data/T2020_RK200/?format=CSV"
urlData = requests.get(url).content
```


```python
energy_df = pd.read_csv(io.StringIO(urlData.decode('utf-8')))
energy_df.iloc[:,0] = energy_df.iloc[:,0].apply(lambda x: x.split(';')[-1])
energy_df.rename(columns={'freq;siec;unit;nrg_bal;geo\TIME_PERIOD': 'CODE'},inplace=True)
energy_df
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
      <th>CODE</th>
      <th>1990</th>
      <th>1991</th>
      <th>1992</th>
      <th>1993</th>
      <th>1994</th>
      <th>1995</th>
      <th>1996</th>
      <th>1997</th>
      <th>1998</th>
      <th>...</th>
      <th>2010</th>
      <th>2011</th>
      <th>2012</th>
      <th>2013</th>
      <th>2014</th>
      <th>2015</th>
      <th>2016</th>
      <th>2017</th>
      <th>2018</th>
      <th>2019</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AL</td>
      <td>527.995</td>
      <td>531.026</td>
      <td>508.966</td>
      <td>469.652</td>
      <td>452.131</td>
      <td>474.021</td>
      <td>516.029</td>
      <td>493.527</td>
      <td>483.591</td>
      <td>...</td>
      <td>489.039</td>
      <td>498.280</td>
      <td>509.086</td>
      <td>580.003</td>
      <td>556.243</td>
      <td>532.287</td>
      <td>496.350</td>
      <td>492.580</td>
      <td>510.484</td>
      <td>:</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AT</td>
      <td>5881.122</td>
      <td>6531.442</td>
      <td>6164.417</td>
      <td>6274.606</td>
      <td>5917.381</td>
      <td>6322.048</td>
      <td>6961.087</td>
      <td>6272.585</td>
      <td>6379.629</td>
      <td>...</td>
      <td>7070.224</td>
      <td>6551.691</td>
      <td>6677.912</td>
      <td>6923.356</td>
      <td>6239.870</td>
      <td>6631.937</td>
      <td>6909.468</td>
      <td>6955.601</td>
      <td>6542.766</td>
      <td>6695.287</td>
    </tr>
    <tr>
      <th>2</th>
      <td>BA</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>...</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>980.816</td>
      <td>1113.030</td>
      <td>1139.361</td>
      <td>1045.641</td>
      <td>1719.424</td>
      <td>:</td>
    </tr>
    <tr>
      <th>3</th>
      <td>BE</td>
      <td>8289.251</td>
      <td>9151.751</td>
      <td>9147.798</td>
      <td>9091.449</td>
      <td>8937.893</td>
      <td>9320.715</td>
      <td>10619.932</td>
      <td>9883.169</td>
      <td>9911.078</td>
      <td>...</td>
      <td>9614.660</td>
      <td>8000.830</td>
      <td>8477.623</td>
      <td>9098.495</td>
      <td>7489.331</td>
      <td>8270.398</td>
      <td>8313.535</td>
      <td>8180.798</td>
      <td>8118.234</td>
      <td>7899.465</td>
    </tr>
    <tr>
      <th>4</th>
      <td>BG</td>
      <td>2336.070</td>
      <td>2479.321</td>
      <td>2626.804</td>
      <td>2829.252</td>
      <td>2443.498</td>
      <td>2447.466</td>
      <td>2685.130</td>
      <td>2213.172</td>
      <td>2424.369</td>
      <td>...</td>
      <td>2243.338</td>
      <td>2374.281</td>
      <td>2352.559</td>
      <td>2241.036</td>
      <td>2164.991</td>
      <td>2192.912</td>
      <td>2252.111</td>
      <td>2318.716</td>
      <td>2229.673</td>
      <td>2159.861</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>35</th>
      <td>SI</td>
      <td>953.347</td>
      <td>1134.819</td>
      <td>1011.722</td>
      <td>1101.065</td>
      <td>1093.883</td>
      <td>1163.921</td>
      <td>1033.136</td>
      <td>1070.152</td>
      <td>1030.235</td>
      <td>...</td>
      <td>1364.215</td>
      <td>1302.665</td>
      <td>1255.831</td>
      <td>1240.442</td>
      <td>1060.707</td>
      <td>1166.185</td>
      <td>1186.482</td>
      <td>1156.364</td>
      <td>1084.519</td>
      <td>1057.443</td>
    </tr>
    <tr>
      <th>36</th>
      <td>SK</td>
      <td>2246.498</td>
      <td>1888.877</td>
      <td>1767.102</td>
      <td>1720.807</td>
      <td>1771.655</td>
      <td>1976.756</td>
      <td>2232.746</td>
      <td>2362.268</td>
      <td>2448.105</td>
      <td>...</td>
      <td>2311.582</td>
      <td>2121.122</td>
      <td>2070.086</td>
      <td>2146.977</td>
      <td>1951.912</td>
      <td>1987.662</td>
      <td>2030.217</td>
      <td>2108.641</td>
      <td>2057.519</td>
      <td>2643.412</td>
    </tr>
    <tr>
      <th>37</th>
      <td>TR</td>
      <td>14627.319</td>
      <td>14942.931</td>
      <td>15702.925</td>
      <td>15763.148</td>
      <td>14946.096</td>
      <td>16329.111</td>
      <td>16805.932</td>
      <td>17504.025</td>
      <td>17103.364</td>
      <td>...</td>
      <td>19414.937</td>
      <td>20809.776</td>
      <td>20187.463</td>
      <td>19807.984</td>
      <td>19166.344</td>
      <td>20153.330</td>
      <td>20711.118</td>
      <td>22167.051</td>
      <td>20557.145</td>
      <td>:</td>
    </tr>
    <tr>
      <th>38</th>
      <td>UK</td>
      <td>37317.942</td>
      <td>40815.072</td>
      <td>40210.237</td>
      <td>41658.464</td>
      <td>40309.277</td>
      <td>39335.159</td>
      <td>44132.057</td>
      <td>41134.181</td>
      <td>42348.175</td>
      <td>...</td>
      <td>45512.575</td>
      <td>37786.091</td>
      <td>40957.945</td>
      <td>41458.259</td>
      <td>35813.099</td>
      <td>37268.868</td>
      <td>37970.686</td>
      <td>36772.290</td>
      <td>38307.568</td>
      <td>38134.984</td>
    </tr>
    <tr>
      <th>39</th>
      <td>XK</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>...</td>
      <td>461.840</td>
      <td>491.178</td>
      <td>474.234</td>
      <td>494.163</td>
      <td>479.943</td>
      <td>476.388</td>
      <td>549.413</td>
      <td>571.465</td>
      <td>573.404</td>
      <td>586.881</td>
    </tr>
  </tbody>
</table>
<p>40 rows × 31 columns</p>
</div>




```python
energy_df = energy_df[['CODE','2019 ']].rename(columns={'2019 ': 'energy cons (kg of oil equivalent per capita)'}).set_index('CODE')
```


```python
url = "https://ec.europa.eu/eurostat/api/dissemination/sdmx/2.1/data/TEN00118/?format=CSV"
urlData = requests.get(url).content
```


```python
gas_df = pd.read_csv(io.StringIO(urlData.decode('utf-8')))
gas_df['type'] = gas_df.iloc[:,0].apply(lambda x: x.split(';')[-2])
gas_df.iloc[:,0] = gas_df.iloc[:,0].apply(lambda x: x.split(';')[-1])
gas_df.rename(columns={'freq;product;currency;unit;indic_en;geo\TIME_PERIOD': 'CODE'},inplace=True)
gas_df
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
      <th>CODE</th>
      <th>2009</th>
      <th>2010</th>
      <th>2011</th>
      <th>2012</th>
      <th>2013</th>
      <th>2014</th>
      <th>2015</th>
      <th>2016</th>
      <th>2017</th>
      <th>2018</th>
      <th>2019</th>
      <th>2020</th>
      <th>type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AT</td>
      <td>18.0300</td>
      <td>17.2900</td>
      <td>19.2900</td>
      <td>21.0500</td>
      <td>21.3200</td>
      <td>20.7800</td>
      <td>20.2800</td>
      <td>19.1700</td>
      <td>18.7100</td>
      <td>18.5850</td>
      <td>18.3400</td>
      <td>17.9895</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>1</th>
      <td>BA</td>
      <td>:</td>
      <td>10.4796</td>
      <td>12.5510</td>
      <td>15.4082</td>
      <td>15.4082</td>
      <td>14.2347</td>
      <td>14.2650</td>
      <td>10.8598</td>
      <td>8.5335</td>
      <td>9.0462</td>
      <td>9.3109</td>
      <td>10.3855</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>2</th>
      <td>BE</td>
      <td>16.8200</td>
      <td>14.7000</td>
      <td>17.6000</td>
      <td>19.1300</td>
      <td>18.3200</td>
      <td>18.2700</td>
      <td>16.2300</td>
      <td>15.1900</td>
      <td>14.5114</td>
      <td>14.8847</td>
      <td>15.3755</td>
      <td>13.7697</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>3</th>
      <td>BG</td>
      <td>13.1404</td>
      <td>10.2107</td>
      <td>11.9440</td>
      <td>13.7233</td>
      <td>14.2397</td>
      <td>13.6261</td>
      <td>13.2580</td>
      <td>10.2260</td>
      <td>9.1778</td>
      <td>10.5356</td>
      <td>12.4631</td>
      <td>11.0110</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>4</th>
      <td>CZ</td>
      <td>13.7480</td>
      <td>13.0395</td>
      <td>15.1247</td>
      <td>18.3111</td>
      <td>17.8029</td>
      <td>15.2285</td>
      <td>15.9493</td>
      <td>16.1879</td>
      <td>15.2685</td>
      <td>15.9675</td>
      <td>16.2682</td>
      <td>15.9069</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>68</th>
      <td>SI</td>
      <td>11.3400</td>
      <td>10.8766</td>
      <td>11.1900</td>
      <td>14.8000</td>
      <td>12.3800</td>
      <td>10.6400</td>
      <td>8.9300</td>
      <td>8.1800</td>
      <td>7.1900</td>
      <td>7.6043</td>
      <td>8.5024</td>
      <td>6.8561</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>69</th>
      <td>SK</td>
      <td>11.1200</td>
      <td>8.7390</td>
      <td>9.2200</td>
      <td>10.6000</td>
      <td>9.8800</td>
      <td>9.9100</td>
      <td>9.2900</td>
      <td>8.1100</td>
      <td>7.4748</td>
      <td>7.6680</td>
      <td>9.1350</td>
      <td>8.1740</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>70</th>
      <td>TR</td>
      <td>7.7118</td>
      <td>6.3672</td>
      <td>5.7822</td>
      <td>6.7818</td>
      <td>8.2623</td>
      <td>6.5550</td>
      <td>7.5261</td>
      <td>6.6187</td>
      <td>5.0315</td>
      <td>4.7541</td>
      <td>5.7577</td>
      <td>5.9961</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>71</th>
      <td>UA</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>7.2733</td>
      <td>6.8203</td>
      <td>7.1643</td>
      <td>4.2910</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>72</th>
      <td>UK</td>
      <td>7.6862</td>
      <td>5.9426</td>
      <td>6.4724</td>
      <td>8.5986</td>
      <td>9.3555</td>
      <td>9.8504</td>
      <td>9.4236</td>
      <td>7.6422</td>
      <td>6.5778</td>
      <td>6.8814</td>
      <td>7.2799</td>
      <td>6.9937</td>
      <td>MSIND</td>
    </tr>
  </tbody>
</table>
<p>73 rows × 14 columns</p>
</div>




```python
gas_df = gas_df.set_index(['CODE','type'])['2019 '].unstack().rename(columns={'MSHH': 'residential gas price in euro','MSIND':'indus gas price in euro'})
```


```python
url = "https://ec.europa.eu/eurostat/api/dissemination/sdmx/2.1/data/TEN00117/?format=CSV"
urlData = requests.get(url).content   
```


```python
elc_df = pd.read_csv(io.StringIO(urlData.decode('utf-8')))
elc_df['type'] = elc_df.iloc[:,0].apply(lambda x: x.split(';')[-2])
elc_df.iloc[:,0] = elc_df.iloc[:,0].apply(lambda x: x.split(';')[-1])
elc_df.rename(columns={'freq;product;currency;unit;indic_en;geo\TIME_PERIOD': 'CODE'},inplace=True)
elc_df
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
      <th>CODE</th>
      <th>2009</th>
      <th>2010</th>
      <th>2011</th>
      <th>2012</th>
      <th>2013</th>
      <th>2014</th>
      <th>2015</th>
      <th>2016</th>
      <th>2017</th>
      <th>2018</th>
      <th>2019</th>
      <th>2020</th>
      <th>type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AL</td>
      <td>:</td>
      <td>:</td>
      <td>0.1152</td>
      <td>0.1163</td>
      <td>0.1156</td>
      <td>0.1156</td>
      <td>0.0812</td>
      <td>0.0824</td>
      <td>0.0844</td>
      <td>:</td>
      <td>0.0920 e</td>
      <td>:</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AT</td>
      <td>0.1909</td>
      <td>0.1967</td>
      <td>0.1986</td>
      <td>0.1975</td>
      <td>0.2082</td>
      <td>0.2021</td>
      <td>0.2009</td>
      <td>0.2034</td>
      <td>0.1950</td>
      <td>0.1966</td>
      <td>0.2034</td>
      <td>0.2102</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>2</th>
      <td>BA</td>
      <td>:</td>
      <td>0.0741</td>
      <td>0.0745</td>
      <td>0.0798</td>
      <td>0.0803</td>
      <td>0.0791</td>
      <td>0.0812</td>
      <td>0.0831</td>
      <td>0.0859</td>
      <td>0.0864</td>
      <td>0.0873</td>
      <td>0.0870</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>3</th>
      <td>BE</td>
      <td>0.1916</td>
      <td>0.1959</td>
      <td>0.2136</td>
      <td>0.2327</td>
      <td>0.2173</td>
      <td>0.2097</td>
      <td>0.2126</td>
      <td>0.2544</td>
      <td>0.2857</td>
      <td>0.2824</td>
      <td>0.2839</td>
      <td>0.2792</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>4</th>
      <td>BG</td>
      <td>0.0823</td>
      <td>0.0813</td>
      <td>0.0826</td>
      <td>0.0846</td>
      <td>0.0924</td>
      <td>0.0832</td>
      <td>0.0942</td>
      <td>0.0956</td>
      <td>0.0955</td>
      <td>0.0979</td>
      <td>0.0997</td>
      <td>0.0997</td>
      <td>MSHH</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>83</th>
      <td>SK</td>
      <td>0.1416</td>
      <td>0.1161</td>
      <td>0.1233</td>
      <td>0.1273</td>
      <td>0.1242</td>
      <td>0.1107</td>
      <td>0.1081</td>
      <td>0.1047</td>
      <td>0.0741</td>
      <td>0.0790</td>
      <td>0.0921</td>
      <td>0.0977</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>84</th>
      <td>TR</td>
      <td>0.0754</td>
      <td>0.0863</td>
      <td>0.0760</td>
      <td>0.0831</td>
      <td>0.0891</td>
      <td>0.0720</td>
      <td>0.0790</td>
      <td>0.0722</td>
      <td>0.0615</td>
      <td>0.0571</td>
      <td>0.0683</td>
      <td>0.0774</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>85</th>
      <td>UA</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>0.0595</td>
      <td>0.0656</td>
      <td>0.0595</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>86</th>
      <td>UK</td>
      <td>0.1077</td>
      <td>0.0947</td>
      <td>0.0939</td>
      <td>0.1095</td>
      <td>0.1124</td>
      <td>0.1246</td>
      <td>0.1184</td>
      <td>0.1042</td>
      <td>0.0938</td>
      <td>0.0970</td>
      <td>0.0998</td>
      <td>0.1065</td>
      <td>MSIND</td>
    </tr>
    <tr>
      <th>87</th>
      <td>XK</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>:</td>
      <td>0.0617</td>
      <td>0.0637</td>
      <td>0.0705</td>
      <td>0.0687</td>
      <td>0.0731</td>
      <td>0.0729</td>
      <td>0.0642</td>
      <td>0.0638</td>
      <td>MSIND</td>
    </tr>
  </tbody>
</table>
<p>88 rows × 14 columns</p>
</div>




```python
 elc_df = elc_df.set_index(['CODE','type'])['2017 '].unstack().rename(columns={'MSHH': 'residential elec price in euro','MSIND':'indus elec price in euro'})
```

### Building stock


```python
stock_df = pd.read_csv('building_stock.csv')
stock_df.rename(columns={'Unnamed: 0': 'Country'},inplace=True)
stock_df
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
      <th>Country</th>
      <th>Residential share in %</th>
      <th>Non-residential share in %</th>
      <th>single family share in %</th>
      <th>mutiple family share in %</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Austria</td>
      <td>0.616</td>
      <td>0.384</td>
      <td>0.475</td>
      <td>0.525</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Belgium</td>
      <td>0.675</td>
      <td>0.325</td>
      <td>0.729</td>
      <td>0.271</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Bulgaria</td>
      <td>0.721</td>
      <td>0.279</td>
      <td>0.549</td>
      <td>0.451</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Croatia</td>
      <td>0.777</td>
      <td>0.223</td>
      <td>0.659</td>
      <td>0.341</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Cyprus</td>
      <td>0.862</td>
      <td>0.138</td>
      <td>0.635</td>
      <td>0.365</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>23</th>
      <td>Slovakia</td>
      <td>0.594</td>
      <td>0.406</td>
      <td>0.492</td>
      <td>0.508</td>
    </tr>
    <tr>
      <th>24</th>
      <td>Slovenia</td>
      <td>0.817</td>
      <td>0.183</td>
      <td>0.607</td>
      <td>0.393</td>
    </tr>
    <tr>
      <th>25</th>
      <td>Spain</td>
      <td>0.827</td>
      <td>0.173</td>
      <td>0.292</td>
      <td>0.708</td>
    </tr>
    <tr>
      <th>26</th>
      <td>Sweden</td>
      <td>0.666</td>
      <td>0.334</td>
      <td>0.441</td>
      <td>0.559</td>
    </tr>
    <tr>
      <th>27</th>
      <td>United Kingdom</td>
      <td>0.762</td>
      <td>0.238</td>
      <td>0.823</td>
      <td>0.177</td>
    </tr>
  </tbody>
</table>
<p>28 rows × 5 columns</p>
</div>




```python
stock_df = stock_df.merge(countries,left_on='Country',right_on='Name',how='left')
stock_df = stock_df[['Code','Residential share in %','Non-residential share in %','single family share in %','mutiple family share in %']].set_index('Code')
stock_df = stock_df*100
stock_df
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
      <th>Residential share in %</th>
      <th>Non-residential share in %</th>
      <th>single family share in %</th>
      <th>mutiple family share in %</th>
    </tr>
    <tr>
      <th>Code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>AT</th>
      <td>61.6</td>
      <td>38.4</td>
      <td>47.5</td>
      <td>52.5</td>
    </tr>
    <tr>
      <th>BE</th>
      <td>67.5</td>
      <td>32.5</td>
      <td>72.9</td>
      <td>27.1</td>
    </tr>
    <tr>
      <th>BG</th>
      <td>72.1</td>
      <td>27.9</td>
      <td>54.9</td>
      <td>45.1</td>
    </tr>
    <tr>
      <th>HR</th>
      <td>77.7</td>
      <td>22.3</td>
      <td>65.9</td>
      <td>34.1</td>
    </tr>
    <tr>
      <th>CY</th>
      <td>86.2</td>
      <td>13.8</td>
      <td>63.5</td>
      <td>36.5</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>SK</th>
      <td>59.4</td>
      <td>40.6</td>
      <td>49.2</td>
      <td>50.8</td>
    </tr>
    <tr>
      <th>SI</th>
      <td>81.7</td>
      <td>18.3</td>
      <td>60.7</td>
      <td>39.3</td>
    </tr>
    <tr>
      <th>ES</th>
      <td>82.7</td>
      <td>17.3</td>
      <td>29.2</td>
      <td>70.8</td>
    </tr>
    <tr>
      <th>SE</th>
      <td>66.6</td>
      <td>33.4</td>
      <td>44.1</td>
      <td>55.9</td>
    </tr>
    <tr>
      <th>GB</th>
      <td>76.2</td>
      <td>23.8</td>
      <td>82.3</td>
      <td>17.7</td>
    </tr>
  </tbody>
</table>
<p>28 rows × 4 columns</p>
</div>



### Building typology


```python
typo_df = pd.read_csv('building_typo.csv')
typo_df.rename(columns={'Unnamed: 0': 'Country'},inplace=True)
typo_df
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
      <th>Country</th>
      <th>single family U value</th>
      <th>multiple family U value</th>
      <th>single family area (m2)</th>
      <th>mutiple family area (m2)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Austria</td>
      <td>0.978513</td>
      <td>0.882793</td>
      <td>145</td>
      <td>418.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Belgium</td>
      <td>1.019217</td>
      <td>1.121330</td>
      <td>220</td>
      <td>1613.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Bulgaria</td>
      <td>1.182875</td>
      <td>0.989048</td>
      <td>172</td>
      <td>495.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Czech Republic</td>
      <td>0.813539</td>
      <td>0.967243</td>
      <td>112</td>
      <td>681.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Denmark</td>
      <td>0.479975</td>
      <td>0.649674</td>
      <td>127</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Poland</td>
      <td>0.699272</td>
      <td>0.729197</td>
      <td>136</td>
      <td>2186.0</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Slovenia</td>
      <td>0.788727</td>
      <td>1.110169</td>
      <td>201</td>
      <td>1362.0</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Spain</td>
      <td>1.680646</td>
      <td>1.661011</td>
      <td>248</td>
      <td>1022.0</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Sweden</td>
      <td>0.255006</td>
      <td>NaN</td>
      <td>121</td>
      <td>1207.0</td>
    </tr>
    <tr>
      <th>16</th>
      <td>United Kingdom</td>
      <td>1.305911</td>
      <td>1.310864</td>
      <td>147</td>
      <td>653.0</td>
    </tr>
  </tbody>
</table>
<p>17 rows × 5 columns</p>
</div>




```python
typo_df = typo_df.merge(countries,left_on='Country',right_on='Name',how='left')
typo_df = typo_df[['Code','single family U value','multiple family U value','single family area (m2)','mutiple family area (m2)']].set_index('Code')
typo_df
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
      <th>single family U value</th>
      <th>multiple family U value</th>
      <th>single family area (m2)</th>
      <th>mutiple family area (m2)</th>
    </tr>
    <tr>
      <th>Code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>AT</th>
      <td>0.978513</td>
      <td>0.882793</td>
      <td>145</td>
      <td>418.0</td>
    </tr>
    <tr>
      <th>BE</th>
      <td>1.019217</td>
      <td>1.121330</td>
      <td>220</td>
      <td>1613.0</td>
    </tr>
    <tr>
      <th>BG</th>
      <td>1.182875</td>
      <td>0.989048</td>
      <td>172</td>
      <td>495.0</td>
    </tr>
    <tr>
      <th>CZ</th>
      <td>0.813539</td>
      <td>0.967243</td>
      <td>112</td>
      <td>681.0</td>
    </tr>
    <tr>
      <th>DK</th>
      <td>0.479975</td>
      <td>0.649674</td>
      <td>127</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>PL</th>
      <td>0.699272</td>
      <td>0.729197</td>
      <td>136</td>
      <td>2186.0</td>
    </tr>
    <tr>
      <th>SI</th>
      <td>0.788727</td>
      <td>1.110169</td>
      <td>201</td>
      <td>1362.0</td>
    </tr>
    <tr>
      <th>ES</th>
      <td>1.680646</td>
      <td>1.661011</td>
      <td>248</td>
      <td>1022.0</td>
    </tr>
    <tr>
      <th>SE</th>
      <td>0.255006</td>
      <td>NaN</td>
      <td>121</td>
      <td>1207.0</td>
    </tr>
    <tr>
      <th>GB</th>
      <td>1.305911</td>
      <td>1.310864</td>
      <td>147</td>
      <td>653.0</td>
    </tr>
  </tbody>
</table>
<p>17 rows × 4 columns</p>
</div>



### Environmental conditions


```python
with xr.open_dataset('tg_0.50deg_reg_1995-2016_v14.0.nc') as file:
    # You can open the data using the code below, however it you use rio.write_crs
    # it will ensure that crs for your data persist throughout your analysis
    # max_temp_xr = file_nc
    dsCDF = file.rio.write_crs("epsg:4326", inplace=True)
    
dsCDF
```




<div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:      (latitude: 101, longitude: 232, time: 7914)
Coordinates:
  * longitude    (longitude) float32 -40.25 -39.75 -39.25 ... 74.25 74.75 75.25
  * latitude     (latitude) float32 25.25 25.75 26.25 ... 74.25 74.75 75.25
  * time         (time) datetime64[ns] 1995-01-01 1995-01-02 ... 2016-08-31
    spatial_ref  int32 0
Data variables:
    tg           (time, latitude, longitude) float32 ...
Attributes:
    Ensembles_ECAD:  14.0
    Conventions:     CF-1.4
    References:      http://www.ecad.eu\nhttp://www.ecad.eu/download/ensemble...
    history:         Mon Oct 17 11:28:47 2016: ncks -a -d time,16436,24349 tg...
    NCO:             &quot;4.5.3&quot;
    grid_mapping:    spatial_ref</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-e40d6304-1e94-4402-98dc-9353dd9ee838' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-e40d6304-1e94-4402-98dc-9353dd9ee838' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>latitude</span>: 101</li><li><span class='xr-has-index'>longitude</span>: 232</li><li><span class='xr-has-index'>time</span>: 7914</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-16614d6d-236b-43b2-a4d6-edbf80c5ca87' class='xr-section-summary-in' type='checkbox'  checked><label for='section-16614d6d-236b-43b2-a4d6-edbf80c5ca87' class='xr-section-summary' >Coordinates: <span>(4)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>longitude</span></div><div class='xr-var-dims'>(longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>-40.25 -39.75 ... 74.75 75.25</div><input id='attrs-411551bf-fe76-4d46-be5e-13a00b86a924' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-411551bf-fe76-4d46-be5e-13a00b86a924' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-2d7887bf-3c13-46ad-979b-3f2919d32b44' class='xr-var-data-in' type='checkbox'><label for='data-2d7887bf-3c13-46ad-979b-3f2919d32b44' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Longitude values</dd><dt><span>units :</span></dt><dd>degrees_east</dd><dt><span>standard_name :</span></dt><dd>longitude</dd></dl></div><div class='xr-var-data'><pre>array([-40.25, -39.75, -39.25, ...,  74.25,  74.75,  75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>latitude</span></div><div class='xr-var-dims'>(latitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>25.25 25.75 26.25 ... 74.75 75.25</div><input id='attrs-b78aa35d-fb9b-431a-a37f-5b3a6ceed329' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-b78aa35d-fb9b-431a-a37f-5b3a6ceed329' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-fd3dc4a4-653f-443f-b415-a72f6b052e00' class='xr-var-data-in' type='checkbox'><label for='data-fd3dc4a4-653f-443f-b415-a72f6b052e00' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Latitude values</dd><dt><span>units :</span></dt><dd>degrees_north</dd><dt><span>standard_name :</span></dt><dd>latitude</dd></dl></div><div class='xr-var-data'><pre>array([25.25, 25.75, 26.25, 26.75, 27.25, 27.75, 28.25, 28.75, 29.25, 29.75,
       30.25, 30.75, 31.25, 31.75, 32.25, 32.75, 33.25, 33.75, 34.25, 34.75,
       35.25, 35.75, 36.25, 36.75, 37.25, 37.75, 38.25, 38.75, 39.25, 39.75,
       40.25, 40.75, 41.25, 41.75, 42.25, 42.75, 43.25, 43.75, 44.25, 44.75,
       45.25, 45.75, 46.25, 46.75, 47.25, 47.75, 48.25, 48.75, 49.25, 49.75,
       50.25, 50.75, 51.25, 51.75, 52.25, 52.75, 53.25, 53.75, 54.25, 54.75,
       55.25, 55.75, 56.25, 56.75, 57.25, 57.75, 58.25, 58.75, 59.25, 59.75,
       60.25, 60.75, 61.25, 61.75, 62.25, 62.75, 63.25, 63.75, 64.25, 64.75,
       65.25, 65.75, 66.25, 66.75, 67.25, 67.75, 68.25, 68.75, 69.25, 69.75,
       70.25, 70.75, 71.25, 71.75, 72.25, 72.75, 73.25, 73.75, 74.25, 74.75,
       75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>time</span></div><div class='xr-var-dims'>(time)</div><div class='xr-var-dtype'>datetime64[ns]</div><div class='xr-var-preview xr-preview'>1995-01-01 ... 2016-08-31</div><input id='attrs-d967d9d9-a4e0-4c5d-87c8-20d8e2e4f0ca' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-d967d9d9-a4e0-4c5d-87c8-20d8e2e4f0ca' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-4fd80a42-9dfd-4238-99d1-50098245b7e6' class='xr-var-data-in' type='checkbox'><label for='data-4fd80a42-9dfd-4238-99d1-50098245b7e6' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Time in days</dd><dt><span>standard_name :</span></dt><dd>time</dd></dl></div><div class='xr-var-data'><pre>array([&#x27;1995-01-01T00:00:00.000000000&#x27;, &#x27;1995-01-02T00:00:00.000000000&#x27;,
       &#x27;1995-01-03T00:00:00.000000000&#x27;, ..., &#x27;2016-08-29T00:00:00.000000000&#x27;,
       &#x27;2016-08-30T00:00:00.000000000&#x27;, &#x27;2016-08-31T00:00:00.000000000&#x27;],
      dtype=&#x27;datetime64[ns]&#x27;)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-40befb34-e1ad-4b84-8e8b-0623252d87da' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-40befb34-e1ad-4b84-8e8b-0623252d87da' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-a766075a-1f01-416a-b2af-8b404cb9d19a' class='xr-var-data-in' type='checkbox'><label for='data-a766075a-1f01-416a-b2af-8b404cb9d19a' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd><dt><span>semi_major_axis :</span></dt><dd>6378137.0</dd><dt><span>semi_minor_axis :</span></dt><dd>6356752.314245179</dd><dt><span>inverse_flattening :</span></dt><dd>298.257223563</dd><dt><span>reference_ellipsoid_name :</span></dt><dd>WGS 84</dd><dt><span>longitude_of_prime_meridian :</span></dt><dd>0.0</dd><dt><span>prime_meridian_name :</span></dt><dd>Greenwich</dd><dt><span>geographic_crs_name :</span></dt><dd>WGS 84</dd><dt><span>grid_mapping_name :</span></dt><dd>latitude_longitude</dd><dt><span>spatial_ref :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-56eca7a9-f58c-4f55-b750-2a576faee6c7' class='xr-section-summary-in' type='checkbox'  checked><label for='section-56eca7a9-f58c-4f55-b750-2a576faee6c7' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(time, latitude, longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>...</div><input id='attrs-9032ff50-ee96-41b6-9e4b-f34c012bc71d' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-9032ff50-ee96-41b6-9e4b-f34c012bc71d' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-854086c0-60e4-46c6-aac8-a8b3fdacba5e' class='xr-var-data-in' type='checkbox'><label for='data-854086c0-60e4-46c6-aac8-a8b3fdacba5e' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>mean temperature</dd><dt><span>units :</span></dt><dd>Celsius</dd><dt><span>standard_name :</span></dt><dd>air_temperature</dd><dt><span>grid_mapping :</span></dt><dd>spatial_ref</dd></dl></div><div class='xr-var-data'><pre>[185440848 values with dtype=float32]</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-a4c29401-6dc6-402d-8a67-6b11a0ab30d4' class='xr-section-summary-in' type='checkbox'  checked><label for='section-a4c29401-6dc6-402d-8a67-6b11a0ab30d4' class='xr-section-summary' >Attributes: <span>(6)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'><dt><span>Ensembles_ECAD :</span></dt><dd>14.0</dd><dt><span>Conventions :</span></dt><dd>CF-1.4</dd><dt><span>References :</span></dt><dd>http://www.ecad.eu\nhttp://www.ecad.eu/download/ensembles/ensembles.php\nhttp://www.ecad.eu/download/ensembles/Haylock_et_al_2008.pdf</dd><dt><span>history :</span></dt><dd>Mon Oct 17 11:28:47 2016: ncks -a -d time,16436,24349 tg_0.50deg_regular_1.nc tg_0.50deg_reg_1995-2016_v14.0.nc
Mon Oct 17 11:28:39 2016: ncks -a --mk_rec_dmn time tg_0.50deg_regular.nc tg_0.50deg_regular_1.nc</dd><dt><span>NCO :</span></dt><dd>&quot;4.5.3&quot;</dd><dt><span>grid_mapping :</span></dt><dd>spatial_ref</dd></dl></div></li></ul></div></div>




```python
#Create typical year from the 20 years'set with daily data point (grid X 366 days)
typical_year = dsCDF.groupby('time.dayofyear').median('time') 
```


```python
display(typical_year)
```


<div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:      (dayofyear: 366, latitude: 101, longitude: 232)
Coordinates:
  * longitude    (longitude) float32 -40.25 -39.75 -39.25 ... 74.25 74.75 75.25
  * latitude     (latitude) float32 25.25 25.75 26.25 ... 74.25 74.75 75.25
    spatial_ref  int32 0
  * dayofyear    (dayofyear) int64 1 2 3 4 5 6 7 ... 360 361 362 363 364 365 366
Data variables:
    tg           (dayofyear, latitude, longitude) float32 nan nan ... nan nan</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-68d00950-9b3f-467c-9c62-252f0b58239a' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-68d00950-9b3f-467c-9c62-252f0b58239a' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>dayofyear</span>: 366</li><li><span class='xr-has-index'>latitude</span>: 101</li><li><span class='xr-has-index'>longitude</span>: 232</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-39fcdfc9-2139-43f0-be38-9365782e53e7' class='xr-section-summary-in' type='checkbox'  checked><label for='section-39fcdfc9-2139-43f0-be38-9365782e53e7' class='xr-section-summary' >Coordinates: <span>(4)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>longitude</span></div><div class='xr-var-dims'>(longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>-40.25 -39.75 ... 74.75 75.25</div><input id='attrs-ae318c7b-e737-4059-8ebf-bd9367e52b59' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-ae318c7b-e737-4059-8ebf-bd9367e52b59' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-977d1dc2-00a8-4cfb-99d1-2ebbee5f9a73' class='xr-var-data-in' type='checkbox'><label for='data-977d1dc2-00a8-4cfb-99d1-2ebbee5f9a73' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Longitude values</dd><dt><span>units :</span></dt><dd>degrees_east</dd><dt><span>standard_name :</span></dt><dd>longitude</dd></dl></div><div class='xr-var-data'><pre>array([-40.25, -39.75, -39.25, ...,  74.25,  74.75,  75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>latitude</span></div><div class='xr-var-dims'>(latitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>25.25 25.75 26.25 ... 74.75 75.25</div><input id='attrs-12394d1d-ef23-4ad9-8e62-72af2c57a405' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-12394d1d-ef23-4ad9-8e62-72af2c57a405' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-174a9ae7-e039-444c-8c2c-02f7228beca2' class='xr-var-data-in' type='checkbox'><label for='data-174a9ae7-e039-444c-8c2c-02f7228beca2' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Latitude values</dd><dt><span>units :</span></dt><dd>degrees_north</dd><dt><span>standard_name :</span></dt><dd>latitude</dd></dl></div><div class='xr-var-data'><pre>array([25.25, 25.75, 26.25, 26.75, 27.25, 27.75, 28.25, 28.75, 29.25, 29.75,
       30.25, 30.75, 31.25, 31.75, 32.25, 32.75, 33.25, 33.75, 34.25, 34.75,
       35.25, 35.75, 36.25, 36.75, 37.25, 37.75, 38.25, 38.75, 39.25, 39.75,
       40.25, 40.75, 41.25, 41.75, 42.25, 42.75, 43.25, 43.75, 44.25, 44.75,
       45.25, 45.75, 46.25, 46.75, 47.25, 47.75, 48.25, 48.75, 49.25, 49.75,
       50.25, 50.75, 51.25, 51.75, 52.25, 52.75, 53.25, 53.75, 54.25, 54.75,
       55.25, 55.75, 56.25, 56.75, 57.25, 57.75, 58.25, 58.75, 59.25, 59.75,
       60.25, 60.75, 61.25, 61.75, 62.25, 62.75, 63.25, 63.75, 64.25, 64.75,
       65.25, 65.75, 66.25, 66.75, 67.25, 67.75, 68.25, 68.75, 69.25, 69.75,
       70.25, 70.75, 71.25, 71.75, 72.25, 72.75, 73.25, 73.75, 74.25, 74.75,
       75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-b525ea30-e637-45b4-9797-084b4f160e65' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-b525ea30-e637-45b4-9797-084b4f160e65' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-f4411837-7ebc-4468-b26e-22dc18e32d09' class='xr-var-data-in' type='checkbox'><label for='data-f4411837-7ebc-4468-b26e-22dc18e32d09' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd><dt><span>semi_major_axis :</span></dt><dd>6378137.0</dd><dt><span>semi_minor_axis :</span></dt><dd>6356752.314245179</dd><dt><span>inverse_flattening :</span></dt><dd>298.257223563</dd><dt><span>reference_ellipsoid_name :</span></dt><dd>WGS 84</dd><dt><span>longitude_of_prime_meridian :</span></dt><dd>0.0</dd><dt><span>prime_meridian_name :</span></dt><dd>Greenwich</dd><dt><span>geographic_crs_name :</span></dt><dd>WGS 84</dd><dt><span>grid_mapping_name :</span></dt><dd>latitude_longitude</dd><dt><span>spatial_ref :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>dayofyear</span></div><div class='xr-var-dims'>(dayofyear)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>1 2 3 4 5 6 ... 362 363 364 365 366</div><input id='attrs-281f8365-24c4-4d6f-97df-c9dd2d089732' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-281f8365-24c4-4d6f-97df-c9dd2d089732' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-c6ce6f72-0759-4e37-9295-de696d736e64' class='xr-var-data-in' type='checkbox'><label for='data-c6ce6f72-0759-4e37-9295-de696d736e64' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([  1,   2,   3, ..., 364, 365, 366], dtype=int64)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-aafc4d4d-7cd1-47a7-97c7-39579b8356fb' class='xr-section-summary-in' type='checkbox'  checked><label for='section-aafc4d4d-7cd1-47a7-97c7-39579b8356fb' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(dayofyear, latitude, longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-dca28d62-322d-4e62-b463-d3fd114a3dfd' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-dca28d62-322d-4e62-b463-d3fd114a3dfd' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-e1a4d75b-b2e7-41fc-8431-11baaba7796d' class='xr-var-data-in' type='checkbox'><label for='data-e1a4d75b-b2e7-41fc-8431-11baaba7796d' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[[nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        ...,
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan]],

       [[nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        ...,
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan]],

       [[nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        ...,
...
        ...,
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan]],

       [[nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        ...,
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan]],

       [[nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        ...,
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan],
        [nan, nan, nan, ..., nan, nan, nan]]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-34721e85-f433-45e2-89d0-6ae5467ec66c' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-34721e85-f433-45e2-89d0-6ae5467ec66c' class='xr-section-summary'  title='Expand/collapse section'>Attributes: <span>(0)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'></dl></div></li></ul></div></div>



```python
#Split typical year into seasonal bins (grid X 2 seasons)
day_bins = [0,106,289,366] #Limits of the 2 seasons (day number over the year)
typical_Seasonal_year = typical_year.groupby_bins('dayofyear',day_bins,labels=['cold1','hot','cold2']).median('dayofyear') #Split the year into 3 bins
Cold = typical_Seasonal_year.sel(dayofyear_bins = ['cold2','cold1']).mean('dayofyear_bins') #Average of the 2 "Cold" season bins
Hot = typical_Seasonal_year.sel(dayofyear_bins = ['hot']).mean('dayofyear_bins')
```


<div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:      (latitude: 101, longitude: 232)
Coordinates:
  * longitude    (longitude) float32 -40.25 -39.75 -39.25 ... 74.25 74.75 75.25
  * latitude     (latitude) float32 25.25 25.75 26.25 ... 74.25 74.75 75.25
    spatial_ref  int32 0
Data variables:
    tg           (latitude, longitude) float32 nan nan nan nan ... nan nan nan</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-479d518b-6d86-4057-93ef-1b2220379032' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-479d518b-6d86-4057-93ef-1b2220379032' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>latitude</span>: 101</li><li><span class='xr-has-index'>longitude</span>: 232</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-0f83aff1-6d16-4563-a66e-e26f8405e983' class='xr-section-summary-in' type='checkbox'  checked><label for='section-0f83aff1-6d16-4563-a66e-e26f8405e983' class='xr-section-summary' >Coordinates: <span>(3)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>longitude</span></div><div class='xr-var-dims'>(longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>-40.25 -39.75 ... 74.75 75.25</div><input id='attrs-57d7dfbf-30cb-4236-8a5e-5572fa678150' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-57d7dfbf-30cb-4236-8a5e-5572fa678150' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-7786b1ab-88df-4491-801c-c160f361a891' class='xr-var-data-in' type='checkbox'><label for='data-7786b1ab-88df-4491-801c-c160f361a891' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Longitude values</dd><dt><span>units :</span></dt><dd>degrees_east</dd><dt><span>standard_name :</span></dt><dd>longitude</dd></dl></div><div class='xr-var-data'><pre>array([-40.25, -39.75, -39.25, ...,  74.25,  74.75,  75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>latitude</span></div><div class='xr-var-dims'>(latitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>25.25 25.75 26.25 ... 74.75 75.25</div><input id='attrs-5ad6d1c3-0142-48ce-9f76-393b9959594e' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-5ad6d1c3-0142-48ce-9f76-393b9959594e' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-13979534-5dbd-4f88-b8e1-9536b3603521' class='xr-var-data-in' type='checkbox'><label for='data-13979534-5dbd-4f88-b8e1-9536b3603521' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Latitude values</dd><dt><span>units :</span></dt><dd>degrees_north</dd><dt><span>standard_name :</span></dt><dd>latitude</dd></dl></div><div class='xr-var-data'><pre>array([25.25, 25.75, 26.25, 26.75, 27.25, 27.75, 28.25, 28.75, 29.25, 29.75,
       30.25, 30.75, 31.25, 31.75, 32.25, 32.75, 33.25, 33.75, 34.25, 34.75,
       35.25, 35.75, 36.25, 36.75, 37.25, 37.75, 38.25, 38.75, 39.25, 39.75,
       40.25, 40.75, 41.25, 41.75, 42.25, 42.75, 43.25, 43.75, 44.25, 44.75,
       45.25, 45.75, 46.25, 46.75, 47.25, 47.75, 48.25, 48.75, 49.25, 49.75,
       50.25, 50.75, 51.25, 51.75, 52.25, 52.75, 53.25, 53.75, 54.25, 54.75,
       55.25, 55.75, 56.25, 56.75, 57.25, 57.75, 58.25, 58.75, 59.25, 59.75,
       60.25, 60.75, 61.25, 61.75, 62.25, 62.75, 63.25, 63.75, 64.25, 64.75,
       65.25, 65.75, 66.25, 66.75, 67.25, 67.75, 68.25, 68.75, 69.25, 69.75,
       70.25, 70.75, 71.25, 71.75, 72.25, 72.75, 73.25, 73.75, 74.25, 74.75,
       75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-4449e413-3322-46d0-81e0-7dd1ce856a02' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-4449e413-3322-46d0-81e0-7dd1ce856a02' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-7911485d-8464-4247-8d3b-8aabda523a44' class='xr-var-data-in' type='checkbox'><label for='data-7911485d-8464-4247-8d3b-8aabda523a44' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd><dt><span>semi_major_axis :</span></dt><dd>6378137.0</dd><dt><span>semi_minor_axis :</span></dt><dd>6356752.314245179</dd><dt><span>inverse_flattening :</span></dt><dd>298.257223563</dd><dt><span>reference_ellipsoid_name :</span></dt><dd>WGS 84</dd><dt><span>longitude_of_prime_meridian :</span></dt><dd>0.0</dd><dt><span>prime_meridian_name :</span></dt><dd>Greenwich</dd><dt><span>geographic_crs_name :</span></dt><dd>WGS 84</dd><dt><span>grid_mapping_name :</span></dt><dd>latitude_longitude</dd><dt><span>spatial_ref :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-610f450d-d7db-44d5-a2fa-14d22f348c26' class='xr-section-summary-in' type='checkbox'  checked><label for='section-610f450d-d7db-44d5-a2fa-14d22f348c26' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(latitude, longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-d18aeb51-5460-4408-ad06-b129400c8384' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-d18aeb51-5460-4408-ad06-b129400c8384' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-7948a652-182e-4298-aa83-34feeb2a2bb7' class='xr-var-data-in' type='checkbox'><label for='data-7948a652-182e-4298-aa83-34feeb2a2bb7' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       ...,
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-63f5ccdc-5589-47cb-8eea-3764921ce2dd' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-63f5ccdc-5589-47cb-8eea-3764921ce2dd' class='xr-section-summary'  title='Expand/collapse section'>Attributes: <span>(0)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'></dl></div></li></ul></div></div>



<div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:      (latitude: 101, longitude: 232)
Coordinates:
  * longitude    (longitude) float32 -40.25 -39.75 -39.25 ... 74.25 74.75 75.25
  * latitude     (latitude) float32 25.25 25.75 26.25 ... 74.25 74.75 75.25
    spatial_ref  int32 0
Data variables:
    tg           (latitude, longitude) float32 nan nan nan nan ... nan nan nan</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-5975261d-66fb-4533-bec0-71d98695fbe2' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-5975261d-66fb-4533-bec0-71d98695fbe2' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>latitude</span>: 101</li><li><span class='xr-has-index'>longitude</span>: 232</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-f5968774-9255-4dd9-8252-711a98f7274a' class='xr-section-summary-in' type='checkbox'  checked><label for='section-f5968774-9255-4dd9-8252-711a98f7274a' class='xr-section-summary' >Coordinates: <span>(3)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>longitude</span></div><div class='xr-var-dims'>(longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>-40.25 -39.75 ... 74.75 75.25</div><input id='attrs-f40b084b-c6f3-4e1b-8934-be3589f68936' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-f40b084b-c6f3-4e1b-8934-be3589f68936' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-53bcc6bb-2ca2-490b-b943-994e20309c6a' class='xr-var-data-in' type='checkbox'><label for='data-53bcc6bb-2ca2-490b-b943-994e20309c6a' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Longitude values</dd><dt><span>units :</span></dt><dd>degrees_east</dd><dt><span>standard_name :</span></dt><dd>longitude</dd></dl></div><div class='xr-var-data'><pre>array([-40.25, -39.75, -39.25, ...,  74.25,  74.75,  75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>latitude</span></div><div class='xr-var-dims'>(latitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>25.25 25.75 26.25 ... 74.75 75.25</div><input id='attrs-e0bf73e0-68fb-4d6c-ac44-d0401a6f6120' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-e0bf73e0-68fb-4d6c-ac44-d0401a6f6120' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-bfedeaf4-33dc-469a-8e33-21fd35ae9093' class='xr-var-data-in' type='checkbox'><label for='data-bfedeaf4-33dc-469a-8e33-21fd35ae9093' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Latitude values</dd><dt><span>units :</span></dt><dd>degrees_north</dd><dt><span>standard_name :</span></dt><dd>latitude</dd></dl></div><div class='xr-var-data'><pre>array([25.25, 25.75, 26.25, 26.75, 27.25, 27.75, 28.25, 28.75, 29.25, 29.75,
       30.25, 30.75, 31.25, 31.75, 32.25, 32.75, 33.25, 33.75, 34.25, 34.75,
       35.25, 35.75, 36.25, 36.75, 37.25, 37.75, 38.25, 38.75, 39.25, 39.75,
       40.25, 40.75, 41.25, 41.75, 42.25, 42.75, 43.25, 43.75, 44.25, 44.75,
       45.25, 45.75, 46.25, 46.75, 47.25, 47.75, 48.25, 48.75, 49.25, 49.75,
       50.25, 50.75, 51.25, 51.75, 52.25, 52.75, 53.25, 53.75, 54.25, 54.75,
       55.25, 55.75, 56.25, 56.75, 57.25, 57.75, 58.25, 58.75, 59.25, 59.75,
       60.25, 60.75, 61.25, 61.75, 62.25, 62.75, 63.25, 63.75, 64.25, 64.75,
       65.25, 65.75, 66.25, 66.75, 67.25, 67.75, 68.25, 68.75, 69.25, 69.75,
       70.25, 70.75, 71.25, 71.75, 72.25, 72.75, 73.25, 73.75, 74.25, 74.75,
       75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-2f46118c-d7b9-4024-a718-34c4d9495ca1' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-2f46118c-d7b9-4024-a718-34c4d9495ca1' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-878c4ca6-c1d6-42d7-a205-7c9af3f667b7' class='xr-var-data-in' type='checkbox'><label for='data-878c4ca6-c1d6-42d7-a205-7c9af3f667b7' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd><dt><span>semi_major_axis :</span></dt><dd>6378137.0</dd><dt><span>semi_minor_axis :</span></dt><dd>6356752.314245179</dd><dt><span>inverse_flattening :</span></dt><dd>298.257223563</dd><dt><span>reference_ellipsoid_name :</span></dt><dd>WGS 84</dd><dt><span>longitude_of_prime_meridian :</span></dt><dd>0.0</dd><dt><span>prime_meridian_name :</span></dt><dd>Greenwich</dd><dt><span>geographic_crs_name :</span></dt><dd>WGS 84</dd><dt><span>grid_mapping_name :</span></dt><dd>latitude_longitude</dd><dt><span>spatial_ref :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-8e686309-c3b2-4b4b-b611-58799425ad78' class='xr-section-summary-in' type='checkbox'  checked><label for='section-8e686309-c3b2-4b4b-b611-58799425ad78' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(latitude, longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-d4f319db-6912-4377-8e05-3a74eb11a8f1' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-d4f319db-6912-4377-8e05-3a74eb11a8f1' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-ebeb195f-a1d6-4d14-b3f6-43ddff6abc45' class='xr-var-data-in' type='checkbox'><label for='data-ebeb195f-a1d6-4d14-b3f6-43ddff6abc45' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       ...,
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-8687330f-080d-4f0d-baf8-fafed7ac6992' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-8687330f-080d-4f0d-baf8-fafed7ac6992' class='xr-section-summary'  title='Expand/collapse section'>Attributes: <span>(0)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'></dl></div></li></ul></div></div>



```python
display(Cold)
```


<div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:      (x: 88, y: 120)
Coordinates:
  * y            (y) float64 1.103e+07 1.097e+07 1.09e+07 ... 3.336e+06 3.27e+06
  * x            (x) float64 -1.998e+06 -1.932e+06 ... 3.611e+06 3.676e+06
    spatial_ref  int32 0
Data variables:
    tg           (y, x) float32 nan nan nan nan nan nan ... nan nan nan nan nan
Attributes:
    grid_mapping:  spatial_ref</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-34974993-64fe-4b15-8a40-7ed8c6845913' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-34974993-64fe-4b15-8a40-7ed8c6845913' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>x</span>: 88</li><li><span class='xr-has-index'>y</span>: 120</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-c04609ec-b5da-488d-991e-d513fd633e42' class='xr-section-summary-in' type='checkbox'  checked><label for='section-c04609ec-b5da-488d-991e-d513fd633e42' class='xr-section-summary' >Coordinates: <span>(3)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>y</span></div><div class='xr-var-dims'>(y)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>1.103e+07 1.097e+07 ... 3.27e+06</div><input id='attrs-3632e6ad-5fea-4baa-af14-1c22c9dde58a' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-3632e6ad-5fea-4baa-af14-1c22c9dde58a' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-860b5d63-dd52-4f10-97c1-79305c9e78d8' class='xr-var-data-in' type='checkbox'><label for='data-860b5d63-dd52-4f10-97c1-79305c9e78d8' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>axis :</span></dt><dd>Y</dd><dt><span>long_name :</span></dt><dd>y coordinate of projection</dd><dt><span>standard_name :</span></dt><dd>projection_y_coordinate</dd><dt><span>units :</span></dt><dd>metre</dd></dl></div><div class='xr-var-data'><pre>array([11031284.311445, 10966066.025924, 10900847.740402, 10835629.454881,
       10770411.16936 , 10705192.883839, 10639974.598317, 10574756.312796,
       10509538.027275, 10444319.741754, 10379101.456232, 10313883.170711,
       10248664.88519 , 10183446.599669, 10118228.314147, 10053010.028626,
        9987791.743105,  9922573.457584,  9857355.172063,  9792136.886541,
        9726918.60102 ,  9661700.315499,  9596482.029978,  9531263.744456,
        9466045.458935,  9400827.173414,  9335608.887893,  9270390.602371,
        9205172.31685 ,  9139954.031329,  9074735.745808,  9009517.460286,
        8944299.174765,  8879080.889244,  8813862.603723,  8748644.318201,
        8683426.03268 ,  8618207.747159,  8552989.461638,  8487771.176117,
        8422552.890595,  8357334.605074,  8292116.319553,  8226898.034032,
        8161679.74851 ,  8096461.462989,  8031243.177468,  7966024.891947,
        7900806.606425,  7835588.320904,  7770370.035383,  7705151.749862,
        7639933.46434 ,  7574715.178819,  7509496.893298,  7444278.607777,
        7379060.322256,  7313842.036734,  7248623.751213,  7183405.465692,
        7118187.180171,  7052968.894649,  6987750.609128,  6922532.323607,
        6857314.038086,  6792095.752564,  6726877.467043,  6661659.181522,
        6596440.896001,  6531222.610479,  6466004.324958,  6400786.039437,
        6335567.753916,  6270349.468394,  6205131.182873,  6139912.897352,
        6074694.611831,  6009476.32631 ,  5944258.040788,  5879039.755267,
        5813821.469746,  5748603.184225,  5683384.898703,  5618166.613182,
        5552948.327661,  5487730.04214 ,  5422511.756618,  5357293.471097,
        5292075.185576,  5226856.900055,  5161638.614533,  5096420.329012,
        5031202.043491,  4965983.75797 ,  4900765.472449,  4835547.186927,
        4770328.901406,  4705110.615885,  4639892.330364,  4574674.044842,
        4509455.759321,  4444237.4738  ,  4379019.188279,  4313800.902757,
        4248582.617236,  4183364.331715,  4118146.046194,  4052927.760672,
        3987709.475151,  3922491.18963 ,  3857272.904109,  3792054.618588,
        3726836.333066,  3661618.047545,  3596399.762024,  3531181.476503,
        3465963.190981,  3400744.90546 ,  3335526.619939,  3270308.334418])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>x</span></div><div class='xr-var-dims'>(x)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>-1.998e+06 -1.932e+06 ... 3.676e+06</div><input id='attrs-5bdf670a-37a4-4b32-822f-28e009e31ff7' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-5bdf670a-37a4-4b32-822f-28e009e31ff7' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-adf5f4e7-bb5e-432a-a9be-705fb0200db3' class='xr-var-data-in' type='checkbox'><label for='data-adf5f4e7-bb5e-432a-a9be-705fb0200db3' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>axis :</span></dt><dd>X</dd><dt><span>long_name :</span></dt><dd>x coordinate of projection</dd><dt><span>standard_name :</span></dt><dd>projection_x_coordinate</dd><dt><span>units :</span></dt><dd>metre</dd></dl></div><div class='xr-var-data'><pre>array([-1997535.38456 , -1932317.099039, -1867098.813517, -1801880.527996,
       -1736662.242475, -1671443.956954, -1606225.671433, -1541007.385911,
       -1475789.10039 , -1410570.814869, -1345352.529348, -1280134.243826,
       -1214915.958305, -1149697.672784, -1084479.387263, -1019261.101741,
        -954042.81622 ,  -888824.530699,  -823606.245178,  -758387.959656,
        -693169.674135,  -627951.388614,  -562733.103093,  -497514.817572,
        -432296.53205 ,  -367078.246529,  -301859.961008,  -236641.675487,
        -171423.389965,  -106205.104444,   -40986.818923,    24231.466598,
          89449.75212 ,   154668.037641,   219886.323162,   285104.608683,
         350322.894205,   415541.179726,   480759.465247,   545977.750768,
         611196.036289,   676414.321811,   741632.607332,   806850.892853,
         872069.178374,   937287.463896,  1002505.749417,  1067724.034938,
        1132942.320459,  1198160.605981,  1263378.891502,  1328597.177023,
        1393815.462544,  1459033.748066,  1524252.033587,  1589470.319108,
        1654688.604629,  1719906.890151,  1785125.175672,  1850343.461193,
        1915561.746714,  1980780.032235,  2045998.317757,  2111216.603278,
        2176434.888799,  2241653.17432 ,  2306871.459842,  2372089.745363,
        2437308.030884,  2502526.316405,  2567744.601927,  2632962.887448,
        2698181.172969,  2763399.45849 ,  2828617.744012,  2893836.029533,
        2959054.315054,  3024272.600575,  3089490.886096,  3154709.171618,
        3219927.457139,  3285145.74266 ,  3350364.028181,  3415582.313703,
        3480800.599224,  3546018.884745,  3611237.170266,  3676455.455788])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-41e71f06-85da-4bdb-bd46-6d518b1ca178' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-41e71f06-85da-4bdb-bd46-6d518b1ca178' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-daec262c-8af8-4f00-8e07-101b0ed08abc' class='xr-var-data-in' type='checkbox'><label for='data-daec262c-8af8-4f00-8e07-101b0ed08abc' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>PROJCS[&quot;WGS 84 / Pseudo-Mercator&quot;,GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]],PROJECTION[&quot;Mercator_1SP&quot;],PARAMETER[&quot;central_meridian&quot;,0],PARAMETER[&quot;scale_factor&quot;,1],PARAMETER[&quot;false_easting&quot;,0],PARAMETER[&quot;false_northing&quot;,0],UNIT[&quot;metre&quot;,1,AUTHORITY[&quot;EPSG&quot;,&quot;9001&quot;]],AXIS[&quot;Easting&quot;,EAST],AXIS[&quot;Northing&quot;,NORTH],EXTENSION[&quot;PROJ4&quot;,&quot;+proj=merc +a=6378137 +b=6378137 +lat_ts=0 +lon_0=0 +x_0=0 +y_0=0 +k=1 +units=m +nadgrids=@null +wktext +no_defs&quot;],AUTHORITY[&quot;EPSG&quot;,&quot;3857&quot;]]</dd><dt><span>spatial_ref :</span></dt><dd>PROJCS[&quot;WGS 84 / Pseudo-Mercator&quot;,GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]],PROJECTION[&quot;Mercator_1SP&quot;],PARAMETER[&quot;central_meridian&quot;,0],PARAMETER[&quot;scale_factor&quot;,1],PARAMETER[&quot;false_easting&quot;,0],PARAMETER[&quot;false_northing&quot;,0],UNIT[&quot;metre&quot;,1,AUTHORITY[&quot;EPSG&quot;,&quot;9001&quot;]],AXIS[&quot;Easting&quot;,EAST],AXIS[&quot;Northing&quot;,NORTH],EXTENSION[&quot;PROJ4&quot;,&quot;+proj=merc +a=6378137 +b=6378137 +lat_ts=0 +lon_0=0 +x_0=0 +y_0=0 +k=1 +units=m +nadgrids=@null +wktext +no_defs&quot;],AUTHORITY[&quot;EPSG&quot;,&quot;3857&quot;]]</dd><dt><span>GeoTransform :</span></dt><dd>-2030144.52732059 65218.28552123656 0.0 11063893.454205379 0.0 -65218.285521236554</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-1654b537-615f-4002-9c5a-4124138682c0' class='xr-section-summary-in' type='checkbox'  checked><label for='section-1654b537-615f-4002-9c5a-4124138682c0' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(y, x)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-efd04359-464b-4e29-b352-df4379fd185b' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-efd04359-464b-4e29-b352-df4379fd185b' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-ac33732b-678c-492e-89a9-ca76d2c95f97' class='xr-var-data-in' type='checkbox'><label for='data-ac33732b-678c-492e-89a9-ca76d2c95f97' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>grid_mapping :</span></dt><dd>spatial_ref</dd><dt><span>_FillValue :</span></dt><dd>-9999.0</dd></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       ...,
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-afac37be-7775-4916-9ab6-1bffe23ad92f' class='xr-section-summary-in' type='checkbox'  checked><label for='section-afac37be-7775-4916-9ab6-1bffe23ad92f' class='xr-section-summary' >Attributes: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'><dt><span>grid_mapping :</span></dt><dd>spatial_ref</dd></dl></div></li></ul></div></div>



```python
display(Hot)
```


<div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:      (x: 88, y: 120)
Coordinates:
  * y            (y) float64 1.103e+07 1.097e+07 1.09e+07 ... 3.336e+06 3.27e+06
  * x            (x) float64 -1.998e+06 -1.932e+06 ... 3.611e+06 3.676e+06
    spatial_ref  int32 0
Data variables:
    tg           (y, x) float32 nan nan nan nan nan nan ... nan nan nan nan nan
Attributes:
    grid_mapping:  spatial_ref</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-8b9efcb2-5842-4393-9496-dfe369ab0dc3' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-8b9efcb2-5842-4393-9496-dfe369ab0dc3' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>x</span>: 88</li><li><span class='xr-has-index'>y</span>: 120</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-a996b537-7dd6-47f6-b82f-4f1698dc9a9d' class='xr-section-summary-in' type='checkbox'  checked><label for='section-a996b537-7dd6-47f6-b82f-4f1698dc9a9d' class='xr-section-summary' >Coordinates: <span>(3)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>y</span></div><div class='xr-var-dims'>(y)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>1.103e+07 1.097e+07 ... 3.27e+06</div><input id='attrs-f85ca785-8b82-4f06-a67e-6e7580b2eb21' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-f85ca785-8b82-4f06-a67e-6e7580b2eb21' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-194d1d00-96fd-451c-9a99-9aea26226d8a' class='xr-var-data-in' type='checkbox'><label for='data-194d1d00-96fd-451c-9a99-9aea26226d8a' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>axis :</span></dt><dd>Y</dd><dt><span>long_name :</span></dt><dd>y coordinate of projection</dd><dt><span>standard_name :</span></dt><dd>projection_y_coordinate</dd><dt><span>units :</span></dt><dd>metre</dd></dl></div><div class='xr-var-data'><pre>array([11031284.311445, 10966066.025924, 10900847.740402, 10835629.454881,
       10770411.16936 , 10705192.883839, 10639974.598317, 10574756.312796,
       10509538.027275, 10444319.741754, 10379101.456232, 10313883.170711,
       10248664.88519 , 10183446.599669, 10118228.314147, 10053010.028626,
        9987791.743105,  9922573.457584,  9857355.172063,  9792136.886541,
        9726918.60102 ,  9661700.315499,  9596482.029978,  9531263.744456,
        9466045.458935,  9400827.173414,  9335608.887893,  9270390.602371,
        9205172.31685 ,  9139954.031329,  9074735.745808,  9009517.460286,
        8944299.174765,  8879080.889244,  8813862.603723,  8748644.318201,
        8683426.03268 ,  8618207.747159,  8552989.461638,  8487771.176117,
        8422552.890595,  8357334.605074,  8292116.319553,  8226898.034032,
        8161679.74851 ,  8096461.462989,  8031243.177468,  7966024.891947,
        7900806.606425,  7835588.320904,  7770370.035383,  7705151.749862,
        7639933.46434 ,  7574715.178819,  7509496.893298,  7444278.607777,
        7379060.322256,  7313842.036734,  7248623.751213,  7183405.465692,
        7118187.180171,  7052968.894649,  6987750.609128,  6922532.323607,
        6857314.038086,  6792095.752564,  6726877.467043,  6661659.181522,
        6596440.896001,  6531222.610479,  6466004.324958,  6400786.039437,
        6335567.753916,  6270349.468394,  6205131.182873,  6139912.897352,
        6074694.611831,  6009476.32631 ,  5944258.040788,  5879039.755267,
        5813821.469746,  5748603.184225,  5683384.898703,  5618166.613182,
        5552948.327661,  5487730.04214 ,  5422511.756618,  5357293.471097,
        5292075.185576,  5226856.900055,  5161638.614533,  5096420.329012,
        5031202.043491,  4965983.75797 ,  4900765.472449,  4835547.186927,
        4770328.901406,  4705110.615885,  4639892.330364,  4574674.044842,
        4509455.759321,  4444237.4738  ,  4379019.188279,  4313800.902757,
        4248582.617236,  4183364.331715,  4118146.046194,  4052927.760672,
        3987709.475151,  3922491.18963 ,  3857272.904109,  3792054.618588,
        3726836.333066,  3661618.047545,  3596399.762024,  3531181.476503,
        3465963.190981,  3400744.90546 ,  3335526.619939,  3270308.334418])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>x</span></div><div class='xr-var-dims'>(x)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>-1.998e+06 -1.932e+06 ... 3.676e+06</div><input id='attrs-1226b4df-1ed7-4c26-a69f-eea6929da015' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-1226b4df-1ed7-4c26-a69f-eea6929da015' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-780d83f1-7996-4015-96f7-5d5e3db70e68' class='xr-var-data-in' type='checkbox'><label for='data-780d83f1-7996-4015-96f7-5d5e3db70e68' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>axis :</span></dt><dd>X</dd><dt><span>long_name :</span></dt><dd>x coordinate of projection</dd><dt><span>standard_name :</span></dt><dd>projection_x_coordinate</dd><dt><span>units :</span></dt><dd>metre</dd></dl></div><div class='xr-var-data'><pre>array([-1997535.38456 , -1932317.099039, -1867098.813517, -1801880.527996,
       -1736662.242475, -1671443.956954, -1606225.671433, -1541007.385911,
       -1475789.10039 , -1410570.814869, -1345352.529348, -1280134.243826,
       -1214915.958305, -1149697.672784, -1084479.387263, -1019261.101741,
        -954042.81622 ,  -888824.530699,  -823606.245178,  -758387.959656,
        -693169.674135,  -627951.388614,  -562733.103093,  -497514.817572,
        -432296.53205 ,  -367078.246529,  -301859.961008,  -236641.675487,
        -171423.389965,  -106205.104444,   -40986.818923,    24231.466598,
          89449.75212 ,   154668.037641,   219886.323162,   285104.608683,
         350322.894205,   415541.179726,   480759.465247,   545977.750768,
         611196.036289,   676414.321811,   741632.607332,   806850.892853,
         872069.178374,   937287.463896,  1002505.749417,  1067724.034938,
        1132942.320459,  1198160.605981,  1263378.891502,  1328597.177023,
        1393815.462544,  1459033.748066,  1524252.033587,  1589470.319108,
        1654688.604629,  1719906.890151,  1785125.175672,  1850343.461193,
        1915561.746714,  1980780.032235,  2045998.317757,  2111216.603278,
        2176434.888799,  2241653.17432 ,  2306871.459842,  2372089.745363,
        2437308.030884,  2502526.316405,  2567744.601927,  2632962.887448,
        2698181.172969,  2763399.45849 ,  2828617.744012,  2893836.029533,
        2959054.315054,  3024272.600575,  3089490.886096,  3154709.171618,
        3219927.457139,  3285145.74266 ,  3350364.028181,  3415582.313703,
        3480800.599224,  3546018.884745,  3611237.170266,  3676455.455788])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-edf96726-4141-476c-bb20-43d7323d2258' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-edf96726-4141-476c-bb20-43d7323d2258' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-1bd3203b-a2ab-4511-adb5-c3a5e804a0b3' class='xr-var-data-in' type='checkbox'><label for='data-1bd3203b-a2ab-4511-adb5-c3a5e804a0b3' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>PROJCS[&quot;WGS 84 / Pseudo-Mercator&quot;,GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]],PROJECTION[&quot;Mercator_1SP&quot;],PARAMETER[&quot;central_meridian&quot;,0],PARAMETER[&quot;scale_factor&quot;,1],PARAMETER[&quot;false_easting&quot;,0],PARAMETER[&quot;false_northing&quot;,0],UNIT[&quot;metre&quot;,1,AUTHORITY[&quot;EPSG&quot;,&quot;9001&quot;]],AXIS[&quot;Easting&quot;,EAST],AXIS[&quot;Northing&quot;,NORTH],EXTENSION[&quot;PROJ4&quot;,&quot;+proj=merc +a=6378137 +b=6378137 +lat_ts=0 +lon_0=0 +x_0=0 +y_0=0 +k=1 +units=m +nadgrids=@null +wktext +no_defs&quot;],AUTHORITY[&quot;EPSG&quot;,&quot;3857&quot;]]</dd><dt><span>spatial_ref :</span></dt><dd>PROJCS[&quot;WGS 84 / Pseudo-Mercator&quot;,GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]],PROJECTION[&quot;Mercator_1SP&quot;],PARAMETER[&quot;central_meridian&quot;,0],PARAMETER[&quot;scale_factor&quot;,1],PARAMETER[&quot;false_easting&quot;,0],PARAMETER[&quot;false_northing&quot;,0],UNIT[&quot;metre&quot;,1,AUTHORITY[&quot;EPSG&quot;,&quot;9001&quot;]],AXIS[&quot;Easting&quot;,EAST],AXIS[&quot;Northing&quot;,NORTH],EXTENSION[&quot;PROJ4&quot;,&quot;+proj=merc +a=6378137 +b=6378137 +lat_ts=0 +lon_0=0 +x_0=0 +y_0=0 +k=1 +units=m +nadgrids=@null +wktext +no_defs&quot;],AUTHORITY[&quot;EPSG&quot;,&quot;3857&quot;]]</dd><dt><span>GeoTransform :</span></dt><dd>-2030144.52732059 65218.28552123656 0.0 11063893.454205379 0.0 -65218.285521236554</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-ca22b045-553d-4fa3-b1de-55b297f81dce' class='xr-section-summary-in' type='checkbox'  checked><label for='section-ca22b045-553d-4fa3-b1de-55b297f81dce' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(y, x)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-e3860ffc-0312-4d1a-ba5b-d8ca9841a704' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-e3860ffc-0312-4d1a-ba5b-d8ca9841a704' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-c13173b8-ca55-4eda-8e06-64f17547d293' class='xr-var-data-in' type='checkbox'><label for='data-c13173b8-ca55-4eda-8e06-64f17547d293' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>grid_mapping :</span></dt><dd>spatial_ref</dd><dt><span>_FillValue :</span></dt><dd>-9999.0</dd></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       ...,
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-8a0ba6bb-6e14-40c7-af87-f95e35a47f00' class='xr-section-summary-in' type='checkbox'  checked><label for='section-8a0ba6bb-6e14-40c7-af87-f95e35a47f00' class='xr-section-summary' >Attributes: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'><dt><span>grid_mapping :</span></dt><dd>spatial_ref</dd></dl></div></li></ul></div></div>


#### mask europe


```python
Hot.rio.write_crs("epsg:4326", inplace=True)
Hot = Hot.rio.reproject(dst_crs='EPSG:3857')
Hot = Hot.rio.clip(eu)
Hot = Hot.where(Hot['tg'] != -9999.)  

Cold.rio.write_crs("epsg:4326", inplace=True)
Cold = Cold.rio.reproject(dst_crs='EPSG:3857')
Cold = Cold.rio.clip(eu)
Cold = Cold.where(Cold['tg'] != -9999.) 
```

#### raster to vector


```python
#Hot.rio.write_crs("epsg:4326", inplace=True)
#Hot = Hot.rio.reproject(dst_crs='EPSG:3857')

hot_bins = Hot.quantile([0,0.25,0.5,0.75,1]).tg.values
Hot_discr = np.digitize(Hot.tg.values, hot_bins)
Hot_discr = Hot_discr.astype(np.int32)
#Hot_discr[Hot_discr == 5] = np.nan

west, east = min(Hot.x.values), max(Hot.x.values)
south, north =  min(Hot.y.values), max(Hot.y.values)
width = len(Hot.x.values)
height = len(Hot.y.values)

res_x = np.diff(Hot.x.values).mean()
res_y = np.diff(Hot.y.values).mean()

transform = rasterio.transform.from_origin(west, north, res_x, -res_y)

data = []
mask = Hot_discr != 5
for shp, val in shapes(Hot_discr,mask=mask, transform=transform):
    record = {'bin': val, 'geometry' : shape(shp)}
    data.append(record)
    
Hot_df = pd.DataFrame.from_records(data)
Hot_df = gpd.GeoDataFrame(Hot_df,crs=3857)
Hot_df = Hot_df.dissolve(by='bin')
```


```python
plt.clf()
ax = plt.figure(figsize=(20,20)).add_subplot(111)
colors = ['blue','yellow','orange','red']
for poly, color in zip(Hot_df['geometry'],colors):
    ax.add_patch(PolygonPatch(poly, fc=color, ec='black', alpha=0.5))
ax.axis('scaled')
plt.locator_params(axis='y', nbins=30)
plt.locator_params(axis='x', nbins=30)
plt.draw()
plt.show()
```


    <Figure size 432x288 with 0 Axes>



    
![png](output_40_1.png)
    



```python
for index, row in Hot_df.iterrows():
    temp_poly = row.geometry.buffer(100*1000)
    temp_poly = temp_poly.intersection(eu)
    other_polys = Hot_df.loc[Hot_df.index != index].geometry.values.unary_union()
    to_remove = temp_poly.intersection(other_polys)
    temp_poly = temp_poly.difference(to_remove)

    # then we snap the geometry of row to the just created multipolygon
    # the snaped geometry overwrites the exisiting geometry in input_gdf
    Hot_df.loc[index,['geometry']]= temp_poly

```


```python
plt.clf()
ax = plt.figure(figsize=(20,20)).add_subplot(111)
colors = ['blue','yellow','orange','red']
for poly, color in zip(Hot_df['geometry'],colors):
    ax.add_patch(PolygonPatch(poly, fc=color, ec='black', alpha=0.5))
ax.axis('scaled')
plt.locator_params(axis='y', nbins=30)
plt.locator_params(axis='x', nbins=30)
plt.draw()
plt.show()
```


    <Figure size 432x288 with 0 Axes>



    
![png](output_42_1.png)
    



```python
#Cold.rio.write_crs("epsg:4326", inplace=True)
#Cold = Cold.rio.reproject(dst_crs='EPSG:3857')

cold_bins = Cold.quantile([0,0.25,0.5,0.75,1]).tg.values
Cold_discr = np.digitize(Cold.tg.values, cold_bins)
Cold_discr = Cold_discr.astype(np.int32)
#Cold_discr[Cold_discr == 5] = np.nan

west, east = min(Cold.x.values), max(Cold.x.values)
south, north =  min(Cold.y.values), max(Cold.y.values)
width = len(Cold.x.values)
height = len(Cold.y.values)

res_x = np.diff(Cold.x.values).mean()
res_y = np.diff(Cold.y.values).mean()

transform = rasterio.transform.from_origin(west, north, res_x, -res_y)

data = []
mask = Cold_discr != 5
for shp, val in shapes(Cold_discr,mask=mask, transform=transform):
    record = {'bin': val, 'geometry' : shape(shp)}
    data.append(record)
    
Cold_df = pd.DataFrame.from_records(data)
Cold_df = gpd.GeoDataFrame(Cold_df,crs=3857)
Cold_df = Cold_df.dissolve(by='bin')
```


```python
for index, row in Cold_df.iterrows():
    temp_poly = row.geometry.buffer(100*1000)
    temp_poly = temp_poly.intersection(eu)
    other_polys = Cold_df.loc[Cold_df.index != index].geometry.values.unary_union()
    to_remove = temp_poly.intersection(other_polys)
    temp_poly = temp_poly.difference(to_remove)

    # then we snap the geometry of row to the just created multipolygon
    # the snaped geometry overwrites the exisiting geometry in input_gdf
    Cold_df.loc[index,['geometry']]= temp_poly

```


```python
plt.clf()
ax = plt.figure(figsize=(20,20)).add_subplot(111)
colors = ['blue','yellow','orange','red']
for poly, color in zip(Cold_df['geometry'],colors):
    ax.add_patch(PolygonPatch(poly, fc=color, ec='black', alpha=0.5))
ax.axis('scaled')
plt.locator_params(axis='y', nbins=30)
plt.locator_params(axis='x', nbins=30)
plt.draw()
plt.show()
```


    <Figure size 432x288 with 0 Axes>



    
![png](output_45_1.png)
    


# Merge Sources


```python
all_data = pd.concat([renew_df,energy_df,gas_df,elc_df,typo_df,stock_df],join='inner',axis=1).replace(': ',np.nan)
```


```python
#eu_df.set_index('Code',inplace=True)
truc = pd.concat([all_data,eu_df[['geometry']]],join='inner',axis=1)
display(truc)
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
      <th>renew share in %</th>
      <th>energy cons (kg of oil equivalent per capita)</th>
      <th>residential gas price in euro</th>
      <th>indus gas price in euro</th>
      <th>residential elec price in euro</th>
      <th>indus elec price in euro</th>
      <th>single family U value</th>
      <th>multiple family U value</th>
      <th>single family area (m2)</th>
      <th>mutiple family area (m2)</th>
      <th>Residential share in %</th>
      <th>Non-residential share in %</th>
      <th>single family share in %</th>
      <th>mutiple family share in %</th>
      <th>geometry</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>AT</th>
      <td>33.626</td>
      <td>6695.287</td>
      <td>18.3400</td>
      <td>7.3744</td>
      <td>0.1950</td>
      <td>0.0621</td>
      <td>0.978513</td>
      <td>0.882793</td>
      <td>145</td>
      <td>418.0</td>
      <td>61.6</td>
      <td>38.4</td>
      <td>47.5</td>
      <td>52.5</td>
      <td>POLYGON ((1687803.014 6264215.777, 1696293.843...</td>
    </tr>
    <tr>
      <th>BE</th>
      <td>9.924</td>
      <td>7899.465</td>
      <td>15.3755</td>
      <td>6.2508</td>
      <td>0.2857</td>
      <td>0.0816</td>
      <td>1.019217</td>
      <td>1.121330</td>
      <td>220</td>
      <td>1613.0</td>
      <td>67.5</td>
      <td>32.5</td>
      <td>72.9</td>
      <td>27.1</td>
      <td>POLYGON ((536053.133 6697902.835, 536858.496 6...</td>
    </tr>
    <tr>
      <th>BG</th>
      <td>21.564</td>
      <td>2159.861</td>
      <td>12.4631</td>
      <td>8.2636</td>
      <td>0.0955</td>
      <td>0.0753</td>
      <td>1.182875</td>
      <td>0.989048</td>
      <td>172</td>
      <td>495.0</td>
      <td>72.1</td>
      <td>27.9</td>
      <td>54.9</td>
      <td>45.1</td>
      <td>POLYGON ((2551393.950 5439823.819, 2575025.606...</td>
    </tr>
    <tr>
      <th>CZ</th>
      <td>16.244</td>
      <td>7006.728</td>
      <td>16.2682</td>
      <td>7.7802</td>
      <td>0.1438</td>
      <td>0.0677</td>
      <td>0.813539</td>
      <td>0.967243</td>
      <td>112</td>
      <td>681.0</td>
      <td>64.9</td>
      <td>35.1</td>
      <td>44.3</td>
      <td>55.7</td>
      <td>POLYGON ((1602756.662 6623613.894, 1605874.568...</td>
    </tr>
    <tr>
      <th>DE</th>
      <td>17.354</td>
      <td>57743.118</td>
      <td>17.5503</td>
      <td>7.7126</td>
      <td>0.3048</td>
      <td>0.0761</td>
      <td>0.722681</td>
      <td>0.861979</td>
      <td>173</td>
      <td>1239.0</td>
      <td>68.4</td>
      <td>31.6</td>
      <td>45.2</td>
      <td>54.8</td>
      <td>MULTIPOLYGON (((750538.061 7090702.816, 751353...</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>IT</th>
      <td>18.181</td>
      <td>31612.006</td>
      <td>21.3645</td>
      <td>8.2732</td>
      <td>0.2132</td>
      <td>0.0829</td>
      <td>1.295267</td>
      <td>1.109430</td>
      <td>154</td>
      <td>884.0</td>
      <td>89.0</td>
      <td>11.0</td>
      <td>26.0</td>
      <td>74.0</td>
      <td>MULTIPOLYGON (((1404993.029 4230990.607, 14038...</td>
    </tr>
    <tr>
      <th>NL</th>
      <td>8.768</td>
      <td>9307.869</td>
      <td>25.5701</td>
      <td>6.1856</td>
      <td>0.1562</td>
      <td>0.0607</td>
      <td>1.075242</td>
      <td>1.544523</td>
      <td>163</td>
      <td>2618.0</td>
      <td>60.7</td>
      <td>39.3</td>
      <td>70.8</td>
      <td>29.2</td>
      <td>MULTIPOLYGON (((-7596140.830 1348758.669, -759...</td>
    </tr>
    <tr>
      <th>PL</th>
      <td>12.164</td>
      <td>18196.434</td>
      <td>13.1310</td>
      <td>9.3669</td>
      <td>0.1446</td>
      <td>0.0673</td>
      <td>0.699272</td>
      <td>0.729197</td>
      <td>136</td>
      <td>2186.0</td>
      <td>67.0</td>
      <td>33.0</td>
      <td>33.1</td>
      <td>66.9</td>
      <td>POLYGON ((2115307.046 7235751.798, 2157060.914...</td>
    </tr>
    <tr>
      <th>SE</th>
      <td>56.391</td>
      <td>7363.914</td>
      <td>32.8557</td>
      <td>8.7466</td>
      <td>0.1936</td>
      <td>0.0643</td>
      <td>0.255006</td>
      <td>NaN</td>
      <td>121</td>
      <td>1207.0</td>
      <td>66.6</td>
      <td>33.4</td>
      <td>44.1</td>
      <td>55.9</td>
      <td>MULTIPOLYGON (((1744966.813 7582362.064, 17458...</td>
    </tr>
    <tr>
      <th>SI</th>
      <td>21.974</td>
      <td>1057.443</td>
      <td>15.8903</td>
      <td>8.5024</td>
      <td>0.1609</td>
      <td>0.0619</td>
      <td>0.788727</td>
      <td>1.110169</td>
      <td>201</td>
      <td>1362.0</td>
      <td>81.7</td>
      <td>18.3</td>
      <td>60.7</td>
      <td>39.3</td>
      <td>POLYGON ((1819341.831 5895544.825, 1820883.526...</td>
    </tr>
  </tbody>
</table>
<p>15 rows × 15 columns</p>
</div>



```python
# clean numeric values
all_data.loc[:,all_data.dtypes == object] = all_data.loc[:,all_data.dtypes == object].apply(lambda col: col.str.extract('(\d+.\d+)', expand=False))
all_data.loc[:,all_data.dtypes == object] = all_data.loc[:,all_data.dtypes == object].astype(float)
```


```python
all_data_dis = all_data.apply(lambda col: pd.qcut(col, 4, labels=False, duplicates = 'drop'))
all_data_dis = all_data_dis+1
```


```python
eu_df.set_index('Code',inplace=True)
all_data_dis = pd.concat([all_data_dis,eu_df[['geometry']]],join='inner',axis=1)
all_data_dis = gpd.GeoDataFrame(all_data_dis)
```


```python
all_data_dis = gpd.GeoDataFrame(all_data_dis)
```


```python
discretization_cols = all_data.columns
gs_data = gpd.GeoDataFrame()
gs_data[discretization_cols] = all_data_dis[discretization_cols].apply(lambda col: all_data_dis.dissolve(by= col)['geometry'])
gs_data #table with polygon of the 4 quartiles of each feature
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
      <th>renew share in %</th>
      <th>energy cons (kg of oil equivalent per capita)</th>
      <th>residential gas price in euro</th>
      <th>indus gas price in euro</th>
      <th>residential elec price in euro</th>
      <th>indus elec price in euro</th>
      <th>single family U value</th>
      <th>multiple family U value</th>
      <th>single family area (m2)</th>
      <th>mutiple family area (m2)</th>
      <th>Residential share in %</th>
      <th>Non-residential share in %</th>
      <th>single family share in %</th>
      <th>mutiple family share in %</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1.0</th>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
      <td>MULTIPOLYGON (((532003.307 6697072.451, 530749...</td>
      <td>MULTIPOLYGON (((-7601494.819 1355961.122, -760...</td>
      <td>MULTIPOLYGON (((2089863.323 6360968.338, 20852...</td>
      <td>MULTIPOLYGON (((-7601494.819 1355961.122, -760...</td>
      <td>MULTIPOLYGON (((747620.997 7090275.595, 745972...</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-6860026.202 1789816.725, -685...</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-1997145.494 3212924.432, -199...</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
    </tr>
    <tr>
      <th>2.0</th>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((1304598.938 7965673.626, 13035...</td>
      <td>MULTIPOLYGON (((748744.339 7090473.944, 747620...</td>
      <td>MULTIPOLYGON (((-1996529.468 3213170.029, -199...</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
      <td>MULTIPOLYGON (((1304598.938 7965673.626, 13035...</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
      <td>MULTIPOLYGON (((749695.555 7090107.764, 748744...</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
      <td>MULTIPOLYGON (((-1996529.468 3213170.029, -199...</td>
      <td>MULTIPOLYGON (((-1149113.468 6782653.514, -115...</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
      <td>MULTIPOLYGON (((748744.339 7090473.944, 747620...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
    </tr>
    <tr>
      <th>3.0</th>
      <td>MULTIPOLYGON (((-1996529.468 3213170.029, -199...</td>
      <td>MULTIPOLYGON (((-7601494.819 1355961.122, -760...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-1996529.468 3213170.029, -199...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-6860026.202 1789816.725, -685...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
      <td>MULTIPOLYGON (((747620.997 7090275.595, 745972...</td>
      <td>MULTIPOLYGON (((942103.975 7442969.926, 940672...</td>
      <td>MULTIPOLYGON (((530116.456 6694743.216, 532475...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((749695.555 7090107.764, 748744...</td>
    </tr>
    <tr>
      <th>4.0</th>
      <td>MULTIPOLYGON (((940971.574 7436628.407, 941814...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-7601209.454 1361599.214, -760...</td>
      <td>MULTIPOLYGON (((-1149113.468 6782653.514, -115...</td>
      <td>MULTIPOLYGON (((-1149113.468 6782653.514, -115...</td>
      <td>MULTIPOLYGON (((-1996529.468 3213170.029, -199...</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
      <td>MULTIPOLYGON (((-1996529.468 3213170.029, -199...</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
      <td>MULTIPOLYGON (((-6859070.456 1788521.753, -685...</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
      <td>MULTIPOLYGON (((-1997145.494 3212924.432, -199...</td>
    </tr>
  </tbody>
</table>
</div>




```python
Hot_df.rename(columns={'geometry':'Hot_temp in C'},index={'bin':None},inplace=True)
Cold_df.rename(columns={'geometry':'Cold_temp in C'},index={'bin':None},inplace=True)

Final_geometry = pd.concat([gs_data,Cold_df,Hot_df],1)
```


```python
Final_quartiles = gpd.GeoDataFrame(columns=Final_geometry.columns, index = Final_geometry.index)
#Create table with quartiles' pairs (edges of each bin)
for col in all_data:
    bins = pd.qcut(all_data[col], 4).values.categories
    for i, _bin in enumerate(bins):
        Final_quartiles[col][i+1]= _bin
        
Final_quartiles['Cold_temp in C'] = pd.qcut(cold_bins,4).categories
Final_quartiles['Hot_temp in C'] = pd.qcut(hot_bins,4).categories
```

# Save all


```python
data = gpd.GeoDataFrame(pd.concat([Final_quartiles.stack(),Final_geometry.stack()],axis=1))
```


```python
data.columns = ['quartiles','geometry']
data.set_geometry('geometry',inplace=True)
#data.set_crs("EPSG:3857",inplace=True)
#data.to_crs("EPSG:4326",inplace=True)
data
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
      <th></th>
      <th>quartiles</th>
      <th>geometry</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="5" valign="top">1.0</th>
      <th>renew share in %</th>
      <td>(8.767000000000001, 12.389]</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
    </tr>
    <tr>
      <th>energy cons (kg of oil equivalent per capita)</th>
      <td>(1057.442, 5023.244]</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
    </tr>
    <tr>
      <th>residential gas price in euro</th>
      <td>(9.601, 15.633]</td>
      <td>MULTIPOLYGON (((532003.307 6697072.451, 530749...</td>
    </tr>
    <tr>
      <th>indus gas price in euro</th>
      <td>(6.185, 7.471]</td>
      <td>MULTIPOLYGON (((-7601494.819 1355961.122, -760...</td>
    </tr>
    <tr>
      <th>residential elec price in euro</th>
      <td>(0.0945, 0.15]</td>
      <td>MULTIPOLYGON (((2089863.323 6360968.338, 20852...</td>
    </tr>
    <tr>
      <th>...</th>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th rowspan="5" valign="top">4.0</th>
      <th>Non-residential share in %</th>
      <td>(33.2, 39.3]</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
    </tr>
    <tr>
      <th>single family share in %</th>
      <td>(61.1, 87.2]</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
    </tr>
    <tr>
      <th>mutiple family share in %</th>
      <td>(55.8, 74.0]</td>
      <td>MULTIPOLYGON (((-1997145.494 3212924.432, -199...</td>
    </tr>
    <tr>
      <th>Cold_temp in C</th>
      <td>(5.103, 13.653]</td>
      <td>MULTIPOLYGON (((-1160953.847 6780231.201, -115...</td>
    </tr>
    <tr>
      <th>Hot_temp in C</th>
      <td>(16.635, 24.56]</td>
      <td>MULTIPOLYGON (((-1010018.459 4695661.164, -101...</td>
    </tr>
  </tbody>
</table>
<p>64 rows × 2 columns</p>
</div>




```python
data['quartiles'] = data['quartiles'].apply(lambda x : str([x.left,x.right]))
```
data.to_file('data.geojson',driver='GeoJSON')
# Check Data


```python
def intersect_all_poly(l_poly):
    final = l_poly[0]
    for i in l_poly[1:]:
        final = i.intersection(final)
    return final 

#intersection = intersect_all_poly(filtered.geometry.values)
```


```python
def clean_poly(intersection):
    filtered_poly = []
    for i in intersection:
        if i.type == 'Polygon':
            filtered_poly.append(i)
    return max(MultiPolygon(filtered_poly), key=lambda a: a.area)

#intersection = clean_poly(intersection)
```


```python
import folium
import ast
import ipywidgets as widgets
from ipywidgets import interact, interactive, fixed, interact_manual
import time
import geopy as gp

geolocator = gp.Nominatim(user_agent="geocluster")
WGS84 = pyproj.CRS('EPSG:4326')
WEB_PROJ = pyproj.CRS('EPSG:3857')
PROJECT = pyproj.Transformer.from_crs(WEB_PROJ, WGS84, always_xy=True).transform
PROJECT_REV = pyproj.Transformer.from_crs(WGS84, WEB_PROJ, always_xy=True).transform

def f(x):
    time.sleep(2)
    location = geolocator.geocode(x)
    center = [location.latitude,location.longitude]
    pt = sh_ops.transform(PROJECT_REV, Point(location.longitude,location.latitude))
    filtered = data[data.intersects(pt)]
    intersection = intersect_all_poly(filtered.geometry.values)
    intersection = clean_poly(intersection)
    intersection = sh_ops.transform(PROJECT, intersection)

    my_map = folium.Map(location=center,  tiles="cartodbpositron",zoom_start=5)

    html = filtered[['quartiles']].to_html()
    test = folium.Html(html, script=True)
    popup = folium.Popup(test, max_width=2650)
    layer = folium.GeoJson(intersection, style_function = lambda x: {'color': 'red'})
    popup.add_to(layer)
    layer.add_to(my_map)

    display(my_map)
    #display(filtered[['quartiles']])
    
interact_manual(f,x=widgets.Text(value='Paris', disabled=False)); 
```


    interactive(children=(Text(value='Paris', description='x'), Button(description='Run Interact', style=ButtonSty…


# Find existing poly

4**16 = 4294967296 possibilities. Hard if not impossible to compute


```python
l,b,r,u = Cold_df.Cold_temp.values.unary_union().bounds
b,l = math.floor(b), math.floor(l)
u,r = math.ceil(u), math.ceil(r)
```


```python
grid_pts = [Point((i[0], i[1])) for i in list(product(Hot.x.values,Hot.y.values))]
```


```python
grid_pts = gpd.GeoDataFrame(grid_pts,columns=['geometry'])
eu_poly = gpd.GeoDataFrame(eu,columns=['geometry'])
```


```python
%%time
pts = gpd.overlay(grid_pts, eu_poly, how='intersection')
```


```python
def intersect_all_poly(l_poly):
    final = l_poly[0]
    for i in l_poly:
        final = i.intersection(final)
    return final 

combined_poly = []
for point in pts:
    polygons = [polygon for polygon in list(Final_geometry_stack.values) if point.within(polygon)]
    if len(polygons)>0:
        intersec = intersect_all_poly(polygons)
        combined_poly.append(intersec)
```


```python
%%time
exist = []
for point in pts.geometry.values:
    ids = []
    for label, content in Final_geometry.iteritems():
        for index in content.index:
            if point.within(content[index]):
                ids.append((index,label))
                break
    if len(ids) > 15:            
        exist.append(ids)
```


```python
def intersect_all_poly(l_poly):
    final = l_poly[0]
    for i in l_poly:
        final = i.intersection(final)
    return final 

truc = np.array(exist)
truc = np.unique(truc,axis=0)

combined_poly = []
for i in truc:
    polygons = []
    #quartiles = []
    for j in i:
        polygons.append(Final_geometry.loc[float(j[0]),j[1]])
        #quartiles.append(Final_quartile.loc[float(j[0]),truc[1]])

    intersec = intersect_all_poly(polygons)
    combined_poly.append(unary_union(intersec))
```


```python
df = gpd.GeoDataFrame(gpd.GeoSeries(combined_poly),columns=['geometry'],crs='EPSG:3857')
```


```python
import folium
import ast

center = eu

df.to_crs('EPSG:4326',inplace=True)

my_map = folium.Map(location=[53, 9],  tiles="cartodbpositron",zoom_start=9)

for geom in df.geometry: 
    folium.GeoJson(geom, style_function = lambda x: {'color': 'red'}).add_to(my_map)

my_map
```

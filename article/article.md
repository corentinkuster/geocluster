# Geocluster

A few years back I had to deliver this task for a European project ([the THERMOSS project](https://cordis.europa.eu/project/id/723562/fr)). THERMOSS was a H2020 project that aimed at facilitating the adoption of District Heating and Cooling (DHC) networks. 

In this project, the first stage was to create  geoclusters for district heating and cooling catalogue. Those geoclusters would have similar characteristics (energy, climate and building typology related) to help stakeholders choose the most suited technologies to implement. The idea behind is that if a technology has proven being efficient within a certain geocluster, it is most likely that the solution can be reproducible in other location of this same geocluster.

At the time, I did not know python so much and used MATLAB for such task. If the result were satisfaying at the time, I now want to reproduce it using python and some well-known GIS packages.

So lets get to it...

## 1. the data

The table below gives the list of features that have been used to create the geoclusters. Those are believed to be the most influential parameters for heating and cooling loads.

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

<br>
For the building typology and stock, some manual work was needed in excel to produce a neat table with only the required information.
Thanks, eurostat provided some nice tables, ready to use. See the example below for renewable energy share.


```python
import requests
import pandas as pd
import io

url = "https://ec.europa.eu/eurostat/api/dissemination/sdmx/2.1/data/T2020_RD330/?format=CSV"
urlData = requests.get(url).content
renew_df = pd.read_csv(io.StringIO(urlData.decode('utf-8')))
renew_df.iloc[:,0] = renew_df.iloc[:,0].apply(lambda x: x.split(';')[-1])
renew_df.rename(columns={'freq;nrg_bal;unit;geo\TIME_PERIOD': 'CODE'},inplace=True)
renew_df
```

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
In addition to that, we will also be using EU countries geoJSON file found [here](https://ec.europa.eu/eurostat/web/gisco/geodata/reference-data/administrative-units-statistical-units/countries).

With all the country codes set to iso alpha 2 (as above), we can merge our data with our geojson. Geopandas will be useful there.

```python
import geopandas as gpd

all_data = pd.concat([renew_df,energy_df,gas_df,elc_df,typo_df,stock_dfeu_df[['geometry']]],join='inner',axis=1).replace(': ',np.nan) #eurostat set nan values as ':'
all_data_gdf  = gpd.GeoDataFrame(all_data)
```

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

## 2. Handling NetCDF

For NetCDF it is a bit more technical. Indeed, NetCdf are multidimensional arrays that allows us to store both temporal and spatial dimension of certain phenomenon (temperature in our case).

![NetCDF](https://geoserver.geo-solutions.it/educational/en/_images/md1.png)

It can also been seen as temporaly related raster layers. For that reason, we are ogoin to use python libraries such as ```xarray```, ```rasterio``` and ```rioxarray```.

First let's use xarray to load our data.

```python
import xarray as xr
import rioxarray
import rasterio

with xr.open_dataset('tg_0.50deg_reg_1995-2016_v14.0.nc') as file:
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



From there, we will create a typical year by grouping data by days and running a median, thus for each long/lat cell.

```python
#Create typical year from the 20 years'set with daily data point (grid X 366 days)
typical_year = dsCDF.groupby('time.dayofyear').median('time') 
typical_year
```


We are now going to slip our typical year into 2 seasons, namely cold and hot seasons so that for each cell we have the median temperature over those seasons.

```python
#Split typical year into seasonal bins (grid X 2 seasons)
day_bins = [0,106,289,366] #Limits of the 2 seasons (day number over the year)
typical_Seasonal_year = typical_year.groupby_bins('dayofyear',day_bins,labels=['cold1','hot','cold2']).median('dayofyear') #Split the year into 3 bins
Cold = typical_Seasonal_year.sel(dayofyear_bins = ['cold2','cold1']).mean('dayofyear_bins') #Average of the 2 "Cold" season bins
Hot = typical_Seasonal_year.sel(dayofyear_bins = ['hot'])
```

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
Dimensions:      (latitude: 101, longitude: 232)
Coordinates:
  * longitude    (longitude) float32 -40.25 -39.75 -39.25 ... 74.25 74.75 75.25
  * latitude     (latitude) float32 25.25 25.75 26.25 ... 74.25 74.75 75.25
    spatial_ref  int32 0
Data variables:
    tg           (latitude, longitude) float32 nan nan nan nan ... nan nan nan</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-b6659d99-8267-433e-ab7b-ea6f49126ed3' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-b6659d99-8267-433e-ab7b-ea6f49126ed3' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>latitude</span>: 101</li><li><span class='xr-has-index'>longitude</span>: 232</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-e14c621c-aa2f-47fd-a24b-cb961e85135c' class='xr-section-summary-in' type='checkbox'  checked><label for='section-e14c621c-aa2f-47fd-a24b-cb961e85135c' class='xr-section-summary' >Coordinates: <span>(3)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>longitude</span></div><div class='xr-var-dims'>(longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>-40.25 -39.75 ... 74.75 75.25</div><input id='attrs-aa4a591e-5efc-4b37-b4bc-898640be6284' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-aa4a591e-5efc-4b37-b4bc-898640be6284' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-8f05b905-4afb-43c9-b286-a85c9fd314ea' class='xr-var-data-in' type='checkbox'><label for='data-8f05b905-4afb-43c9-b286-a85c9fd314ea' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Longitude values</dd><dt><span>units :</span></dt><dd>degrees_east</dd><dt><span>standard_name :</span></dt><dd>longitude</dd></dl></div><div class='xr-var-data'><pre>array([-40.25, -39.75, -39.25, ...,  74.25,  74.75,  75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>latitude</span></div><div class='xr-var-dims'>(latitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>25.25 25.75 26.25 ... 74.75 75.25</div><input id='attrs-499607cd-3a6f-469e-953e-314a29af14bb' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-499607cd-3a6f-469e-953e-314a29af14bb' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-65427ad4-2040-485e-b21e-9366ba1de7a6' class='xr-var-data-in' type='checkbox'><label for='data-65427ad4-2040-485e-b21e-9366ba1de7a6' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Latitude values</dd><dt><span>units :</span></dt><dd>degrees_north</dd><dt><span>standard_name :</span></dt><dd>latitude</dd></dl></div><div class='xr-var-data'><pre>array([25.25, 25.75, 26.25, 26.75, 27.25, 27.75, 28.25, 28.75, 29.25, 29.75,
       30.25, 30.75, 31.25, 31.75, 32.25, 32.75, 33.25, 33.75, 34.25, 34.75,
       35.25, 35.75, 36.25, 36.75, 37.25, 37.75, 38.25, 38.75, 39.25, 39.75,
       40.25, 40.75, 41.25, 41.75, 42.25, 42.75, 43.25, 43.75, 44.25, 44.75,
       45.25, 45.75, 46.25, 46.75, 47.25, 47.75, 48.25, 48.75, 49.25, 49.75,
       50.25, 50.75, 51.25, 51.75, 52.25, 52.75, 53.25, 53.75, 54.25, 54.75,
       55.25, 55.75, 56.25, 56.75, 57.25, 57.75, 58.25, 58.75, 59.25, 59.75,
       60.25, 60.75, 61.25, 61.75, 62.25, 62.75, 63.25, 63.75, 64.25, 64.75,
       65.25, 65.75, 66.25, 66.75, 67.25, 67.75, 68.25, 68.75, 69.25, 69.75,
       70.25, 70.75, 71.25, 71.75, 72.25, 72.75, 73.25, 73.75, 74.25, 74.75,
       75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-561eaf40-9437-4ccc-8dfb-fe57e4e427a0' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-561eaf40-9437-4ccc-8dfb-fe57e4e427a0' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-6397d1dd-3e90-44cd-84d9-afcfa332101b' class='xr-var-data-in' type='checkbox'><label for='data-6397d1dd-3e90-44cd-84d9-afcfa332101b' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd><dt><span>semi_major_axis :</span></dt><dd>6378137.0</dd><dt><span>semi_minor_axis :</span></dt><dd>6356752.314245179</dd><dt><span>inverse_flattening :</span></dt><dd>298.257223563</dd><dt><span>reference_ellipsoid_name :</span></dt><dd>WGS 84</dd><dt><span>longitude_of_prime_meridian :</span></dt><dd>0.0</dd><dt><span>prime_meridian_name :</span></dt><dd>Greenwich</dd><dt><span>geographic_crs_name :</span></dt><dd>WGS 84</dd><dt><span>grid_mapping_name :</span></dt><dd>latitude_longitude</dd><dt><span>spatial_ref :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-73f1b69e-495e-497b-938b-48ecc7e71d84' class='xr-section-summary-in' type='checkbox'  checked><label for='section-73f1b69e-495e-497b-938b-48ecc7e71d84' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(latitude, longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-e94bafd1-ea17-4aac-894d-411de8470f3c' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-e94bafd1-ea17-4aac-894d-411de8470f3c' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-15497eb7-8e8a-4a87-aeaa-cded8167776e' class='xr-var-data-in' type='checkbox'><label for='data-15497eb7-8e8a-4a87-aeaa-cded8167776e' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       ...,
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-959e6299-7e82-4716-88ec-2791a01d2e0c' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-959e6299-7e82-4716-88ec-2791a01d2e0c' class='xr-section-summary'  title='Expand/collapse section'>Attributes: <span>(0)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'></dl></div></li></ul></div></div>

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
Dimensions:      (latitude: 101, longitude: 232)
Coordinates:
  * longitude    (longitude) float32 -40.25 -39.75 -39.25 ... 74.25 74.75 75.25

  * latitude     (latitude) float32 25.25 25.75 26.25 ... 74.25 74.75 75.25
    spatial_ref  int32 0
Data variables:
    tg           (latitude, longitude) float32 nan nan nan nan ... nan nan nan</pre><div class='xr-wrap' hidden><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-b9f06a96-a516-40c5-8f72-84554e59c003' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-b9f06a96-a516-40c5-8f72-84554e59c003' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>latitude</span>: 101</li><li><span class='xr-has-index'>longitude</span>: 232</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-1d80c0d1-a3b6-444d-af21-a3ceb1eb4b0d' class='xr-section-summary-in' type='checkbox'  checked><label for='section-1d80c0d1-a3b6-444d-af21-a3ceb1eb4b0d' class='xr-section-summary' >Coordinates: <span>(3)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>longitude</span></div><div class='xr-var-dims'>(longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>-40.25 -39.75 ... 74.75 75.25</div><input id='attrs-328bec64-6b08-4406-bfa6-e996bf9e282b' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-328bec64-6b08-4406-bfa6-e996bf9e282b' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-27bde3f9-cfdb-4aaa-8d3c-c7b8e1a381f1' class='xr-var-data-in' type='checkbox'><label for='data-27bde3f9-cfdb-4aaa-8d3c-c7b8e1a381f1' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Longitude values</dd><dt><span>units :</span></dt><dd>degrees_east</dd><dt><span>standard_name :</span></dt><dd>longitude</dd></dl></div><div class='xr-var-data'><pre>array([-40.25, -39.75, -39.25, ...,  74.25,  74.75,  75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>latitude</span></div><div class='xr-var-dims'>(latitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>25.25 25.75 26.25 ... 74.75 75.25</div><input id='attrs-3ee50b7f-154c-4703-8a99-1938f18daf88' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-3ee50b7f-154c-4703-8a99-1938f18daf88' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-52431376-1eff-412b-8890-dce015b431bd' class='xr-var-data-in' type='checkbox'><label for='data-52431376-1eff-412b-8890-dce015b431bd' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>long_name :</span></dt><dd>Latitude values</dd><dt><span>units :</span></dt><dd>degrees_north</dd><dt><span>standard_name :</span></dt><dd>latitude</dd></dl></div><div class='xr-var-data'><pre>array([25.25, 25.75, 26.25, 26.75, 27.25, 27.75, 28.25, 28.75, 29.25, 29.75,
       30.25, 30.75, 31.25, 31.75, 32.25, 32.75, 33.25, 33.75, 34.25, 34.75,
       35.25, 35.75, 36.25, 36.75, 37.25, 37.75, 38.25, 38.75, 39.25, 39.75,
       40.25, 40.75, 41.25, 41.75, 42.25, 42.75, 43.25, 43.75, 44.25, 44.75,
       45.25, 45.75, 46.25, 46.75, 47.25, 47.75, 48.25, 48.75, 49.25, 49.75,
       50.25, 50.75, 51.25, 51.75, 52.25, 52.75, 53.25, 53.75, 54.25, 54.75,
       55.25, 55.75, 56.25, 56.75, 57.25, 57.75, 58.25, 58.75, 59.25, 59.75,
       60.25, 60.75, 61.25, 61.75, 62.25, 62.75, 63.25, 63.75, 64.25, 64.75,
       65.25, 65.75, 66.25, 66.75, 67.25, 67.75, 68.25, 68.75, 69.25, 69.75,
       70.25, 70.75, 71.25, 71.75, 72.25, 72.75, 73.25, 73.75, 74.25, 74.75,
       75.25], dtype=float32)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>spatial_ref</span></div><div class='xr-var-dims'>()</div><div class='xr-var-dtype'>int32</div><div class='xr-var-preview xr-preview'>0</div><input id='attrs-23810994-459d-4f60-86f8-8089d8359e52' class='xr-var-attrs-in' type='checkbox' ><label for='attrs-23810994-459d-4f60-86f8-8089d8359e52' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-8aa85516-fa63-4622-890f-63b929ff4040' class='xr-var-data-in' type='checkbox'><label for='data-8aa85516-fa63-4622-890f-63b929ff4040' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'><dt><span>crs_wkt :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd><dt><span>semi_major_axis :</span></dt><dd>6378137.0</dd><dt><span>semi_minor_axis :</span></dt><dd>6356752.314245179</dd><dt><span>inverse_flattening :</span></dt><dd>298.257223563</dd><dt><span>reference_ellipsoid_name :</span></dt><dd>WGS 84</dd><dt><span>longitude_of_prime_meridian :</span></dt><dd>0.0</dd><dt><span>prime_meridian_name :</span></dt><dd>Greenwich</dd><dt><span>geographic_crs_name :</span></dt><dd>WGS 84</dd><dt><span>grid_mapping_name :</span></dt><dd>latitude_longitude</dd><dt><span>spatial_ref :</span></dt><dd>GEOGCS[&quot;WGS 84&quot;,DATUM[&quot;WGS_1984&quot;,SPHEROID[&quot;WGS 84&quot;,6378137,298.257223563,AUTHORITY[&quot;EPSG&quot;,&quot;7030&quot;]],AUTHORITY[&quot;EPSG&quot;,&quot;6326&quot;]],PRIMEM[&quot;Greenwich&quot;,0,AUTHORITY[&quot;EPSG&quot;,&quot;8901&quot;]],UNIT[&quot;degree&quot;,0.0174532925199433,AUTHORITY[&quot;EPSG&quot;,&quot;9122&quot;]],AXIS[&quot;Latitude&quot;,NORTH],AXIS[&quot;Longitude&quot;,EAST],AUTHORITY[&quot;EPSG&quot;,&quot;4326&quot;]]</dd></dl></div><div class='xr-var-data'><pre>array(0)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-6d1aa318-338e-41f1-b3ca-1d7c9f0cca6e' class='xr-section-summary-in' type='checkbox'  checked><label for='section-6d1aa318-338e-41f1-b3ca-1d7c9f0cca6e' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>tg</span></div><div class='xr-var-dims'>(latitude, longitude)</div><div class='xr-var-dtype'>float32</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-f5ffff7c-ba18-4819-9985-ba33cd71532a' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-f5ffff7c-ba18-4819-9985-ba33cd71532a' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-5b4bbd3b-fc8d-4723-811b-e5f27d9186e5' class='xr-var-data-in' type='checkbox'><label for='data-5b4bbd3b-fc8d-4723-811b-e5f27d9186e5' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       ...,
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]], dtype=float32)</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-cda943a5-78ec-43d1-b25d-93732afb5dd7' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-cda943a5-78ec-43d1-b25d-93732afb5dd7' class='xr-section-summary'  title='Expand/collapse section'>Attributes: <span>(0)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'></dl></div></li></ul></div></div>
    
       

Also, the area covered by the NetCDF is greater than we actually need. Let's clip the Hot and Cold raster to the area of interest (EU countries).

```python
#Note that this could have been done before defining Hot and Cold ¯\_ツ_/¯
Hot.rio.write_crs("epsg:4326", inplace=True) #set crs to our raster
Hot = Hot.rio.reproject(dst_crs='EPSG:3857') #reproject because geojson in EPSG:3857
Hot = Hot.rio.clip(eu)
Hot = Hot.where(Hot['tg'] != -9999.)  #conserve only necesaary values

Cold.rio.write_crs("epsg:4326", inplace=True)
... #idem
```

## 3. binning

Ok, so, we have 16 features that we must transform into 'categorical features'. For that we must split them into equals bins. With 4 bins per feature, we have 4<sup>16</sup> = 4294967296 geoclusters possibilities. Plenty enough! The reality will be way less. Testes have shown that 4 bins per feature would result into 116 geoclusters which is sufficient for now.

```python
hot_bins = Hot.quantile([0,0.25,0.5,0.75,1]).tg.values
Hot_discr = np.digitize(Hot.tg.values, hot_bins)
Hot_discr = Hot_discr.astype(np.int32)

#idem for Cold raster
```

<img src="C:\Users\corentink\OneDrive\website\projects\geocluster\article\output_40_1.png" alt="Hot raster" style="zoom:70%;" />

Now let's vectorize those rasters

```python
from rasterio.features import shapes

west, east = min(Hot.x.values), max(Hot.x.values) #get corners coordinates
south, north =  min(Hot.y.values), max(Hot.y.values) #get corners coordinates
width = len(Hot.x.values)
height = len(Hot.y.values)

res_x = np.diff(Hot.x.values).mean() #get the x gradient
res_y = np.diff(Hot.y.values).mean() #get the y gradient

transform = rasterio.transform.from_origin(west, north, res_x, -res_y) #Return an Affine transformation for a georeferenced raster 

data = []
mask = Cold_discr != 5
for shp, val in shapes(Cold_discr,mask=mask, transform=transform):
    record = {'properties':{'bin': val}, 'geometry' : shp}
    data.append(record)

#let's make a geodataframe from it
Hot_df = gpd.GeoDataFrame.from_features(data,crs=3857) 
Hot_df = Hot_df.dissolve(by='bin')
```

81 rows × 2 columns

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Hot_temp in C</th>
    </tr>
    <tr>
      <th>bin</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1.0</th>
      <td>MULTIPOLYGON (((-727543.440 7693137.472, -7242...</td>
    </tr>
    <tr>
      <th>2.0</th>
      <td>MULTIPOLYGON (((-352714.079 7639933.464, -3604...</td>
    </tr>
    <tr>
      <th>3.0</th>
      <td>MULTIPOLYGON (((-986903.059 5272566.153, -9883...</td>
    </tr>
    <tr>
      <th>4.0</th>
      <td>MULTIPOLYGON (((-1010018.459 4695661.164, -101...</td>
    </tr>
  </tbody>
</table>

Also let's fit data to countries outline

```python
for index, row in Hot_df.iterrows(): #for each bin
    temp_poly = row.geometry.buffer(100*1000) #buffer the current geometry
    temp_poly = temp_poly.intersection(eu)    #intersect it with EU geometry (outline)
    other_polys = Hot_df.loc[Hot_df.index != index].geometry.values.unary_union() #union of all the other polygon
    to_remove = temp_poly.intersection(other_polys) #intersection to retrieve what to remove (because of the previous buffer)
    temp_poly = temp_poly.difference(to_remove) #removal of teh extra bits

    Hot_df.loc[index,['geometry']]= temp_poly #overwrite geometry in dataframe
```

| Hot_df                                                       | Cold_df                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src="C:\Users\corentink\OneDrive\website\projects\geocluster\article\output_42_1.png" alt="hot_df" style="zoom:50%;" /> | <img src="C:\Users\corentink\OneDrive\website\projects\geocluster\article\output_45_1.png" alt="cold_df" style="zoom:50%;" /> |

We binned our rasters. Let's bin our other variables.

```python
all_data_dis = all_data.apply(lambda col: pd.qcut(col, 4, labels=False, duplicates = 'drop')) #qcut is a Quantile-based discretization function.
all_data_dis = all_data_dis+1 #we add 1 so the value range from 1 to 4 (and not 0 to 3)
```

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
      <td>4</td>
      <td>2</td>
      <td>2</td>
      <td>1</td>
      <td>3</td>
      <td>1</td>
      <td>2</td>
      <td>2.0</td>
      <td>2</td>
      <td>1.0</td>
      <td>1</td>
      <td>4</td>
      <td>2</td>
      <td>3</td>
      <td>POLYGON ((1687803.014 6264215.777, 1696293.843...</td>
    </tr>
    <tr>
      <th>BE</th>
      <td>1</td>
      <td>3</td>
      <td>1</td>
      <td>1</td>
      <td>4</td>
      <td>4</td>
      <td>3</td>
      <td>4.0</td>
      <td>4</td>
      <td>4.0</td>
      <td>2</td>
      <td>3</td>
      <td>4</td>
      <td>1</td>
      <td>POLYGON ((536053.133 6697902.835, 536858.496 6...</td>
    </tr>
    <tr>
      <th>BG</th>
      <td>3</td>
      <td>1</td>
      <td>1</td>
      <td>3</td>
      <td>1</td>
      <td>3</td>
      <td>4</td>
      <td>3.0</td>
      <td>3</td>
      <td>1.0</td>
      <td>3</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>POLYGON ((2551393.950 5439823.819, 2575025.606...</td>
    </tr>
    <tr>
      <th>CZ</th>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>1</td>
      <td>2</td>
      <td>2</td>
      <td>2.0</td>
      <td>1</td>
      <td>2.0</td>
      <td>1</td>
      <td>4</td>
      <td>2</td>
      <td>3</td>
      <td>POLYGON ((1602756.662 6623613.894, 1605874.568...</td>
    </tr>
    <tr>
      <th>DE</th>
      <td>2</td>
      <td>4</td>
      <td>2</td>
      <td>2</td>
      <td>4</td>
      <td>3</td>
      <td>1</td>
      <td>2.0</td>
      <td>4</td>
      <td>3.0</td>
      <td>2</td>
      <td>3</td>
      <td>2</td>
      <td>3</td>
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
      <td>3</td>
      <td>4</td>
      <td>4</td>
      <td>3</td>
      <td>3</td>
      <td>4</td>
      <td>4</td>
      <td>3.0</td>
      <td>3</td>
      <td>2.0</td>
      <td>4</td>
      <td>1</td>
      <td>1</td>
      <td>4</td>
      <td>MULTIPOLYGON (((1404993.029 4230990.607, 14038...</td>
    </tr>
    <tr>
      <th>NL</th>
      <td>1</td>
      <td>3</td>
      <td>4</td>
      <td>1</td>
      <td>2</td>
      <td>1</td>
      <td>4</td>
      <td>4.0</td>
      <td>3</td>
      <td>4.0</td>
      <td>1</td>
      <td>4</td>
      <td>4</td>
      <td>1</td>
      <td>MULTIPOLYGON (((-7596140.830 1348758.669, -759...</td>
    </tr>
    <tr>
      <th>PL</th>
      <td>1</td>
      <td>4</td>
      <td>1</td>
      <td>4</td>
      <td>1</td>
      <td>2</td>
      <td>1</td>
      <td>1.0</td>
      <td>2</td>
      <td>4.0</td>
      <td>2</td>
      <td>3</td>
      <td>1</td>
      <td>4</td>
      <td>POLYGON ((2115307.046 7235751.798, 2157060.914...</td>
    </tr>
    <tr>
      <th>SE</th>
      <td>4</td>
      <td>2</td>
      <td>4</td>
      <td>4</td>
      <td>2</td>
      <td>2</td>
      <td>1</td>
      <td>NaN</td>
      <td>1</td>
      <td>3.0</td>
      <td>1</td>
      <td>4</td>
      <td>1</td>
      <td>4</td>
      <td>MULTIPOLYGON (((1744966.813 7582362.064, 17458...</td>
    </tr>
    <tr>
      <th>SI</th>
      <td>4</td>
      <td>1</td>
      <td>2</td>
      <td>4</td>
      <td>2</td>
      <td>1</td>
      <td>2</td>
      <td>4.0</td>
      <td>4</td>
      <td>3.0</td>
      <td>4</td>
      <td>1</td>
      <td>3</td>
      <td>2</td>
      <td>POLYGON ((1819341.831 5895544.825, 1820883.526...</td>
    </tr>
  </tbody>
</table>

## 4. merge all in a final table

Now, all our data are ready to come together. I am going to spare you dataframe manipulations because it isn't the topic here but at the end, the final table would look like that:

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
      <td>[8.767000000000001, 12.389]</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
    </tr>
    <tr>
      <th>energy cons (kg of oil equivalent per capita)</th>
      <td>[1057.442, 5023.244]</td>
      <td>MULTIPOLYGON (((-1146332.292 6785384.989, -114...</td>
    </tr>
    <tr>
      <th>residential gas price in euro</th>
      <td>[9.601, 15.633]</td>
      <td>MULTIPOLYGON (((532003.307 6697072.451, 530749...</td>
    </tr>
    <tr>
      <th>indus gas price in euro</th>
      <td>[6.185, 7.471]</td>
      <td>MULTIPOLYGON (((-7601494.819 1355961.122, -760...</td>
    </tr>
    <tr>
      <th>residential elec price in euro</th>
      <td>[0.0945, 0.15]</td>
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
      <td>[33.2, 39.3]</td>
      <td>MULTIPOLYGON (((-7601703.180 1358036.485, -760...</td>
    </tr>
    <tr>
      <th>single family share in %</th>
      <td>[61.1, 87.2]</td>
      <td>MULTIPOLYGON (((-7600964.855 1353927.589, -760...</td>
    </tr>
    <tr>
      <th>mutiple family share in %</th>
      <td>[55.8, 74.0]</td>
      <td>MULTIPOLYGON (((-1997145.494 3212924.432, -199...</td>
    </tr>
    <tr>
      <th>Cold_temp in C</th>
      <td>[5.103, 13.653]</td>
      <td>MULTIPOLYGON (((-1160953.847 6780231.201, -115...</td>
    </tr>
    <tr>
      <th>Hot_temp in C</th>
      <td>[16.635, 24.56]</td>
      <td>MULTIPOLYGON (((-1010018.459 4695661.164, -101...</td>
    </tr>
  </tbody>
</table>

The final result can then be store in a postgresql DB, geoJSON, Shapefile or more...

```python
final_data.to_postgis(name, con, schema=None, if_exists='fail', index=False, index_label=None, chunksize=None, dtype=None)

final_data.to_file("data.shp")

final_data.to_file("data.geojson", driver='GeoJSON')

...
```

## 5. Plotting

I first intended to use javascript for the plotting using ```mapbox gl```, ```turf``` and ```d3``` for a more interactive output but I got into some issues with the polygons intersections. So I went for something simpler using python ```folium```, ```geopy``` and ```ipywidgets```.

![plot](C:\Users\corentink\OneDrive\website\projects\geocluster\article\ezgif.com-gif-maker (5).gif)


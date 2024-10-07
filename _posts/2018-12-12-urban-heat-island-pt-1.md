---
layout: post
title: Designing Out the Urban Heat Island with Dragonfly
subtitle: Dragonfly 101
header:
  image: /assets/img/urban-heat-island/blue_dragonfly.jpeg
tags: [ladybug-tools]
---

> TL;DR: Dragonfly modelling 101. Here's the repository where all code is saved. https://github.com/AntoineDao/lbt-dragonfly-tutorial

# Introducing Dragonfly
Dragonfly is the latest addition to the Ladybug Tools' bug ecosystem, it is python library for urban climate and energy modeling. This library can help us model an UHI by servin as an API layer on top of the Python [Urban Weather Generator (UWG) library](https://github.com/ladybug-tools/uwg) in a similar way to how Honeybee interracts with Radiance and Butterfly with OpenFoam. 

The UWG library, [originally written in Matlab](http://urbanmicroclimate.scripts.mit.edu/uwg.php), simulates the Urban Heat Island effect of an urban area at a district or neighborhood scale. Michael Street found in his [master's thesis](https://dspace.mit.edu/handle/1721.1/82284) that the size of an urban areas accurately characterized by the UWG was ~500 meters x 500 meters. Any larger and you will have urban subsets with different temperatures than the average. Any smaller, and the mixing of air of your simultated area with neighboring areas will probably dimminish the result of your simualtion.

In the case study covered below we will use Dragonfly to calculate the impact of an Urban Heat Island on it's local climate. You can fork/clone the [tutorial repository](https://github.com/AntoineDao/lbt-dragonfly-tutorial) or if you are just wanting to try it out online why not give Binder a go? Look for the `Baseline_UHI.ipynb` file.

[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/AntoineDao/lbt-dragonfly-tutorial/master)

In this example we will carry out the following tasks:

1. Create Building Typology models
2. Create an urban District model
3. Run the UHI Simulation
4. Compare the original EPW file with the new District EPW

## 1. Building Typologies
![typologies!](/assets/img/urban-heat-island/typology.jpeg)

Dragonfly simulates district level urban areas. Rather than simulating all buildings seperately it uses the concept of a `typology`. We will be creating two typologies in this case study based on a neighborhood in Malaga called La Luz:
* Residential Typology
* Small Retail Typology


The code snippet below covers how these typologies are created, if you want more info on all the attributes you can set for these I'd reccomend checking out the [documentation for this class](http://www.ladybug.tools/apidoc/dragonfly/dragonfly.html#module-dragonfly.typology). You will notice no geometry is actually used here. You can use building geometry when creating typologies from Rhino/Grasshopper and we will also cover how to do so from Open Street Map data and GeoJson in a later example.

Let's first create our residential typology:


```python
from dragonfly.typology import Typology

# The average height of a single floor (in meters)
floor_to_floor = 3.05

# The average number of floors for residential buildings 
resi_average_floors = 8
resi_average_height = floor_to_floor * resi_average_floors

# The total footprint area of all residential buildings in the district (in square meters)
resi_footprint_area = 52000

# The total area of exposed facade (in square meters)
resi_facade_area = 221000

# The average glazing ratio of residential buildings 
resi_glazing_ratio = 0.3

residential_typology = Typology(average_height=resi_average_height,
                                footprint_area=resi_footprint_area,
                                floor_to_floor=floor_to_floor,
                                facade_area=resi_facade_area,
                                glz_ratio=resi_glazing_ratio,
                                bldg_program='MidRiseApartment', # one of the 16 DOE building program types 
                                bldg_era='Pre1980s' # used to determine what constructions make up the walls, roofs, and windows based on international building codes over the last several decades
                               )

residential_typology
```




    Building Typology: 
      MidRiseApartment, Pre1980s
      Average Height: 24 m
      Number of Stories: 8
      Floor Area: 416,000 m2
      Footprint Area: 52,000 m2
      Facade Area: 221,000 m2
      Glazing Ratio: 30 %



Not too bad. We can see the properties of the typology we have just created above. Next step if to create the small retail typology. This time we will do so without comments to save time and space as well as input the average height directly.


```python
retail_average_height = 8
retail_footprint_area = 2256
retail_facade_area = 2424
retail_glazing_ratio = 0.4

small_retail_typology = Typology(average_height=retail_average_height,
                                 footprint_area=retail_footprint_area,
                                 facade_area=retail_facade_area,
                                 glz_ratio=retail_glazing_ratio,
                                 bldg_program='StandAloneRetail',
                                 bldg_era='Pre1980s'
                                )

small_retail_typology
```




    Building Typology: 
      StandAloneRetail, Pre1980s
      Average Height: 8 m
      Number of Stories: 3
      Floor Area: 6,768 m2
      Footprint Area: 2,256 m2
      Facade Area: 2,424 m2
      Glazing Ratio: 40 %



## 2. Dragonfly District Recipe
![recipes!](/assets/img/urban-heat-island/district.jpeg)

Dragonfly models an urban area by creating a `district` object, which contains the typologies we have just created plus some information about the local climate zone, traffic and greenery.

### Traffic Parameters
The traffic heat emission rate and weekly schedule. We will leave the schedule values empty to set the defaults.


```python
from dragonfly.uwg.districtpar import TrafficPar

# The maximum sensible anthropogenic heat flux of the urban area in watts per square meter.
traffic_heat = 4

# Note that we leave the schedules blank, this will set the default schedule values for traffic
traffic_parameters = TrafficPar(sensible_heat=traffic_heat,
                                weekday_schedule=[],
                                saturday_schedule=[],
                                sunday_schedule=[])

traffic_parameters
```




    Traffic Parameters: 
      Max Heat: 4 W/m2
      Weekday Avg Heat: 2.1999999999999997 W/m2
      Saturday Avg Heat: 1.7166666666666666 W/m2
      Sunday Avg Heat: 1.316666666666667 W/m2



### Pavement Parameters
The makeup of pavement within the urban area. We will set all the values to `None` in order to create an object with the default values for pavements. 


```python
from dragonfly.uwg.districtpar import PavementPar

# We will leave all these values as None which will set the default values for each
pav_albedo = None
pav_thickness = None
pav_conductivity = None
pav_volumetric_heat_capacity = None

pavement_parameters = PavementPar(albedo=pav_albedo,
                                  thickness=pav_thickness,
                                  conductivity=pav_conductivity,
                                  volumetric_heat_capacity=pav_volumetric_heat_capacity)

# Note: we can declare the pavement parameter object like this also as we are using default values
pavement_parameters = PavementPar()

pavement_parameters
```




    Pavement Parameters: 
      Albedo: 0.1
      Thickness: 0.5
      Conductivity: 1
      Vol Heat Capacity: 1600000



### Vegetation Parameters
The behaviour of vegetation within an urban area. 



```python
from dragonfly.uwg.districtpar import VegetationPar

# The average albedo of grass and trees
vegetation_albedo=0.25
# The month of the year where leaves grow on trees (1-12)
vegetation_start_month=4
# The month of the year where leaves fall from trees
vegetation_end_month=11
# The fraction of absorbed solar energy by trees that is given off as latent heat (evapotranspiration)
tree_latent_fraction=0.7
# Same as above but for grass
grass_latent_fraction=0.5

vegetation_parameters = VegetationPar(vegetation_albedo=vegetation_albedo,
                                      vegetation_start_month=vegetation_start_month,
                                      vegetation_end_month=vegetation_end_month,
                                      tree_latent_fraction=tree_latent_fraction,
                                      grass_latent_fraction=grass_latent_fraction)

vegetation_parameters
```




    Vegetation Parameters: 
      Albedo: 0.25
      Vegetation Time: Apr - Nov
      Tree | Grass Latent: 0.7 | 0.5



### Making a District
![we haz district!](/assets/img/urban-heat-island/icanhazdistrict.jpg)

Now that we have set up our typologies and district level parameters we can combine them all together and create a district object. We still need to set a few variables before creating the district:
* Site Area: The total area of the site modelled
* Climate Zone: The ASHRAE climate zone. This is used to set default constructions.
* Tree coverage ratio: The fraction of the urban area (including both pavement and roofs) that is covered by trees. 
* Grass coverage ratio: The fraction of the urban area (including both pavement and roofs) that is covered by grass/vegetation.

You will notice that some new values appear when we return the district:
* Site coverage ratio
* Building typologies split by percentage of the total built area they represent


```python
from dragonfly.district import District

site_area = 160000

climate_zone = '3A'

tree_coverage_ratio = 0
grass_coverage_ratio = 0

district = District(site_area=site_area,
                    climate_zone=climate_zone,
                    tree_coverage_ratio=tree_coverage_ratio,
                    grass_coverage_ratio=grass_coverage_ratio,
                    building_typologies=[residential_typology, small_retail_typology],
                    traffic_parameters=traffic_parameters, 
                    pavement_parameters=pavement_parameters,
                    vegetation_parameters=vegetation_parameters)

district
```




    District: 
      Average Bldg Height: 23 m
      Site Coverage Ratio: 0.34
      Facade-to-Site Ratio: 1.4
      Tree Coverage Ratio: 0
      Grass Coverage Ratio: 0
      ------------------------
      Building Typologies:
         0.98 - MidRiseApartment,Pre1980s
         0.02 - StandAloneRetail,Pre1980s



## 3. Bringing It All Together (running the simulation)
![COMPUTE!](/assets/img/urban-heat-island/bringing_it_together.jpeg)

Right, we now have an urban district model to run an UHI simulation with. In order to run the simulation we will need an EPW. We will also set some parameters for the original EPW file to provide some extra context to the UWG engine.

### Reference EPW Site Parameters
The Reference EPW Site Parameters refer to the context of the original EPW site and how the different measurements were taken. Values that we leave blank will be defaulted.


```python
from dragonfly.uwg.regionpar import RefEPWSitePar

# We set the reference epw site parameters with the information we know about site
# In this case we only know that the vegetation covers roughly 70% of the measurement site
ref_epw_site_par = RefEPWSitePar(average_obstacle_height=None,
                                 vegetation_coverage=0.7,
                                 temp_measure_height=None,
                                 wind_measure_height=None)

ref_epw_site_par
```




    Reference EPW Site Parameters:
      Obstacle Height: 0.1 m
      Vegetation Coverage: 0.7
      Measurement Height (Temp | Wind): 10 m | 10 m



### Create the Run Manager
The run manager object combines all the model objects and parameters into one object ready to be run. 


```python
from dragonfly.uwg.run import RunManager

epw_file='data/ESP_MALAGA-AP_084820_IW2.epw'

rm = RunManager(epw_file=epw_file,
                district=district,
               epw_site_par=ref_epw_site_par)
rm
```




    UWG RunManager: 
     Rural EPW: data/ESP_MALAGA-AP_084820_IW2.epw
     Analysis Period: 1/1 to 12/31 between 0 to 23 @1
     District: District: 
      Average Bldg Height: 23 m
      Site Coverage Ratio: 0.34
      Facade-to-Site Ratio: 1.4
      Tree Coverage Ratio: 0
      Grass Coverage Ratio: 0
      ------------------------
      Building Typologies:
         0.98 - MidRiseApartment,Pre1980s
         0.02 - StandAloneRetail,Pre1980s



### Run the Simulation
All we have left to do is run the simulation. The output of an UWG simulation is a morphed EPW file, by default this file has the same name as the original EPW with an `_URBAN` at the end of it. Because we are saving the file to a different folder we will name it manually.

The simulation will take roughly 1-3 minutes to complete.


```python
urban_epw_file = 'data/ESP_MALAGA-AP_084820_IW2_URBAN.epw'

rm.run(urban_epw_file)
```

    
    Simulating new temperature and humidity values for 365 days from 1/1.
    
    New climate file 'ESP_MALAGA-AP_084820_IW2_URBAN.epw' is generated at data.





    'data/ESP_MALAGA-AP_084820_IW2_URBAN.epw'



Neat! We now have a new morphed EPW sitting in our `data` folder waiting to be analysed. 

## 4. Results Analysis

![Look at the data!](/assets/img/urban-heat-island/look_at_the_data.jpeg)


Now that we have an urban EPW to compare to our original EPW we can run some data analysis. To keep this first tutorial short we will simply look at the differences between the two weather files. We can later build upon this to measure energy and comfort impacts.

To run our comparison we will load the two EPW files into dataframes: `df_original` for the original EPW and `df_urban` for the EPW generated by our simulation. We will then create another dataframe called `diff` which will be the result of `df_urban` - `df_original`. Any positive value will therefore show an increase over the original EPW.

### EPW as a DataFrame
We first load our dependencies and an EPW import utility function `epw_to_df`. The EPW import code is saved in the `lib` file of the code's repository, do check it out if you want to see how to load an EPW file into a dataframe using `pandas`.


```python
import pandas as pd
import datetime
import matplotlib.pyplot as plt
import seaborn as sns

from lib.epw_utils import *

%matplotlib inline
```

First we import the original epw.


```python
# Create a DataFrame from the original EPW file
df_original = epw_to_df(epw_file)

# Add a python datetime column for easier timeseries analysis
df_original['datetime'] = df_original.apply(lambda row: get_datetime(row), axis=1)

# Add a Day of Year column to facilitate heatmap plotting
df_original['doy'] = df_original.apply(lambda row: get_doy(row), axis=1)

df_original.head()
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
      <th>Index</th>
      <th>Year</th>
      <th>Month</th>
      <th>Day</th>
      <th>Hour</th>
      <th>Minute</th>
      <th>Remove</th>
      <th>Dry_Bulb_Temperature</th>
      <th>Dew_Point_Temperature</th>
      <th>Relative_Humidity</th>
      <th>...</th>
      <th>Present_Weather_Codes</th>
      <th>Precipitable_Water</th>
      <th>Aerosol_Optical_Depth</th>
      <th>Snow_Depth</th>
      <th>Days_Since_Last_Snowfall</th>
      <th>Albedo</th>
      <th>Liquid_Precipitation_Depth</th>
      <th>Liquid_Precipitation_Quantity</th>
      <th>datetime</th>
      <th>doy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>11.6</td>
      <td>10.7</td>
      <td>94</td>
      <td>...</td>
      <td>999999999</td>
      <td>200</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>2017-01-01 00:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>8.9</td>
      <td>7.4</td>
      <td>90</td>
      <td>...</td>
      <td>999999999</td>
      <td>160</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>2017-01-01 01:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>3</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>7.0</td>
      <td>6.0</td>
      <td>93</td>
      <td>...</td>
      <td>999999999</td>
      <td>139</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>2017-01-01 02:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>4</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>9.8</td>
      <td>8.6</td>
      <td>92</td>
      <td>...</td>
      <td>999999999</td>
      <td>170</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>2017-01-01 03:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>7.0</td>
      <td>6.0</td>
      <td>93</td>
      <td>...</td>
      <td>999999999</td>
      <td>139</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>2017-01-01 04:00:00</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 38 columns</p>
</div>



Then we import the urban EPW generated by the simulation.


```python
df_urban = epw_to_df(urban_epw_file)

df_urban['datetime'] = df_urban.apply(lambda row: get_datetime(row), axis=1)
df_urban['doy'] = df_urban.apply(lambda row: get_doy(row), axis=1)

df_urban.head()
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
      <th>Index</th>
      <th>Year</th>
      <th>Month</th>
      <th>Day</th>
      <th>Hour</th>
      <th>Minute</th>
      <th>Remove</th>
      <th>Dry_Bulb_Temperature</th>
      <th>Dew_Point_Temperature</th>
      <th>Relative_Humidity</th>
      <th>...</th>
      <th>Present_Weather_Codes</th>
      <th>Precipitable_Water</th>
      <th>Aerosol_Optical_Depth</th>
      <th>Snow_Depth</th>
      <th>Days_Since_Last_Snowfall</th>
      <th>Albedo</th>
      <th>Liquid_Precipitation_Depth</th>
      <th>Liquid_Precipitation_Quantity</th>
      <th>datetime</th>
      <th>doy</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>11.8</td>
      <td>10.7</td>
      <td>92.6</td>
      <td>...</td>
      <td>999999999</td>
      <td>200</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2017-01-01 00:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>11.4</td>
      <td>7.4</td>
      <td>76.1</td>
      <td>...</td>
      <td>999999999</td>
      <td>160</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2017-01-01 01:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>3</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>10.5</td>
      <td>6.0</td>
      <td>73.3</td>
      <td>...</td>
      <td>999999999</td>
      <td>139</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2017-01-01 02:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>4</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>10.1</td>
      <td>8.6</td>
      <td>90.3</td>
      <td>...</td>
      <td>999999999</td>
      <td>170</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2017-01-01 03:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>2001</td>
      <td>1</td>
      <td>1</td>
      <td>5</td>
      <td>0</td>
      <td>?9?9?9?9E0?9?9?9?9*9?9?9?9?9?9?9?9?9?9*_*9*9*9...</td>
      <td>9.4</td>
      <td>6.0</td>
      <td>79.2</td>
      <td>...</td>
      <td>999999999</td>
      <td>139</td>
      <td>0.0</td>
      <td>0</td>
      <td>88</td>
      <td>999.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2017-01-01 04:00:00</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 38 columns</p>
</div>



### Diff Analysis

The UWG simulation only impacts the following EPW metrics:
* Dry_Bulb_Temperature
* Dew_Point_Temperature
* Relative_Humidity
* Wind_Speed

We will drop all other measurements and only substract the value we are interested as shown below:


```python
impacted_measurements = ['Dry_Bulb_Temperature',
                         'Dew_Point_Temperature',
                         'Relative_Humidity',
                         'Wind_Speed']

diff = df_original[['Hour','datetime' ,'doy']]

diff[impacted_measurements] = df_urban[impacted_measurements] - df_original[impacted_measurements]

diff.head()
```

    /home/kakistocrat/.local/lib/python3.6/site-packages/pandas/core/frame.py:2352: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy
      self[k1] = value[k2]





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
      <th>Hour</th>
      <th>datetime</th>
      <th>doy</th>
      <th>Dry_Bulb_Temperature</th>
      <th>Dew_Point_Temperature</th>
      <th>Relative_Humidity</th>
      <th>Wind_Speed</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>2017-01-01 00:00:00</td>
      <td>1</td>
      <td>0.2</td>
      <td>0.0</td>
      <td>-1.4</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>2017-01-01 01:00:00</td>
      <td>1</td>
      <td>2.5</td>
      <td>0.0</td>
      <td>-13.9</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>2017-01-01 02:00:00</td>
      <td>1</td>
      <td>3.5</td>
      <td>0.0</td>
      <td>-19.7</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>2017-01-01 03:00:00</td>
      <td>1</td>
      <td>0.3</td>
      <td>0.0</td>
      <td>-1.7</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>2017-01-01 04:00:00</td>
      <td>1</td>
      <td>2.4</td>
      <td>0.0</td>
      <td>-13.8</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>



We can then run a quick aggregation to measure the amount of difference between the two EPWs. The code snippet below demonstrates how run a quick statistical study of monthly aggregations. Here is a list of initial observations:
* Maximum `Dry Bulb Temperature` difference of 10C in May
* Mean increase in `Dry Bulb Temperature` is between 0.6C and 1C throughout the year
* `Dew Point Temperature` and `Wind Speed` are not much affected by the urban model
* `Relative Humidity` is generally a bit lower by roughly 4%


```python
diff.groupby(pd.Grouper(key='datetime', freq='M'))['Dry_Bulb_Temperature',
                                                   'Dew_Point_Temperature',
                                                   'Relative_Humidity',
                                                   'Wind_Speed'].agg(['mean', 'median', 'min', 'max'])
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
    <tr>
      <th></th>
      <th colspan="4" halign="left">Dry_Bulb_Temperature</th>
      <th colspan="4" halign="left">Dew_Point_Temperature</th>
      <th colspan="4" halign="left">Relative_Humidity</th>
      <th colspan="4" halign="left">Wind_Speed</th>
    </tr>
    <tr>
      <th></th>
      <th>mean</th>
      <th>median</th>
      <th>min</th>
      <th>max</th>
      <th>mean</th>
      <th>median</th>
      <th>min</th>
      <th>max</th>
      <th>mean</th>
      <th>median</th>
      <th>min</th>
      <th>max</th>
      <th>mean</th>
      <th>median</th>
      <th>min</th>
      <th>max</th>
    </tr>
    <tr>
      <th>datetime</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2017-01-31</th>
      <td>0.688038</td>
      <td>0.3</td>
      <td>-3.9</td>
      <td>7.3</td>
      <td>0.048387</td>
      <td>0.00</td>
      <td>-0.2</td>
      <td>0.2</td>
      <td>-3.538575</td>
      <td>-1.30</td>
      <td>-35.5</td>
      <td>19.3</td>
      <td>0.086694</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-02-28</th>
      <td>0.658929</td>
      <td>0.3</td>
      <td>-3.9</td>
      <td>5.5</td>
      <td>0.046577</td>
      <td>0.00</td>
      <td>-0.6</td>
      <td>0.2</td>
      <td>-3.426488</td>
      <td>-1.50</td>
      <td>-26.3</td>
      <td>20.9</td>
      <td>0.096726</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-03-31</th>
      <td>0.939382</td>
      <td>0.5</td>
      <td>-3.4</td>
      <td>7.4</td>
      <td>0.059812</td>
      <td>0.10</td>
      <td>-0.1</td>
      <td>0.4</td>
      <td>-4.764651</td>
      <td>-2.10</td>
      <td>-31.3</td>
      <td>14.4</td>
      <td>0.141129</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-04-30</th>
      <td>0.975139</td>
      <td>0.4</td>
      <td>-3.9</td>
      <td>5.6</td>
      <td>0.067222</td>
      <td>0.10</td>
      <td>-0.2</td>
      <td>0.4</td>
      <td>-4.568889</td>
      <td>-1.70</td>
      <td>-26.5</td>
      <td>12.1</td>
      <td>0.232639</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-05-31</th>
      <td>1.083065</td>
      <td>0.4</td>
      <td>-2.9</td>
      <td>10.1</td>
      <td>0.053629</td>
      <td>0.05</td>
      <td>-0.4</td>
      <td>0.3</td>
      <td>-4.540457</td>
      <td>-1.70</td>
      <td>-39.3</td>
      <td>11.4</td>
      <td>0.057392</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-06-30</th>
      <td>0.738056</td>
      <td>0.4</td>
      <td>-2.5</td>
      <td>5.6</td>
      <td>0.047917</td>
      <td>0.00</td>
      <td>-0.3</td>
      <td>0.4</td>
      <td>-2.955694</td>
      <td>-1.25</td>
      <td>-21.2</td>
      <td>10.7</td>
      <td>0.127083</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-07-31</th>
      <td>1.007527</td>
      <td>0.5</td>
      <td>-2.5</td>
      <td>8.4</td>
      <td>0.024059</td>
      <td>0.00</td>
      <td>-0.2</td>
      <td>0.4</td>
      <td>-4.213575</td>
      <td>-1.55</td>
      <td>-38.4</td>
      <td>8.1</td>
      <td>0.072984</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-08-31</th>
      <td>0.838978</td>
      <td>0.6</td>
      <td>-3.6</td>
      <td>7.6</td>
      <td>0.023925</td>
      <td>0.00</td>
      <td>-0.2</td>
      <td>0.3</td>
      <td>-3.313306</td>
      <td>-2.00</td>
      <td>-28.5</td>
      <td>8.3</td>
      <td>0.077957</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-09-30</th>
      <td>0.842917</td>
      <td>0.5</td>
      <td>-2.4</td>
      <td>5.7</td>
      <td>0.020694</td>
      <td>0.00</td>
      <td>-0.2</td>
      <td>0.3</td>
      <td>-3.799861</td>
      <td>-2.10</td>
      <td>-23.6</td>
      <td>9.0</td>
      <td>0.211806</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-10-31</th>
      <td>0.821505</td>
      <td>0.6</td>
      <td>-4.0</td>
      <td>4.5</td>
      <td>0.044489</td>
      <td>0.00</td>
      <td>-0.1</td>
      <td>0.3</td>
      <td>-3.926478</td>
      <td>-2.10</td>
      <td>-19.8</td>
      <td>15.9</td>
      <td>0.190860</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-11-30</th>
      <td>0.692222</td>
      <td>0.4</td>
      <td>-3.4</td>
      <td>5.0</td>
      <td>0.029444</td>
      <td>0.00</td>
      <td>-0.7</td>
      <td>0.3</td>
      <td>-3.364306</td>
      <td>-1.80</td>
      <td>-20.3</td>
      <td>14.3</td>
      <td>0.068194</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2017-12-31</th>
      <td>0.629167</td>
      <td>0.2</td>
      <td>-3.7</td>
      <td>6.6</td>
      <td>0.052419</td>
      <td>0.00</td>
      <td>-0.1</td>
      <td>0.3</td>
      <td>-3.234140</td>
      <td>-1.00</td>
      <td>-30.8</td>
      <td>16.8</td>
      <td>0.043683</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
    </tr>
  </tbody>
</table>
</div>



### Data Visualisation (!!)
You have now made it through the more tedious number examination section, it's time for a data visualisation reward. Those of you using Ladybug in Grasshopper will recognise the beloved `Yearly Heatmap plot` which we use below to visualise `Dry Bulb Temperature` and `Relative Humidity`. We focus on these two metrics because we have already observed that the other two are not much affected by the urban model.

#### Dry Bulb Temperature

We notice that throughout the summer months the urban model pushes temperatures close to 30C much later into the evening and increases the occurence of more extreme temperatures. The diff plot also shows us an interesting seasonality pattern whereby the difference in temperature is least earlier in the morning during Summer.


```python
yearly_heatmap_plot(df_original, 'Dry_Bulb_Temperature', 'Original EPW Dry Bulb Temperature')
```


![png](/assets/img/urban-heat-island/original_epw_dry_bulb.png)



```python
yearly_heatmap_plot(df_urban, 'Dry_Bulb_Temperature', 'Urban Dry Bulb Temperature')
```


![png](/assets/img/urban-heat-island/urban_epw_dry_bulb.png)



```python
yearly_heatmap_plot(diff, 'Dry_Bulb_Temperature', 'Difference between Original and Urban Dry Bulb Temperature')
```


![png](/assets/img/urban-heat-island/diff_dry_bulb.png)


#### Relative Humidity

The impact of the Urban environment on `Relative Humidity` also follows a seasonal pattern whereby Summers are distinctly drier, especially once the sun has set.


```python
yearly_heatmap_plot(df_original, 'Relative_Humidity', 'Original EPW Relative Humidity')
```


![png](/assets/img/urban-heat-island/original_epw_rel_hum.png)



```python
yearly_heatmap_plot(df_urban, 'Relative_Humidity', 'Urban Relative Humidity')
```


![png](/assets/img/urban-heat-island/urban_epw_rel_hum.png)



```python
yearly_heatmap_plot(diff, 'Relative_Humidity', 'Difference between Original and Urban Relative Humidity')
```


![png](/assets/img/urban-heat-island/diff_rel_hum.png)


## Wrapping it up

And that's us done folks. Hope you enjoyed the tutorial and look forward to showing you more of this stuff in Part 2 of this tutorial series where we will be running a parametric study using Dragonfly to inform urban design.

---
title: "Creating a CF-1.6 timeSeries using pocean"
layout: notebook

---


IOOS recommends to data providers that their netCDF files follow the CF-1.6 standard. In this notebook we will create a [CF-1.6 compliant](http://cfconventions.org/latest.html) file that follows file that follows the [Discrete Sampling Geometries](http://cfconventions.org/Data/cf-conventions/cf-conventions-1.7/build/ch09.html) (DSG) of a `timeSeries` from a pandas DataFrame.

The `pocean` module can handle all the DSGs described in the CF-1.6 document: `point`, `timeSeries`, `trajectory`, `profile`, `timeSeriesProfile`, and `trajectoryProfile`. These DSGs array may be represented in the netCDF file as:

- **orthogonal multidimensional**: when the coordinates along the element axis of the features are identical;
- **incomplete multidimensional**: when the features within a collection do not all have the same number but space is not an issue and using longest feature to all features is convenient;
- **contiguous ragged**: can be used if the size of each feature is known;
- **indexed ragged**: stores the features interleaved along the sample dimension in the data variable.

Here we will use the orthogonal multidimensional array to represent time-series data from am hypothetical current meter. We'll use fake data for this example for convenience.

Our fake data represents a current meter located at 10 meters depth collected last week.

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
from datetime import datetime, timedelta

import numpy as np
import pandas as pd


x = np.arange(100, 110, 0.1)
start = datetime.now() - timedelta(days=7)

df = pd.DataFrame({
    'time': [start + timedelta(days=n) for n in range(len(x))],
    'longitude': -48.6256,
    'latitude': -27.5717,
    'depth': 10,
    'u': np.sin(x),
    'v': np.cos(x),
    'station': 'fake buoy',
})


df.tail()
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
      <th>time</th>
      <th>longitude</th>
      <th>latitude</th>
      <th>depth</th>
      <th>u</th>
      <th>v</th>
      <th>station</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>95</th>
      <td>2019-05-17 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.440129</td>
      <td>-0.897934</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>96</th>
      <td>2019-05-18 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.348287</td>
      <td>-0.937388</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>97</th>
      <td>2019-05-19 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.252964</td>
      <td>-0.967476</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>98</th>
      <td>2019-05-20 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.155114</td>
      <td>-0.987897</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>99</th>
      <td>2019-05-21 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.055714</td>
      <td>-0.998447</td>
      <td>fake buoy</td>
    </tr>
  </tbody>
</table>
</div>



Let's take a look at our fake data.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%matplotlib inline


import matplotlib.pyplot as plt
from oceans.plotting import stick_plot

q = stick_plot(
    [t.to_pydatetime() for t in df['time']],
    df['u'],
    df['v']
)

ref = 1
qk = plt.quiverkey(q, 0.1, 0.85, ref,
                  "%s m s$^{-1}$" % ref,
                  labelpos='N', coordinates='axes')

_ = plt.xticks(rotation=70)
```
<div class="warning" style="border:thin solid red">
    /home/filipe/miniconda3/envs/IOOS/lib/python3.7/site-
packages/pandas/plotting/_converter.py:129: FutureWarning: Using an implicitly
registered datetime converter for a matplotlib plotting method. The converter
was registered by pandas on import. Future versions of pandas will require you
to explicitly register matplotlib converters.

    To register the converters:
        >>> from pandas.plotting import register_matplotlib_converters
        >>> register_matplotlib_converters()
      warnings.warn(msg, FutureWarning)

</div>

![png](data:image/png;base64,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
)


`pocean.dsg` is relatively simple to use. The user must provide a DataFrame, like the one above, and a dictionary of attributes that maps to the data and adhere to the DSG conventions desired. 

Because we want the file to work seamlessly with ERDDAP we also added some ERDDAP specific attributes like `cdm_timeseries_variables`, and `subsetVariables`.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
attributes = {
    'global': {
        'title': 'Fake mooring',
        'summary': 'Vector current meter ADCP @ 10 m',
        'institution': 'Restaurant at the end of the universe',
        'cdm_timeseries_variables': 'station',
        'subsetVariables': 'depth',
    },
    'longitude': {
        'units': 'degrees_east',
        'standard_name': 'longitude',
    },
    'latitude': {
        'units': 'degrees_north',
        'standard_name': 'latitude',
    },
    'z': {
        'units': 'm',
        'standard_name': 'depth',
        'positive': 'down',
    },
    'u': {
        'units': 'm/s',
        'standard_name': 'eastward_sea_water_velocity',
    },
    'v': {
        'units': 'm/s',
        'standard_name': 'northward_sea_water_velocity',
    },
    'station': {
        'cf_role': 'timeseries_id'
    },
}
```

We also need to map the our data axes to [`pocean`'s defaults](https://github.com/pyoceans/pocean-core/blob/master/pocean/utils.py#L50-L59). This step is not needed if the data axes are already named like the default ones.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
axes = {'t': 'time', 'x': 'longitude', 'y': 'latitude', 'z': 'depth'}
```

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from pocean.dsg.timeseries.om import OrthogonalMultidimensionalTimeseries
from pocean.utils import downcast_dataframe


df = downcast_dataframe(df)  # safely cast depth np.int64 to np.int32
dsg = OrthogonalMultidimensionalTimeseries.from_dataframe(
    df,
    output='fake_buoy.nc',
    attributes=attributes,
    axes=axes,
)
```

The `OrthogonalMultidimensionalTimeseries` saves the DataFrame into a CF-1.6 TimeSeries DSG.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
!ncdump -h fake_buoy.nc
```
<div class="output_area"><div class="prompt"></div>
<pre>
    netcdf fake_buoy {
    dimensions:
    	station = 1 ;
    	time = 100 ;
    variables:
    	int crs ;
    	double time(time) ;
    		time:units = "seconds since 1990-01-01 00:00:00Z" ;
    		time:standard_name = "time" ;
    		time:axis = "T" ;
    	string station(station) ;
    		station:cf_role = "timeseries_id" ;
    		station:long_name = "station identifier" ;
    	double latitude(station) ;
    		latitude:axis = "Y" ;
    		latitude:units = "degrees_north" ;
    		latitude:standard_name = "latitude" ;
    	double longitude(station) ;
    		longitude:axis = "X" ;
    		longitude:units = "degrees_east" ;
    		longitude:standard_name = "longitude" ;
    	int depth(station) ;
    		depth:_FillValue = -9999 ;
    		depth:axis = "Z" ;
    	double u(station, time) ;
    		u:_FillValue = -9999.9 ;
    		u:units = "m/s" ;
    		u:standard_name = "eastward_sea_water_velocity" ;
    		u:coordinates = "time depth longitude latitude" ;
    	double v(station, time) ;
    		v:_FillValue = -9999.9 ;
    		v:units = "m/s" ;
    		v:standard_name = "northward_sea_water_velocity" ;
    		v:coordinates = "time depth longitude latitude" ;
    
    // global attributes:
    		:Conventions = "CF-1.6" ;
    		:date_created = "2019-02-18T20:14:00Z" ;
    		:featureType = "timeseries" ;
    		:cdm_data_type = "Timeseries" ;
    		:title = "Fake mooring" ;
    		:summary = "Vector current meter ADCP @ 10 m" ;
    		:institution = "Restaurant at the end of the universe" ;
    		:cdm_timeseries_variables = "station" ;
    		:subsetVariables = "depth" ;
    }

</pre>
</div>
 It also outputs the dsg object for inspection. Let us check a few things to see if our objects was created as expected. (Note that some of the metadata was "free" due t the built-in defaults in `pocean`.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
dsg.getncattr('featureType')
```




    'timeseries'



<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
type(dsg)
```




    pocean.dsg.timeseries.om.OrthogonalMultidimensionalTimeseries



In addition to standard `netCDF4-python` object `.variables` method `pocean`'s DSGs provides an "categorized" version of the variables in the `data_vars`, `ancillary_vars`, and the DSG axes methods.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
[(v.standard_name) for v in dsg.data_vars()]
```




    ['eastward_sea_water_velocity', 'northward_sea_water_velocity']



<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
dsg.axes('T')
```




    [<class 'netCDF4._netCDF4.Variable'>
     float64 time(time)
         units: seconds since 1990-01-01 00:00:00Z
         standard_name: time
         axis: T
     unlimited dimensions: 
     current shape = (100,)
     filling on, default _FillValue of 9.969209968386869e+36 used]



<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
dsg.axes('Z')
```




    [<class 'netCDF4._netCDF4.Variable'>
     int32 depth(station)
         _FillValue: -9999
         axis: Z
     unlimited dimensions: 
     current shape = (1,)
     filling on]



<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
dsg.vatts('station')
```




    {'cf_role': 'timeseries_id', 'long_name': 'station identifier'}



<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
dsg['station'][:]
```




    array(['fake buoy'], dtype=object)



<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
dsg.vatts('u')
```




    {'_FillValue': -9999.9,
     'units': 'm/s',
     'standard_name': 'eastward_sea_water_velocity',
     'coordinates': 'time depth longitude latitude'}



We can easily round-trip back to the pandas DataFrame object.

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
dsg.to_dataframe().head()
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
      <th>t</th>
      <th>x</th>
      <th>y</th>
      <th>z</th>
      <th>station</th>
      <th>u</th>
      <th>v</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2019-02-11 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.506366</td>
      <td>0.862319</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2019-02-12 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.417748</td>
      <td>0.908563</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2019-02-13 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.324956</td>
      <td>0.945729</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2019-02-14 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.228917</td>
      <td>0.973446</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2019-02-15 17:14:40.880577</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.130591</td>
      <td>0.991436</td>
    </tr>
  </tbody>
</table>
</div>



For more information on `pocean` please check the [API docs](https://pyoceans.github.io/pocean-core/docs/api/pocean.html).
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2018-02-27-pocean-timeSeries-demo.ipynb)
this notebook, or click [here](https://mybinder.org/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2018-02-27-pocean-timeSeries-demo.ipynb) to run a live instance of this notebook.
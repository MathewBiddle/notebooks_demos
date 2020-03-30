---
title: "The Boston Light Swim temperature analysis with Python"
layout: notebook

---

In the past we demonstrated how to perform a CSW catalog search with [`OWSLib`](https://ioos.github.io/notebooks_demos//notebooks/2016-12-19-exploring_csw),
and how to obtain near real-time data with [`pyoos`](https://ioos.github.io/notebooks_demos//notebooks/2016-10-12-fetching_data).
In this notebook we will use both to find all observations and model data around the Boston Harbor to access the sea water temperature.


This workflow is part of an example to advise swimmers of the annual [Boston lighthouse swim](http://bostonlightswim.org/) of the Boston Harbor water temperature conditions prior to the race. For more information regarding the workflow presented here see [Signell, Richard P.; Fernandes, Filipe; Wilcox, Kyle.   2016. "Dynamic Reusable Workflows for Ocean Science." *J. Mar. Sci. Eng.* 4, no. 4: 68](http://dx.doi.org/10.3390/jmse4040068).

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import warnings

# Suppresing warnings for a "pretty output."
warnings.simplefilter("ignore")
```

This notebook is quite big and complex,
so to help us keep things organized we'll define a cell with the most important options and switches.

Below we can define the date,
bounding box, phenomena `SOS` and `CF` names and units,
and the catalogs we will search.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%%writefile config.yaml

# Specify a YYYY-MM-DD hh:mm:ss date or integer day offset.
# If both start and stop are offsets they will be computed relative to datetime.today() at midnight.
# Use the dates commented below to reproduce the last Boston Light Swim event forecast.
date:
    start: -5 # 2016-8-16 00:00:00
    stop: +4 # 2016-8-29 00:00:00

run_name: 'latest'

# Boston harbor.
region:
    bbox: [-71.3, 42.03, -70.57, 42.63]
    # Try the bounding box below to see how the notebook will behave for a different region.
    #bbox: [-74.5, 40, -72., 41.5]
    crs: 'urn:ogc:def:crs:OGC:1.3:CRS84'

sos_name: 'sea_water_temperature'

cf_names:
    - sea_water_temperature
    - sea_surface_temperature
    - sea_water_potential_temperature
    - equivalent_potential_temperature
    - sea_water_conservative_temperature
    - pseudo_equivalent_potential_temperature

units: 'celsius'

catalogs:
    - https://data.ioos.us/csw
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Overwriting config.yaml

</pre>
</div>
We'll print some of the search configuration options along the way to keep track of them.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
import os
import shutil
from datetime import datetime

from ioos_tools.ioos import parse_config

config = parse_config("config.yaml")

# Saves downloaded data into a temporary directory.
save_dir = os.path.abspath(config["run_name"])
if os.path.exists(save_dir):
    shutil.rmtree(save_dir)
os.makedirs(save_dir)

fmt = "{:*^64}".format
print(fmt("Saving data inside directory {}".format(save_dir)))
print(fmt(" Run information "))
print("Run date: {:%Y-%m-%d %H:%M:%S}".format(datetime.utcnow()))
print("Start: {:%Y-%m-%d %H:%M:%S}".format(config["date"]["start"]))
print("Stop: {:%Y-%m-%d %H:%M:%S}".format(config["date"]["stop"]))
print(
    "Bounding box: {0:3.2f}, {1:3.2f},"
    "{2:3.2f}, {3:3.2f}".format(*config["region"]["bbox"])
)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Saving data inside directory /home/filipe/IOOS/notebooks_demos/notebooks/latest
    *********************** Run information ************************
    Run date: 2019-07-03 16:59:01
    Start: 2019-06-28 00:00:00
    Stop: 2019-07-07 00:00:00
    Bounding box: -71.30, 42.03,-70.57, 42.63

</pre>
</div>
We already created an `OWSLib.fes` filter [before](https://ioos.github.io/notebooks_demos//notebooks/2016-12-19-exploring_csw).
The main difference here is that we do not want the atmosphere model data,
so we are filtering out all the `GRIB-2` data format.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
def make_filter(config):
    from owslib import fes
    from ioos_tools.ioos import fes_date_filter

    kw = dict(
        wildCard="*", escapeChar="\\", singleChar="?", propertyname="apiso:AnyText"
    )

    or_filt = fes.Or(
        [fes.PropertyIsLike(literal=("*%s*" % val), **kw) for val in config["cf_names"]]
    )

    not_filt = fes.Not([fes.PropertyIsLike(literal="GRIB-2", **kw)])

    begin, end = fes_date_filter(config["date"]["start"], config["date"]["stop"])
    bbox_crs = fes.BBox(config["region"]["bbox"], crs=config["region"]["crs"])
    filter_list = [fes.And([bbox_crs, begin, end, or_filt, not_filt])]
    return filter_list


filter_list = make_filter(config)
```

In the cell below we ask the catalog for all the returns that match the filter and have an OPeNDAP endpoint.

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from ioos_tools.ioos import get_csw_records, service_urls
from owslib.csw import CatalogueServiceWeb

dap_urls = []
print(fmt(" Catalog information "))
for endpoint in config["catalogs"]:
    print("URL: {}".format(endpoint))
    try:
        csw = CatalogueServiceWeb(endpoint, timeout=120)
    except Exception as e:
        print("{}".format(e))
        continue
    csw = get_csw_records(csw, filter_list, esn="full")
    OPeNDAP = service_urls(csw.records, identifier="OPeNDAP:OPeNDAP")
    odp = service_urls(
        csw.records, identifier="urn:x-esri:specification:ServiceType:odp:url"
    )
    dap = OPeNDAP + odp
    dap_urls.extend(dap)

    print("Number of datasets available: {}".format(len(csw.records.keys())))

    for rec, item in csw.records.items():
        print("{}".format(item.title))
    if dap:
        print(fmt(" DAP "))
        for url in dap:
            print("{}.html".format(url))
    print("\n")

# Get only unique endpoints.
dap_urls = list(set(dap_urls))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Catalog information **********************
    URL: https://data.ioos.us/csw
    Number of datasets available: 30
    NECOFS Massachusetts (FVCOM) - Massachusetts Coastal - Latest Forecast
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 ACCELEROMETER Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 CTD1m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 CTD20m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 MET Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 OPTICS3m Massachusetts Bay
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 OPTODE51m Massachusetts Bay
    NOAA Coral Reef Watch Operational Daily Near-Real-Time Global 5-km Satellite Coral Bleaching Monitoring Products
    ROMS doppio Real-Time Operational PSAS Forecast System Version 1 FMRC Averages
    ROMS doppio Real-Time Operational PSAS Forecast System Version 1 FMRC History
    A01 Accelerometer - Waves
    A01 Directional Waves
    A01 Met - Meteorology
    A01 Optics - Chlorophyll / Turbidity
    A01 Optode - Oxygen
    A01 SBE16 Oxygen
    A01 Sbe37 - CTD
    BOSTON 16 NM East of Boston, MA
    Buoy A01 - Massachusetts Bay
    Department of Physical Oceanography, School of Marine Sciences, University of Maine A01 Accelerometer Buoy Sensor
    Department of Physical Oceanography, School of Marine Sciences, University of Maine A01 Met Buoy Sensor
    Department of Physical Oceanography, School of Marine Sciences, University of Maine A01 Optode 51m Buoy Sensor
    Department of Physical Oceanography, School of Marine Sciences, University of Maine A01 Sbe37 1m Buoy Sensor
    Department of Physical Oceanography, School of Marine Sciences, University of Maine A01 Sbe37 20m Buoy Sensor
    Directional wave and sea surface temperature measurements collected in situ by Datawell DWR-M3 directional buoy located near CLATSOP SPIT, OR from 2018/02/07 22:03:45 to 2019/07/02 18:00:25.
    Directional wave and sea surface temperature measurements collected in situ by Datawell DWR-M3 directional buoy located near GRAYS HARBOR, WA from 2018/06/28 19:03:45 to 2019/07/02 18:00:25.
    Directional wave and sea surface temperature measurements collected in situ by Datawell DWR-M3 directional buoy located near KAUMALAPAU SOUTHWEST, LANAI, HI from 2018/10/02 16:03:45 to 2019/07/02 17:30:25.
    Latest CDIP real-time buoy observations, 3-day aggregate
    NDBC Standard Meteorological Buoy Data, 1970-present
    NECOFS (FVCOM) - Scituate - Latest Forecast
    ***************************** DAP ******************************
    http://oos.soest.hawaii.edu/thredds/dodsC/hioos/satellite/dhw_5km.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/doppio/2017_da/avg/Averages_Best.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/doppio/2017_da/his/History_Best.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/036p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/162p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/239p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/latest_gudb_3day.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/Accelerometer/HistoricRealtime.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/CTD1m/HistoricRealtime.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/CTD20m/HistoricRealtime.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/Met/HistoricRealtime.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/OPTICS_S3m/HistoricRealtime.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/OPTODE51m/HistoricRealtime.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.accelerometer.realtime.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.met.realtime.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.optode.realtime.51m.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.1m.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.20m.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html
    
    

</pre>
</div>
We found some models, and observations from NERACOOS there.
However, we do know that there are some buoys from NDBC and CO-OPS available too.
Also, those NERACOOS observations seem to be from a [CTD](http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/CTD1m/HistoricRealtime/Agg.ncml.html) mounted at 65 meters below the sea surface. Rendering them useless from our purpose.

So let's use the catalog only for the models by filtering the observations with `is_station` below.
And we'll rely `CO-OPS` and `NDBC` services for the observations.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
from ioos_tools.ioos import is_station
from timeout_decorator import TimeoutError

# Filter out some station endpoints.
non_stations = []
for url in dap_urls:
    url = f"{url}#fillmismatch"
    try:
        if not is_station(url):
            non_stations.append(url)
    except (IOError, OSError, RuntimeError, TimeoutError) as e:
        print("Could not access URL {}.html\n{!r}".format(url, e))

dap_urls = non_stations

print(fmt(" Filtered DAP "))
for url in dap_urls:
    print("{}.html".format(url))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ************************* Filtered DAP *************************
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/doppio/2017_da/his/History_Best#fillmismatch.html
    http://oos.soest.hawaii.edu/thredds/dodsC/hioos/satellite/dhw_5km#fillmismatch.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.1m.nc#fillmismatch.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.met.realtime.nc#fillmismatch.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.accelerometer.realtime.nc#fillmismatch.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/doppio/2017_da/avg/Averages_Best#fillmismatch.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.20m.nc#fillmismatch.html
    http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.optode.realtime.51m.nc#fillmismatch.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc#fillmismatch.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc#fillmismatch.html

</pre>
</div>
Now we can use `pyoos` collectors for `NdbcSos`,

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
from pyoos.collectors.ndbc.ndbc_sos import NdbcSos

collector_ndbc = NdbcSos()

collector_ndbc.set_bbox(config["region"]["bbox"])
collector_ndbc.end_time = config["date"]["stop"]
collector_ndbc.start_time = config["date"]["start"]
collector_ndbc.variables = [config["sos_name"]]

ofrs = collector_ndbc.server.offerings
title = collector_ndbc.server.identification.title
print(fmt(" NDBC Collector offerings "))
print("{}: {} offerings".format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ******************* NDBC Collector offerings *******************
    National Data Buoy Center SOS: 1062 offerings

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
import pandas as pd
from ioos_tools.ioos import collector2table

ndbc = collector2table(
    collector=collector_ndbc, config=config, col="sea_water_temperature (C)"
)

if ndbc:
    data = dict(
        station_name=[s._metadata.get("station_name") for s in ndbc],
        station_code=[s._metadata.get("station_code") for s in ndbc],
        sensor=[s._metadata.get("sensor") for s in ndbc],
        lon=[s._metadata.get("lon") for s in ndbc],
        lat=[s._metadata.get("lat") for s in ndbc],
        depth=[s._metadata.get("depth") for s in ndbc],
    )

table = pd.DataFrame(data).set_index("station_code")
table
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
      <th>station_name</th>
      <th>sensor</th>
      <th>lon</th>
      <th>lat</th>
      <th>depth</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44013</th>
      <td>BOSTON 16 NM East of Boston, MA</td>
      <td>urn:ioos:sensor:wmo:44013::watertemp1</td>
      <td>-70.651</td>
      <td>42.346</td>
      <td>0.6</td>
    </tr>
  </tbody>
</table>
</div>



and `CoopsSos`.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
from pyoos.collectors.coops.coops_sos import CoopsSos

collector_coops = CoopsSos()

collector_coops.set_bbox(config["region"]["bbox"])
collector_coops.end_time = config["date"]["stop"]
collector_coops.start_time = config["date"]["start"]
collector_coops.variables = [config["sos_name"]]

ofrs = collector_coops.server.offerings
title = collector_coops.server.identification.title
print(fmt(" Collector offerings "))
print("{}: {} offerings".format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Collector offerings **********************
    NOAA.NOS.CO-OPS SOS: 1227 offerings

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
coops = collector2table(
    collector=collector_coops, config=config, col="sea_water_temperature (C)"
)

if coops:
    data = dict(
        station_name=[s._metadata.get("station_name") for s in coops],
        station_code=[s._metadata.get("station_code") for s in coops],
        sensor=[s._metadata.get("sensor") for s in coops],
        lon=[s._metadata.get("lon") for s in coops],
        lat=[s._metadata.get("lat") for s in coops],
        depth=[s._metadata.get("depth") for s in coops],
    )

table = pd.DataFrame(data).set_index("station_code")
table
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
      <th>station_name</th>
      <th>sensor</th>
      <th>lon</th>
      <th>lat</th>
      <th>depth</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44013</th>
      <td>BOSTON 16 NM East of Boston, MA</td>
      <td>urn:ioos:sensor:wmo:44013::watertemp1</td>
      <td>-70.651</td>
      <td>42.346</td>
      <td>0.6</td>
    </tr>
  </tbody>
</table>
</div>



We will join all the observations into an uniform series, interpolated to 1-hour interval, for the model-data comparison.

This step is necessary because the observations can be 7 or 10 minutes resolution,
while the models can be 30 to 60 minutes.

<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
data = ndbc + coops

index = pd.date_range(
    start=config["date"]["start"].replace(tzinfo=None),
    end=config["date"]["stop"].replace(tzinfo=None),
    freq="1H",
)

# Preserve metadata with `reindex`.
observations = []
for series in data:
    _metadata = series._metadata
    series.index = series.index.tz_localize(None)
    series.index = series.index.tz_localize(None)
    obs = series.reindex(index=index, limit=1, method="nearest")
    obs._metadata = _metadata
    observations.append(obs)
```

In this next cell we will save the data for quicker access later.

<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
import iris
from ioos_tools.tardis import series2cube

attr = dict(
    featureType="timeSeries",
    Conventions="CF-1.6",
    standard_name_vocabulary="CF-1.6",
    cdm_data_type="Station",
    comment="Data from http://opendap.co-ops.nos.noaa.gov",
)


cubes = iris.cube.CubeList([series2cube(obs, attr=attr) for obs in observations])

outfile = os.path.join(save_dir, "OBS_DATA.nc")
iris.save(cubes, outfile)
```

Taking a quick look at the observations:

<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
%matplotlib inline

ax = pd.concat(data).plot(figsize=(11, 2.25))
```


![png](data:image/png;base64,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
)


Now it is time to loop the models we found above,

<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
from ioos_tools.ioos import get_model_name
from ioos_tools.tardis import get_surface, is_model, proc_cube, quick_load_cubes
from iris.exceptions import ConstraintMismatchError, CoordinateNotFoundError, MergeError

print(fmt(" Models "))
cubes = dict()
for k, url in enumerate(dap_urls):
    print("\n[Reading url {}/{}]: {}".format(k + 1, len(dap_urls), url))
    try:
        cube = quick_load_cubes(url, config["cf_names"], callback=None, strict=True)
        if is_model(cube):
            cube = proc_cube(
                cube,
                bbox=config["region"]["bbox"],
                time=(config["date"]["start"], config["date"]["stop"]),
                units=config["units"],
            )
        else:
            print("[Not model data]: {}".format(url))
            continue
        cube = get_surface(cube)
        mod_name = get_model_name(url)
        cubes.update({mod_name: cube})
    except (
        RuntimeError,
        ValueError,
        ConstraintMismatchError,
        CoordinateNotFoundError,
        IndexError,
    ) as e:
        print("Cannot get cube for: {}\n{}".format(url, e))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    **************************** Models ****************************
    
    [Reading url 1/10]: http://tds.marine.rutgers.edu/thredds/dodsC/roms/doppio/2017_da/his/History_Best#fillmismatch
    
    [Reading url 2/10]: http://oos.soest.hawaii.edu/thredds/dodsC/hioos/satellite/dhw_5km#fillmismatch
    
    [Reading url 3/10]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.1m.nc#fillmismatch
    [Not model data]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.1m.nc#fillmismatch
    
    [Reading url 4/10]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.met.realtime.nc#fillmismatch
    Cannot get cube for: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.met.realtime.nc#fillmismatch
    Cannot find ['sea_water_temperature', 'sea_surface_temperature', 'sea_water_potential_temperature', 'equivalent_potential_temperature', 'sea_water_conservative_temperature', 'pseudo_equivalent_potential_temperature'] in http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.met.realtime.nc#fillmismatch.
    
    [Reading url 5/10]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.accelerometer.realtime.nc#fillmismatch
    Cannot get cube for: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.accelerometer.realtime.nc#fillmismatch
    Cannot find ['sea_water_temperature', 'sea_surface_temperature', 'sea_water_potential_temperature', 'equivalent_potential_temperature', 'sea_water_conservative_temperature', 'pseudo_equivalent_potential_temperature'] in http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.accelerometer.realtime.nc#fillmismatch.
    
    [Reading url 6/10]: http://tds.marine.rutgers.edu/thredds/dodsC/roms/doppio/2017_da/avg/Averages_Best#fillmismatch
    
    [Reading url 7/10]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.20m.nc#fillmismatch
    [Not model data]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.sbe37.realtime.20m.nc#fillmismatch
    
    [Reading url 8/10]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.optode.realtime.51m.nc#fillmismatch
    [Not model data]: http://www.neracoos.org/thredds/dodsC/UMO/Realtime/SOS/A01/DSG_A0141.optode.realtime.51m.nc#fillmismatch
    
    [Reading url 9/10]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc#fillmismatch
    
    [Reading url 10/10]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc#fillmismatch

</pre>
</div>
Next, we will match them with the nearest observed time-series. The `max_dist=0.08` is in degrees, that is roughly 8 kilometers.

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
import iris
from ioos_tools.tardis import (
    add_station,
    ensure_timeseries,
    get_nearest_water,
    make_tree,
    remove_ssh,
)
from iris.pandas import as_series

for mod_name, cube in cubes.items():
    fname = "{}.nc".format(mod_name)
    fname = os.path.join(save_dir, fname)
    print(fmt(" Downloading to file {} ".format(fname)))
    try:
        tree, lon, lat = make_tree(cube)
    except CoordinateNotFoundError:
        print("Cannot make KDTree for: {}".format(mod_name))
        continue
    # Get model series at observed locations.
    raw_series = dict()
    for obs in observations:
        obs = obs._metadata
        station = obs["station_code"]
        try:
            kw = dict(k=10, max_dist=0.08, min_var=0.01)
            args = cube, tree, obs["lon"], obs["lat"]
            try:
                series, dist, idx = get_nearest_water(*args, **kw)
            except RuntimeError as e:
                print("Cannot download {!r}.\n{}".format(cube, e))
                series = None
        except ValueError:
            status = "No Data"
            print("[{}] {}".format(status, obs["station_name"]))
            continue
        if not series:
            status = "Land   "
        else:
            raw_series.update({station: series})
            series = as_series(series)
            status = "Water  "
        print("[{}] {}".format(status, obs["station_name"]))
    if raw_series:  # Save cube.
        for station, cube in raw_series.items():
            cube = add_station(cube, station)
            cube = remove_ssh(cube)
        try:
            cube = iris.cube.CubeList(raw_series.values()).merge_cube()
        except MergeError as e:
            print(e)
        ensure_timeseries(cube)
        try:
            iris.save(cube, fname)
        except AttributeError:
            # FIXME: we should patch the bad attribute instead of removing everything.
            cube.attributes = {}
            iris.save(cube, fname)
        del cube
    print("Finished processing [{}]".format(mod_name))
```
<div class="output_area"><div class="prompt"></div>
<pre>
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/roms_doppio_2017_da-History_Best#fillmismatch.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [roms_doppio_2017_da-History_Best#fillmismatch]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/hioos_satellite-dhw_5km#fillmismatch.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [hioos_satellite-dhw_5km#fillmismatch]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/roms_doppio_2017_da_avg-Averages_Best#fillmismatch.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [roms_doppio_2017_da_avg-Averages_Best#fillmismatch]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc#fillmismatch.nc 
    [No Data] BOSTON 16 NM East of Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc#fillmismatch]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc#fillmismatch.nc 
    Cannot download <iris 'Cube' of sea_water_potential_temperature / (degrees_C) (time: 144; -- : 31358)>.
    NetCDF: file not found
    [Land   ] BOSTON 16 NM East of Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc#fillmismatch]

</pre>
</div>
Now it is possible to compute some simple comparison metrics. First we'll calculate the model mean bias:

$$ \text{MB} = \mathbf{\overline{m}} - \mathbf{\overline{o}}$$

<div class="prompt input_prompt">
In&nbsp;[16]:
</div>

```python
from ioos_tools.ioos import stations_keys


def rename_cols(df, config):
    cols = stations_keys(config, key="station_name")
    return df.rename(columns=cols)
```

<div class="prompt input_prompt">
In&nbsp;[17]:
</div>

```python
from ioos_tools.ioos import load_ncs
from ioos_tools.skill_score import apply_skill, mean_bias

dfs = load_ncs(config)

df = apply_skill(dfs, mean_bias, remove_mean=False, filter_tides=False)
skill_score = dict(mean_bias=df.to_dict())

# Filter out stations with no valid comparison.
df.dropna(how="all", axis=1, inplace=True)
df = df.applymap("{:.2f}".format).replace("nan", "--")
```

And the root mean squared rrror of the deviations from the mean:
$$ \text{CRMS} = \sqrt{\left(\mathbf{m'} - \mathbf{o'}\right)^2}$$

where: $\mathbf{m'} = \mathbf{m} - \mathbf{\overline{m}}$ and $\mathbf{o'} = \mathbf{o} - \mathbf{\overline{o}}$

<div class="prompt input_prompt">
In&nbsp;[18]:
</div>

```python
from ioos_tools.skill_score import rmse

dfs = load_ncs(config)

df = apply_skill(dfs, rmse, remove_mean=True, filter_tides=False)
skill_score["rmse"] = df.to_dict()

# Filter out stations with no valid comparison.
df.dropna(how="all", axis=1, inplace=True)
df = df.applymap("{:.2f}".format).replace("nan", "--")
```

The next 2 cells make the scores "pretty" for plotting.

<div class="prompt input_prompt">
In&nbsp;[19]:
</div>

```python
import pandas as pd

# Stringfy keys.
for key in skill_score.keys():
    skill_score[key] = {str(k): v for k, v in skill_score[key].items()}

mean_bias = pd.DataFrame.from_dict(skill_score["mean_bias"])
mean_bias = mean_bias.applymap("{:.2f}".format).replace("nan", "--")

skill_score = pd.DataFrame.from_dict(skill_score["rmse"])
skill_score = skill_score.applymap("{:.2f}".format).replace("nan", "--")
```

<div class="prompt input_prompt">
In&nbsp;[20]:
</div>

```python
import folium
from ioos_tools.ioos import get_coordinates


def make_map(bbox, **kw):
    line = kw.pop("line", True)
    layers = kw.pop("layers", True)
    zoom_start = kw.pop("zoom_start", 5)

    lon = (bbox[0] + bbox[2]) / 2
    lat = (bbox[1] + bbox[3]) / 2
    m = folium.Map(
        width="100%", height="100%", location=[lat, lon], zoom_start=zoom_start
    )

    if layers:
        url = "http://oos.soest.hawaii.edu/thredds/wms/hioos/satellite/dhw_5km"
        w = folium.WmsTileLayer(
            url,
            name="Sea Surface Temperature",
            fmt="image/png",
            layers="CRW_SST",
            attr="PacIOOS TDS",
            overlay=True,
            transparent=True,
        )
        w.add_to(m)

    if line:
        p = folium.PolyLine(
            get_coordinates(bbox), color="#FF0000", weight=2, opacity=0.9,
        )
        p.add_to(m)
    return m
```

<div class="prompt input_prompt">
In&nbsp;[21]:
</div>

```python
bbox = config["region"]["bbox"]

m = make_map(bbox, zoom_start=11, line=True, layers=True)
```

The cells from `[20]` to `[25]` create a [`folium`](https://github.com/python-visualization/folium) map with [`bokeh`](http://bokeh.pydata.org/en/latest/) for the time-series at the observed points.

Note that we did mark the nearest model cell location used in the comparison.

<div class="prompt input_prompt">
In&nbsp;[22]:
</div>

```python
all_obs = stations_keys(config)

from glob import glob
from operator import itemgetter

import iris
from folium.plugins import MarkerCluster

iris.FUTURE.netcdf_promote = True

big_list = []
for fname in glob(os.path.join(save_dir, "*.nc")):
    if "OBS_DATA" in fname:
        continue
    cube = iris.load_cube(fname)
    model = os.path.split(fname)[1].split("-")[-1].split(".")[0]
    lons = cube.coord(axis="X").points
    lats = cube.coord(axis="Y").points
    stations = cube.coord("station_code").points
    models = [model] * lons.size
    lista = zip(models, lons.tolist(), lats.tolist(), stations.tolist())
    big_list.extend(lista)

big_list.sort(key=itemgetter(3))
df = pd.DataFrame(big_list, columns=["name", "lon", "lat", "station"])
df.set_index("station", drop=True, inplace=True)
groups = df.groupby(df.index)


locations, popups = [], []
for station, info in groups:
    sta_name = all_obs[station]
    for lat, lon, name in zip(info.lat, info.lon, info.name):
        locations.append([lat, lon])
        popups.append(
            "[{}]: {}".format(name.rstrip("fillmismatch").rstrip("#"), sta_name)
        )

MarkerCluster(locations=locations, popups=popups, name="Cluster").add_to(m)
```

Here we use a dictionary with some models we expect to find so we can create a better legend for the plots. If any new models are found, we will use its filename in the legend as a default until we can go back and add a short name to our library.

<div class="prompt input_prompt">
In&nbsp;[23]:
</div>

```python
titles = {
    "coawst_4_use_best": "COAWST_4",
    "global": "HYCOM",
    "NECOFS_GOM3_FORECAST": "NECOFS_GOM3",
    "NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST": "NECOFS_MassBay",
    "OBS_DATA": "Observations",
}
```

<div class="prompt input_prompt">
In&nbsp;[24]:
</div>

```python
from itertools import cycle

from bokeh.embed import file_html
from bokeh.models import HoverTool
from bokeh.palettes import Category20
from bokeh.plotting import figure
from bokeh.resources import CDN
from folium import IFrame

# Plot defaults.
colors = Category20[20]
colorcycler = cycle(colors)
tools = "pan,box_zoom,reset"
width, height = 750, 250


def make_plot(df, station):
    p = figure(
        toolbar_location="above",
        x_axis_type="datetime",
        width=width,
        height=height,
        tools=tools,
        title=str(station),
    )
    for column, series in df.iteritems():
        series.dropna(inplace=True)
        if not series.empty:
            if "OBS_DATA" not in column:
                bias = mean_bias[str(station)][column]
                skill = skill_score[str(station)][column]
                line_color = next(colorcycler)
                kw = dict(alpha=0.65, line_color=line_color)
            else:
                skill = bias = "NA"
                kw = dict(alpha=1, color="crimson")
            legend = f"{titles.get(column, column)}"
            legend = legend.rstrip("fillmismatch").rstrip("#")
            line = p.line(
                x=series.index,
                y=series.values,
                legend=legend,
                line_width=5,
                line_cap="round",
                line_join="round",
                **kw,
            )
            p.add_tools(
                HoverTool(
                    tooltips=[
                        ("Name", "{}".format(titles.get(column, column))),
                        ("Bias", bias),
                        ("Skill", skill),
                    ],
                    renderers=[line],
                )
            )
    return p


def make_marker(p, station):
    lons = stations_keys(config, key="lon")
    lats = stations_keys(config, key="lat")

    lon, lat = lons[station], lats[station]
    html = file_html(p, CDN, station)
    iframe = IFrame(html, width=width + 40, height=height + 80)

    popup = folium.Popup(iframe, max_width=2650)
    icon = folium.Icon(color="green", icon="stats")
    marker = folium.Marker(location=[lat, lon], popup=popup, icon=icon)
    return marker
```

<div class="prompt input_prompt">
In&nbsp;[25]:
</div>

```python
dfs = load_ncs(config)

for station in dfs:
    sta_name = all_obs[station]
    df = dfs[station]
    if df.empty:
        continue
    p = make_plot(df, station)
    marker = make_marker(p, station)
    marker.add_to(m)

folium.LayerControl().add_to(m)

m
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgCiAgICAgICAgPHNjcmlwdD4KICAgICAgICAgICAgTF9OT19UT1VDSCA9IGZhbHNlOwogICAgICAgICAgICBMX0RJU0FCTEVfM0QgPSBmYWxzZTsKICAgICAgICA8L3NjcmlwdD4KICAgIAogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY2RuLmpzZGVsaXZyLm5ldC9ucG0vbGVhZmxldEAxLjQuMC9kaXN0L2xlYWZsZXQuanMiPjwvc2NyaXB0PgogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY29kZS5qcXVlcnkuY29tL2pxdWVyeS0xLjEyLjQubWluLmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9qcy9ib290c3RyYXAubWluLmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9MZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy8yLjAuMi9sZWFmbGV0LmF3ZXNvbWUtbWFya2Vycy5qcyI+PC9zY3JpcHQ+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vY2RuLmpzZGVsaXZyLm5ldC9ucG0vbGVhZmxldEAxLjQuMC9kaXN0L2xlYWZsZXQuY3NzIi8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vYm9vdHN0cmFwLzMuMi4wL2Nzcy9ib290c3RyYXAubWluLmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9mb250LWF3ZXNvbWUvNC42LjMvY3NzL2ZvbnQtYXdlc29tZS5taW4uY3NzIi8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL0xlYWZsZXQuYXdlc29tZS1tYXJrZXJzLzIuMC4yL2xlYWZsZXQuYXdlc29tZS1tYXJrZXJzLmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2Nkbi5naXRoYWNrLmNvbS9weXRob24tdmlzdWFsaXphdGlvbi9mb2xpdW0vbWFzdGVyL2ZvbGl1bS90ZW1wbGF0ZXMvbGVhZmxldC5hd2Vzb21lLnJvdGF0ZS5jc3MiLz4KICAgIDxzdHlsZT5odG1sLCBib2R5IHt3aWR0aDogMTAwJTtoZWlnaHQ6IDEwMCU7bWFyZ2luOiAwO3BhZGRpbmc6IDA7fTwvc3R5bGU+CiAgICA8c3R5bGU+I21hcCB7cG9zaXRpb246YWJzb2x1dGU7dG9wOjA7Ym90dG9tOjA7cmlnaHQ6MDtsZWZ0OjA7fTwvc3R5bGU+CiAgICAKICAgICAgICAgICAgPG1ldGEgbmFtZT0idmlld3BvcnQiIGNvbnRlbnQ9IndpZHRoPWRldmljZS13aWR0aCwKICAgICAgICAgICAgICAgIGluaXRpYWwtc2NhbGU9MS4wLCBtYXhpbXVtLXNjYWxlPTEuMCwgdXNlci1zY2FsYWJsZT1ubyIgLz4KICAgICAgICAgICAgPHN0eWxlPgogICAgICAgICAgICAgICAgI21hcF8xZDc3NDhjYmFiYWQ0N2MzYWQzNDFkYzViZGNlMTBiZCB7CiAgICAgICAgICAgICAgICAgICAgcG9zaXRpb246IHJlbGF0aXZlOwogICAgICAgICAgICAgICAgICAgIHdpZHRoOiAxMDAuMCU7CiAgICAgICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICAgICAgbGVmdDogMC4wJTsKICAgICAgICAgICAgICAgICAgICB0b3A6IDAuMCU7CiAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgIDwvc3R5bGU+CiAgICAgICAgCiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvbGVhZmxldC5tYXJrZXJjbHVzdGVyLzEuMS4wL2xlYWZsZXQubWFya2VyY2x1c3Rlci5qcyI+PC9zY3JpcHQ+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjEuMC9NYXJrZXJDbHVzdGVyLmNzcyIvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMS4xLjAvTWFya2VyQ2x1c3Rlci5EZWZhdWx0LmNzcyIvPgo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgICAgICAgICA8ZGl2IGNsYXNzPSJmb2xpdW0tbWFwIiBpZD0ibWFwXzFkNzc0OGNiYWJhZDQ3YzNhZDM0MWRjNWJkY2UxMGJkIiA+PC9kaXY+CiAgICAgICAgCjwvYm9keT4KPHNjcmlwdD4gICAgCiAgICAKICAgICAgICAgICAgdmFyIG1hcF8xZDc3NDhjYmFiYWQ0N2MzYWQzNDFkYzViZGNlMTBiZCA9IEwubWFwKAogICAgICAgICAgICAgICAgIm1hcF8xZDc3NDhjYmFiYWQ0N2MzYWQzNDFkYzViZGNlMTBiZCIsCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgY2VudGVyOiBbNDIuMzMsIC03MC45MzVdLAogICAgICAgICAgICAgICAgICAgIGNyczogTC5DUlMuRVBTRzM4NTcsCiAgICAgICAgICAgICAgICAgICAgem9vbTogMTEsCiAgICAgICAgICAgICAgICAgICAgem9vbUNvbnRyb2w6IHRydWUsCiAgICAgICAgICAgICAgICAgICAgcHJlZmVyQ2FudmFzOiBmYWxzZSwKICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgKTsKCiAgICAgICAgICAgIAoKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgdGlsZV9sYXllcl9lZDcxYmRmOTE4Zjc0NGM0YWVkM2M0MWNkYmJjNzRiMCA9IEwudGlsZUxheWVyKAogICAgICAgICAgICAgICAgImh0dHBzOi8ve3N9LnRpbGUub3BlbnN0cmVldG1hcC5vcmcve3p9L3t4fS97eX0ucG5nIiwKICAgICAgICAgICAgICAgIHsiYXR0cmlidXRpb24iOiAiRGF0YSBieSBcdTAwMjZjb3B5OyBcdTAwM2NhIGhyZWY9XCJodHRwOi8vb3BlbnN0cmVldG1hcC5vcmdcIlx1MDAzZU9wZW5TdHJlZXRNYXBcdTAwM2MvYVx1MDAzZSwgdW5kZXIgXHUwMDNjYSBocmVmPVwiaHR0cDovL3d3dy5vcGVuc3RyZWV0bWFwLm9yZy9jb3B5cmlnaHRcIlx1MDAzZU9EYkxcdTAwM2MvYVx1MDAzZS4iLCAiZGV0ZWN0UmV0aW5hIjogZmFsc2UsICJtYXhOYXRpdmVab29tIjogMTgsICJtYXhab29tIjogMTgsICJtaW5ab29tIjogMCwgIm5vV3JhcCI6IGZhbHNlLCAib3BhY2l0eSI6IDEsICJzdWJkb21haW5zIjogImFiYyIsICJ0bXMiOiBmYWxzZX0KICAgICAgICAgICAgKS5hZGRUbyhtYXBfMWQ3NzQ4Y2JhYmFkNDdjM2FkMzQxZGM1YmRjZTEwYmQpOwogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBtYWNyb19lbGVtZW50XzRkMDc3ZWQ3Mjc4YTQ3MzM4MjBkODM0NmY2YzE3ZjI1ID0gTC50aWxlTGF5ZXIud21zKAogICAgICAgICAgICAgICAgImh0dHA6Ly9vb3Muc29lc3QuaGF3YWlpLmVkdS90aHJlZGRzL3dtcy9oaW9vcy9zYXRlbGxpdGUvZGh3XzVrbSIsCiAgICAgICAgICAgICAgICB7ImF0dHJpYnV0aW9uIjogIlBhY0lPT1MgVERTIiwgImZvcm1hdCI6ICJpbWFnZS9wbmciLCAibGF5ZXJzIjogIkNSV19TU1QiLCAic3R5bGVzIjogIiIsICJ0cmFuc3BhcmVudCI6IHRydWUsICJ2ZXJzaW9uIjogIjEuMS4xIn0KICAgICAgICAgICAgKS5hZGRUbyhtYXBfMWQ3NzQ4Y2JhYmFkNDdjM2FkMzQxZGM1YmRjZTEwYmQpOwogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb2x5X2xpbmVfYTAwZDhhMzNkYzdlNGYyMjgzYTRkM2M4YzY1OGQyMGQgPSBMLnBvbHlsaW5lKAogICAgICAgICAgICAgICAgW1s0Mi4wMywgLTcxLjNdLCBbNDIuMDMsIC03MC41N10sIFs0Mi42MywgLTcwLjU3XSwgWzQyLjYzLCAtNzEuM10sIFs0Mi4wMywgLTcxLjNdXSwKICAgICAgICAgICAgICAgIHsiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsICJjb2xvciI6ICIjRkYwMDAwIiwgImRhc2hBcnJheSI6IG51bGwsICJkYXNoT2Zmc2V0IjogbnVsbCwgImZpbGwiOiBmYWxzZSwgImZpbGxDb2xvciI6ICIjRkYwMDAwIiwgImZpbGxPcGFjaXR5IjogMC4yLCAiZmlsbFJ1bGUiOiAiZXZlbm9kZCIsICJsaW5lQ2FwIjogInJvdW5kIiwgImxpbmVKb2luIjogInJvdW5kIiwgIm5vQ2xpcCI6IGZhbHNlLCAib3BhY2l0eSI6IDAuOSwgInNtb290aEZhY3RvciI6IDEuMCwgInN0cm9rZSI6IHRydWUsICJ3ZWlnaHQiOiAyfQogICAgICAgICAgICApLmFkZFRvKG1hcF8xZDc3NDhjYmFiYWQ0N2MzYWQzNDFkYzViZGNlMTBiZCk7CiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIG1hcmtlcl9jbHVzdGVyXzUwMmFkOThlOTE0YjQzOWU5ZWU2NDhmZjc5NmRlM2IxID0gTC5tYXJrZXJDbHVzdGVyR3JvdXAoCiAgICAgICAgICAgICAgICB7fQogICAgICAgICAgICApOwogICAgICAgICAgICBtYXBfMWQ3NzQ4Y2JhYmFkNDdjM2FkMzQxZGM1YmRjZTEwYmQuYWRkTGF5ZXIobWFya2VyX2NsdXN0ZXJfNTAyYWQ5OGU5MTRiNDM5ZTllZTY0OGZmNzk2ZGUzYjEpOwogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBtYXJrZXJfYjUxYWY3NGU3OGE5NDQ3NjliYThmYmZmNjRmYTMyMTMgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zMzcyMTA0MDg2MTk0NywgLTcwLjY4OTg5MDY2MDA5OTQyXSwKICAgICAgICAgICAgICAgIHt9CiAgICAgICAgICAgICkuYWRkVG8obWFya2VyX2NsdXN0ZXJfNTAyYWQ5OGU5MTRiNDM5ZTllZTY0OGZmNzk2ZGUzYjEpOwogICAgICAgIAogICAgCiAgICAgICAgdmFyIHBvcHVwXzhkZDM4OTBkMGI2YTQyNmI4YmFkODNjODk0YzYyYmUwID0gTC5wb3B1cCh7Im1heFdpZHRoIjogIjEwMCUifSk7CgogICAgICAgIAogICAgICAgICAgICB2YXIgaHRtbF81NGVmNGFhMTQ0MmQ0MDFhODYxNjViMTIxNjViYjk4MSA9ICQoYDxkaXYgaWQ9Imh0bWxfNTRlZjRhYTE0NDJkNDAxYTg2MTY1YjEyMTY1YmI5ODEiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPltIaXN0b3J5X0Jlc3RdOiBCT1NUT04gMTYgTk0gRWFzdCBvZiBCb3N0b24sIE1BPC9kaXY+YClbMF07CiAgICAgICAgICAgIHBvcHVwXzhkZDM4OTBkMGI2YTQyNmI4YmFkODNjODk0YzYyYmUwLnNldENvbnRlbnQoaHRtbF81NGVmNGFhMTQ0MmQ0MDFhODYxNjViMTIxNjViYjk4MSk7CiAgICAgICAgCgogICAgICAgIG1hcmtlcl9iNTFhZjc0ZTc4YTk0NDc2OWJhOGZiZmY2NGZhMzIxMy5iaW5kUG9wdXAocG9wdXBfOGRkMzg5MGQwYjZhNDI2YjhiYWQ4M2M4OTRjNjJiZTApCiAgICAgICAgOwoKICAgICAgICAKICAgIAogICAgCiAgICAgICAgICAgIHZhciBtYXJrZXJfMzk4ZmUyN2EwZTkwNDc4ZTk0NGE5ZGMxNTQ5NGRjZjAgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zMzcyMTA0MDg2MTk0NywgLTcwLjY4OTg5MDY2MDA5OTQyXSwKICAgICAgICAgICAgICAgIHt9CiAgICAgICAgICAgICkuYWRkVG8obWFya2VyX2NsdXN0ZXJfNTAyYWQ5OGU5MTRiNDM5ZTllZTY0OGZmNzk2ZGUzYjEpOwogICAgICAgIAogICAgCiAgICAgICAgdmFyIHBvcHVwX2ZlOTc0MTliNWFjNzRhMzNhY2ZmMTQwNTQ4YjIzMTk3ID0gTC5wb3B1cCh7Im1heFdpZHRoIjogIjEwMCUifSk7CgogICAgICAgIAogICAgICAgICAgICB2YXIgaHRtbF85ZTJiMGE4MWI1MGQ0M2MyOTAwNDUxYjdlODU4N2E4NiA9ICQoYDxkaXYgaWQ9Imh0bWxfOWUyYjBhODFiNTBkNDNjMjkwMDQ1MWI3ZTg1ODdhODYiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPltBdmVyYWdlc19CZXN0XTogQk9TVE9OIDE2IE5NIEVhc3Qgb2YgQm9zdG9uLCBNQTwvZGl2PmApWzBdOwogICAgICAgICAgICBwb3B1cF9mZTk3NDE5YjVhYzc0YTMzYWNmZjE0MDU0OGIyMzE5Ny5zZXRDb250ZW50KGh0bWxfOWUyYjBhODFiNTBkNDNjMjkwMDQ1MWI3ZTg1ODdhODYpOwogICAgICAgIAoKICAgICAgICBtYXJrZXJfMzk4ZmUyN2EwZTkwNDc4ZTk0NGE5ZGMxNTQ5NGRjZjAuYmluZFBvcHVwKHBvcHVwX2ZlOTc0MTliNWFjNzRhMzNhY2ZmMTQwNTQ4YjIzMTk3KQogICAgICAgIDsKCiAgICAgICAgCiAgICAKICAgIAogICAgICAgICAgICB2YXIgbWFya2VyX2Y3YzhlZDAxYWFhMjQxN2Y5NjU2NTVkY2JkODhhYWEyID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzI1MDA0NTc3NjM2NzUsIC03MC42NzQ5OTU0MjIzNjMzMl0sCiAgICAgICAgICAgICAgICB7fQogICAgICAgICAgICApLmFkZFRvKG1hcmtlcl9jbHVzdGVyXzUwMmFkOThlOTE0YjQzOWU5ZWU2NDhmZjc5NmRlM2IxKTsKICAgICAgICAKICAgIAogICAgICAgIHZhciBwb3B1cF8wNTZmODFhNTkyMTQ0MmVhOTYwZDk1YjI0NDlhYzM0OCA9IEwucG9wdXAoeyJtYXhXaWR0aCI6ICIxMDAlIn0pOwoKICAgICAgICAKICAgICAgICAgICAgdmFyIGh0bWxfYTZiODkyMWIwNTRiNDI3MzhmMzI1OWI0ODc1MTRkODkgPSAkKGA8ZGl2IGlkPSJodG1sX2E2Yjg5MjFiMDU0YjQyNzM4ZjMyNTliNDg3NTE0ZDg5IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5bZGh3XzVrbV06IEJPU1RPTiAxNiBOTSBFYXN0IG9mIEJvc3RvbiwgTUE8L2Rpdj5gKVswXTsKICAgICAgICAgICAgcG9wdXBfMDU2ZjgxYTU5MjE0NDJlYTk2MGQ5NWIyNDQ5YWMzNDguc2V0Q29udGVudChodG1sX2E2Yjg5MjFiMDU0YjQyNzM4ZjMyNTliNDg3NTE0ZDg5KTsKICAgICAgICAKCiAgICAgICAgbWFya2VyX2Y3YzhlZDAxYWFhMjQxN2Y5NjU2NTVkY2JkODhhYWEyLmJpbmRQb3B1cChwb3B1cF8wNTZmODFhNTkyMTQ0MmVhOTYwZDk1YjI0NDlhYzM0OCkKICAgICAgICA7CgogICAgICAgIAogICAgCiAgICAKICAgICAgICAgICAgdmFyIG1hcmtlcl9jMjM0Njc4NDNjNWU0NmU5YTVmMTg5OTBlNzZkOGE0MyA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzQyLjM0NjAwMDAwMDAwMDAwNCwgLTcwLjY1MTAwMDAwMDAwMDAxXSwKICAgICAgICAgICAgICAgIHt9CiAgICAgICAgICAgICkuYWRkVG8obWFwXzFkNzc0OGNiYWJhZDQ3YzNhZDM0MWRjNWJkY2UxMGJkKTsKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgaWNvbl82OTEzOGEyMmM5ZjY0ZTg1OGY1MTZiYzdlZmM2ZDU2NSA9IEwuQXdlc29tZU1hcmtlcnMuaWNvbigKICAgICAgICAgICAgICAgIHsiZXh0cmFDbGFzc2VzIjogImZhLXJvdGF0ZS0wIiwgImljb24iOiAic3RhdHMiLCAiaWNvbkNvbG9yIjogIndoaXRlIiwgIm1hcmtlckNvbG9yIjogImdyZWVuIiwgInByZWZpeCI6ICJnbHlwaGljb24ifQogICAgICAgICAgICApOwogICAgICAgICAgICBtYXJrZXJfYzIzNDY3ODQzYzVlNDZlOWE1ZjE4OTkwZTc2ZDhhNDMuc2V0SWNvbihpY29uXzY5MTM4YTIyYzlmNjRlODU4ZjUxNmJjN2VmYzZkNTY1KTsKICAgICAgICAKICAgIAogICAgICAgIHZhciBwb3B1cF9lN2I2ZDg0NDI4MTg0NGNjYTkxOGNhYTU5ZjU2NzdjOCA9IEwucG9wdXAoeyJtYXhXaWR0aCI6IDI2NTB9KTsKCiAgICAgICAgCiAgICAgICAgICAgIHZhciBpX2ZyYW1lX2FkZmYzZWI4NDRlYjQ3MWQ5ZTAyMjk2MWFjOTA4ZDJmID0gJChgPGlmcmFtZSBzcmM9ImRhdGE6dGV4dC9odG1sO2NoYXJzZXQ9dXRmLTg7YmFzZTY0LENpQWdJQ0FLQ2dvS1BDRkVUME5VV1ZCRklHaDBiV3crQ2p4b2RHMXNJR3hoYm1jOUltVnVJajRLSUNBS0lDQThhR1ZoWkQ0S0lDQWdJQW9nSUNBZ0lDQThiV1YwWVNCamFHRnljMlYwUFNKMWRHWXRPQ0krQ2lBZ0lDQWdJRHgwYVhSc1pUNDBOREF4TXp3dmRHbDBiR1UrQ2lBZ0lDQWdJQW9nSUNBZ0lDQUtJQ0FnSUNBZ0lDQUtJQ0FnSUNBZ0lDQWdJQW9nSUNBZ0lDQWdJRHhzYVc1cklISmxiRDBpYzNSNWJHVnphR1ZsZENJZ2FISmxaajBpYUhSMGNITTZMeTlqWkc0dWNIbGtZWFJoTG05eVp5OWliMnRsYUM5eVpXeGxZWE5sTDJKdmEyVm9MVEV1TWk0d0xtMXBiaTVqYzNNaUlIUjVjR1U5SW5SbGVIUXZZM056SWlBdlBnb2dJQ0FnSUNBZ0lBb2dJQ0FnSUNBZ0lBb2dJQ0FnSUNBZ0lDQWdDaUFnSUNBZ0lDQWdQSE5qY21sd2RDQjBlWEJsUFNKMFpYaDBMMnBoZG1GelkzSnBjSFFpSUhOeVl6MGlhSFIwY0hNNkx5OWpaRzR1Y0hsa1lYUmhMbTl5Wnk5aWIydGxhQzl5Wld4bFlYTmxMMkp2YTJWb0xURXVNaTR3TG0xcGJpNXFjeUkrUEM5elkzSnBjSFErQ2lBZ0lDQWdJQ0FnUEhOamNtbHdkQ0IwZVhCbFBTSjBaWGgwTDJwaGRtRnpZM0pwY0hRaVBnb2dJQ0FnSUNBZ0lDQWdJQ0JDYjJ0bGFDNXpaWFJmYkc5blgyeGxkbVZzS0NKcGJtWnZJaWs3Q2lBZ0lDQWdJQ0FnUEM5elkzSnBjSFErQ2lBZ0lDQWdJQ0FnQ2lBZ0lDQWdJQW9nSUNBZ0lDQUtJQ0FnSUFvZ0lEd3ZhR1ZoWkQ0S0lDQUtJQ0FLSUNBOFltOWtlVDRLSUNBZ0lBb2dJQ0FnSUNBS0lDQWdJQ0FnSUNBS0lDQWdJQ0FnSUNBZ0lBb2dJQ0FnSUNBZ0lDQWdDaUFnSUNBZ0lDQWdJQ0FnSUFvZ0lDQWdJQ0FnSUNBZ0lDQWdJRHhrYVhZZ1kyeGhjM005SW1KckxYSnZiM1FpSUdsa1BTSTJNMk01TmpNNE5TMWhNRFpsTFRSa09Ea3RPR0psTmkweVkyWXhORE0wWm1NMk1UWWlJR1JoZEdFdGNtOXZkQzFwWkQwaU1UQXdNU0krUEM5a2FYWStDaUFnSUNBZ0lDQWdJQ0FnSUFvZ0lDQWdJQ0FnSUNBZ0NpQWdJQ0FnSUNBZ0NpQWdJQ0FnSUFvZ0lDQWdJQ0FLSUNBZ0lDQWdJQ0E4YzJOeWFYQjBJSFI1Y0dVOUltRndjR3hwWTJGMGFXOXVMMnB6YjI0aUlHbGtQU0l4TXpnd0lqNEtJQ0FnSUNBZ0lDQWdJSHNpWlRnMFpESTJOekl0T1dFd1lpMDBNRGxsTFdGaE1qa3ROakl6TW1KbU16azBNbU15SWpwN0luSnZiM1J6SWpwN0luSmxabVZ5Wlc1alpYTWlPbHQ3SW1GMGRISnBZblYwWlhNaU9uc2lZMkZzYkdKaFkyc2lPbTUxYkd3c0ltUmhkR0VpT25zaWVDSTZleUpmWDI1a1lYSnlZWGxmWHlJNklrRkJRMmRIV1ZNMlpHdEpRVUZKYVVsb04zQXlVV2RCUVdOUVpVdDFibHBESWl3aVpIUjVjR1VpT2lKbWJHOWhkRFkwSWl3aWMyaGhjR1VpT2xzelhYMHNJbmtpT25zaVgxOXVaR0Z5Y21GNVgxOGlPaUpCUVVGQlowMUpNVTFWUVVGQlFVTkJkMnBWZUZGQlFVRkJTVVJEVGxSR1FTSXNJbVIwZVhCbElqb2labXh2WVhRMk5DSXNJbk5vWVhCbElqcGJNMTE5ZlN3aWMyVnNaV04wWldRaU9uc2lhV1FpT2lJeE1UY3dJaXdpZEhsd1pTSTZJbE5sYkdWamRHbHZiaUo5TENKelpXeGxZM1JwYjI1ZmNHOXNhV041SWpwN0ltbGtJam9pTVRFM01TSXNJblI1Y0dVaU9pSlZibWx2YmxKbGJtUmxjbVZ5Y3lKOWZTd2lhV1FpT2lJeE1URTBJaXdpZEhsd1pTSTZJa052YkhWdGJrUmhkR0ZUYjNWeVkyVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liVzl1ZEdoeklqcGJNQ3d4TERJc015dzBMRFVzTml3M0xEZ3NPU3d4TUN3eE1WMTlMQ0pwWkNJNklqRXdORGNpTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltWnZjbTFoZEhSbGNpSTZleUpwWkNJNklqRXdNelVpTENKMGVYQmxJam9pUkdGMFpYUnBiV1ZVYVdOclJtOXliV0YwZEdWeUluMHNJblJwWTJ0bGNpSTZleUpwWkNJNklqRXdNVE1pTENKMGVYQmxJam9pUkdGMFpYUnBiV1ZVYVdOclpYSWlmWDBzSW1sa0lqb2lNVEF4TWlJc0luUjVjR1VpT2lKRVlYUmxkR2x0WlVGNGFYTWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liVzl1ZEdoeklqcGJNQ3cyWFgwc0ltbGtJam9pTVRBMU1DSXNJblI1Y0dVaU9pSk5iMjUwYUhOVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWkdGMFlWOXpiM1Z5WTJVaU9uc2lhV1FpT2lJeE1ESTVJaXdpZEhsd1pTSTZJa052YkhWdGJrUmhkR0ZUYjNWeVkyVWlmU3dpWjJ4NWNHZ2lPbnNpYVdRaU9pSXhNRE13SWl3aWRIbHdaU0k2SWt4cGJtVWlmU3dpYUc5MlpYSmZaMng1Y0dnaU9tNTFiR3dzSW0xMWRHVmtYMmRzZVhCb0lqcHVkV3hzTENKdWIyNXpaV3hsWTNScGIyNWZaMng1Y0dnaU9uc2lhV1FpT2lJeE1ETXhJaXdpZEhsd1pTSTZJa3hwYm1VaWZTd2ljMlZzWldOMGFXOXVYMmRzZVhCb0lqcHVkV3hzTENKMmFXVjNJanA3SW1sa0lqb2lNVEF6TXlJc0luUjVjR1VpT2lKRFJGTldhV1YzSW4xOUxDSnBaQ0k2SWpFd016SWlMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZTMwc0ltbGtJam9pTVRBeU1pSXNJblI1Y0dVaU9pSlFZVzVVYjI5c0luMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteHBibVZmWVd4d2FHRWlPakF1TmpVc0lteHBibVZmWTJGd0lqb2ljbTkxYm1RaUxDSnNhVzVsWDJOdmJHOXlJam9pSTJabU4yWXdaU0lzSW14cGJtVmZhbTlwYmlJNkluSnZkVzVrSWl3aWJHbHVaVjkzYVdSMGFDSTZOU3dpZUNJNmV5Sm1hV1ZzWkNJNkluZ2lmU3dpZVNJNmV5Sm1hV1ZzWkNJNklua2lmWDBzSW1sa0lqb2lNVEV4TlNJc0luUjVjR1VpT2lKTWFXNWxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbUpsYkc5M0lqcGJleUpwWkNJNklqRXdNVElpTENKMGVYQmxJam9pUkdGMFpYUnBiV1ZCZUdsekluMWRMQ0pqWlc1MFpYSWlPbHQ3SW1sa0lqb2lNVEF4TmlJc0luUjVjR1VpT2lKSGNtbGtJbjBzZXlKcFpDSTZJakV3TWpFaUxDSjBlWEJsSWpvaVIzSnBaQ0o5TEhzaWFXUWlPaUl4TURVeUlpd2lkSGx3WlNJNklreGxaMlZ1WkNKOVhTd2liR1ZtZENJNlczc2lhV1FpT2lJeE1ERTNJaXdpZEhsd1pTSTZJa3hwYm1WaGNrRjRhWE1pZlYwc0luQnNiM1JmYUdWcFoyaDBJam95TlRBc0luQnNiM1JmZDJsa2RHZ2lPamMxTUN3aWNtVnVaR1Z5WlhKeklqcGJleUpwWkNJNklqRXdNeklpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlMSHNpYVdRaU9pSXhNRFU1SWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmU3g3SW1sa0lqb2lNVEE0TnlJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjBzZXlKcFpDSTZJakV4TVRjaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFN3aWRHbDBiR1VpT25zaWFXUWlPaUl4TURBeUlpd2lkSGx3WlNJNklsUnBkR3hsSW4wc0luUnZiMnhpWVhJaU9uc2lhV1FpT2lJeE1ESTFJaXdpZEhsd1pTSTZJbFJ2YjJ4aVlYSWlmU3dpZEc5dmJHSmhjbDlzYjJOaGRHbHZiaUk2SW1GaWIzWmxJaXdpZUY5eVlXNW5aU0k2ZXlKcFpDSTZJakV3TURRaUxDSjBlWEJsSWpvaVJHRjBZVkpoYm1kbE1XUWlmU3dpZUY5elkyRnNaU0k2ZXlKcFpDSTZJakV3TURnaUxDSjBlWEJsSWpvaVRHbHVaV0Z5VTJOaGJHVWlmU3dpZVY5eVlXNW5aU0k2ZXlKcFpDSTZJakV3TURZaUxDSjBlWEJsSWpvaVJHRjBZVkpoYm1kbE1XUWlmU3dpZVY5elkyRnNaU0k2ZXlKcFpDSTZJakV3TVRBaUxDSjBlWEJsSWpvaVRHbHVaV0Z5VTJOaGJHVWlmWDBzSW1sa0lqb2lNVEF3TVNJc0luTjFZblI1Y0dVaU9pSkdhV2QxY21VaUxDSjBlWEJsSWpvaVVHeHZkQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZlMzBzSW1sa0lqb2lNVEExTVNJc0luUjVjR1VpT2lKWlpXRnljMVJwWTJ0bGNpSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJGc2NHaGhJam93TGpFc0lteHBibVZmWTJGd0lqb2ljbTkxYm1RaUxDSnNhVzVsWDJOdmJHOXlJam9pSXpGbU56ZGlOQ0lzSW14cGJtVmZhbTlwYmlJNkluSnZkVzVrSWl3aWJHbHVaVjkzYVdSMGFDSTZOU3dpZUNJNmV5Sm1hV1ZzWkNJNkluZ2lmU3dpZVNJNmV5Sm1hV1ZzWkNJNklua2lmWDBzSW1sa0lqb2lNVEV4TmlJc0luUjVjR1VpT2lKTWFXNWxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbVJoZEdGZmMyOTFjbU5sSWpwN0ltbGtJam9pTVRFeE5DSXNJblI1Y0dVaU9pSkRiMngxYlc1RVlYUmhVMjkxY21ObEluMHNJbWRzZVhCb0lqcDdJbWxrSWpvaU1URXhOU0lzSW5SNWNHVWlPaUpNYVc1bEluMHNJbWh2ZG1WeVgyZHNlWEJvSWpwdWRXeHNMQ0p0ZFhSbFpGOW5iSGx3YUNJNmJuVnNiQ3dpYm05dWMyVnNaV04wYVc5dVgyZHNlWEJvSWpwN0ltbGtJam9pTVRFeE5pSXNJblI1Y0dVaU9pSk1hVzVsSW4wc0luTmxiR1ZqZEdsdmJsOW5iSGx3YUNJNmJuVnNiQ3dpZG1sbGR5STZleUpwWkNJNklqRXhNVGdpTENKMGVYQmxJam9pUTBSVFZtbGxkeUo5ZlN3aWFXUWlPaUl4TVRFM0lpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYVhSbGJYTWlPbHQ3SW1sa0lqb2lNVEExTXlJc0luUjVjR1VpT2lKTVpXZGxibVJKZEdWdEluMHNleUpwWkNJNklqRXdPREVpTENKMGVYQmxJam9pVEdWblpXNWtTWFJsYlNKOUxIc2lhV1FpT2lJeE1URXhJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlN4N0ltbGtJam9pTVRFME15SXNJblI1Y0dVaU9pSk1aV2RsYm1SSmRHVnRJbjFkZlN3aWFXUWlPaUl4TURVeUlpd2lkSGx3WlNJNklreGxaMlZ1WkNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKc2FXNWxYMkZzY0doaElqb3dMalkxTENKc2FXNWxYMk5oY0NJNkluSnZkVzVrSWl3aWJHbHVaVjlqYjJ4dmNpSTZJaU14WmpjM1lqUWlMQ0pzYVc1bFgycHZhVzRpT2lKeWIzVnVaQ0lzSW14cGJtVmZkMmxrZEdnaU9qVXNJbmdpT25zaVptbGxiR1FpT2lKNEluMHNJbmtpT25zaVptbGxiR1FpT2lKNUluMTlMQ0pwWkNJNklqRXdNekFpTENKMGVYQmxJam9pVEdsdVpTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnVkVzFmYldsdWIzSmZkR2xqYTNNaU9qVXNJblJwWTJ0bGNuTWlPbHQ3SW1sa0lqb2lNVEEwTUNJc0luUjVjR1VpT2lKQlpHRndkR2wyWlZScFkydGxjaUo5TEhzaWFXUWlPaUl4TURReElpd2lkSGx3WlNJNklrRmtZWEIwYVhabFZHbGphMlZ5SW4wc2V5SnBaQ0k2SWpFd05ESWlMQ0owZVhCbElqb2lRV1JoY0hScGRtVlVhV05yWlhJaWZTeDdJbWxrSWpvaU1UQTBNeUlzSW5SNWNHVWlPaUpFWVhselZHbGphMlZ5SW4wc2V5SnBaQ0k2SWpFd05EUWlMQ0owZVhCbElqb2lSR0Y1YzFScFkydGxjaUo5TEhzaWFXUWlPaUl4TURRMUlpd2lkSGx3WlNJNklrUmhlWE5VYVdOclpYSWlmU3g3SW1sa0lqb2lNVEEwTmlJc0luUjVjR1VpT2lKRVlYbHpWR2xqYTJWeUluMHNleUpwWkNJNklqRXdORGNpTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpwWkNJNklqRXdORGdpTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpwWkNJNklqRXdORGtpTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpwWkNJNklqRXdOVEFpTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpwWkNJNklqRXdOVEVpTENKMGVYQmxJam9pV1dWaGNuTlVhV05yWlhJaWZWMTlMQ0pwWkNJNklqRXdNVE1pTENKMGVYQmxJam9pUkdGMFpYUnBiV1ZVYVdOclpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2ljMjkxY21ObElqcDdJbWxrSWpvaU1URXhOQ0lzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjE5TENKcFpDSTZJakV4TVRnaUxDSjBlWEJsSWpvaVEwUlRWbWxsZHlKOUxIc2lZWFIwY21saWRYUmxjeUk2ZTMwc0ltbGtJam9pTVRFME1TSXNJblI1Y0dVaU9pSlRaV3hsWTNScGIyNGlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR0ZpWld3aU9uc2lkbUZzZFdVaU9pSkJkbVZ5WVdkbGMxOUNaWE4wSW4wc0luSmxibVJsY21WeWN5STZXM3NpYVdRaU9pSXhNRE15SWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmVjE5TENKcFpDSTZJakV3TlRNaUxDSjBlWEJsSWpvaVRHVm5aVzVrU1hSbGJTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmhZM1JwZG1WZlpISmhaeUk2SW1GMWRHOGlMQ0poWTNScGRtVmZhVzV6Y0dWamRDSTZJbUYxZEc4aUxDSmhZM1JwZG1WZmJYVnNkR2tpT201MWJHd3NJbUZqZEdsMlpWOXpZM0p2Ykd3aU9pSmhkWFJ2SWl3aVlXTjBhWFpsWDNSaGNDSTZJbUYxZEc4aUxDSjBiMjlzY3lJNlczc2lhV1FpT2lJeE1ESXlJaXdpZEhsd1pTSTZJbEJoYmxSdmIyd2lmU3g3SW1sa0lqb2lNVEF5TXlJc0luUjVjR1VpT2lKQ2IzaGFiMjl0Vkc5dmJDSjlMSHNpYVdRaU9pSXhNREkwSWl3aWRIbHdaU0k2SWxKbGMyVjBWRzl2YkNKOUxIc2lhV1FpT2lJeE1EVTBJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5TEhzaWFXUWlPaUl4TURneUlpd2lkSGx3WlNJNklraHZkbVZ5Vkc5dmJDSjlMSHNpYVdRaU9pSXhNVEV5SWl3aWRIbHdaU0k2SWtodmRtVnlWRzl2YkNKOUxIc2lhV1FpT2lJeE1UUTBJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5WFgwc0ltbGtJam9pTVRBeU5TSXNJblI1Y0dVaU9pSlViMjlzWW1GeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN2ZTd2lhV1FpT2lJeE1ERTRJaXdpZEhsd1pTSTZJa0poYzJsalZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3ZlN3aWFXUWlPaUl4TVRReUlpd2lkSGx3WlNJNklsVnVhVzl1VW1WdVpHVnlaWEp6SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14aFltVnNJanA3SW5aaGJIVmxJam9pWkdoM1h6VnJiU0o5TENKeVpXNWtaWEpsY25NaU9sdDdJbWxrSWpvaU1URXhOeUlzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMWRmU3dpYVdRaU9pSXhNVFF6SWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lJeE1URTNJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZWMHNJblJ2YjJ4MGFYQnpJanBiV3lKT1lXMWxJaXdpWkdoM1h6VnJiU05tYVd4c2JXbHpiV0YwWTJnaVhTeGJJa0pwWVhNaUxDSXRNQzR6TmlKZExGc2lVMnRwYkd3aUxDSXdMakl4SWwxZGZTd2lhV1FpT2lJeE1UUTBJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzYVc1bFgyRnNjR2hoSWpvd0xqRXNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpvaUl6Rm1OemRpTkNJc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2TlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pTVRBMU9DSXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3ZlN3aWFXUWlPaUl4TURNMUlpd2lkSGx3WlNJNklrUmhkR1YwYVcxbFZHbGphMFp2Y20xaGRIUmxjaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzYVc1bFgyRnNjR2hoSWpvd0xqWTFMQ0pzYVc1bFgyTmhjQ0k2SW5KdmRXNWtJaXdpYkdsdVpWOWpiMnh2Y2lJNklpTmhaV00zWlRnaUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT2pVc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SWpFd05UY2lMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKbWIzSnRZWFIwWlhJaU9uc2lhV1FpT2lJeE1ETTNJaXdpZEhsd1pTSTZJa0poYzJsalZHbGphMFp2Y20xaGRIUmxjaUo5TENKMGFXTnJaWElpT25zaWFXUWlPaUl4TURFNElpd2lkSGx3WlNJNklrSmhjMmxqVkdsamEyVnlJbjE5TENKcFpDSTZJakV3TVRjaUxDSjBlWEJsSWpvaVRHbHVaV0Z5UVhocGN5SjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmpZV3hzWW1GamF5STZiblZzYkN3aWNtVnVaR1Z5WlhKeklqcGJleUpwWkNJNklqRXdNeklpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYU3dpZEc5dmJIUnBjSE1pT2x0YklrNWhiV1VpTENKQmRtVnlZV2RsYzE5Q1pYTjBJMlpwYkd4dGFYTnRZWFJqYUNKZExGc2lRbWxoY3lJc0lqQXVORE1pWFN4YklsTnJhV3hzSWl3aU1DNDBPQ0pkWFgwc0ltbGtJam9pTVRBMU5DSXNJblI1Y0dVaU9pSkliM1psY2xSdmIyd2lmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZMkZzYkdKaFkyc2lPbTUxYkd4OUxDSnBaQ0k2SWpFd01EWWlMQ0owZVhCbElqb2lSR0YwWVZKaGJtZGxNV1FpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWTJGc2JHSmhZMnNpT201MWJHeDlMQ0pwWkNJNklqRXdNRFFpTENKMGVYQmxJam9pUkdGMFlWSmhibWRsTVdRaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW1SaGRHRWlPbnNpZUNJNmV5SmZYMjVrWVhKeVlYbGZYeUk2SWtGQlFrRkhOMkUxWkd0SlFVRkRhVXQxWW13eVVXZEJRVVZRYlRoMVdGcERRVUZFTkZvNFF6VmthMGxCUVU5RVYzYzNiREpSWjBGQmVVVllTSFZZV2tOQlFVTjNkRTF4TldSclNVRkJTbWRxZW5Kc01sRm5RVUZuU2t4U2RWaGFRMEZCUW05QlpGYzFaR3RKUVVGR1FuY3lUR3d5VVdkQlFVOU9MMkoxV0ZwRFFVRkJaMVIwS3pWa2EwbEJRVUZwT1RSeWJESlJaMEZCT0VOMmJYVllXa05CUVVSWmJYVnROV1JyU1VGQlRVRktOMkpzTWxGblFVRnhTR3AzZFZoYVEwRkJRMUUxTDA4MVpHdEpRVUZJYUZjNU4yd3lVV2RCUVZsTldEWjFXRnBEUVVGQ1NVNVFOalZrYTBsQlFVUkRha0ZpY0RKUlowRkJSMEpKUm5WdVdrTkJRVUZCWjFGcE5tUnJTVUZCVDJwMlF6ZHdNbEZuUVVFd1JqUlFkVzVhUTBGQlF6UjZVa3MyWkd0SlFVRkxRVGhHY25BeVVXZEJRV2xMYzFwMWJscERRVUZDZDBkb01qWmthMGxCUVVacFNrbE1jREpSWjBGQlVWQm5hblZ1V2tOQlFVRnZXbmxsTm1SclNVRkJRa1JYUzNKd01sRm5RVUVyUlZGMWRXNWFRMEZCUkdkemVrYzJaR3RKUVVGTloybE9ZbkF5VVdkQlFYTktSVFIxYmxwRFFVRkRXVUZFZVRaa2EwbEJRVWxDZGxBM2NESlJaMEZCWVU0MVEzVnVXa05CUVVKUlZGVmhObVJyU1VGQlJHazRVMkp3TWxGblFVRkpRM1JPZFc1YVEwRkJRVWx0YkVNMlpHdEpRVUZRUVVsV1RIQXlVV2RCUVRKSVpGaDFibHBEUVVGRVFUVnNjVFprYTBsQlFVdG9WbGh5Y0RKUlowRkJhMDFTYUhWdVdrTkJRVUkwVFRKWE5tUnJTVUZCUjBOcFlVeHdNbEZuUVVGVFFrWnpkVzVhUTBGQlFYZG5SeXMyWkd0SlFVRkNhblpqY25BeVVXZEJRVUZHTlRKMWJscERRVUZFYjNwSWJUWmthMGxCUVU1Qk4yWmljREpSWjBGQmRVdHhRWFZ1V2tOQlFVTm5SMWxUTm1SclNVRkJTV2xKYURkd01sRm5RVUZqVUdWTGRXNWFRMEZCUWxsYWJ6WTJaR3RKUVVGRlJGWnJZbkF5VVdkQlFVdEZVMVoxYmxwRFFVRkJVWE0xYVRaa2EwbEJRVkJuYUc1TWNESlJaMEZCTkVwRFpuVnVXa05CUVVSSkx6WkxObVJyU1VGQlRFSjFjSEp3TWxGblFVRnRUakp3ZFc1YVEwRkJRMEZVU3pJMlpHdEpRVUZIYVRkelRIQXlVV2RCUVZWRGNUQjFibHBEUVVGQk5HMWlaVFprYTBsQlFVTkJTWFUzY0RKUlowRkJRMGhsSzNWdVdrTkJRVVIzTldOSE5tUnJTVUZCVG1oVmVHSndNbEZuUVVGM1RWQkpkVzVhUTBGQlEyOU5jM2syWkd0SlFVRktRMmg2TjNBeVVXZEJRV1ZDUkZSMWJscERRVUZDWjJZNVlUWmthMGxCUVVWcWRUSmljREpSWjBGQlRVWXpaSFZ1V2tOQlFVRlplazlETm1SclNVRkJRVUUzTlV4d01sRm5RVUUyUzI1dWRXNWFRMEZCUkZGSFQzVTJaR3RKUVVGTWFVZzNjbkF5VVdkQlFXOVFZbmgxYmxwRFFVRkRTVnBtVnpaa2EwbEJRVWhFVlN0TWNESlJaMEZCVjBWUU9IVnVXa05CUVVKQmMzWXJObVJyU1VGQlEyZG9RVGQwTWxGblFVRkZTa0ZIZFROYVEwRkJSRFF2WjIwM1pHdEpRVUZQUW5SRVluUXlVV2RCUVhsT2QxRjFNMXBEUVVGRGQxTjRVemRrYTBsQlFVcHBOa1kzZERKUlowRkJaME5yWW5VeldrTkJRVUp2YlVJMk4yUnJTVUZCUmtGSVNYSjBNbEZuUVVGUFNGbHNkVE5hUTBGQlFXYzFVMmszWkd0SlFVRkJhRlZNVEhReVVXZEJRVGhOU1haMU0xcERRVUZFV1UxVVR6ZGthMGxCUVUxRFowNXlkREpSWjBGQmNVRTROblV6V2tOQlFVTlJabW95TjJSclNVRkJTR3AwVVV4ME1sRm5RVUZaUm5oRmRUTmFRMEZCUWtsNU1HVTNaR3RKUVVGRVFUWlROM1F5VVdkQlFVZExiRTkxTTFwRFFVRkJRVWRHU3pka2EwbEJRVTlwUjFaaWRESlJaMEZCTUZCV1dYVXpXa05CUVVNMFdrWjVOMlJyU1VGQlMwUlVXRGQwTWxGblFVRnBSVXBxZFROYVEwRkJRbmR6VjJFM1pHdEpRVUZHWjJkaGNuUXlVV2RCUVZGSk9YUjFNMXBEUVVGQmJ5OXVRemRrYTBsQlFVSkNkR1JNZERKUlowRkJLMDUwTTNVeldrTkJRVVJuVTI1MU4yUnJTVUZCVFdrMVpuSjBNbEZuUVVGelEybERkVE5hUTBGQlExbHNORmMzWkd0SlFVRkpRVWRwWW5ReVVXZEJRV0ZJVjAxMU0xcERRVUZDVVRWSkt6ZGthMGxCUVVSb1ZHczNkREpSWjBGQlNVMUxWM1V6V2tOQlFVRkpUVnB4TjJSclNVRkJVRU5tYm1KME1sRm5RVUV5UVRab2RUTmFRMEZCUkVGbVlWTTNaR3RKUVVGTGFuTndOM1F5VVdkQlFXdEdkWEoxTTFwRFFVRkNOSGx4Tmpka2EwbEJRVWRCTlhOeWRESlJaMEZCVTB0cE1YVXpXa05CUVVGM1JqZHROMlJyU1VGQlFtbEhka3gwTWxGblFVRkJVRmN2ZFROYVEwRkJSRzlaT0U4M1pHdEpRVUZPUkZONGNuUXlVV2RCUVhWRlNFdDFNMXBEUVVGRFozTk5NamRrYTBsQlFVbG5aakJpZERKUlowRkJZMGszVlhVeldrTkJRVUpaTDJSbE4yUnJTVUZCUlVKek1qZDBNbEZuUVVGTFRuWmxkVE5hUTBGQlFWRlRkVXMzWkd0SlFVRlFhVFExWW5ReVVXZEJRVFJEWm5CMU0xcERRVUZFU1d4MWVUZGthMGxCUVV4QlJqaE1kREpSWjBGQmJVaFVlblV6V2tOQlFVTkJOQzloTjJSclNVRkJSMmhUSzNKME1sRm5RVUZWVFVnNWRUTmFRMEZCUVRSTlFVYzRaR3RKUVVGRFEyWkNUSGd5VVdkQlFVTkJORWwyU0ZwRFFVRkVkMlpCZFRoa2EwbEJRVTVxY2tSeWVESlJaMEZCZDBadlUzWklXa05CUVVOdmVWSlhPR1JyU1VGQlNrRTBSMko0TWxGblFVRmxTMk5qZGtoYVEwRkJRbWRHYVVNNFpHdEpRVUZGYVVaSk4zZ3lVV2RCUVUxUVVXMTJTRnBEUVVGQldWbDVjVGhrYTBsQlFVRkVVMHhpZURKUlowRkJOa1ZCZUhaSVdrTkJRVVJSY25wVE9HUnJTVUZCVEdkbFQweDRNbEZuUVVGdlNUQTNka2hhUTBGQlEwa3ZSRFk0Wkd0SlFVRklRbkpSY25neVVXZEJRVmRPY0VaMlNGcERRVUZDUVZOVmJUaGthMGxCUVVOcE5GUk1lREpSWjBGQlJVTmtVWFpJV2tOQlFVUTBiRlpQT0dSclNVRkJUMEZGVmpkNE1sRm5RVUY1U0U1aGRraGFRMEZCUTNjMGJESTRaR3RKUVVGS2FGSlpZbmd5VVdkQlFXZE5RbXQyU0ZwRFFVRkNiMHd5YVRoa2EwbEJRVVpEWldFM2VESlJaMEZCVDBFeGRuWklXa05CUVVGblpraExPR1JyU1VGQlFXcHlaR0o0TWxGblBUMGlMQ0prZEhsd1pTSTZJbVpzYjJGME5qUWlMQ0p6YUdGd1pTSTZXekl3TmwxOUxDSjVJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZCUVRSSFdsSk1NRUZCUVVGRFFWQmFkM1pSUVVGQlFVOURVbXRET1VGQlFVRkJaMDlIVEV3d1FVRkJRVU5uT1Zrd2RsRkJRVUZCVFVOaWEyazVRVUZCUVVGQlNEWmFUREJCUVVGQlFVRktZVWwyVVVGQlFVRkxRV055YVRsQlFVRkJRVkZKWlRGTU1FRkJRVUZCUVZsTlFYWlJRVUZCUVVsQ01qSkRPVUZCUVVGQlFVcDZPRXd3UVVGQlFVRm5OWGhqZDFGQlFVRkJRMEY1VUVSQ1FVRkJRVUZuUTJ4cVRVVkJRVUZCUkdkU1dVVjNVVUZCUVVGSFJEbHVha0pCUVVGQlFXZE1jVE5OUlVGQlFVRkRRVzVOYjNkUlFVRkJRVUZCUlRGNlFrRkJRVUZCTkVKRVpFMUZRVUZCUVVObmNDdEJkMUZCUVVGQlEwRk1OVlJDUVVGQlFVRTBRWHB2VFVWQlFVRkJRa0ZvZUUxNFVVRkJRVUZMUVRSR2FrWkJRVUZCUVhkSk9GcE5WVUZCUVVGQ1FUaG9NSGhSUVVGQlFVMUVSRXBFUmtGQlFVRkJORTlCY1UxVlFVRkJRVUZuTldrNGVGRkJRVUZCUTBNeFRYcEdRVUZCUVVGdlJtTXlUVlZCUVVGQlFXZERhbXQ0VVVGQlFVRkJRa0pRVkVaQlFVRkJRVzlMVmtWTlZVRkJRVUZCWjJSRk9IaFJRVUZCUVVsQlVGaHFSa0ZCUVVGQlNVZG9kazFWUVVGQlFVRkJTVzlSZUZGQlFVRkJTMEpHYm1wR1FVRkJRVUZuUkhrM1RWVkJRVUZCUkVGMlRsbDRVVUZCUVVGRFF6QTNha1pCUVVGQlFUUlBjaXROVlVGQlFVRkNaMGxuWjNsUlFVRkJRVWRCVmtOVVNrRkJRVUZCU1U5WlFVMXJRVUZCUVVOblMxY3dlVkZCUVVGQlEwRTBZa1JLUVVGQlFVRm5TMUoxVFd0QlFVRkJSRUV5U0VGNVVVRkJRVUZQUTNkalJFcEJRVUZCUVhkS09YVk5hMEZCUVVGQloybHRhM2xSUVVGQlFVVkRSbGxVU2tGQlFVRkJTVVYwVVUxclFVRkJRVVJCUTBSM2VWRkJRVUZCUTBGcVRrUktRVUZCUVVGQlQwVjFUV3RCUVVGQlJHZEVVM041VVVGQlFVRkhRWGhMUkVwQlFVRkJRVmxRZDIxTmEwRkJRVUZEWnpGcFkzbFJRVUZCUVVkRE0wdFVTa0ZCUVVGQlNVOVpjazFyUVVGQlFVRkJRVk0wZVZGQlFVRkJRVU5CVEhwS1FVRkJRVUZKU2tsMlRXdEJRVUZCUTBGQlF6QjVVVUZCUVVGSlFYaEtWRXBCUVVGQlFWRkRZMVJOYTBGQlFVRkVRVzlvYjNsUlFVRkJRVUZEUlVKVVNrRkJRVUZCZDBsNmVVMVZRVUZCUVVKblpqazRlRkZCUVVGQlIwTTRlbFJHUVVGQlFVRkJRM0VyVFZWQlFVRkJRa0V6WWtGNFVVRkJRVUZEUW5Sd1ZFWkJRVUZCUVRST0syRk5WVUZCUVVGRVFXTmFUWGhSUVVGQlFVRkJSV3Q2UmtGQlFVRkJTVTgyV2sxVlFVRkJRVUpuYm1GcmVGRkJRVUZCUjBNdmQycEdRVUZCUVVGblNDOXJUVlZCUVVGQlFXZHpkekI1VVVGQlFVRkxRekJRZWtwQlFVRkJRVUZMYkhwTmEwRkJRVUZDWnpJMlNYbFJRVUZCUVV0Q2JIaDZTa0ZCUVVGQlowSjJZMDFyUVVGQlFVSm5NMlZGZVZGQlFVRkJTMEowTTNwS1FVRkJRVUZKUkdaV1RXdEJRVUZCUWtGaU9HTjVVVUZCUVVGQlFYSjFWRXBCUVVGQlFWRkRSM0ZOYTBGQlFVRkNaMWR3YzNsUlFVRkJRVWxFY0dwVVNrRkJRVUZCWjBkSFEwMXJRVUZCUVVOQlJWaG5lVkZCUVVGQlRVSk5ZbnBLUVVGQlFVRkpRa3B3VFd0QlFVRkJRMmR1YlZsNVVVRkJRVUZOUkdOaGVrcEJRVUZCUVdkRVFqUk5hMEZCUVVGRFFXVkpjM2xSUVVGQlFVZEJTM0JVU2tGQlFVRkJVVXBxUkUxclFVRkJRVVJuTXl0UmVWRkJRVUZCUTBKblFtcE9RVUZCUVVGQlNGbHRUVEJCUVVGQlFrRTBSVWw2VVVGQlFVRk5RVGxXZWs1QlFVRkJRVFJDVW1oTk1FRkJRVUZEUVhkWFJYcFJRVUZCUVVkQ1ZGZDZUa0ZCUVVGQlFVZGtVVTB3UVVGQlFVRkJTWHB6ZWxGQlFVRkJUVUprUzFST1FVRkJRVUZuVEdOaVRUQkJRVUZCUTJkVWFFbDZVVUZCUVVGUFFrWkRhazVCUVVGQlFXZElORU5OTUVGQlFVRkRaMFZtYzNsUlFVRkJRVU5EUlRoNlNrRkJRVUZCUVV4dWMwMXJRVUZCUVVKbmNDdHJlVkZCUVVGQlFVTkNPRlJLUVVGQlFVRlpRelJHVFRCQlFVRkJRMmRrYVZGNlVVRkJRVUZEUTJOVWFrNUJRVUZCUVhkTU5rRk5NRUZCUVVGQ1FWTk1aM3BSUVVGQlFVMUNXVGRFVGtGQlFVRkJTVkJ6VDA1RlFVRkJRVUZuZDFSQk1GRkJRVUZCUlVFMlUwUlNRVUZCUVVGdlF6RlVUa1ZCUVVGQlFtZFRNVUV3VVVGQlFVRkxRekpSYWxKQlFVRkJRV2RQUVd4T1JVRkJRVUZEWjJWQlNUQlJRVUZCUVU5Q0t6VnFUa0ZCUVVGQldVWjJVMDB3UVVGQlFVUkJLM05KZWxGQlFVRkJUME5yZEZST1FVRkJRVUZaVFZOdlRUQkJRVUZCUTBGTWNIZDZVVUZCUVVGUFFscHJSRTVCUVVGQlFYZEhRMFpOTUVGQlFVRkRaeXRZTkhwUlFVRkJRVXREYldoRVRrRkJRVUZCVVVWRFYwMHdRVUZCUVVOQlIweFJlbEZCUVVGQlQwTm1NMVJPUVVGQlFVRnZURkZQVGtWQlFVRkJRV2N6VkRBd1VVRkJRVUZIUkZGaFZGSkJRVUZCUVVGRVMxWk9SVUZCUVVGQlFVMXliekJSUVVGQlFVMUVkVEZxVWtGQlFVRkJORWxxY0U1RlFVRkJRVU5uYW5VME1GRkJRVUZCUzBKMU5VUlNRVUZCUVVGM1VHWklUa1ZCUVVGQlJHZHJZVUV3VVVGQlFVRkxRV0psVkZKQlFVRkJRWGRIVGxKT1JVRkJRVUZEWjBwRE5EQlJRVUZCUVVORVpFVkVVa0ZCUVVGQlFVaFlNMDB3UVVGQlFVRm5XaXRCZWxGQlFVRkJUME5xZVhwT1FVRkJRVUUwVFNzMVRUQkJRVUZCUTJkcWNUQjZVVUZCUVVGRlFYWnlWRTVCUVVGQlFXOUdLek5OTUVGQlFVRkVaMmhqYTNwUlFVRkJRVU5DVWpSVVRrRkJRVUZCV1VsTU9FMHdRVUZCUVVKbmMxSnJNRkZCUVVGQlNVRXJUbnBTUVVGQlFVRm5SR3hWVGtWQlFVRkJRVUU1TWpRd1VVRkJRVUZCUTFobmVsSkJRVUZCUVc5Q01rOU9SVUZCUVVGRVFVZHZiekJSUVVGQlFVOUNSbVZFVWtGQlFVRkJaMEZLWlU1RlFVRkJRVUpuUWxWRk1GRkJRVUZCUjBGSVNXcFNRVUZCUVVGQlMzSTRUVEJCUVVGQlJFRkVaR042VVVGQlFVRkxSQ3QwYWs1QlFVRkJRVWxGVTJWTk1FRkJRVUZDWjBWSmMzcFJRVUZCUVU5RWFHVnFUa0ZCUVVGQldVUm9kRTB3UVVGQlFVRm5kekpOZWxGQlFVRkJUMEpoV0hwT1FVRkJRVUUwUm5CbVRUQkJRVUZCUkdkWGJEaDZVVUU5UFNJc0ltUjBlWEJsSWpvaVpteHZZWFEyTkNJc0luTm9ZWEJsSWpwYk1qQTJYWDE5TENKelpXeGxZM1JsWkNJNmV5SnBaQ0k2SWpFeE1Ea2lMQ0owZVhCbElqb2lVMlZzWldOMGFXOXVJbjBzSW5ObGJHVmpkR2x2Ymw5d2IyeHBZM2tpT25zaWFXUWlPaUl4TVRFd0lpd2lkSGx3WlNJNklsVnVhVzl1VW1WdVpHVnlaWEp6SW4xOUxDSnBaQ0k2SWpFd05UWWlMQ0owZVhCbElqb2lRMjlzZFcxdVJHRjBZVk52ZFhKalpTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmUzMHNJbWxrSWpvaU1UQTNPU0lzSW5SNWNHVWlPaUpUWld4bFkzUnBiMjRpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpZEdsamEyVnlJanA3SW1sa0lqb2lNVEF4TXlJc0luUjVjR1VpT2lKRVlYUmxkR2x0WlZScFkydGxjaUo5ZlN3aWFXUWlPaUl4TURFMklpd2lkSGx3WlNJNklrZHlhV1FpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWW1GelpTSTZOakFzSW0xaGJuUnBjM05oY3lJNld6RXNNaXcxTERFd0xERTFMREl3TERNd1hTd2liV0Y0WDJsdWRHVnlkbUZzSWpveE9EQXdNREF3TGpBc0ltMXBibDlwYm5SbGNuWmhiQ0k2TVRBd01DNHdMQ0p1ZFcxZmJXbHViM0pmZEdsamEzTWlPakI5TENKcFpDSTZJakV3TkRFaUxDSjBlWEJsSWpvaVFXUmhjSFJwZG1WVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdsdVpWOWhiSEJvWVNJNk1DNHhMQ0pzYVc1bFgyTmhjQ0k2SW5KdmRXNWtJaXdpYkdsdVpWOWpiMnh2Y2lJNklpTXhaamMzWWpRaUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT2pVc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SWpFd016RWlMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiQ3dpWkdGMFlTSTZleUo0SWpwN0lsOWZibVJoY25KaGVWOWZJam9pUVVGQloxUjBLelZrYTBsQlFVRnBPVFJ5YkRKUlowRkJPRU4yYlhWWVdrTkJRVVJuYzNwSE5tUnJTVUZCVFdkcFRtSndNbEZuUVVGelNrVTBkVzVhUTBGQlEyZEhXVk0yWkd0SlFVRkphVWxvTjNBeVVXZEJRV05RWlV0MWJscERRVUZDWjJZNVlUWmthMGxCUVVWcWRUSmljREpSWjBGQlRVWXpaSFZ1V2tOQlFVRm5OVk5wTjJSclNVRkJRV2hWVEV4ME1sRm5RVUU0VFVsMmRUTmFRMEZCUkdkVGJuVTNaR3RKUVVGTmFUVm1jblF5VVdkQlFYTkRhVU4xTTFwRFFVRkRaM05OTWpka2EwbEJRVWxuWmpCaWRESlJaMEZCWTBrM1ZYVXpXa01pTENKa2RIbHdaU0k2SW1ac2IyRjBOalFpTENKemFHRndaU0k2V3pJeFhYMHNJbmtpT25zaVgxOXVaR0Z5Y21GNVgxOGlPaUpCUVVGQlowVjNORTFGUVVGQlFVTkJVVVZWZDFGQlFVRkJTMEV3VldwQ1FVRkJRVUZaUXpGMlRWVkJRVUZCUWtGRU0yZDRVVUZCUVVGQlJIaG5SRVpCUVVGQlFVRkdiRVZOYTBGQlFVRkNaMVZGU1hsUlFVRkJRVXRDU0ZGRVNrRkJRVUZCUVVsclZFMXJRVUZCUVVOQlUyaHplVkZCUVVGQlFVRk5TWHBLUVVGQlFVRlpTek5PVFd0QlFVRkJSR2QxT1ZGNVVVRkJRVUZIUkVzeWVrcEJRVUZCUVhkQmFETk5NRUZCUVVGRVoycElNSHBSUVVGQlFVTkJVbWhFVGtGQlFVRkJTVWN3VkU1RlFVRkJRVUZuWWxKTk1GRkJRVUZCUTBKMFJYcFNRU0lzSW1SMGVYQmxJam9pWm14dllYUTJOQ0lzSW5Ob1lYQmxJanBiTWpGZGZYMHNJbk5sYkdWamRHVmtJanA3SW1sa0lqb2lNVEEzT1NJc0luUjVjR1VpT2lKVFpXeGxZM1JwYjI0aWZTd2ljMlZzWldOMGFXOXVYM0J2YkdsamVTSTZleUpwWkNJNklqRXdPREFpTENKMGVYQmxJam9pVlc1cGIyNVNaVzVrWlhKbGNuTWlmWDBzSW1sa0lqb2lNVEF5T1NJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW0xaGJuUnBjM05oY3lJNld6RXNNaXcxWFN3aWJXRjRYMmx1ZEdWeWRtRnNJam8xTURBdU1Dd2liblZ0WDIxcGJtOXlYM1JwWTJ0eklqb3dmU3dpYVdRaU9pSXhNRFF3SWl3aWRIbHdaU0k2SWtGa1lYQjBhWFpsVkdsamEyVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbVJoZEdGZmMyOTFjbU5sSWpwN0ltbGtJam9pTVRBMU5pSXNJblI1Y0dVaU9pSkRiMngxYlc1RVlYUmhVMjkxY21ObEluMHNJbWRzZVhCb0lqcDdJbWxrSWpvaU1UQTFOeUlzSW5SNWNHVWlPaUpNYVc1bEluMHNJbWh2ZG1WeVgyZHNlWEJvSWpwdWRXeHNMQ0p0ZFhSbFpGOW5iSGx3YUNJNmJuVnNiQ3dpYm05dWMyVnNaV04wYVc5dVgyZHNlWEJvSWpwN0ltbGtJam9pTVRBMU9DSXNJblI1Y0dVaU9pSk1hVzVsSW4wc0luTmxiR1ZqZEdsdmJsOW5iSGx3YUNJNmJuVnNiQ3dpZG1sbGR5STZleUpwWkNJNklqRXdOakFpTENKMGVYQmxJam9pUTBSVFZtbGxkeUo5ZlN3aWFXUWlPaUl4TURVNUlpd2lkSGx3WlNJNklrZHNlWEJvVW1WdVpHVnlaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnQ5TENKcFpDSTZJakV3TXpjaUxDSjBlWEJsSWpvaVFtRnphV05VYVdOclJtOXliV0YwZEdWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltUnBiV1Z1YzJsdmJpSTZNU3dpZEdsamEyVnlJanA3SW1sa0lqb2lNVEF4T0NJc0luUjVjR1VpT2lKQ1lYTnBZMVJwWTJ0bGNpSjlmU3dpYVdRaU9pSXhNREl4SWl3aWRIbHdaU0k2SWtkeWFXUWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2ljMjkxY21ObElqcDdJbWxrSWpvaU1UQTFOaUlzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjE5TENKcFpDSTZJakV3TmpBaUxDSjBlWEJsSWpvaVEwUlRWbWxsZHlKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKa1lYbHpJanBiTVN3eE5WMTlMQ0pwWkNJNklqRXdORFlpTENKMGVYQmxJam9pUkdGNWMxUnBZMnRsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKa1lYbHpJanBiTVN3NExERTFMREl5WFgwc0ltbGtJam9pTVRBME5TSXNJblI1Y0dVaU9pSkVZWGx6VkdsamEyVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdmU3dpYVdRaU9pSXhNVGN4SWl3aWRIbHdaU0k2SWxWdWFXOXVVbVZ1WkdWeVpYSnpJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdmU3dpYVdRaU9pSXhNREkwSWl3aWRIbHdaU0k2SWxKbGMyVjBWRzl2YkNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKemIzVnlZMlVpT25zaWFXUWlPaUl4TURJNUlpd2lkSGx3WlNJNklrTnZiSFZ0YmtSaGRHRlRiM1Z5WTJVaWZYMHNJbWxrSWpvaU1UQXpNeUlzSW5SNWNHVWlPaUpEUkZOV2FXVjNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdmU3dpYVdRaU9pSXhNRGd3SWl3aWRIbHdaU0k2SWxWdWFXOXVVbVZ1WkdWeVpYSnpJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhoWW1Wc0lqcDdJblpoYkhWbElqb2lTR2x6ZEc5eWVWOUNaWE4wSW4wc0luSmxibVJsY21WeWN5STZXM3NpYVdRaU9pSXhNRFU1SWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmVjE5TENKcFpDSTZJakV3T0RFaUxDSjBlWEJsSWpvaVRHVm5aVzVrU1hSbGJTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnZkbVZ5YkdGNUlqcDdJbWxrSWpvaU1UQXpPU0lzSW5SNWNHVWlPaUpDYjNoQmJtNXZkR0YwYVc5dUluMTlMQ0pwWkNJNklqRXdNak1pTENKMGVYQmxJam9pUW05NFdtOXZiVlJ2YjJ3aWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVltOTBkRzl0WDNWdWFYUnpJam9pYzJOeVpXVnVJaXdpWm1sc2JGOWhiSEJvWVNJNmV5SjJZV3gxWlNJNk1DNDFmU3dpWm1sc2JGOWpiMnh2Y2lJNmV5SjJZV3gxWlNJNklteHBaMmgwWjNKbGVTSjlMQ0pzWldaMFgzVnVhWFJ6SWpvaWMyTnlaV1Z1SWl3aWJHVjJaV3dpT2lKdmRtVnliR0Y1SWl3aWJHbHVaVjloYkhCb1lTSTZleUoyWVd4MVpTSTZNUzR3ZlN3aWJHbHVaVjlqYjJ4dmNpSTZleUoyWVd4MVpTSTZJbUpzWVdOckluMHNJbXhwYm1WZlpHRnphQ0k2V3pRc05GMHNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pKOUxDSnlaVzVrWlhKZmJXOWtaU0k2SW1OemN5SXNJbkpwWjJoMFgzVnVhWFJ6SWpvaWMyTnlaV1Z1SWl3aWRHOXdYM1Z1YVhSeklqb2ljMk55WldWdUluMHNJbWxrSWpvaU1UQXpPU0lzSW5SNWNHVWlPaUpDYjNoQmJtNXZkR0YwYVc5dUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltUmhlWE1pT2xzeExESXNNeXcwTERVc05pdzNMRGdzT1N3eE1Dd3hNU3d4TWl3eE15d3hOQ3d4TlN3eE5pd3hOeXd4T0N3eE9Td3lNQ3d5TVN3eU1pd3lNeXd5TkN3eU5Td3lOaXd5Tnl3eU9Dd3lPU3d6TUN3ek1WMTlMQ0pwWkNJNklqRXdORE1pTENKMGVYQmxJam9pUkdGNWMxUnBZMnRsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiQ3dpY21WdVpHVnlaWEp6SWpwYmV5SnBaQ0k2SWpFd05Ua2lMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOVhTd2lkRzl2YkhScGNITWlPbHRiSWs1aGJXVWlMQ0pJYVhOMGIzSjVYMEpsYzNRalptbHNiRzFwYzIxaGRHTm9JbDBzV3lKQ2FXRnpJaXdpTFRBdU5UVWlYU3hiSWxOcmFXeHNJaXdpTUM0Mk5TSmRYWDBzSW1sa0lqb2lNVEE0TWlJc0luUjVjR1VpT2lKSWIzWmxjbFJ2YjJ3aWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWRHVjRkQ0k2SWpRME1ERXpJbjBzSW1sa0lqb2lNVEF3TWlJc0luUjVjR1VpT2lKVWFYUnNaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpqWVd4c1ltRmpheUk2Ym5Wc2JDd2laR0YwWVNJNmV5SjRJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZDUVVjM1lUVmthMGxCUVVOcFMzVmliREpSWjBGQlJWQnRPSFZZV2tOQlFVUTBXamhETldSclNVRkJUMFJYZHpkc01sRm5RVUY1UlZoSWRWaGFRMEZCUTNkMFRYRTFaR3RKUVVGS1oycDZjbXd5VVdkQlFXZEtURkoxV0ZwRFFVRkNiMEZrVnpWa2EwbEJRVVpDZHpKTWJESlJaMEZCVDA0dlluVllXa05CUVVGblZIUXJOV1JyU1VGQlFXazVOSEpzTWxGblFVRTRRM1p0ZFZoYVEwRkJSRmx0ZFcwMVpHdEpRVUZOUVVvM1ltd3lVV2RCUVhGSWFuZDFXRnBEUVVGRFVUVXZUelZrYTBsQlFVaG9WemszYkRKUlowRkJXVTFZTm5WWVdrTkJRVUpKVGxBMk5XUnJTVUZCUkVOcVFXSndNbEZuUVVGSFFrbEdkVzVhUTBGQlFVRm5VV2syWkd0SlFVRlBhblpETjNBeVVXZEJRVEJHTkZCMWJscERRVUZETkhwU1N6WmthMGxCUVV0Qk9FWnljREpSWjBGQmFVdHpXblZ1V2tOQlFVSjNSMmd5Tm1SclNVRkJSbWxLU1V4d01sRm5RVUZSVUdkcWRXNWFRMEZCUVc5YWVXVTJaR3RKUVVGQ1JGZExjbkF5VVdkQlFTdEZVWFYxYmxwRFFVRkVaM042Unpaa2EwbEJRVTFuYVU1aWNESlJaMEZCYzBwRk5IVnVXa05CUVVOWlFVUjVObVJyU1VGQlNVSjJVRGR3TWxGblFVRmhUalZEZFc1YVEwRkJRbEZVVldFMlpHdEpRVUZFYVRoVFluQXlVV2RCUVVsRGRFNTFibHBEUVVGQlNXMXNRelprYTBsQlFWQkJTVlpNY0RKUlowRkJNa2hrV0hWdVdrTkJRVVJCTld4eE5tUnJTVUZCUzJoV1dISndNbEZuUVVGclRWSm9kVzVhUTBGQlFqUk5NbGMyWkd0SlFVRkhRMmxoVEhBeVVXZEJRVk5DUm5OMWJscERRVUZCZDJkSEt6WmthMGxCUVVKcWRtTnljREpSWjBGQlFVWTFNblZ1V2tOQlFVUnZla2h0Tm1SclNVRkJUa0UzWm1Kd01sRm5RVUYxUzNGQmRXNWFRMEZCUTJkSFdWTTJaR3RKUVVGSmFVbG9OM0F5VVdkQlFXTlFaVXQxYmxwRFFVRkNXVnB2Tmpaa2EwbEJRVVZFVm10aWNESlJaMEZCUzBWVFZuVnVXa05CUVVGUmN6VnBObVJyU1VGQlVHZG9ia3h3TWxGblFVRTBTa05tZFc1YVEwRkJSRWt2TmtzMlpHdEpRVUZNUW5Wd2NuQXlVV2RCUVcxT01uQjFibHBEUVVGRFFWUkxNalprYTBsQlFVZHBOM05NY0RKUlowRkJWVU54TUhWdVdrTkJRVUUwYldKbE5tUnJTVUZCUTBGSmRUZHdNbEZuUVVGRFNHVXJkVzVhUTBGQlJIYzFZMGMyWkd0SlFVRk9hRlY0WW5BeVVXZEJRWGROVUVsMWJscERRVUZEYjAxemVUWmthMGxCUVVwRGFIbzNjREpSWjBGQlpVSkVWSFZ1V2tOQlFVSm5aamxoTm1SclNVRkJSV3AxTW1Kd01sRm5RVUZOUmpOa2RXNWFRMEZCUVZsNlQwTTJaR3RKUVVGQlFUYzFUSEF5VVdkQlFUWkxibTUxYmxwRFFVRkVVVWRQZFRaa2EwbEJRVXhwU0RkeWNESlJaMEZCYjFCaWVIVnVXa05CUVVOSldtWlhObVJyU1VGQlNFUlZLMHh3TWxGblFVRlhSVkE0ZFc1YVEwRkJRa0Z6ZGlzMlpHdEpRVUZEWjJoQk4zUXlVV2RCUVVWS1FVZDFNMXBEUVVGRU5DOW5iVGRrYTBsQlFVOUNkRVJpZERKUlowRkJlVTUzVVhVeldrTkJRVU4zVTNoVE4yUnJTVUZCU21rMlJqZDBNbEZuUVVGblEydGlkVE5hUTBGQlFtOXRRalkzWkd0SlFVRkdRVWhKY25ReVVXZEJRVTlJV1d4MU0xcERRVUZCWnpWVGFUZGthMGxCUVVGb1ZVeE1kREpSWjBGQk9FMUpkblV6V2tOQlFVUlpUVlJQTjJSclNVRkJUVU5uVG5KME1sRm5RVUZ4UVRnMmRUTmFRMEZCUTFGbWFqSTNaR3RKUVVGSWFuUlJUSFF5VVdkQlFWbEdlRVYxTTFwRFFVRkNTWGt3WlRka2EwbEJRVVJCTmxNM2RESlJaMEZCUjB0c1QzVXpXa05CUVVGQlIwWkxOMlJyU1VGQlQybEhWbUowTWxGblFVRXdVRlpaZFROYVEwRkJRelJhUm5rM1pHdEpRVUZMUkZSWU4zUXlVV2RCUVdsRlNtcDFNMXBEUVVGQ2QzTlhZVGRrYTBsQlFVWm5aMkZ5ZERKUlowRkJVVWs1ZEhVeldrTkJRVUZ2TDI1RE4yUnJTVUZCUWtKMFpFeDBNbEZuUVVFclRuUXpkVE5hUTBGQlJHZFRiblUzWkd0SlFVRk5hVFZtY25ReVVXZEJRWE5EYVVOMU0xcERRVUZEV1d3MFZ6ZGthMGxCUVVsQlIybGlkREpSWnowOUlpd2laSFI1Y0dVaU9pSm1iRzloZERZMElpd2ljMmhoY0dVaU9sc3hNemRkZlN3aWVTSTZleUpmWDI1a1lYSnlZWGxmWHlJNklscHRXbTFhYldKdFRVVkNiVnB0V20xYWRWbDNVVWRhYlZwdFdtMDFha0pCV20xYWJWcHRZbTFOUlVST2VrMTZUWHBOZDNkUlRUTk5lazE2VFhwRVFrRjZZM3BOZWsxNlRVMUZSRTU2VFhwTmVrMTNkMUZFVFhwTmVrMTZjM3BDUVVGQlFVRkJRVU5CVFVWQlFVRkJRVUZCUVVGM1VVZGFiVnB0V20xYWFUbEJlbU42VFhwTmVrMU1NRU5oYlZwdFdtMVNhM2RSUkUxNlRYcE5lbk42UWtGTmVrMTZUWHBQZWsxRlFVRkJRVUZCUVVGQmVGRkhXbTFhYlZwdFdtcEdRVzF3YlZwdFdtdGFUVlZCUVVGQlFVRkJTVUY0VVVkYWJWcHRXbTAxYWtKQmJYQnRXbTFhYlZwTlJVTmhiVnB0V20xYWEzZFJSRTE2VFhwTmVuTjZRa0Z0Y0cxYWJWcHRXazFGUVhwTmVrMTZUVGROZDFGRVRYcE5lazE2YzNwQ1FXMXdiVnB0V20xYVRVVkJRVUZCUVVGQlNVRjNVVUZCUVVGQlFVRm5SRUpCYlhCdFdtMWFiVnBOUlVOaGJWcHRXbTFhYTNkUlFVRkJRVUZCUVdkRVFrRjZZM3BOZWsxNFRVMUZRWHBOZWsxNlRYcE5kMUZOTTAxNlRYcE5WRVJDUVhwamVrMTZUWGhOVFVWQ2JWcHRXbTFhYlZsM1VVcHhXbTFhYlZwdFZFSkJlbU42VFhwTmVrMU5SVUp0V20xYWJWcDFXWGRSUVVGQlFVRkJRV2RFUmtGQlFVRkJRVUZCUVUxclJFNTZUWHBOZWtWM2VWRktjVnB0V20xYVIxUktRVUZCUVVGQlFVRkJUV3RDYlZwdFdtMWFkVmw0VVUwelRYcE5lazE2UkVaQlFVRkJRVUZCUTBGTlZVSnRXbTFhYlZwMVdYaFJUVE5OZWsxNlRYcEVSa0Z0Y0cxYWJWcHRXazFWUVVGQlFVRkJRVWxCZUZGQlFVRkJRVUZCWjBSR1FWcHRXbTFhYlZwdFRWVkJlazE2VFhwTmVrMTRVVXB4V20xYWJWcEhWRVpCYlhCdFdtMWFhMXBOVlVOaGJWcHRXbTFTYTNoUlFVRkJRVUZCUVVGRVJrRjZZM3BOZWsxNFRVMVZRMkZ0V20xYWJWcHJlRkZOTTAxNlRYcE5la1JHUVcxd2JWcHRXbXRhVFd0RFlXMWFiVnB0V210NVVVcHhXbTFhYlZwdFZFcEJUWHBOZWsxNlRYcE5hMEY2VFhwTmVrMTZUWGxSUVVGQlFVRkJRV2RFU2tGNlkzcE5lazE0VFUxclJFNTZUWHBOZWtWM2VWRkJRVUZCUVVGQlFVUktRVUZCUVVGQlFVRkJUV3REWVcxYWJWcHRVbXQ1VVVkYWJWcHRXbTAxYWtaQlRYcE5lazE2VDNwTlZVRjZUWHBOZWswM1RYaFJUVE5OZWsxNlRYcEVSa0ZhYlZwdFdtMWliVTFWUTJGdFdtMWFiVkpyZVZGS2NWcHRXbTFhUjFSS1FYcGplazE2VFhwTlRWVkRZVzFhYlZwdFdtdDRVVWRhYlZwdFdtMDFha1pCUVVGQlFVRkJRVUZOYTBGNlRYcE5lazE2VFhsUlIxcHRXbTFhYlZwcVNrRkJRVUZCUVVGRFFVMXJRbTFhYlZwdFduVlplVkZCUVVGQlFVRkJaMFJPUVcxd2JWcHRXbXRhVGtWRFlXMWFiVnB0VW1zd1VVRkJRVUZCUVVGQlJGSkJiWEJ0V20xYWExcE9SVU5oYlZwdFdtMWFhM3BSVFROTmVrMTZUVlJFVGtGdGNHMWFiVnByV2swd1FVRkJRVUZCUVVGQmVsRkJRVUZCUVVGQlFVUk9RVUZCUVVGQlFVRkJUVEJCUVVGQlFVRkJRVUY2VVVkYWJWcHRXbTAxYWtwQmVtTjZUWHBOZWsxTmEwRjZUWHBOZWswM1RYbFJTbkZhYlZwdFdtMVVTa0Z0Y0cxYWJWcHRXazFyUTJGdFdtMWFiVnByZVZGQlFVRkJRVUZCWjBSS1FVRkJRVUZCUVVOQlRXdEJRVUZCUVVGQlNVRjVVVTB6VFhwTmVrMTZSRXBCV20xYWJWcHRZbTFOYTBOaGJWcHRXbTFTYTNwUlJFMTZUWHBOZWsxNlRrRmFiVnB0V20xYWJVMHdRMkZ0V20xYWJWcHJlbEZOTTAxNlRYcE5la1JPUVhwamVrMTZUWGhOVGtWRVRucE5lazE2Ulhjd1VVZGFiVnB0V20xYWFsSkJlbU42VFhwTmVFMU9SVUY2VFhwTmVrMTZUVEJSUVVGQlFVRkJRVUZFVWtGNlkzcE5lazE2VFUwd1JFNTZUWHBOZWsxM2VsRkVUWHBOZWsxNmMzcE9RVzF3YlZwdFdtMWFUVEJCUVVGQlFVRkJTVUY2VVVkYWJWcHRXbTFhYWs1QldtMWFiVnB0V20xTk1FUk9lazE2VFhwRmQzcFJUVE5OZWsxNlRWUkVUa0ZhYlZwdFdtMWFiVTB3UVVGQlFVRkJRVWxCZWxGQlFVRkJRVUZCUVVSU1FVRkJRVUZCUVVGQlRsVkVUbnBOZWsxNlRYY3dVVUU5UFNJc0ltUjBlWEJsSWpvaVpteHZZWFEyTkNJc0luTm9ZWEJsSWpwYk1UTTNYWDE5TENKelpXeGxZM1JsWkNJNmV5SnBaQ0k2SWpFeE5ERWlMQ0owZVhCbElqb2lVMlZzWldOMGFXOXVJbjBzSW5ObGJHVmpkR2x2Ymw5d2IyeHBZM2tpT25zaWFXUWlPaUl4TVRReUlpd2lkSGx3WlNJNklsVnVhVzl1VW1WdVpHVnlaWEp6SW4xOUxDSnBaQ0k2SWpFd09EUWlMQ0owZVhCbElqb2lRMjlzZFcxdVJHRjBZVk52ZFhKalpTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2SW1OeWFXMXpiMjRpTENKc2FXNWxYMnB2YVc0aU9pSnliM1Z1WkNJc0lteHBibVZmZDJsa2RHZ2lPalVzSW5naU9uc2labWxsYkdRaU9pSjRJbjBzSW5raU9uc2labWxsYkdRaU9pSjVJbjE5TENKcFpDSTZJakV3T0RVaUxDSjBlWEJsSWpvaVRHbHVaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzYVc1bFgyRnNjR2hoSWpvd0xqRXNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpvaUl6Rm1OemRpTkNJc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2TlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pTVRBNE5pSXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1SaGRHRmZjMjkxY21ObElqcDdJbWxrSWpvaU1UQTROQ0lzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjBzSW1kc2VYQm9JanA3SW1sa0lqb2lNVEE0TlNJc0luUjVjR1VpT2lKTWFXNWxJbjBzSW1odmRtVnlYMmRzZVhCb0lqcHVkV3hzTENKdGRYUmxaRjluYkhsd2FDSTZiblZzYkN3aWJtOXVjMlZzWldOMGFXOXVYMmRzZVhCb0lqcDdJbWxrSWpvaU1UQTROaUlzSW5SNWNHVWlPaUpNYVc1bEluMHNJbk5sYkdWamRHbHZibDluYkhsd2FDSTZiblZzYkN3aWRtbGxkeUk2ZXlKcFpDSTZJakV3T0RnaUxDSjBlWEJsSWpvaVEwUlRWbWxsZHlKOWZTd2lhV1FpT2lJeE1EZzNJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWMyOTFjbU5sSWpwN0ltbGtJam9pTVRBNE5DSXNJblI1Y0dVaU9pSkRiMngxYlc1RVlYUmhVMjkxY21ObEluMTlMQ0pwWkNJNklqRXdPRGdpTENKMGVYQmxJam9pUTBSVFZtbGxkeUo5TEhzaVlYUjBjbWxpZFhSbGN5STZlMzBzSW1sa0lqb2lNVEF4TUNJc0luUjVjR1VpT2lKTWFXNWxZWEpUWTJGc1pTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmtZWGx6SWpwYk1TdzBMRGNzTVRBc01UTXNNVFlzTVRrc01qSXNNalVzTWpoZGZTd2lhV1FpT2lJeE1EUTBJaXdpZEhsd1pTSTZJa1JoZVhOVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnQ5TENKcFpDSTZJakV4TURraUxDSjBlWEJsSWpvaVUyVnNaV04wYVc5dUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN2ZTd2lhV1FpT2lJeE1UY3dJaXdpZEhsd1pTSTZJbE5sYkdWamRHbHZiaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUppWVhObElqb3lOQ3dpYldGdWRHbHpjMkZ6SWpwYk1Td3lMRFFzTml3NExERXlYU3dpYldGNFgybHVkR1Z5ZG1Gc0lqbzBNekl3TURBd01DNHdMQ0p0YVc1ZmFXNTBaWEoyWVd3aU9qTTJNREF3TURBdU1Dd2liblZ0WDIxcGJtOXlYM1JwWTJ0eklqb3dmU3dpYVdRaU9pSXhNRFF5SWl3aWRIbHdaU0k2SWtGa1lYQjBhWFpsVkdsamEyVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdmU3dpYVdRaU9pSXhNVEV3SWl3aWRIbHdaU0k2SWxWdWFXOXVVbVZ1WkdWeVpYSnpJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhoWW1Wc0lqcDdJblpoYkhWbElqb2lUMkp6WlhKMllYUnBiMjRpZlN3aWNtVnVaR1Z5WlhKeklqcGJleUpwWkNJNklqRXdPRGNpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYWDBzSW1sa0lqb2lNVEV4TVNJc0luUjVjR1VpT2lKTVpXZGxibVJKZEdWdEluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN2ZTd2lhV1FpT2lJeE1EQTRJaXdpZEhsd1pTSTZJa3hwYm1WaGNsTmpZV3hsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW0xdmJuUm9jeUk2V3pBc05DdzRYWDBzSW1sa0lqb2lNVEEwT1NJc0luUjVjR1VpT2lKTmIyNTBhSE5VYVdOclpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZMkZzYkdKaFkyc2lPbTUxYkd3c0luSmxibVJsY21WeWN5STZXM3NpYVdRaU9pSXhNRGczSWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmVjBzSW5SdmIyeDBhWEJ6SWpwYld5Sk9ZVzFsSWl3aVQySnpaWEoyWVhScGIyNXpJbDBzV3lKQ2FXRnpJaXdpVGtFaVhTeGJJbE5yYVd4c0lpd2lUa0VpWFYxOUxDSnBaQ0k2SWpFeE1USWlMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbTF2Ym5Sb2N5STZXekFzTWl3MExEWXNPQ3d4TUYxOUxDSnBaQ0k2SWpFd05EZ2lMQ0owZVhCbElqb2lUVzl1ZEdoelZHbGphMlZ5SW4xZExDSnliMjkwWDJsa2N5STZXeUl4TURBeElsMTlMQ0owYVhSc1pTSTZJa0p2YTJWb0lFRndjR3hwWTJGMGFXOXVJaXdpZG1WeWMybHZiaUk2SWpFdU1pNHdJbjE5Q2lBZ0lDQWdJQ0FnUEM5elkzSnBjSFErQ2lBZ0lDQWdJQ0FnUEhOamNtbHdkQ0IwZVhCbFBTSjBaWGgwTDJwaGRtRnpZM0pwY0hRaVBnb2dJQ0FnSUNBZ0lDQWdLR1oxYm1OMGFXOXVLQ2tnZXdvZ0lDQWdJQ0FnSUNBZ0lDQjJZWElnWm00Z1BTQm1kVzVqZEdsdmJpZ3BJSHNLSUNBZ0lDQWdJQ0FnSUNBZ0lDQkNiMnRsYUM1ellXWmxiSGtvWm5WdVkzUnBiMjRvS1NCN0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBb1puVnVZM1JwYjI0b2NtOXZkQ2tnZXdvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNCbWRXNWpkR2x2YmlCbGJXSmxaRjlrYjJOMWJXVnVkQ2h5YjI5MEtTQjdDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnQ2lBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUhaaGNpQmtiMk56WDJwemIyNGdQU0JrYjJOMWJXVnVkQzVuWlhSRmJHVnRaVzUwUW5sSlpDZ25NVE00TUNjcExuUmxlSFJEYjI1MFpXNTBPd29nSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0IyWVhJZ2NtVnVaR1Z5WDJsMFpXMXpJRDBnVzNzaVpHOWphV1FpT2lKbE9EUmtNalkzTWkwNVlUQmlMVFF3T1dVdFlXRXlPUzAyTWpNeVltWXpPVFF5WXpJaUxDSnliMjkwY3lJNmV5SXhNREF4SWpvaU5qTmpPVFl6T0RVdFlUQTJaUzAwWkRnNUxUaGlaVFl0TW1ObU1UUXpOR1pqTmpFMkluMTlYVHNLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnY205dmRDNUNiMnRsYUM1bGJXSmxaQzVsYldKbFpGOXBkR1Z0Y3loa2IyTnpYMnB6YjI0c0lISmxibVJsY2w5cGRHVnRjeWs3Q2lBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnZlFvZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNCcFppQW9jbTl2ZEM1Q2IydGxhQ0FoUFQwZ2RXNWtaV1pwYm1Wa0tTQjdDaUFnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnWlcxaVpXUmZaRzlqZFcxbGJuUW9jbTl2ZENrN0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lIMGdaV3h6WlNCN0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdkbUZ5SUdGMGRHVnRjSFJ6SUQwZ01Ec0tJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0IyWVhJZ2RHbHRaWElnUFNCelpYUkpiblJsY25aaGJDaG1kVzVqZEdsdmJpaHliMjkwS1NCN0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0JwWmlBb2NtOXZkQzVDYjJ0bGFDQWhQVDBnZFc1a1pXWnBibVZrS1NCN0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUdWdFltVmtYMlJ2WTNWdFpXNTBLSEp2YjNRcE93b2dJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNCamJHVmhja2x1ZEdWeWRtRnNLSFJwYldWeUtUc0tJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUgwS0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJR0YwZEdWdGNIUnpLeXM3Q2lBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQnBaaUFvWVhSMFpXMXdkSE1nUGlBeE1EQXBJSHNLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdZMjl1YzI5c1pTNXNiMmNvSWtKdmEyVm9PaUJGVWxKUFVqb2dWVzVoWW14bElIUnZJSEoxYmlCQ2IydGxhRXBUSUdOdlpHVWdZbVZqWVhWelpTQkNiMnRsYUVwVElHeHBZbkpoY25rZ2FYTWdiV2x6YzJsdVp5SXBPd29nSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQmpiR1ZoY2tsdWRHVnlkbUZzS0hScGJXVnlLVHNLSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lIMEtJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ0lDQWdJQ0I5TENBeE1Dd2djbTl2ZENrS0lDQWdJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ2ZRb2dJQ0FnSUNBZ0lDQWdJQ0FnSUNBZ2ZTa29kMmx1Wkc5M0tUc0tJQ0FnSUNBZ0lDQWdJQ0FnSUNCOUtUc0tJQ0FnSUNBZ0lDQWdJQ0FnZlRzS0lDQWdJQ0FnSUNBZ0lDQWdhV1lnS0dSdlkzVnRaVzUwTG5KbFlXUjVVM1JoZEdVZ0lUMGdJbXh2WVdScGJtY2lLU0JtYmlncE93b2dJQ0FnSUNBZ0lDQWdJQ0JsYkhObElHUnZZM1Z0Wlc1MExtRmtaRVYyWlc1MFRHbHpkR1Z1WlhJb0lrUlBUVU52Ym5SbGJuUk1iMkZrWldRaUxDQm1iaWs3Q2lBZ0lDQWdJQ0FnSUNCOUtTZ3BPd29nSUNBZ0lDQWdJRHd2YzJOeWFYQjBQZ29nSUNBZ0NpQWdQQzlpYjJSNVBnb2dJQW84TDJoMGJXdysiIHdpZHRoPSI3OTAiIHN0eWxlPSJib3JkZXI6bm9uZSAhaW1wb3J0YW50OyIgaGVpZ2h0PSIzMzAiPjwvaWZyYW1lPmApWzBdOwogICAgICAgICAgICBwb3B1cF9lN2I2ZDg0NDI4MTg0NGNjYTkxOGNhYTU5ZjU2NzdjOC5zZXRDb250ZW50KGlfZnJhbWVfYWRmZjNlYjg0NGViNDcxZDllMDIyOTYxYWM5MDhkMmYpOwogICAgICAgIAoKICAgICAgICBtYXJrZXJfYzIzNDY3ODQzYzVlNDZlOWE1ZjE4OTkwZTc2ZDhhNDMuYmluZFBvcHVwKHBvcHVwX2U3YjZkODQ0MjgxODQ0Y2NhOTE4Y2FhNTlmNTY3N2M4KQogICAgICAgIDsKCiAgICAgICAgCiAgICAKICAgIAogICAgICAgICAgICB2YXIgbGF5ZXJfY29udHJvbF82ZjI0MDVhNWEyZWI0ZjQ0OGQ0MjA5ZDk2YjA4ZjIxMCA9IHsKICAgICAgICAgICAgICAgIGJhc2VfbGF5ZXJzIDogewogICAgICAgICAgICAgICAgICAgICJvcGVuc3RyZWV0bWFwIiA6IHRpbGVfbGF5ZXJfZWQ3MWJkZjkxOGY3NDRjNGFlZDNjNDFjZGJiYzc0YjAsCiAgICAgICAgICAgICAgICB9LAogICAgICAgICAgICAgICAgb3ZlcmxheXMgOiAgewogICAgICAgICAgICAgICAgICAgICJTZWEgU3VyZmFjZSBUZW1wZXJhdHVyZSIgOiBtYWNyb19lbGVtZW50XzRkMDc3ZWQ3Mjc4YTQ3MzM4MjBkODM0NmY2YzE3ZjI1LAogICAgICAgICAgICAgICAgICAgICJDbHVzdGVyIiA6IG1hcmtlcl9jbHVzdGVyXzUwMmFkOThlOTE0YjQzOWU5ZWU2NDhmZjc5NmRlM2IxLAogICAgICAgICAgICAgICAgfSwKICAgICAgICAgICAgfTsKICAgICAgICAgICAgTC5jb250cm9sLmxheWVycygKICAgICAgICAgICAgICAgIGxheWVyX2NvbnRyb2xfNmYyNDA1YTVhMmViNGY0NDhkNDIwOWQ5NmIwOGYyMTAuYmFzZV9sYXllcnMsCiAgICAgICAgICAgICAgICBsYXllcl9jb250cm9sXzZmMjQwNWE1YTJlYjRmNDQ4ZDQyMDlkOTZiMDhmMjEwLm92ZXJsYXlzLAogICAgICAgICAgICAgICAgeyJhdXRvWkluZGV4IjogdHJ1ZSwgImNvbGxhcHNlZCI6IHRydWUsICJwb3NpdGlvbiI6ICJ0b3ByaWdodCJ9CiAgICAgICAgICAgICkuYWRkVG8obWFwXzFkNzc0OGNiYWJhZDQ3YzNhZDM0MWRjNWJkY2UxMGJkKTsKICAgICAgICAKPC9zY3JpcHQ+" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>



Now we can navigate the map and click on the markers to explorer our findings.

The green markers locate the observations locations. They pop-up an interactive plot with the time-series and scores for the models (hover over the lines to se the scores). The blue markers indicate the nearest model grid point found for the comparison.
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2016-12-22-boston_light_swim.ipynb)
this notebook, or click [here](https://binder.pangeo.io/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2016-12-22-boston_light_swim.ipynb) to run a live instance of this notebook.
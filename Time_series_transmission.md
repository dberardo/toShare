
# Thoughts on time series transmission

The simulation models used as part of the MiST project may generate lots of time series.
Such a time series usually includes multiple data points each of which consist of a time stamp and an associated measurement value.
Those time series have to transmitted to the Client respectively Browser in order to be displayed.
Depending on the number of data points a time series might be really large thus the application's loading times and the resulting user experience may suffer.

This document discusses different strategies to transmit those time series in order to save bandwidth and time.


```python
import numpy as np
import json
import struct

# Let's define a generator that creates a time series with n data points
def generate_timeseries(n=10):
    for i in range(0, n):
        yield {
            "time": i*1000,
            "data": np.random.random_sample()
        }
```

### Plain human-readable JSON

The most naive approach would be to generate human-readable JSON which of course consumes quite a lot of space.


```python
timeseries = list(generate_timeseries())
timeseries_json = json.dumps(timeseries, indent=2, separators=(',', ': '))
print(timeseries_json)
print('Length:', len(timeseries_json))
```

    [
      {
        "data": 0.9303125920977541,
        "time": 0
      },
      {
        "data": 0.0667782230883297,
        "time": 1000
      },
      {
        "data": 0.6171465798217687,
        "time": 2000
      },
      {
        "data": 0.15409131228793504,
        "time": 3000
      },
      {
        "data": 0.680590895538983,
        "time": 4000
      },
      {
        "data": 0.054276471577991425,
        "time": 5000
      },
      {
        "data": 0.5016513931759689,
        "time": 6000
      },
      {
        "data": 0.2064370783135262,
        "time": 7000
      },
      {
        "data": 0.8676049167773978,
        "time": 8000
      },
      {
        "data": 0.6196511476562787,
        "time": 9000
      }
    ]
    Length: 581


### Stripping white spaces from JSON

Stripping all the white spaces can reduce the amount of space needed but readability suffers.
In the following for each alternative approach we will print the amount of space that could be saved in contrast to the the naive approach before.


```python
timeseries_stripped_json = json.dumps(timeseries, separators=(',',':'))
print(timeseries_stripped_json)
print('Length:', len(timeseries_stripped_json))
print('Reduction:', len(timeseries_stripped_json)/len(timeseries_json))
```

    [{"data":0.9303125920977541,"time":0},{"data":0.0667782230883297,"time":1000},{"data":0.6171465798217687,"time":2000},{"data":0.15409131228793504,"time":3000},{"data":0.680590895538983,"time":4000},{"data":0.054276471577991425,"time":5000},{"data":0.5016513931759689,"time":6000},{"data":0.2064370783135262,"time":7000},{"data":0.8676049167773978,"time":8000},{"data":0.6196511476562787,"time":9000}]
    Length: 400
    Reduction: 0.6884681583476764


### Stripping keys from JSON objects

We do not necessarily have to use objects for each data point, instead we can use simple arrays.
The downside of this approach is that the client has to know the position of the timestamp and the data field in the array.


```python
timeseries_without_keys = list(zip([ x["time"] for x in timeseries ], [ x["data"] for x in timeseries ]))

timeseries_without_keys_json = json.dumps(timeseries_without_keys, separators=(',',':'))
print(timeseries_without_keys_json)
print('Length:', len(timeseries_without_keys_json))
print('Reduction:', len(timeseries_without_keys_json)/len(timeseries_json))
```

    [[0,0.9303125920977541],[1000,0.0667782230883297],[2000,0.6171465798217687],[3000,0.15409131228793504],[4000,0.680590895538983],[5000,0.054276471577991425],[6000,0.5016513931759689],[7000,0.2064370783135262],[8000,0.8676049167773978],[9000,0.6196511476562787]]
    Length: 260
    Reduction: 0.4475043029259897


### Gaining advantage of equidistant time stamps

Now let's assume that the timestamps are equidistant.
In that particular case we only have to transmit a start offset and the difference between two timestamps.
This allows us to greatly save space when dealing with a lot of data points.


```python
timeseries_without_keys_and_timestamps = {
    'ts_start': 0,
    'ts_diff': 1000,
    'data': [ x[1] for x in timeseries_without_keys ]
}

timeseries_without_keys_and_timestamps_json = json.dumps(timeseries_without_keys_and_timestamps, separators=(',',':'))
print(timeseries_without_keys_and_timestamps_json)
print('Length:', len(timeseries_without_keys_and_timestamps_json))
print('Reduction:', len(timeseries_without_keys_and_timestamps_json)/len(timeseries_json))
```

    {"ts_start":0,"data":[0.9303125920977541,0.0667782230883297,0.6171465798217687,0.15409131228793504,0.680590895538983,0.054276471577991425,0.5016513931759689,0.2064370783135262,0.8676049167773978,0.6196511476562787],"ts_diff":1000}
    Length: 230
    Reduction: 0.3958691910499139


### Using seekable binary packed data

Instead of encapsulating data into JSON we could pack it into a binary representation.
Again we have to tell the client the start offset and the difference of the timestamps.
Unlike the JSON encoded data that is not seekable due to the fact that the string length of each data point object might be different, the binary data format here can be seeked.
Thus it might be possible to only fetch the latest data points or some points somewhere in the middle of the time series.
Also it greatly saves space.
Unfortunately the data is no longer human-readable and on the client side the data has to be unpacked.


```python
# ts_start and ts_diff should go into the header
timeseries_binary = struct.pack(
    'iI',
    timeseries_without_keys_and_timestamps['ts_start'],
    timeseries_without_keys_and_timestamps['ts_diff']
)

# now loop over every data point and pack it
for data in timeseries_without_keys_and_timestamps['data']:
    timeseries_binary += struct.pack('f', data)

print(timeseries_binary)
print('Length:', len(timeseries_binary))
print('Reduction:', len(timeseries_binary)/len(timeseries_json))
```

    b'\x00\x00\x00\x00\xe8\x03\x00\x00\xf7(n?\x05\xc3\x88=Q\xfd\x1d?\x1d\xca\x1d>4;.?\x01Q^=:l\x00?>dS>[\x1b^?u\xa1\x1e?'
    Length: 48
    Reduction: 0.08261617900172118


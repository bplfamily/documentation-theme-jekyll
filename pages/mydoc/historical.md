---
title: About the theme's author
keywords: documentation theme, jekyll, technical writers, help authoring tools, hat replacements
last_updated: July 3, 2016
tags: [getting_started]
summary: "Historical data"
sidebar: mydoc_sidebar
permalink: historical.html
folder: mydoc
---
# CoinGecko Historical Data

Let's set up the environment first. We need the coingecko API to fetch prices, and pandas to work with timeseries (and tabular data in general).


```python
from pycoingecko import CoinGeckoAPI
gecko = CoinGeckoAPI()
```


```python
import pandas as pd
```


```python
import matplotlib.pyplot as plt
%matplotlib notebook
```


```python
import datetime as dt
```


```python
import requests
```


```python
from IPython.display import clear_output
```

# Experiments

First, let's set up the correct URL and parsing and all that jazz.


```python
def get_historical_data_url(coin_id, currency_id, start_date, end_date):
    return 'https://www.coingecko.com/en/coins/{}/historical_data/{}?end_date={}&start_date={}'.format(coin_id, currency_id, end_date.isoformat(), start_date.isoformat()) 
```


```python
def get_historical_data(coin_id, currency_id, start_date_s, end_date_s):
    start_date = dt.date.fromisoformat(start_date_s)
    end_date = dt.date.fromisoformat(end_date_s)

    url = get_historical_data_url('bitcoin', 'usd', start_date, end_date)
    r = requests.get(url, timeout=5, stream=True)
    dfs = pd.read_html(r.content, parse_dates=['Date'])
    assert len(dfs) == 1

    df = dfs[0][::-1].set_index("Date")
    df = df.replace({'[$,]': ''}, regex=True).apply(pd.to_numeric)
    df.columns = pd.MultiIndex.from_product([[coin_id], df.columns])
    return df
```


```python
df = get_historical_data('bitcoin', 'usd', '2013-01-01', '2021-04-01')
```


```python
df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead tr th {
        text-align: left;
    }

    .dataframe thead tr:last-of-type th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th colspan="4" halign="left">bitcoin</th>
    </tr>
    <tr>
      <th></th>
      <th>Market Cap</th>
      <th>Volume</th>
      <th>Open</th>
      <th>Close</th>
    </tr>
    <tr>
      <th>Date</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2013-04-28</th>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
    </tr>
    <tr>
      <th>2013-04-29</th>
      <td>1.575032e+09</td>
      <td>0.000000e+00</td>
      <td>141.96</td>
      <td>135.30</td>
    </tr>
    <tr>
      <th>2013-04-30</th>
      <td>1.501657e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>117.00</td>
    </tr>
    <tr>
      <th>2013-05-01</th>
      <td>1.298952e+09</td>
      <td>0.000000e+00</td>
      <td>117.00</td>
      <td>103.43</td>
    </tr>
    <tr>
      <th>2013-05-02</th>
      <td>1.148668e+09</td>
      <td>0.000000e+00</td>
      <td>103.43</td>
      <td>91.01</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>2021-03-24</th>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
    </tr>
    <tr>
      <th>2021-03-25</th>
      <td>9.827974e+11</td>
      <td>7.302905e+10</td>
      <td>52527.00</td>
      <td>51417.00</td>
    </tr>
    <tr>
      <th>2021-03-26</th>
      <td>9.591553e+11</td>
      <td>6.340164e+10</td>
      <td>51417.00</td>
      <td>55033.00</td>
    </tr>
    <tr>
      <th>2021-03-27</th>
      <td>1.027210e+12</td>
      <td>5.544256e+10</td>
      <td>55033.00</td>
      <td>55832.00</td>
    </tr>
    <tr>
      <th>2021-03-28</th>
      <td>1.042184e+12</td>
      <td>4.728575e+10</td>
      <td>55832.00</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>2890 rows × 4 columns</p>
</div>




```python
coins_list = pd.DataFrame(gecko.get_coins_list())
```


```python
# df_complete = pd.DataFrame(index=pd.date_range(start='20130101', freq='1D', end='20210401'))

df_complete = None
coins_list_ids = coins_list['id']
coins_list_ids_len = len(coins_list_ids)

for i in range(coins_list_ids_len):
    coin = coins_list_ids[i]
    df = get_historical_data(coin, 'usd', '2013-01-01', '2021-04-01')
    if df_complete is None:
        df_complete = df
    else:
        df_complete = df_complete.join(df, how='left')
    
    clear_output(wait=True)
    print ("Downloaded", coin, "{}/{}".format(i, coins_list_ids_len))
```

    Downloaded chartex 1321/6545



```python
df_complete
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead tr th {
        text-align: left;
    }

    .dataframe thead tr:last-of-type th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th colspan="4" halign="left">01coin</th>
      <th colspan="4" halign="left">0-5x-long-algorand-token</th>
    </tr>
    <tr>
      <th></th>
      <th>Market Cap</th>
      <th>Volume</th>
      <th>Open</th>
      <th>Close</th>
      <th>Market Cap</th>
      <th>Volume</th>
      <th>Open</th>
      <th>Close</th>
    </tr>
    <tr>
      <th>Date</th>
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
      <th>2013-04-28</th>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
    </tr>
    <tr>
      <th>2013-04-29</th>
      <td>1.575032e+09</td>
      <td>0.000000e+00</td>
      <td>141.96</td>
      <td>135.30</td>
      <td>1.575032e+09</td>
      <td>0.000000e+00</td>
      <td>141.96</td>
      <td>135.30</td>
    </tr>
    <tr>
      <th>2013-04-30</th>
      <td>1.501657e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>117.00</td>
      <td>1.501657e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>117.00</td>
    </tr>
    <tr>
      <th>2013-05-01</th>
      <td>1.298952e+09</td>
      <td>0.000000e+00</td>
      <td>117.00</td>
      <td>103.43</td>
      <td>1.298952e+09</td>
      <td>0.000000e+00</td>
      <td>117.00</td>
      <td>103.43</td>
    </tr>
    <tr>
      <th>2013-05-02</th>
      <td>1.148668e+09</td>
      <td>0.000000e+00</td>
      <td>103.43</td>
      <td>91.01</td>
      <td>1.148668e+09</td>
      <td>0.000000e+00</td>
      <td>103.43</td>
      <td>91.01</td>
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
    </tr>
    <tr>
      <th>2021-03-24</th>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
    </tr>
    <tr>
      <th>2021-03-25</th>
      <td>9.827974e+11</td>
      <td>7.302905e+10</td>
      <td>52527.00</td>
      <td>51417.00</td>
      <td>9.827974e+11</td>
      <td>7.302905e+10</td>
      <td>52527.00</td>
      <td>51417.00</td>
    </tr>
    <tr>
      <th>2021-03-26</th>
      <td>9.591553e+11</td>
      <td>6.340164e+10</td>
      <td>51417.00</td>
      <td>55033.00</td>
      <td>9.591553e+11</td>
      <td>6.340164e+10</td>
      <td>51417.00</td>
      <td>55033.00</td>
    </tr>
    <tr>
      <th>2021-03-27</th>
      <td>1.027210e+12</td>
      <td>5.544256e+10</td>
      <td>55033.00</td>
      <td>55832.00</td>
      <td>1.027210e+12</td>
      <td>5.544256e+10</td>
      <td>55033.00</td>
      <td>55832.00</td>
    </tr>
    <tr>
      <th>2021-03-28</th>
      <td>1.042184e+12</td>
      <td>4.728575e+10</td>
      <td>55832.00</td>
      <td>NaN</td>
      <td>1.042184e+12</td>
      <td>4.728575e+10</td>
      <td>55832.00</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>2890 rows × 8 columns</p>
</div>



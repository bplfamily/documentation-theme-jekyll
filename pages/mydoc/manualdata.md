---
title: About the theme's author
keywords: documentation theme, jekyll, technical writers, help authoring tools, hat replacements
last_updated: July 3, 2016
tags: [getting_started]
summary: "Gathered data manually"
sidebar: mydoc_sidebar
permalink: manual.html
folder: mydoc
---
# Manual

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
from time import sleep
```


```python
from IPython.display import clear_output
```

# Experiments

First, let's set up the correct URL and parsing and all that jazz.


```python
def __get_historical_data_url(coin_id, currency_id, start_date, end_date):
    return 'https://www.coingecko.com/en/coins/{}/historical_data/{}?end_date={}&start_date={}'.format(coin_id, currency_id, end_date.isoformat(), start_date.isoformat()) 
```


```python
def get_historical_data_scraper(coin_id, currency_id, start_date_s, end_date_s):
    start_date = dt.date.fromisoformat(start_date_s)
    end_date = dt.date.fromisoformat(end_date_s)

    url = __get_historical_data_url(coin_id, currency_id, start_date, end_date)
    r = requests.get(url, timeout=5, stream=True)
    
    try:
        dfs = pd.read_html(r.content, parse_dates=['Date'])
    except:
        return None
    if len(dfs) != 1:
        return None

    df = dfs[0][::-1].set_index("Date")
    df = df.replace({'[$,]': ''}, regex=True).apply(pd.to_numeric)
    df.columns = pd.MultiIndex.from_product([[coin_id], df.columns])

    return df
```


```python
def get_historical_data_gecko(coin_id, currency_id, start_date_s, end_date_s):
    p = gecko.get_coin_market_chart_range_by_id(coin_id, currency_id, dt.datetime.fromisoformat(start_date_s).strftime("%s"), dt.datetime.fromisoformat(end_date_s).strftime("%s"))

    df_mcaps = pd.DataFrame(p['market_caps'], columns=['Date', 'Market Cap'])
    df_mcaps['Date'] = pd.to_datetime(df_mcaps['Date'].apply(lambda x: dt.datetime.utcfromtimestamp(x/1000).date()))
    df_mcaps.set_index("Date", inplace=True)

    df_vol = pd.DataFrame(p['total_volumes'], columns=['Date', 'Volume'])
    df_vol['Date'] = pd.to_datetime(df_vol['Date'].apply(lambda x: dt.datetime.utcfromtimestamp(x/1000).date()))
    df_vol.set_index("Date", inplace=True)

    df_prices = pd.DataFrame(p['prices'], columns=['Date', 'Open'])
    df_prices['Date'] = pd.to_datetime(df_prices['Date'].apply(lambda x: dt.datetime.utcfromtimestamp(x/1000).date()))
    df_prices.set_index("Date", inplace=True)

    df = pd.concat([df_mcaps, df_vol, df_prices], axis=1)
    df['Close'] = df['Open'].shift(-1)

    df.columns = pd.MultiIndex.from_product([[coin_id], df.columns])

    return df
```


```python
def get_historical_data(coin_id, currency_id, start_date_s, end_date_s):
    df = None
    try:
        df = get_historical_data_gecko(coin_id, currency_id, start_date_s, end_date_s)
    except:
        try:
            df = get_historical_data_scraper(coin_id, currency_id, start_date_s, end_date_s)
        except:
            return None
    return df
```

Let's test the functions.


```python
df = get_historical_data('bitcoin', 'usd', '2013-01-01', '2021-04-01')
df.head()
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
      <td>0.0</td>
      <td>135.30</td>
      <td>141.96</td>
    </tr>
    <tr>
      <th>2013-04-29</th>
      <td>1.575032e+09</td>
      <td>0.0</td>
      <td>141.96</td>
      <td>135.30</td>
    </tr>
    <tr>
      <th>2013-04-30</th>
      <td>1.501657e+09</td>
      <td>0.0</td>
      <td>135.30</td>
      <td>117.00</td>
    </tr>
    <tr>
      <th>2013-05-01</th>
      <td>1.298952e+09</td>
      <td>0.0</td>
      <td>117.00</td>
      <td>103.43</td>
    </tr>
    <tr>
      <th>2013-05-02</th>
      <td>1.148668e+09</td>
      <td>0.0</td>
      <td>103.43</td>
      <td>91.01</td>
    </tr>
  </tbody>
</table>
</div>




```python
coins_list = pd.DataFrame(gecko.get_coins_list())
```

# Download it all

Finally, let's download the daily data for all tokens, across all time.


```python
df_complete = None
coins_list_ids = coins_list['id']
coins_list_ids_len = len(coins_list_ids)

i = 2206
timeout = 1
while i < coins_list_ids_len: 
    coin = coins_list_ids[i]
    
    df = get_historical_data(coin, 'usd', '2013-01-01', '2021-04-01')
    if df is None:
        print ("Failed to download", coin)
        sleep(timeout)
        continue
    
    i = i + 1
    if df_complete is None:
        df_complete = df
    else:
        df_complete = df_complete.join(df, how='outer')
    
    clear_output(wait=True)
    print ("Downloaded", coin, "{}/{}".format(i, coins_list_ids_len - 1))
```

    Downloaded etf-dao 2205/6547



    ---------------------------------------------------------------------------

    KeyboardInterrupt                         Traceback (most recent call last)

    <ipython-input-15-a1cc93c97937> in <module>
         18         df_complete = df
         19     else:
    ---> 20         df_complete = df_complete.join(df, how='outer')
         21 
         22     clear_output(wait=True)


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/frame.py in join(self, other, on, how, lsuffix, rsuffix, sort)
       8109         """
       8110         return self._join_compat(
    -> 8111             other, on=on, how=how, lsuffix=lsuffix, rsuffix=rsuffix, sort=sort
       8112         )
       8113 


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/frame.py in _join_compat(self, other, on, how, lsuffix, rsuffix, sort)
       8141                 right_index=True,
       8142                 suffixes=(lsuffix, rsuffix),
    -> 8143                 sort=sort,
       8144             )
       8145         else:


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/reshape/merge.py in merge(left, right, how, on, left_on, right_on, left_index, right_index, sort, suffixes, copy, indicator, validate)
         87         validate=validate,
         88     )
    ---> 89     return op.get_result()
         90 
         91 


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/reshape/merge.py in get_result(self)
        695             axes=[llabels.append(rlabels), join_index],
        696             concat_axis=0,
    --> 697             copy=self.copy,
        698         )
        699 


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/internals/concat.py in concatenate_block_managers(mgrs_indexers, axes, concat_axis, copy)
         51     """
         52     concat_plans = [
    ---> 53         _get_mgr_concatenation_plan(mgr, indexers) for mgr, indexers in mgrs_indexers
         54     ]
         55     concat_plan = _combine_concat_plans(concat_plans, concat_axis)


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/internals/concat.py in <listcomp>(.0)
         51     """
         52     concat_plans = [
    ---> 53         _get_mgr_concatenation_plan(mgr, indexers) for mgr, indexers in mgrs_indexers
         54     ]
         55     concat_plan = _combine_concat_plans(concat_plans, concat_axis)


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/internals/concat.py in _get_mgr_concatenation_plan(mgr, indexers)
        122 
        123         ax0_indexer = None
    --> 124         blknos = mgr.blknos
        125         blklocs = mgr.blklocs
        126 


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/internals/managers.py in blknos(self)
        167         if self._blknos is None:
        168             # Note: these can be altered by other BlockManager methods.
    --> 169             self._rebuild_blknos_and_blklocs()
        170 
        171         return self._blknos


    ~/docs/projects/by-year-month/2021/03/bplfamily/notebooks/.venv/lib/python3.7/site-packages/pandas/core/internals/managers.py in _rebuild_blknos_and_blklocs(self)
        242             rl = blk.mgr_locs
        243             new_blknos[rl.indexer] = blkno
    --> 244             new_blklocs[rl.indexer] = np.arange(len(rl))
        245 
        246         if (new_blknos == -1).any():


    KeyboardInterrupt: 



```python
# df_complete.to_parquet("CoinGecko_2013-04-28_2021-03-28_2205.parquet")
```


```python
df = get_historical_data('0-5x-long-shitcoin-index-token', 'usd', '2013-01-01', '2021-04-01')
```


```python
df_complete.head()
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
      <th colspan="2" halign="left">0-5x-long-altcoin-index-token</th>
      <th>...</th>
      <th colspan="2" halign="left">1x-short-theta-network-token</th>
      <th colspan="4" halign="left">1x-short-tomochain-token</th>
      <th colspan="4" halign="left">1x-short-trx-token</th>
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
      <th>Market Cap</th>
      <th>Volume</th>
      <th>...</th>
      <th>Open</th>
      <th>Close</th>
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
      <th>2017-06-14</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2017-06-15</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2017-06-16</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2017-06-17</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2017-06-18</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 280 columns</p>
</div>




```python
df_complete_first
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
      <th colspan="2" halign="left">0-5x-long-altcoin-index-token</th>
      <th>...</th>
      <th colspan="2" halign="left">engine</th>
      <th colspan="4" halign="left">enigma</th>
      <th colspan="4" halign="left">enjincoin</th>
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
      <th>Market Cap</th>
      <th>Volume</th>
      <th>...</th>
      <th>Open</th>
      <th>Close</th>
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
      <th>2013-04-28</th>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>135.30</td>
      <td>141.96</td>
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
      <td>1.575032e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>141.96</td>
      <td>135.30</td>
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
      <td>1.501657e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>135.30</td>
      <td>117.00</td>
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
      <td>1.298952e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>117.00</td>
      <td>103.43</td>
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
      <td>1.148668e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>103.43</td>
      <td>91.01</td>
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
      <th>2021-03-24</th>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>...</td>
      <td>54585.00</td>
      <td>52527.00</td>
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
      <td>9.827974e+11</td>
      <td>7.302905e+10</td>
      <td>...</td>
      <td>52527.00</td>
      <td>51417.00</td>
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
      <td>9.591553e+11</td>
      <td>6.340164e+10</td>
      <td>...</td>
      <td>51417.00</td>
      <td>55033.00</td>
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
      <td>1.027210e+12</td>
      <td>5.544256e+10</td>
      <td>...</td>
      <td>55033.00</td>
      <td>55832.00</td>
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
      <td>1.042184e+12</td>
      <td>4.728575e+10</td>
      <td>...</td>
      <td>55832.00</td>
      <td>NaN</td>
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
<p>2890 rows × 8604 columns</p>
</div>




```python
df_complete_second
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
      <th colspan="4" halign="left">enkronos</th>
      <th colspan="4" halign="left">enq-enecuum</th>
      <th colspan="2" halign="left">en-tan-mo</th>
      <th>...</th>
      <th colspan="2" halign="left">shitcoin-token</th>
      <th colspan="4" halign="left">shivers</th>
      <th colspan="4" halign="left">shopping-io</th>
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
      <th>Market Cap</th>
      <th>Volume</th>
      <th>...</th>
      <th>Open</th>
      <th>Close</th>
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
      <th>2013-04-28</th>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>135.30</td>
      <td>141.96</td>
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
      <td>1.575032e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>141.96</td>
      <td>135.30</td>
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
      <td>1.501657e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>135.30</td>
      <td>117.00</td>
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
      <td>1.298952e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>117.00</td>
      <td>103.43</td>
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
      <td>1.148668e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>103.43</td>
      <td>91.01</td>
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
      <th>2021-03-24</th>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>...</td>
      <td>54585.00</td>
      <td>52527.00</td>
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
      <td>9.827974e+11</td>
      <td>7.302905e+10</td>
      <td>...</td>
      <td>52527.00</td>
      <td>51417.00</td>
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
      <td>9.591553e+11</td>
      <td>6.340164e+10</td>
      <td>...</td>
      <td>51417.00</td>
      <td>55033.00</td>
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
      <td>1.027210e+12</td>
      <td>5.544256e+10</td>
      <td>...</td>
      <td>55033.00</td>
      <td>55832.00</td>
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
      <td>1.042184e+12</td>
      <td>4.728575e+10</td>
      <td>...</td>
      <td>55832.00</td>
      <td>NaN</td>
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
<p>2890 rows × 11464 columns</p>
</div>




```python
df_complete_third
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
      <th colspan="4" halign="left">shorty</th>
      <th colspan="4" halign="left">showhand</th>
      <th colspan="2" halign="left">shrimp-capital</th>
      <th>...</th>
      <th colspan="2" halign="left">zyx</th>
      <th colspan="4" halign="left">zzz-finance</th>
      <th colspan="4" halign="left">zzz-finance-v2</th>
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
      <th>Market Cap</th>
      <th>Volume</th>
      <th>...</th>
      <th>Open</th>
      <th>Close</th>
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
      <th>2013-04-28</th>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>135.30</td>
      <td>141.96</td>
      <td>1.500518e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>135.30</td>
      <td>141.96</td>
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
      <td>1.575032e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>141.96</td>
      <td>135.30</td>
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
      <td>1.501657e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>135.30</td>
      <td>117.00</td>
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
      <td>1.298952e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>117.00</td>
      <td>103.43</td>
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
      <td>1.148668e+09</td>
      <td>0.000000e+00</td>
      <td>...</td>
      <td>103.43</td>
      <td>91.01</td>
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
      <th>2021-03-24</th>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>54585.00</td>
      <td>52527.00</td>
      <td>1.017637e+12</td>
      <td>6.038338e+10</td>
      <td>...</td>
      <td>54585.00</td>
      <td>52527.00</td>
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
      <td>9.827974e+11</td>
      <td>7.302905e+10</td>
      <td>...</td>
      <td>52527.00</td>
      <td>51417.00</td>
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
      <td>9.591553e+11</td>
      <td>6.340164e+10</td>
      <td>...</td>
      <td>51417.00</td>
      <td>55033.00</td>
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
      <td>1.027210e+12</td>
      <td>5.544256e+10</td>
      <td>...</td>
      <td>55033.00</td>
      <td>55832.00</td>
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
      <td>1.042184e+12</td>
      <td>4.728575e+10</td>
      <td>...</td>
      <td>55832.00</td>
      <td>NaN</td>
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
<p>2890 rows × 6112 columns</p>
</div>




```python
df_complete.info()
```

    <class 'pandas.core.frame.DataFrame'>
    DatetimeIndex: 2890 entries, 2013-04-28 to 2021-03-28
    Columns: 11464 entries, ('enkronos', 'Market Cap') to ('shopping-io', 'Close')
    dtypes: float64(11464)
    memory usage: 252.9 MB



```python
coins_list.iloc[109]
```




    id        3x-long-midcap-index-token
    symbol                       midbull
    name      3X Long Midcap Index Token
    Name: 109, dtype: object




```python
df_complete = df_complete_first.join(df_complete_second.join(df_complete_third, how='outer'), how='outer')
```


```python
df_complete.to_parquet("CoinGecko_2013-04-28_2021-03-28.parquet")
```


```python
df_spendcoin = get_historical_data('spendcoin', 'usd', '2010-01-01', '2021-03-28')
```


```python
df_spendcoin['spendcoin']['Close'].plot()
```


    <IPython.core.display.Javascript object>



<img src="data:image/png;base64,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" width="936">





    <AxesSubplot:xlabel='Date'>




```python
coin_id = 'bitcoin'
currency_id = 'usd'
start_date_s = '2013-01-01'
end_date_s = '2021-04-01'

p = gecko.get_coin_market_chart_range_by_id(coin_id, currency_id, dt.datetime.fromisoformat(start_date_s).strftime("%s"), dt.datetime.fromisoformat(end_date_s).strftime("%s"))

df_mcaps = pd.DataFrame(p['market_caps'], columns=['Date', 'Market Cap'])
df_mcaps['Date'] = pd.to_datetime(df_mcaps['Date'].apply(lambda x: dt.datetime.utcfromtimestamp(x/1000).date()))
df_mcaps.set_index("Date", inplace=True)

df_vol = pd.DataFrame(p['total_volumes'], columns=['Date', 'Volume'])
df_vol['Date'] = pd.to_datetime(df_vol['Date'].apply(lambda x: dt.datetime.utcfromtimestamp(x/1000).date()))
df_vol.set_index("Date", inplace=True)

df_prices = pd.DataFrame(p['prices'], columns=['Date', 'Open'])
df_prices['Date'] = pd.to_datetime(df_prices['Date'].apply(lambda x: dt.datetime.utcfromtimestamp(x/1000).date()))
df_prices.set_index("Date", inplace=True)

df = pd.concat([df_mcaps, df_vol, df_prices], axis=1)
df['Close'] = df['Open'].shift(-1)

df.columns = pd.MultiIndex.from_product([[coin_id], df.columns])
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
      <td>0.0</td>
      <td>135.30</td>
      <td>141.96</td>
    </tr>
    <tr>
      <th>2013-04-29</th>
      <td>1.575032e+09</td>
      <td>0.0</td>
      <td>141.96</td>
      <td>135.30</td>
    </tr>
    <tr>
      <th>2013-04-30</th>
      <td>1.501657e+09</td>
      <td>0.0</td>
      <td>135.30</td>
      <td>117.00</td>
    </tr>
    <tr>
      <th>2013-05-01</th>
      <td>1.298952e+09</td>
      <td>0.0</td>
      <td>117.00</td>
      <td>103.43</td>
    </tr>
    <tr>
      <th>2013-05-02</th>
      <td>1.148668e+09</td>
      <td>0.0</td>
      <td>103.43</td>
      <td>91.01</td>
    </tr>
    <tr>
      <th>2013-05-03</th>
      <td>1.011066e+09</td>
      <td>0.0</td>
      <td>91.01</td>
      <td>111.25</td>
    </tr>
    <tr>
      <th>2013-05-04</th>
      <td>1.236352e+09</td>
      <td>0.0</td>
      <td>111.25</td>
      <td>116.79</td>
    </tr>
    <tr>
      <th>2013-05-05</th>
      <td>1.298378e+09</td>
      <td>0.0</td>
      <td>116.79</td>
      <td>118.33</td>
    </tr>
    <tr>
      <th>2013-05-06</th>
      <td>1.315992e+09</td>
      <td>0.0</td>
      <td>118.33</td>
      <td>106.40</td>
    </tr>
    <tr>
      <th>2013-05-07</th>
      <td>1.183766e+09</td>
      <td>0.0</td>
      <td>106.40</td>
      <td>112.64</td>
    </tr>
  </tbody>
</table>
</div>




```python
df = get_historical_data('bitcoin', 'usd', '2013-01-01', '2021-04-01')
df.head()
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
      <td>0.0</td>
      <td>135.30</td>
      <td>141.96</td>
    </tr>
    <tr>
      <th>2013-04-29</th>
      <td>1.575032e+09</td>
      <td>0.0</td>
      <td>141.96</td>
      <td>135.30</td>
    </tr>
    <tr>
      <th>2013-04-30</th>
      <td>1.501657e+09</td>
      <td>0.0</td>
      <td>135.30</td>
      <td>117.00</td>
    </tr>
    <tr>
      <th>2013-05-01</th>
      <td>1.298952e+09</td>
      <td>0.0</td>
      <td>117.00</td>
      <td>103.43</td>
    </tr>
    <tr>
      <th>2013-05-02</th>
      <td>1.148668e+09</td>
      <td>0.0</td>
      <td>103.43</td>
      <td>91.01</td>
    </tr>
  </tbody>
</table>
</div>



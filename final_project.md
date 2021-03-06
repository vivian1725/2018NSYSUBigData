
# 主題： 比幣特價格預測

![Alt text](bitcoin.jpg)
--------

# 動機：

### 這幾年加密貨幣興起，幣價大起大落，上沖下洗，這樣波動幅度巨大的交易標的一項吸引許多投(ㄉㄨˇ)資(ㄊㄨˊ)人的關注。
### 如果能利用機器學習的方式去預估給一個上漲或是下跌的波段，將帶來巨大的財富。

### 所以這次期末報告就決定嘗試利用不同的方法來預估比特幣的價格，再來看是否能有效地預測。

-------

# 計畫摘要：

-------

#### 1.利用網路爬蟲技術取得比特幣歷史價格
#### 2.使用深度學習預測
#### 3.建構LINE BOT 機器人取得預測結果


# 實作部份：
-------
#### 1. 建構LINEBOT 機器人放至Heroku 可查詢當日比特幣價格
#### 2. 使用程式捉取Quandl各交易所的比特幣平均金額
#### 3. 再捉取 Poloniex 上不同的虛擬幣值一起放入
#### 4. 使用時間序列預測的LSTM神經網路預測價格


===================

## 建一個LINEBOT 機器人可以查詢目前比特幣價格

#### app.py / Procfile / requirements.txt
--------

###  申請 LINE Messaging API：

#### 利用原來LINEID 申請一個 Messaging API 先
#### 需要 ISSU Channel secret／Channel access token (long-lived)  之後程式會用到需要先記下來

![Alt text](LINE1.png)

###  申請 Heroku 將程式佈署上去：

#### 先申請帳號
#### 並建立一個 APP，自己命名

![Alt text](LINE2.png)

#### 把已命名的APP回填至 申請的Messaging API webHook裡

![Alt text](LINE3.png)
-------

### 程式說明：
-------
####  requirements.txt ： 需要安裝的元件
####  app.py ： 1.裡頭需對應 Messaging API給的 Channel secret／Channel access token (long-lived)
![Alt text](LINE4.png)
####                   2.捉取比特幣今日價格，若使用者有詢問時可回覆其價格
####                   3.若使用者有詢問時，可回覆imgur上產生好的比特幣價格trend chart

                 
#### 佈署至Heroku 方法 ：
####                                    1.需先下載Heroku CLI 
####                                    2.確認Local程式位置後
####                                    3.cmd 下指令 （git add. git push heroku master)                     

![Alt text](LINE5.png)

===============

## 使用套件

  * json
  * requests
  * pandas 
  * matplotlib
  * numpy 
  * pickle
  * quandl
  * keras 
  



# 使用的套件
#### 這次主要是使用Keras作為LSTM模型建立的套件
#### 並輔以numpy與pandas做資料處理
#### 再利用matplotlib做繪圖
#### 資料來源的部分有使用到資料來源專用的接口套件 quandl


```python
import json
import requests
import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
import os
import pickle 
import quandl
import codecs
from datetime import datetime
from keras.models import load_model
from keras.models import Sequential
from keras.layers.recurrent import LSTM
from keras.layers import Dropout
from keras.layers import Dense
from keras.layers import Activation
from scipy import stats
import statsmodels.api as sm
import warnings
from itertools import product
from datetime import datetime
warnings.filterwarnings('ignore')
plt.style.use('seaborn-poster')
```

    Using TensorFlow backend.


# 定義函數

### 下載來自Quandl的 Bitcoin 資料集
### Quandl本身提供了Python的資料擷取套件 ，所以只需要簡單的設定就可以輕鬆地獲得資料
### 中間利用pickle作資料的備份
### api_key可以不用填沒問題，但會有連接上線 ，所以還是建議到Quandl申請免費的帳號獲取API KEY


```python
def get_quandl_data(quandl_id):
    '''Download and cache Quandl dataseries''' 
    cache_path = '{}.pkl'.format(quandl_id).replace('/','-') 
    
    #Enter quandl API Key
    quandl.ApiConfig.api_key = "zQm4uFHeJru86SyaLs6v"
    
    print('Downloading {} from Quandl'.format(quandl_id)) 
    df = quandl.get(quandl_id, returns="pandas") 
    df.to_pickle(cache_path) 
    print('Cached {} at {}'.format(quandl_id, cache_path)) 
    return df
```

## 從不同的DataFrame萃取出特定的欄位合併成新的DataFrame


```python
def merge_dfs_on_column(dataframes, labels, col):
    '''Merge a single column of each dataframe into a new combined dataframe''' 
    series_dict = {} 
    for index in range(len(dataframes)): 
        series_dict[labels[index]] = dataframes[index][col] 
    return pd.DataFrame(series_dict)
```

## 從Poloniex抓取更多其他虛擬貨幣的價格


### 讀取JSON檔
#### 藉由Poloniex提供的Web service來抓去幣價的JSON檔 ，所以先建立一個函式用抓取JSON


```python
def get_json_data(json_url, cache_path):
    '''Download and cache JSON data, return as a dataframe.''' 
    print('Downloading {}'.format(json_url)) 
    #df = pd.read_json(codecs.open(json_url,'r','utf-8'))
    json=requests.get(json_url, verify=True).text
    df = pd.read_json(json) 
    df.to_pickle(cache_path) 
    print('Cached {} at {}'.format(json_url, cache_path)) 
    return df
```

### Poloniex抓取資料
#### 實際利用上面的函式來抓去JSON檔再利用pandas儲存成DataFrame的格式方便後續的操作


```python
def get_crypto_data(poloniex_pair): 
    base_polo_url = 'https://poloniex.com/public?command=returnChartData&currencyPair={}&start={}&end={}&period={}' 
    start_date = datetime.strptime('2016-01-01', '%Y-%m-%d') # get data from the start of 2016
    end_date = datetime.now() # up until today
    pediod = 86400 # pull daily data (86,400 seconds per day) 
    '''Retrieve cryptocurrency data from poloniex''' 
    json_url = base_polo_url.format(poloniex_pair, start_date.timestamp(), end_date.timestamp(), pediod) 
    data_df = get_json_data(json_url, poloniex_pair) 
    data_df = data_df.set_index('date')
    return data_df 
```

## 資料分集
#### 把資料其切分成訓練集與測試集 ，這邊的設定是抓90%數據訓練，10%數據測試


```python
def train_test_split(df, test_size=0.1):
    split_row = len(df) - int(test_size * len(df))
    train_data = df.iloc[:split_row]
    test_data = df.iloc[split_row:]
    return train_data, test_data
```

## 繪圖


```python
def line_plot_s(line1, label1, title):
    fig, ax = plt.subplots(1, figsize=(16, 9))
    ax.plot(line1, label=label1, linewidth=2)
    ax.set_ylabel('price [USD]', fontsize=14)
    ax.set_title(title, fontsize=18)
    ax.legend(loc='best', fontsize=18)
```


```python
def line_plot(line1, line2, label1=None, label2=None, title=''):
    fig, ax = plt.subplots(1, figsize=(16, 9))
    ax.plot(line1, label=label1, linewidth=2)
    ax.plot(line2, label=label2, linewidth=2)
    ax.set_ylabel('price [USD]', fontsize=14)
    ax.set_title(title, fontsize=18)
    ax.legend(loc='best', fontsize=18)
```

## 對資料集做normailise的函數
### 因為後續不是只有BTC的價錢，還有加入其他的虛擬貨幣一起 
### 所以為了避免受不同幣種幣價原本的高低影響，所以做normailise
#### 那這邊的做法是讓價錢變成跟第一天的價錢的漲跌幅
#### 也就是說假設第一天價錢是8000，第二天價錢變成10000，第三天價錢又變回8000
#### 則在normailise之後就會變成0、0.25、0 ，那14天的資料也都是以照這樣的邏輯進行


```python
def normalise_zero_base(df):
    """ Normalise dataframe column-wise to reflect changes with
        respect to first entry.
    """
    return df / df.iloc[0] - 1
```

## 建立函數快速LSTM建模所需的資料模式
### 預設是14天週期，並使用normailise
#### 假設今天是7/15，透過這函數就會從資料集中收集7/1-7/14共14天的資料並做normailise
#### 在合併到訓練用的模型中


```python
def extract_window_data(df, window=14, zero_base=True):
    """ Convert dataframe to overlapping sequences/windows of
        length `window`.
    """
    window_data = []
    for idx in range(len(df) - window):
        tmp = df[idx: (idx + window)].copy()
        if zero_base:
            tmp = normalise_zero_base(tmp)
        window_data.append(tmp.values)
    return np.array(window_data)
```

## 綜合上述的函式
#### 做到一鍵分割、合併、normailise訓練與測試用的目標與資料集 


```python
def prepare_data(df, window=14, zero_base=True, test_size=0.1):
    """ Prepare data for LSTM. """
    # train test split
    train_data, test_data = train_test_split(df, test_size)
    
    # extract window data
    X_train = extract_window_data(train_data, window, zero_base)
    X_test = extract_window_data(test_data, window, zero_base)
    
    # extract targets
    y_train = train_data.average[window:].values
    y_test = test_data.average[window:].values
    if zero_base:
        y_train = y_train / train_data.average[:-window].values - 1
        y_test = y_test / test_data.average[:-window].values - 1
    return train_data, test_data, X_train, X_test, y_train, y_test
```

## LSTM建模函數
#### 利用Keras內的模型建立我們的LSTM模型


```python
def build_lstm_model(input_data, output_size, neurons=20,
                     activ_func='linear', dropout=0.25,
                     loss='mae', optimizer='adam'):
    model = Sequential()
    model.add(LSTM(neurons, input_shape=(
              input_data.shape[1], input_data.shape[2])))
    model.add(Dropout(dropout))
    model.add(Dense(units=output_size))
    model.add(Activation(activ_func))
    model.compile(loss=loss, optimizer=optimizer)
    return model
```

===========================================================

# 抓取各交易所比特幣交易資料
### 這邊利用Quandl提供的資料集來做為資料集
#### Quandl上有持續在更新且以美金報價的交易所為下列4所
#### 因為虛擬貨幣的交易在各國的規範日益嚴苛 , 所以能使用法幣出入金的交易所不多


```python
# Pull pricing data form 4 BTC exchanges 
exchanges = ['COINBASE','BITSTAMP','ITBIT','KRAKEN'] 
exchange_data = {} 
for exchange in exchanges: 
    exchange_code = 'BCHARTS/{}USD'.format(exchange) 
    btc_exchange_df = get_quandl_data(exchange_code) 
    exchange_data[exchange] = btc_exchange_df 
```

    Downloading BCHARTS/COINBASEUSD from Quandl
    Cached BCHARTS/COINBASEUSD at BCHARTS-COINBASEUSD.pkl
    Downloading BCHARTS/BITSTAMPUSD from Quandl
    Cached BCHARTS/BITSTAMPUSD at BCHARTS-BITSTAMPUSD.pkl
    Downloading BCHARTS/ITBITUSD from Quandl
    Cached BCHARTS/ITBITUSD at BCHARTS-ITBITUSD.pkl
    Downloading BCHARTS/KRAKENUSD from Quandl
    Cached BCHARTS/KRAKENUSD at BCHARTS-KRAKENUSD.pkl


### 僅捉取2016年之後的加權價格 , 並增加一欄平均值作為訓練的目標價格
### 只使用2016年之後的價格主要是為了配合下面引進其他虛擬貨幣的部分
### 捉取的資料期有部分缺失或為0的資料 , 缺失或為0的資料則利用當天其他交易所的平均價格替補


```python
# merge the  BTC price dataseries' into a single dataframe
btc_usd_datasets = merge_dfs_on_column(list(exchange_data.values()), list(exchange_data.keys()), 'Weighted Price')
# extract data after 2016
btc_usd_datasets = btc_usd_datasets.loc[btc_usd_datasets.index >= '2016-01-01']
# Remove "0" values 
btc_usd_datasets.replace(0, np.nan, inplace=True)
# replace nan with row mean
fill_value = pd.DataFrame({col: btc_usd_datasets.mean(axis=1) for col in btc_usd_datasets.columns})
btc_usd_datasets = btc_usd_datasets.fillna(value=fill_value)
#
btc_usd_datasets['average'] = btc_usd_datasets.mean(axis=1)
exchange_Volume = ['COINBASE-V', 'BITSTAMP-V', 'ITBIT-V', 'KRAKEN-V']
```


```python
btc_usd_datasets_Volume = merge_dfs_on_column(list(exchange_data.values()), exchange_Volume , 'Volume (BTC)')
btc_usd_datasets_Volume = btc_usd_datasets_Volume.loc[btc_usd_datasets_Volume.index >= '2016-01-01']
btc_usd_datasets_Volume['sum_V'] = btc_usd_datasets_Volume.sum(axis=1)
btc_usd_datasets_Volume =pd.DataFrame(btc_usd_datasets_Volume['sum_V'])
btc_usd_datasets=pd.merge(btc_usd_datasets,btc_usd_datasets_Volume , left_index=True, right_index=True)
btc_usd_datasets_Volume=normalise_zero_base(btc_usd_datasets)
```


```python
btc_usd_datasets
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>BITSTAMP</th>
      <th>COINBASE</th>
      <th>ITBIT</th>
      <th>KRAKEN</th>
      <th>average</th>
      <th>sum_V</th>
    </tr>
    <tr>
      <th>Date</th>
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
      <th>2016-01-01</th>
      <td>433.086003</td>
      <td>433.358101</td>
      <td>431.854189</td>
      <td>433.197419</td>
      <td>432.873928</td>
      <td>8749.673805</td>
    </tr>
    <tr>
      <th>2016-01-02</th>
      <td>433.292697</td>
      <td>435.286346</td>
      <td>433.211856</td>
      <td>432.989873</td>
      <td>433.695193</td>
      <td>6591.310053</td>
    </tr>
    <tr>
      <th>2016-01-03</th>
      <td>428.595713</td>
      <td>430.697845</td>
      <td>429.316383</td>
      <td>427.434460</td>
      <td>429.011100</td>
      <td>9678.402186</td>
    </tr>
    <tr>
      <th>2016-01-04</th>
      <td>432.834487</td>
      <td>433.938214</td>
      <td>432.178487</td>
      <td>430.631979</td>
      <td>432.395792</td>
      <td>13463.971066</td>
    </tr>
    <tr>
      <th>2016-01-05</th>
      <td>432.053592</td>
      <td>433.300199</td>
      <td>432.729037</td>
      <td>430.513601</td>
      <td>432.149107</td>
      <td>10381.184703</td>
    </tr>
    <tr>
      <th>2018-06-20</th>
      <td>6679.719281</td>
      <td>6692.957851</td>
      <td>6681.077994</td>
      <td>6679.288215</td>
      <td>6683.260835</td>
      <td>17914.311412</td>
    </tr>
    <tr>
      <th>2018-06-21</th>
      <td>6731.905067</td>
      <td>6731.627631</td>
      <td>6730.904546</td>
      <td>6729.491059</td>
      <td>6730.982076</td>
      <td>13265.411911</td>
    </tr>
    <tr>
      <th>2018-06-22</th>
      <td>6283.202163</td>
      <td>6241.882496</td>
      <td>6287.060492</td>
      <td>6253.716124</td>
      <td>6266.465319</td>
      <td>52492.267032</td>
    </tr>
    <tr>
      <th>2018-06-23</th>
      <td>6137.488639</td>
      <td>6133.479208</td>
      <td>6121.887751</td>
      <td>6133.505460</td>
      <td>6131.590265</td>
      <td>17195.259088</td>
    </tr>
    <tr>
      <th>2018-06-24</th>
      <td>5987.551809</td>
      <td>6020.179715</td>
      <td>6008.756939</td>
      <td>5990.530329</td>
      <td>6001.754698</td>
      <td>39287.958124</td>
    </tr>
    <tr>
      <th>2018-06-25</th>
      <td>6204.806754</td>
      <td>6212.208982</td>
      <td>6197.104192</td>
      <td>6209.353804</td>
      <td>6205.868433</td>
      <td>28681.231501</td>
    </tr>
    <tr>
      <th>2018-06-26</th>
      <td>6186.030207</td>
      <td>6175.888527</td>
      <td>6184.709258</td>
      <td>6172.424748</td>
      <td>6179.763185</td>
      <td>22865.746868</td>
    </tr>
    <tr>
      <th>2018-06-27</th>
      <td>6088.770534</td>
      <td>6090.949848</td>
      <td>6087.267937</td>
      <td>6091.902635</td>
      <td>6089.722739</td>
      <td>21026.721350</td>
    </tr>
  </tbody>
</table>
<p>909 rows × 6 columns</p>
</div>




```python
line_plot(btc_usd_datasets_Volume.average,btc_usd_datasets_Volume.sum_V, 'avage', 'Volume')
```


![png](output_32_0.png)


 ## 並從 poloniex 下載其它虛擬貨幣資料拉入當欄位
 ### 這邊選擇除了BTC以外幾個比較知名的虛擬貨幣
 ### 包含了乙太坊、萊特幣、瑞波幣、門羅幣等
 ### 這4種分別都有各自知名的成因與背後支持的技術
 #### 但因為前面有提到法幣出入金的限制
 #### 所以這邊報價選擇USDT，USDT是一款"號稱"與美金1:1掛鉤的虛擬貨幣
 #### 雖然實際上USDT的價值可能不一定是1美金，但相差不大，姑且作為美金報價


```python
# 從Poloniex下載交易資料 我們將下載4個虛擬貨幣： Ethereum，Litecoin，Ripple，Monero的交易資料
altcoins = ['ETH','LTC','XRP','XMR']
altcoin_data = {}
for altcoin in altcoins:
    coinpair = 'USDT_{}'.format(altcoin)
    crypto_price_df = get_crypto_data(coinpair)
    altcoin_data[altcoin] = crypto_price_df
```

    Downloading https://poloniex.com/public?command=returnChartData&currencyPair=USDT_ETH&start=1451577600.0&end=1530179875.045544&period=86400
    Cached https://poloniex.com/public?command=returnChartData&currencyPair=USDT_ETH&start=1451577600.0&end=1530179875.045544&period=86400 at USDT_ETH
    Downloading https://poloniex.com/public?command=returnChartData&currencyPair=USDT_LTC&start=1451577600.0&end=1530179876.887559&period=86400
    Cached https://poloniex.com/public?command=returnChartData&currencyPair=USDT_LTC&start=1451577600.0&end=1530179876.887559&period=86400 at USDT_LTC
    Downloading https://poloniex.com/public?command=returnChartData&currencyPair=USDT_XRP&start=1451577600.0&end=1530179877.934&period=86400
    Cached https://poloniex.com/public?command=returnChartData&currencyPair=USDT_XRP&start=1451577600.0&end=1530179877.934&period=86400 at USDT_XRP
    Downloading https://poloniex.com/public?command=returnChartData&currencyPair=USDT_XMR&start=1451577600.0&end=1530179879.295549&period=86400
    Cached https://poloniex.com/public?command=returnChartData&currencyPair=USDT_XMR&start=1451577600.0&end=1530179879.295549&period=86400 at USDT_XMR



```python
# merge price dataseries' into a single dataframe
altcoin_usd_datasets = merge_dfs_on_column(list(altcoin_data.values()), list(altcoin_data.keys()), 'weightedAverage')
# Remove "0" values 
altcoin_usd_datasets.replace(0, np.nan, inplace=True)
```


```python
# PLOTS
fig = plt.figure(figsize=[15, 7])
plt.suptitle('Cryptocurrency,  USD', fontsize=22)

plt.subplot(221)
plt.plot(altcoin_usd_datasets.ETH, '-', label='ETH')
plt.legend()

plt.subplot(222)
plt.plot(altcoin_usd_datasets.LTC, '-', label='LTC')
plt.legend()

plt.subplot(223)
plt.plot(altcoin_usd_datasets.XMR, '-', label='XMR')
plt.legend()

plt.subplot(224)
plt.plot(altcoin_usd_datasets.XRP, '-', label='XRP')
plt.legend()

# plt.tight_layout()
plt.show()
```


![png](output_36_0.png)


## 資料整合：加入其它貨幣合併
#### 合併上面抓取到的比特幣報價與其他虛擬貨幣的報價，整合為單一的DataFrame做為資料集


```python
hist = pd.merge(btc_usd_datasets,altcoin_usd_datasets, left_index=True, right_index=True)
```

## 將14天的價格變化資料分為訓練及測試集
#### 利用前面建立的函數快速輕鬆地建立好訓練與測試用的資料集


```python
train, test, X_train, X_test, y_train, y_test = prepare_data(hist)
```

## 訓練LSTM 模型


```python
model = load_model('10萬epochs.octet-stream')
#model = build_lstm_model(X_train, output_size=1)
##history = model.fit(X_train, y_train, epochs=10, batch_size=4)
```

## 還原結果
#### 因為資料集都是經過normailise的數值
#### 所以這邊需要再把normailise的數值還原成實際的價格


```python
target_col='average'
window=14
targets = test[target_col][window:]
preds = model.predict(X_test).squeeze()
preds = test.average.values[:-window] * (preds + 1)
preds = pd.Series(index=targets.index, data=preds)
```

## 繪製30天比較圖


```python
n = 30
line_plot(targets[-n:], preds[-n:], 'actual', 'prediction')
```


![png](output_46_0.png)

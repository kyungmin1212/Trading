## Table of Contents

- [데이터 수집](#1)
- [Timestamp를 시간으로 변경하기(밀리세컨드)](#2)
- [TA Library(Technical Analysis)](#3)
- [Binance API (ccxt)](#4)

---
## #1

### 데이터 수집
- 크롤링 이용
    - 크롤링을 위하여 크롬 개발자 툴 사용    
        ![](./img/img1.jpg)    
        - 주소 우클릭후 open in new tap 클릭하여 json 파일 확인해보기
    - XRPUSDT, 1분 데이터 1000개 가져오기
        ```python
        import time
        import datetime
        import pandas as pd
        import requests

        url = 'https://www.binance.com/fapi/v1/continuousKlines?limit=1000&pair=XRPUSDT&contractType=PERPETUAL&interval=1m'
        webpage = requests.get(url)
        webpage.content
        '''
        b'[[1673910120000,"0.3869","0.3873","0.3869","0.3869","828579.2",1673910179999,"320707.42747",370,"422302.6","163468.90759","0"],[1673910180000,"0.3870","0.3872","0.3851","0.3859","8189318.3",1673910239999,"3160034.27121",2537,"2918960.9","1126481.04774","0"], ....]]
        '''
        ```
    - Byte 데이터를 DataFrame으로 읽어오기
        ```python
        import time
        import datetime
        import pandas as pd
        import requests
        from io import BytesIO

        url = 'https://www.binance.com/fapi/v1/continuousKlines?limit=1000&pair=XRPUSDT&contractType=PERPETUAL&interval=1m'
        df = pd.read_json(BytesIO(webpage.content))
        df
        ```    
        ![](./img/img2.jpg)    

    - startTime을 설정해주면 설정한 기간부터 limit만큼 데이터를 가져올 수 있음
        ```python
        import time
        import datetime
        import pandas as pd
        import requests
        from io import BytesIO

        url = 'https://www.binance.com/fapi/v1/continuousKlines?limit=1000&pair=XRPUSDT&contractType=PERPETUAL&interval=1m&startTime={1673970180000}'
        webpage = requests.get(url)
        df = pd.read_json(BytesIO(webpage.content))
        df
        ```    
        ![](./img/img3.jpg)    

- ccxt를 이용한 데이터 수집
    - ccxt 는 timestamp를 인자로 받아서 데이터를 출력
    - btc_ohlcv는 timestamp를 반환하고 여기에 pd.to_datetime을 적용하면 UTC가 결과로 나옴. UCT기준 9시간 뒤인 한국시간으로 맞춰주면 됨
        ```python
        import datetime
        import time
        import ccxt 
        import pandas as pd 

        def date_to_timestamp(str_date):

            dt = datetime.datetime.strptime(str_date,'%Y-%m-%d %H:%M:%S')
            ts = time.mktime(dt.timetuple())

            return int(ts*1000)

        binance = ccxt.binance()
        btc_ohlcv = binance.fetch_ohlcv("BTC/USDT",since=date_to_timestamp('2023-01-17 00:00:00'))

        df = pd.DataFrame(btc_ohlcv, columns=['datetime', 'open', 'high', 'low', 'close', 'volume'])
        df['datetime'] = pd.to_datetime(df['datetime'], unit='ms') + datetime.timedelta(hours = 9)
        df.set_index('datetime', inplace=True)
        print(df)
        '''
                         open      high       low     close     volume
        datetime                                                              
        2023-01-17 00:00:00  20860.68  20873.64  20856.99  20866.07  145.69792
        2023-01-17 00:01:00  20866.66  20883.00  20847.56  20847.88  298.07373
        2023-01-17 00:02:00  20847.88  20850.74  20829.25  20842.63  265.42819
        2023-01-17 00:03:00  20842.63  20856.18  20832.27  20835.33  232.53609
        2023-01-17 00:04:00  20833.84  20849.79  20830.00  20842.51  162.59315
        ...                       ...       ...       ...       ...        ...
        2023-01-17 08:15:00  21139.02  21161.88  21138.04  21157.04   90.56335
        2023-01-17 08:16:00  21155.37  21162.18  21155.37  21160.12   39.09142
        2023-01-17 08:17:00  21160.11  21168.70  21154.58  21161.77  112.85471
        2023-01-17 08:18:00  21162.99  21186.84  21161.77  21176.64  109.72950
        2023-01-17 08:19:00  21176.64  21192.59  21174.97  21189.19   94.53721

        [500 rows x 5 columns]
        '''
        ```    
        - fetch_ohlcv 의 timeframe의 default 값은 1m, limit의 default 값은 500 (변경을 원할경우 `btc_ohlcv = binance.fetch_ohlcv(symbol="BTC/USDT", timeframe='1d', limit=10` 와 같이 입력))
        - 현재 시간보다 미래를 입력하게 되면 btc_ohlcv는 빈 데이터 프레임 반환(아래 코드 동작 시간 2023-01-18 22:10:00. 현재시간보다 먼 미래를 조회했기 때문에 아무것도 나오지 않음)
            ```python
            import ccxt 
            import pandas as pd 


            binance = ccxt.binance()
            btc_ohlcv = binance.fetch_ohlcv("BTC/USDT",since=date_to_timestamp('2023-01-18 23:00:00'))

            df = pd.DataFrame(btc_ohlcv, columns=['datetime', 'open', 'high', 'low', 'close', 'volume'])
            df['datetime'] = pd.to_datetime(df['datetime'], unit='ms') + datetime.timedelta(hours = 9)
            # to_datetime은 Timestamp가 들어가면 UTC 기준으로 반환해줌 -> timedelta를 이용하여 UTC 기준 9시간 뒤인 한국시간으로 맞춰주기
            df.set_index('datetime', inplace=True)
            print(df)
            '''
            Empty DataFrame
            Columns: [open, high, low, close, volume]
            Index: []
            '''
            ```
- ccxt를 이용해 원하는 기간 데이터 수집하기(data_collect.ipynb)
    ```python
    import datetime
    import time
    import ccxt 
    import pandas as pd 

    def date_to_timestamp(date,utc=False):
        """
        str형태의 date를 timestamp로 만들어주기
        :params (str or datetime) date : '%Y-%m-%d %H:%M:%S'형태의 데이터. ex)'2023-01-18 23:00:00'
        :params bool utc : True로 설정할시에 date를 utc 시간이라고 생각
        :return timestamp시간(단위 ms) ex)1674050400000
        :rtype int
        
        ex) date_to_timestamp('2023-01-18 23:00:00') -> 1674050400000
        """
        if type(date) == str: # str인 경우 datetime으로 변환해주기
            dt = datetime.datetime.strptime(date,'%Y-%m-%d %H:%M:%S')
        else: # datetime으로 들어온경우
            dt = date
            
        # time.mktime은 local 타임 기준으로 timestamp를 변경함. 따라서 utc일 경우 +9시간해서 한국시간으로 설정해줘야함
        if utc:
            dt = date + datetime.timedelta(hours = 9)
        
        ts = time.mktime(dt.timetuple()) 
        
        return int(ts*1000)

    def make_csv_data(coin_name,period,start_time,end_time):
        """
        coin이름, 수집하고 싶은 봉의 기준 기간, 시작, 끝 시간을 지정해주면 그 기간까지의 데이터를 수집하여 csv 파일로 반환
        :params str coin_name : 코인이름 ex)"BTC/USDT"
        :params str period : 수집기준기간 ex) "1m"
        :params str start_time : 수집 시작 시간 ex) '2022-01-01 00:00:00' (한국시간기준)
        :params str end_time : 수집 끝 시간 ex) '2023-01-01 00:00:00' (한국시간기준)
        :return None
        :rtype None
        
        f'{coin_name}_{period}_{start_time}_{end_time}.csv' 파일로 저장됨
        """
        
        # 입력 end_time은 한국시간기준이므로 utc기준으로 변경해주기. pd.to_datetime 이 UTC 시간으로 반환하기때문에 UTC 시간 기준으로 체크를 해줘야함 (UTC에서 KST로 변경은 제일 마지막에 함)
        utc_end_time = datetime.datetime.strptime(end_time,'%Y-%m-%d %H:%M:%S') - datetime.timedelta(hours = 9)
        
        binance = ccxt.binance()
        btc_ohlcv = binance.fetch_ohlcv(coin_name,period,since=date_to_timestamp(start_time))

        df = pd.DataFrame(btc_ohlcv, columns=['datetime', 'open', 'high', 'low', 'close', 'volume'])
        df['datetime'] = pd.to_datetime(df['datetime'], unit='ms')
        df.set_index('datetime', inplace=True)

        while len(df)==0:  # 정기 점검이 있는 시간대에는 조회를 해도 결과가 나오지 않음. 500개씩 조회되므로 데이터가 조회될때까지 500개씩 건너뛰기
            if 'd' in period:
                start_time=datetime.datetime.strptime(start_time,'%Y-%m-%d %H:%M:%S') + datetime.timedelta(days=int(f"{period[:-1]}*500")) 
            elif 'm' in period:
                start_time=datetime.datetime.strptime(start_time,'%Y-%m-%d %H:%M:%S') + datetime.timedelta(minutes=int(f"{period[:-1]}*500")) 
            else:
                assert False, '일봉과 분봉만 조회 가능합니다.'

            btc_ohlcv = binance.fetch_ohlcv(coin_name,period,since=date_to_timestamp(start_time))

            df=pd.DataFrame(btc_ohlcv,columns=['datetime','open','high','low','close','volume'])
            df['datetime']=pd.to_datetime(df['datetime'],unit='ms')
            df.set_index('datetime',inplace=True)
        
        total_df = df
        
        check_count = 0
        
        while True:
            
            check_count+=1
            if check_count%10 == 0:
                print(total_df.index[-1])
            
            if utc_end_time <= df.index[-1]:
                break
            
            if 'd' in period:
                time_later=df.index[-1] + datetime.timedelta(days=int(f"{period[:-1]}")) 
            elif 'm' in period:
                time_later=df.index[-1] + datetime.timedelta(minutes=int(f"{period[:-1]}")) 
            else:
                assert False, '일봉과 분봉만 조회 가능합니다.'
            
            # pd.to_datetime는 utc기준으로 date를 반환함
            btc_ohlcv = binance.fetch_ohlcv(coin_name,period,since=date_to_timestamp(time_later,utc=True))

            df=pd.DataFrame(btc_ohlcv,columns=['datetime','open','high','low','close','volume'])
            df['datetime']=pd.to_datetime(df['datetime'],unit='ms')
            df.set_index('datetime',inplace=True)
            
            while len(df)==0:  # 정기 점검이 있는 시간대에는 조회를 해도 결과가 나오지 않음. 500개씩 조회되므로 데이터가 조회될때까지 500개씩 건너뛰기
                if 'd' in period:
                    time_later=time_later + datetime.timedelta(days=int(f"{period[:-1]}*500")) 
                elif 'm' in period:
                    time_later=time_later + datetime.timedelta(minutes=int(f"{period[:-1]}*500")) 
                else:
                    assert False, '일봉과 분봉만 조회 가능합니다.'
                
                btc_ohlcv = binance.fetch_ohlcv(coin_name,period,since=date_to_timestamp(time_later,utc=True))

                df=pd.DataFrame(btc_ohlcv,columns=['datetime','open','high','low','close','volume'])
                df['datetime']=pd.to_datetime(df['datetime'],unit='ms')
                df.set_index('datetime',inplace=True)
                
            total_df = pd.concat([total_df,df])
            
        total_df.index = total_df.index + datetime.timedelta(hours = 9)
        total_df = total_df[:end_time]
        
        coin_name = "".join(coin_name.split("/"))
        s_time = "-".join("-".join(str(start_time).split(" ")).split(":"))
        e_time = "_".join("-".join(str(end_time).split(" ")).split(":"))
        total_df.to_csv(f'./{coin_name}_{period}_{s_time}_{e_time}.csv')
    ```
    ```python
    make_csv_data(coin_name= 'BTC/USDT', period= '1m', start_time= '2022-12-01 00:00:00', end_time= '2023-01-01 00:00:00')
    ```

#### References
- https://www.inflearn.com/course/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8-%EC%84%A0%EB%AC%BC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%ED%8A%B8%EB%A0%88%EC%9D%B4%EB%94%A9/dashboard
- https://wikidocs.net/120392
---

## #2

### Timestamp를 시간으로 변경하기(밀리세컨드)

- timestamp -> date
    ```python
    import datetime
    import pytz

    def timestamp_to_date(mili_time):
        utc_dt = datetime.datetime.utcfromtimestamp(mili_time / 1000.0)
        utc_dt = pytz.utc.localize(utc_dt) # utc_df가 utc시 시간이라는것을 

        korea = pytz.timezone('Asia/Seoul')
        korea_dt = korea.normalize(utc_dt.astimezone(korea))

        timeline = str(korea_dt.strftime('%Y-%m-%d %H:%M:%S'))  #(1) 출력형식 지정
        return timeline

    timestamp_to_date(1673979762000)
    '''
    '2023-01-18 03:22:42'
    '''
    ```
    - 참고 : localize 안한경우(utc시간을 가져왔지만 utc라고 설정하지 않았기때문에 local 피시 시간에 맞춰 utc 시간이 아닌 한국시간으로 생각함. 따라서 한국시간.astimezone(한국존) 이므로 어떠한 시간 변화없이 그 시간 자체가 출력됨 -> 오류)
        ```python
        import datetime
        import pytz

        def timestamp_to_date(mili_time):
            utc_dt = datetime.datetime.utcfromtimestamp(mili_time / 1000.0)
        #     utc_dt = pytz.utc.localize(utc_dt) # utc_df가 utc시 시간이라는것을 

            korea = pytz.timezone('Asia/Seoul')
            korea_dt = korea.normalize(utc_dt.astimezone(korea))

            timeline = str(korea_dt.strftime('%Y-%m-%d %H:%M:%S'))  #(1) 출력형식 지정
            return timeline

        timestamp_to_date(1673979762000)
        '''
        '2023-01-17 18:22:42'
        '''
        ```
    - pandas를 이용할 경우 pd.to_datetime을 통해 timestamp를 utc 시간 기준 datetime으로 바로 변경 가능
        ```python
        df['datetime']=pd.to_datetime(df['datetime'],unit='ms')
        ```
- date -> timestamp 
    ```python
    import datetime
    import time

    def date_to_timestamp(date,utc=False):
        """
        str형태의 date를 timestamp로 만들어주기
        :params (str or datetime) date : '%Y-%m-%d %H:%M:%S'형태의 데이터. ex)'2023-01-18 23:00:00'
        :params bool utc : True로 설정할시에 date를 utc 시간이라고 생각
        :return timestamp시간(단위 ms) ex)1674050400000
        :rtype int
        
        ex) date_to_timestamp('2023-01-18 23:00:00') -> 1674050400000
        """
        if type(date) == str: # str인 경우 datetime으로 변환해주기
            dt = datetime.datetime.strptime(date,'%Y-%m-%d %H:%M:%S')
        else: # datetime으로 들어온경우
            dt = date
            
        # time.mktime은 local 타임 기준으로 timestamp를 변경함. 따라서 utc일 경우 +9시간해서 한국시간으로 설정해줘야함
        if utc:
            dt = date + datetime.timedelta(hours = 9)
        
        ts = time.mktime(dt.timetuple()) 
        
        return int(ts*1000)

    date_to_timestamp('2023-01-18 03:22:42')
    '''
    1673979762000.0
    '''
    ```
    - time 모듈은 로컬 시간으로 date를 자동으로 인식하기 때문에 timestamp로 변경할때 알아서 utc값으로 변경해서 반환해줌
    - print(time.tzname) 를 통해 운영체제나 언어의 시간대 지역을 파악가능
        ```python
        print(time.tzname)
        '''
        ('대한민국 표준시', '대한민국 일광 절약 시간')
        '''
        ```

#### References
- https://github.com/kyungmin1212/Quiz_Study/blob/main/study/1-python.md#22

---

## #3

### TA Library(Technical Analysis)
- https://github.com/bukosabino/ta
- https://technical-analysis-library-in-python.readthedocs.io/en/latest/
- 설치 : `pip install --upgrade ta`
- 단순 이동 평균(Simple Moving Average)
    ```python
    from ta.trend import SMAIndicator

    df['sma7'] = SMAIndicator(df['close'],window=7).sma_indicator()
    df['sma25'] = SMAIndicator(df['close'],window=25).sma_indicator()
    df['sma99'] = SMAIndicator(df['close'],window=99).sma_indicator()
    df.head(10)
    ```
- 가중 이동 평균(Weighted Moving Average)
    ```python
    from ta.trend import WMAIndicator

    df['wma7'] = WMAIndicator(df['close'], window=7).wma()
    df['wma25'] = WMAIndicator(df['close'], window=25).wma()
    df['wma99'] = WMAIndicator(df['close'], window=99).wma()
    df.head(10)
    ```
- 지수 이동 평균(Exponential Moving Average)
    ```python
    from ta.trend import EMAIndicator

    df['ema7'] = EMAIndicator(df['close'], window=7).ema_indicator()
    df['ema25'] = EMAIndicator(df['close'], window=25).ema_indicator()
    df['ema99'] = EMAIndicator(df['close'], window=99).ema_indicator()
    df.head(10)
    ```
- MACD
    - MACD Line : Fast 지수이동평균 - Slow 지수이동평균 (ex. Fast : 12, Slow : 26)
    - Signal Line : MACD Line Sign 지수이동평균 (ex. Sign : 9)
    - Diff Line : MACD Line - Signal Line
    ```python
    from ta.trend import MACD

    macd = MACD(df['close'], window_slow=26, window_fast=12, window_sign=9)
    df['macd'] = macd.macd()
    df['macd_s'] = macd.macd_signal()
    df['macd_d'] = macd.macd_diff()
    df.head(10)
    ```
- RSI
    ```python
    from ta.trend import MACD

    macd = MACD(df['close'], window_slow=26, window_fast=12, window_sign=9)
    df['macd'] = macd.macd()
    df['macd_s'] = macd.macd_signal()
    df['macd_d'] = macd.macd_diff()
    df.head(10)
    ```
- StochRSI 
    - StochRSI : (현시점 RSI - 최저점 RSI) / (최고점 RSI - 최저점 RSI)  (일반적으로 14기간 사용)
    - Smooth K : StochRSI의 이동 평균 기간 (% K 라인)
    - Smooth D : Smooth K의 이동 평균 기간 (% D 라인)
    ```python
    from ta.momentum import StochRSIIndicator

    stochRSI = StochRSIIndicator(df['close'], window=14, smooth1=3, smooth2=3)
    df['srsi'] = stochRSI.stochrsi()
    df['srsik'] = stochRSI.stochrsi_k()
    df['srsid'] = stochRSI.stochrsi_d()
    df.tail(10)
    ```
- Bollinger Bands
    - 상단밴드 : M일 단순 이동 평균(SMA) + (M일 표준편차*N)
    - 중간선 : M일 단순 이동 평균(SMA)
    - 하단밴드 : M일 단순 이동 평균(SMA) - (M일 표준편차*N)
    - M-> window , N-> window_dev
    ```python
    from ta.volatility import BollingerBands

    bb = BollingerBands(df['close'], window=20, window_dev=2)
    df['bh'] = bb.bollinger_hband() #high band
    df['bhi'] = bb.bollinger_hband_indicator() #high band 보다 가격이 높으면 1, 아니면 0
    df['bl'] = bb.bollinger_lband() #low band
    df['bli'] = bb.bollinger_lband_indicator() #low band 보다 가격이 낮으면 1, 아니면 0
    df['bm'] = bb.bollinger_mavg() #middle band
    df['bw'] = bb.bollinger_wband() #band width

    df.tail(10)
    ```
- VWAP
    - VWAP = 시그마(대표가격*거래량) / 시그마(거래량) (시그마 기간이 window 값)
    - 대표가격 : (고가+저가+종가) / 3
    ```python
    from ta.volume import VolumeWeightedAveragePrice

    vwap = VolumeWeightedAveragePrice(high=df['high'], low=df['low'], close=df['close'], volume=df['volume'], window=14)
    df['vwap'] = vwap.volume_weighted_average_price()
    df.tail(10)
    ```
#### References
- https://www.inflearn.com/course/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8-%EC%84%A0%EB%AC%BC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%ED%8A%B8%EB%A0%88%EC%9D%B4%EB%94%A9/dashboard

---

## #4

### Binance API (ccxt)
- 선물 API 신청한 후 Enable Futures 체크해주기
- ccxt 이용 바이낸스 단순 거래
    - api key 값 가져오기
        ```python
        with open("./binance.key") as f:
            lines = f.readlines()
            api_key = lines[0].strip()
            api_secret = lines[1].strip()
        ```
    - 거래소 객체 생성(`ccxt.binance`)
        ```python
        import ccxt

        exchange = ccxt.binance(config={
            'apiKey': api_key,
            'secret': api_secret,
            'enableRateLimit': True, # 시장가 주문을 불가능하게 함
            'options': {
                'defaultType': 'future'       # 선물 거래
            }
        })
        print(exchange)
        '''
        Binance
        '''
        ```
    - usdt 시장에서 거래되고 있는 암호화폐들의 심볼 가져오기(`fetch_tickers()`)
        ```python
        import ccxt

        with open("./binance.key") as f:
            lines = f.readlines()
            api_key = lines[0].strip()
            api_secret = lines[1].strip()

        exchange = ccxt.binance(config={
            'apiKey': api_key,
            'secret': api_secret,
            'enableRateLimit': True,
            'options': {
                'defaultType': 'future'
            }
        })

        tickers = exchange.fetch_tickers()
        symbols = tickers.keys()
        usdt_symbols = [x for x in symbols if x.endswith('USDT')]
        print(usdt_symbols)
        print(len(usdt_symbols))
        '''
        ['BAKE/USDT:USDT', 'NKN/USDT:USDT', 'XEM/USDT:USDT', 'FOOTBALL/USDT:USDT', 'LRC/USDT:USDT', 'ZEC/USDT:USDT', 'LINA/USDT:USDT', ... ,'GMT/USDT:USDT', 'CELR/USDT:USDT', 'BAL/USDT:USDT', 'DENT/USDT:USDT', 'LPT/USDT:USDT', 'SNX/USDT:USDT']
        156
        '''
        ```
    - 잔고 조회(`exchange.fetch_balance()`)
        ```python
        import ccxt
        import pprint

        with open("./binance.key") as f:
            lines = f.readlines()
            api_key = lines[0].strip()
            api_secret = lines[1].strip()

        exchange = ccxt.binance(config={
            'apiKey': api_key,
            'secret': api_secret,
            'enableRateLimit': True,
            'options': {
                'defaultType': 'future'
            }
        })

        # balance
        balance = exchange.fetch_balance()
        usdt_balance = balance['USDT']
        pprint.pprint(usdt_balance) 
        '''
        {'free': 157.79005017, 'total': 157.79005017, 'used': 0.0}
        '''
        ```
    - 선물 주문
        - 롱 포지션 진입과 정리
            - 시장가로 롱 포지션 진입 : `create_market_buy_order`
            - 지정가로 롱 포지션 진입 : `create_limit_buy_order`
            - 시장가로 롱 포지션 정리 : `create_market_sell_order`
            - 지정가로 롱 포지션 정리 : `create_limit_sell_order`
        - 숏 포지션 진입과 정리
            - 시장가로 숏 포지션 진입 : `create_market_sell_order`
            - 지정가로 숏 포지션 진입 : `create_limit_sell_order`
            - 시장가로 롱 포지션 정리 : `create_market_buy_order`
            - 지정가로 롱 포지션 정리 : `create_limit_buy_order`
        - TP/SL (Take Profit, Stop Loss)
            - ex) 19600$에 시장가로 롱 포지션을 오픈한 후 익절은 19800$, 손절은 19400$에 하고 싶다면 TP/SL 체크한 후 TP와 SL에 각각 19800,19400 입력
            - 익절이나 손절이 되면 반대 주문은 자동으로 취소됨
        - 시장가 주문과 TP/SL
            ```python
            import ccxt
            import pprint

            with open("./binance.key") as f:
                lines = f.readlines()
                api_key = lines[0].strip()
                secret  = lines[1].strip()

            binance = ccxt.binance(config={
                'apiKey': api_key,
                'secret': secret,
                'enableRateLimit': True,
                'options': {
                    'defaultType': 'future'
                }
            })

            orders = [None] * 3

            # market price (ex: 19500$)
            orders[0] = binance.create_order(
                symbol="BTC/USDT",
                type="MARKET",
                side="buy",
                amount=0.001
            )

            # take profit
            orders[1] = binance.create_order(
                symbol="BTC/USDT",
                type="TAKE_PROFIT_MARKET",
                side="sell",
                amount=0.001,
                params={'stopPrice': 22950}
            )

            # stop loss
            orders[2] = binance.create_order(
                symbol="BTC/USDT",
                type="STOP_MARKET",
                side="sell",
                amount=0.001,
                params={'stopPrice': 22900}
            )

            for order in orders:
                pprint.pprint(order)
            ```
        - 지정가 주문과 TP/SL
            ```python
            import ccxt
            import pprint

            with open("./binance.key") as f:
                lines = f.readlines()
                api_key = lines[0].strip()
                secret  = lines[1].strip()

            binance = ccxt.binance(config={
                'apiKey': api_key,
                'secret': secret,
                'enableRateLimit': True,
                'options': {
                    'defaultType': 'future'
                }
            })

            orders = [None] * 3
            price = 22930

            # limit price
            orders[0] = binance.create_order(
                symbol="BTC/USDT",
                type="LIMIT",
                side="buy",
                amount=0.001,
                price=price
            )

            # take profit
            orders[1] = binance.create_order(
                symbol="BTC/USDT",
                type="TAKE_PROFIT",
                side="sell",
                amount=0.001,
                price=price,
                params={'stopPrice': 22950}
            )

            # stop loss
            orders[2] = binance.create_order(
                symbol="BTC/USDT",
                type="STOP",
                side="sell",
                amount=0.001,
                price=price,
                params={'stopPrice': 22900}
            )

            for order in orders:
                pprint.pprint(order)
            ```

#### References
- https://wikidocs.net/178885


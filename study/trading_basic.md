## Table of Contents

- [데이터 수집](#1)
- [Timestamp를 시간으로 변경하기(밀리세컨드)](#2)
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
- ccxt를 이용해 원하는 기간 데이터 수집하기
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
        
        # 입력 end_time은 한국시간기준이므로 utc기준으로 변경해주기 
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

###

#### References
- https://www.inflearn.com/course/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8-%EC%84%A0%EB%AC%BC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%ED%8A%B8%EB%A0%88%EC%9D%B4%EB%94%A9/dashboard
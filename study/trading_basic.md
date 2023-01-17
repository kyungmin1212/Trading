## Table of Contents

- [](#1)

---
## #1

### 분데이터 크롤링
- 크롤링 기본
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

    - startTime을 설정해주면 설정한 기간부터 데이터를 가져올 수 있음
        ```python

        ```

---

## #2

### Timestamp를 시간으로 변경하기(밀리세컨드)
- 참고 : https://github.com/kyungmin1212/Quiz_Study/blob/main/study/1-python.md#22

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
- date -> timestamp 
    ```python
    import datetime
    import time

    def date_to_timestamp(str_date):

        dt = datetime.datetime.strptime(str_date,'%Y-%m-%d %H:%M:%S')
        ts = time.mktime(dt.timetuple())
        
        return ts*1000

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
- https://www.inflearn.com/course/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8-%EC%84%A0%EB%AC%BC-%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98-%ED%8A%B8%EB%A0%88%EC%9D%B4%EB%94%A9/dashboard
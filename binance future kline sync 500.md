```python
import pandas as pd
 
from binance.um_futures import UMFutures
from db import sub_s, db

umf = UMFutures()

coins = sub_s.find()
i_list = ['15m', '1h', '4h', '1d']

for index, coin in coins.iterrows():
    s = coin['s']
    for i in i_list:
        data = umf.klines(symbol=s, interval=i)
        for d in data:
            t = int(d[0])
            o = d[1]
            h = d[2]
            l = d[3]
            c = d[4]
            v = d[5]
            q = d[7]
            sql = f"replace into future_kline (s,i,t,o,c,h,l,v,q) value ('{s}','{i}'," \
                  f"{t},{o},{c},{h},{l},{v},{q}) "
            db.execute(sql)

```


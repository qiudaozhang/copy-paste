# V1



```py
import pandas as pd
import talib

from daotest.BaseTest import BaseTest
from findex import nake

pd.set_option('display.max_columns', 5000000)
# 显示所有行
pd.set_option('display.max_rows', 5000000)
# 加长宽度
pd.set_option('display.width', 500000)

LONG = 'LONG'
SHORT = 'SHORT'


class Break_It(BaseTest):

    def __init__(self, data, balance, margin, leverage, commission, fix_win, fix_loss):
        super().__init__(data, balance, margin, leverage, commission)
        self.fix_win = fix_win
        self.fix_loss = fix_loss

    def close_try(self):
        if self.has_position:
            if self.max_loss() < self.fix_loss:
                self.close_profit(self.fix_loss)
                print("k inner loss")

    def next(self):
        super().next()
        if self.has_position:
            self.out()
        else:
            self.entry()

    def entry(self):
        df = self.df[:-1]
        last = df.iloc[-1]
        k = self.now_k
        if last['ma'] > last['ma2'] > last['ma3']:
            self.buy_long(k['o'])
            return
        if last['ma'] < last['ma2'] < last['ma3']:
            self.buy_short(k['o'])
            return

    def out(self):
        df = self.df[:-1]
        last = df.iloc[-1]
        k = self.now_k
        if self.is_long():
            if last['ma'] < last['ma2'] < last['ma3']:
                self.close(k['o'])
                return

        if self.is_short():
            if last['ma'] > last['ma2'] > last['ma3']:
                self.close(k['o'])
                return

from util import timeutil as tu
 
t1 = tu.now_ms()
df = pd.read_csv(path, low_memory=False, header=None, usecols=[0, 1, 2, 3, 4, 5], names=['t', 'o', 'h', 'l', 'c', 'v'])

df = df.sort_values('t', ascending=True)
df = df[1:]
dt = pd.to_numeric(df['t'])
df['t'] = dt
t2 = tu.now_ms()
h8 = 8 * 60 * 60 * 1000
df['t'] = df['t'] + h8
df['date'] = pd.to_datetime(df['t'], unit='ms')
df = nake.nake_convert(df)
df['ma'] = talib.SMA(df['c'], 20)
df['ma2'] = talib.SMA(df['c'], 60)
df['ma3'] = talib.SMA(df['c'], 120)
all = 3000
margin = 1000
fix_win = 100 / 100
fix_loss = -50 / 100

bt = Break_It(df[100:], all, margin, 3, 5 / 10000, fix_win, fix_loss)
bt.run()
print(
    f"now balance {bt.balance:.2f} total times {bt.entry_times} total fee {bt.total_commission:.2f} fix_win {fix_win}，"
    f" fix_loss {fix_loss} total balance {all}  margin {margin}  ")
print(f"win times  {bt.win_times} loss times {bt.loss_times} ")
t3 = tu.now_ms()
print((t3 - t2) / 1000)

# for o in bt.orders:
#     print(o)
# output = pd.DataFrame(bt.orders)
#
# output.to_csv('record.csv')
```





# v2

```python
class Break_It(BaseTest):

    def __init__(self, data, balance, margin, leverage, commission, fix_win, fix_loss):
        super().__init__(data, balance, margin, leverage, commission)
        self.fix_win = fix_win
        self.fix_loss = fix_loss

    def close_try(self):
        if self.has_position:
            if self.max_loss() < self.fix_loss:
                self.close_profit(self.fix_loss)
                print("k inner loss")

    def next(self):
        super().next()
        if self.has_position:
            self.out()
        else:
            self.entry()

    def entry(self):
        df = self.df[:-1]
        last = df.iloc[-1]
        k = self.now_k
        gp = ema.gold_pos_v2(df, 'ma', 'ma2')
        dp = ema.dead_pos_v2(df, 'ma', 'ma2')

        if 0 < gp < 3:
            self.buy_long(k['o'])
            return

        if 0 < dp < 3:
            self.buy_short(k['o'])
            return

    def out(self):
        df = self.df[:-1]
        last = df.iloc[-1]
        k = self.now_k
        gp = ema.gold_pos_v2(df, 'ma', 'ma2')
        dp = ema.dead_pos_v2(df, 'ma', 'ma2')
        if self.is_long():
            if 1 < dp:
                self.close(k['o'])
                return
        if self.is_short():
            if gp > 0:
                self.close(k['o'])
                return
```



条件

- 15m
- 60
- 120

 

now balance 2251.14 total times 123 total fee 369.00 fix_win 1.0， fix_loss -0.5 total balance 3000  margin 1000  
win times  46 loss times 77 



条件

- 15m
- 20
- 60



now balance 3134.05 total times 284 total fee 852.00 fix_win 1.0， fix_loss -0.5 total balance 3000  margin 1000  
win times  102 loss times 182 



# v3






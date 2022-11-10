# v1



```python
def find_deviation_top(df: DataFrame):
    # d1 = df[-300:]
    d1 = df.copy()
    index1 = d1['macd'].idxmax()
    p1 = df.iloc[index1]
    p1_max = p1['h']
    d2 = df[index1 + 1:]
    if len(d2) == 0:
        return None
    d2_max = d2['h'].max()
    index2 = d2['h'].idxmax()
    p2 = df.iloc[index2]
    if d2_max > p1_max:
        if p2['macd'] < p1['macd']:
            return (p1, p2)
        return None

    else:
        return None


def find_deviation_bottom(df: DataFrame):
    # d1 = df[-300:]
    d1 = df.copy()
    index1 = d1['macd'].idxmin()
    p1 = df.iloc[index1]
    p1_min = p1['l']
    d2 = df[index1 + 1:]
    if len(d2) == 0:
        return None
    d2_min = d2['l'].min()
    index2 = d2['l'].idxmin()
    p2 = df.iloc[index2]

    if d2_min < p1_min:
        if p2['macd'] > p1['macd']:
            return None
        return (p1, p2)

    else:
        return None
```



date    2022-10-26 20:00:00

date    2022-11-04 20:00:00

代码显示这两个点满足



# V2





```python
def find_deviation_top(df: DataFrame):
    d1 = df[-300:]
    # d1 = df
    index1 = d1['macd'].idxmax()
    p1 = df.iloc[index1]
    p1_max = p1['h']
    d2 = df[index1 + 5:]
    if len(d2) == 0:
        return None
    index2 = d2['macd'].idxmax()
    p2 = df.iloc[index2]
    if p2['h'] > p1_max:
        if p2['macd'] < p1['macd']:
            return (p1, p2)
        return None

    else:
        return None


def find_deviation_bottom(df: DataFrame):
    d1 = df[-300:]
    # d1 = df
    index1 = d1['macd'].idxmin()
    p1 = df.iloc[index1]
    d2 = df[index1 + 5:]
    if len(d2) == 0:
        return None
    index2 = d2['macd'].idxmin()()
    p2 = df.iloc[index2]
    if p2['l'] < p1['l']:
        if p2['macd'] > p1['macd']:
            return [p1, p2]
        return None

    else:
        return None

```



test eth 4h



date    2022-10-26 20:00:00

date    2022-11-05 08:00:00

代码显示这两个点满足





# v3

沿用的是2，但是取值上不固定死









# 15m test



```python
import pandas as pd

from daotest.BaseTest import BaseTest
from findex import nake, macd

pd.set_option('display.max_columns', 5000000)
# 显示所有行
pd.set_option('display.max_rows', 5000000)
# 加长宽度
pd.set_option('display.width', 500000)

LONG = 'LONG'
SHORT = 'SHORT'


class B2_1(BaseTest):

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
            if len(self.df) > 300:
                self.entry()

    def entry(self):
        k = self.now_k
        # print(self.df)
        top = macd.find_deviation_top(self.df[-300:])
        if top is not None:
            # print(top)
            if top[1]['t'] == k['t']:
                self.buy_short(k['c'],self.df[-20:]['h'].max())
                return
        bottom = macd.find_deviation_bottom(self.df)
        if bottom is not None:
            # print(bottom)
            if bottom[1]['t'] == k['t']:
                self.buy_long(k['c'],self.df[-20:]['l'].min())
                return

    def out(self):
        # super().out()
        if self.has_position:
            if self.max_loss() < self.fix_loss:
                self.close_profit(self.fix_loss, 'final loss')
                return
            if self.max_win() > self.fix_win:
                self.close_profit(self.fix_win, 'final win')
                return

 


from util import timeutil as tu
 

path = "E:\\coding\\coin_data\\15m\\all.csv"
 
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
df = macd.cal_macd(df)
all = 3000
margin = 1000
fix_win = 20 / 100
fix_loss = -40 / 100

 

bt = B2_1(df[100:], all, margin, 5, 5 / 10000, fix_win, fix_loss)
bt.run()
print(
    f"now balance {bt.balance:.2f} total times {bt.entry_times} total fee {bt.total_commission:.2f} fix_win {fix_win}，"
    f" fix_loss {fix_loss} total balance {all}  margin {margin}  ")
print(f"win times  {bt.win_times} loss times {bt.loss_times} ")
t3 = tu.now_ms()
print((t3 - t2) / 1000)
 
output = pd.DataFrame(bt.orders)

output.to_csv('record.csv')

```





now balance 4800.00 total times 120 total fee 600.00 fix_win 0.2， fix_loss -0.4 total balance 3000  margin 1000  
win times  84 loss times 36 

 

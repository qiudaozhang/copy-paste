# V1



```python
import talib
from loguru import logger

from db.fkline import find_csv
from findex import macd, nake, ema
from strategy.BaseStrategy import BaseStrategy


def o_key(tag):
    return f"engine_one_{tag}"


class EngineOne(BaseStrategy):

    def __init__(self, _id, uid, s, margin, sid, mode, ext):
        super().__init__(_id, uid, s, margin, sid, mode, ext)
        df = find_csv(s, '1m', limit=1000)
        df['ma'] = talib.EMA(df['c'], 73)
        df['ma2'] = talib.EMA(df['c'], 166)
        df = macd.cal_macd(df)
        df = nake.nake_convert(df)
        df['bias'] = df['c'] - df['ma']
        df['bias2'] = df['c'] - df['ma2']
        self.df = df
        self.k = df.iloc[-1]
        self.gp = ema.gold_pos_v2(df, 'ma', 'ma2')
        self.dp = ema.dead_pos_v2(df, 'ma', 'ma2')
       

    def handle_in(self):
        k = self.k
        gp = self.gp
        dp = self.dp
        p1 = self.df[-50:]
        p2 = self.df[-150:-50]

        p1_max = p1['h'].max()
        p2_max = p2['h'].max()
        p1_min = p1['l'].min()
        p2_min = p2['l'].min()
        gap = k['ma'] - k['ma2']
        logger.info(f"{k['c']} {k['bias']:.2f} {gap:.2f}  {gp} {dp}")
        use = 3050
        bias_v = 1.3
        min = 2
        if min < gp < use:
            if p1_min > p2_min:
                logger.info('low high')
                if k['bias'] > 0:
                    if k['bias'] < bias_v:
                        self.open_long()
                        return

        if min < dp < use:
            if p1_max < p2_max:
                if k['bias'] < 0:
                    if k['bias'] < -bias_v:
                        self.open_short()
                        return

    def handle_win_out(self):
        super().handle_win_out()
        k = self.k
        pm = self.get_price_move() 
        if abs(pm) > 36:
            self.close()
            return
        if abs(k['bias']) > 16:
            self.close()
            return
        gp = self.gp
        dp = self.dp
        if self.is_now_long():
            if dp > 1:
                self.close()
                return
        if self.is_now_short():
            if gp > 1:
                self.close()
                return

    def handle_loss_out(self):
        super().handle_loss_out()
        k = self.k
        pm = self.get_price_move() 
        if self.is_now_long():
            if self.dp > 2:
                self.close()
                return
        if self.is_now_short():
            if self.gp > 2:
                self.close()
                return
```





这里暂时没有使用百分比的价格， 所以一些数字参数需要经常调整，也可以考虑转换为百分比，只不过没那么直观。



# V2



```python
import talib
from loguru import logger

from db.fkline import find_csv
from findex import macd, nake, ema
from strategy.BaseStrategy import BaseStrategy


class EngineOne(BaseStrategy):

    def __init__(self, _id, uid, s, margin, sid, mode, ext):
        super().__init__(_id, uid, s, margin, sid, mode, ext)
        df = find_csv(s, '1m', limit=2000)
        df['ma'] = talib.EMA(df['c'], 73)
        df['ma2'] = talib.EMA(df['c'], 166)
        df = macd.cal_macd(df)
        df = nake.nake_convert(df)
        df['bias'] = df['c'] - df['ma']
        df['bias2'] = df['c'] - df['ma2']
        df['bias2_rate'] = df['bias2'] / df['ma2']
        self.df = df
        self.k = df.iloc[-1]
        self.gp = ema.gold_pos_v2(df, 'ma', 'ma2')
        self.dp = ema.dead_pos_v2(df, 'ma', 'ma2')
 

    def handle_in(self):
        k = self.k
        gp = self.gp
        dp = self.dp
        p1 = self.df[-50:]
        p2 = self.df[-150:-50]

        p1_max = p1['h'].max()
        p2_max = p2['h'].max()
        p1_min = p1['l'].min()
        p2_min = p2['l'].min()
        gap = k['ma'] - k['ma2'] 
        use = 500
        bias_v = 1.3
        min = 2
        if min < gp < use:
            if len(p1.query('bias2_rate < 0 ')) > 5:
                return

            if len(p1.query('bias2_rate < -0.01 ')) > 2:
                return
 
            if p1_min > p2_min:
                logger.info('low high')
                if k['bias'] > 0:
                    if k['bias'] < bias_v:
                        self.open_long()
                        return

        if min < dp < use:
            if len(p1.query('bias2 > 0 ')) > 5:
                return
            if len(p1.query('bias2_rate > 0.01 ')) > 2:
                return

            if p1_max < p2_max:
                if k['bias'] < 0:
                    if k['bias'] < -bias_v:
                        self.open_short()
                        return

    def handle_win_out(self):
        super().handle_win_out()
        k = self.k
        pm = self.get_price_move()
        style = 1
        if style == 1:
            if abs(pm) / k['c'] > 3.8 / 100:
                self.close()
                return
            if abs(k['bias']) / k['ma'] > 1.8 / 100:
                self.close()
                return
            gp = self.gp
            dp = self.dp
            if self.is_now_long():
                if dp > 1:
                    self.close()
                    return
            if self.is_now_short():
                if gp > 1:
                    self.close()
                    return
        elif style == 2:
            self.ma_reverse_close()

    def handle_loss_out(self):
        super().handle_loss_out()
        k = self.k
        if self.has_order:
            pm = self.get_price_move()
            self.ma_reverse_close()

    def ma_reverse_close(self):
        if self.has_order:
            if self.is_now_long():
                if self.dp > 2:
                    self.close()
                    return
            if self.is_now_short():
                if self.gp > 2:
                    self.close()
                    return
```


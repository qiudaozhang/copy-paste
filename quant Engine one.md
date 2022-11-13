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



# V3

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
        i = '1m'
        self.i = i
        df = find_csv(s, '1m', limit=1000)
        # df = find_csv(s, '3m', limit=2000)
        df['ma'] = talib.EMA(df['c'], 73)
        df['ma2'] = talib.EMA(df['c'], 133)
        df['ma3'] = talib.EMA(df['c'], 166)
        df = macd.cal_macd(df)
        df = nake.nake_convert(df)
        # df15m = find_csv(s, '15m')
        df['bias'] = df['c'] - df['ma']
        df['bias2'] = df['c'] - df['ma2']
        df['bias3'] = df['c'] - df['ma3']
        df['bias_rate'] = df['bias'] / df['ma']
        df['bias2_rate'] = df['bias2'] / df['ma2']
        df['bias3_rate'] = df['bias3'] / df['ma3']

        self.df = df
        self.k = df.iloc[-1]
        self.gp = ema.gold_pos_v2(df, 'ma', 'ma2')
        self.dp = ema.dead_pos_v2(df, 'ma', 'ma2')
        self.gp2 = ema.gold_pos_v2(df, 'ma', 'ma3')
        self.dp2 = ema.dead_pos_v2(df, 'ma', 'ma3')
        # self.df15m = df15m

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
        # logger.info(f"{k['c']} {k['bias']:.2f} {gap:.2f}  {gp} {dp}")
        use = 800
        s1 = 300
        s2 = 500
        bias_v = 1 / 1000
        min = 2
        if self.i == '1m':
            s1 = 200
            s2 = 300
            use = 500
        if self.i == '3m':
            s1 = 100
            s2 = 200
            use = 300

        if gp > min:
            if p1_min > p2_min:
                if gp < s1:
                    if gp > 20:
                        if len(p1[-20:].query('bias2_rate < 0 ')) > 5:
                            return
                        if len(p1[-20:].query('bias2_rate < -0.01 ')) > 2:
                            return
                    if k['bias_rate'] < 3 / 1000:
                        self.open_long()
                        return
                else:
                    if gp < s2:
                        if len(p1.query('bias2_rate < 0 ')) > 5:
                            return
                        if len(p1.query('bias2_rate < -0.01 ')) > 2:
                            return
                        if k['bias2_rate'] < bias_v:
                            self.open_long()
                            return
                    elif gp < use:
                        if self.gp2 > 0:
                            # logger.info(k['bias3'])
                            if len(p1.query('bias3_rate < -0.004 ')) > 4:
                                logger.info("ma3 support fail")
                                return
                            if k['bias3_rate'] < 1 / 1000:
                                self.open_long()
                                self.save_win_price(k['c'] + k['c'] * 3 / 100)
                                return

        if dp > min:
            if p1_max < p2_max:
                if dp < s1:
                    if dp > 20:
                        if len(p1[-20:].query('bias2_rate > 0 ')) > 5:
                            return
                        if len(p1[-20:].query('bias2_rate > 0.01 ')) > 2:
                            return
                    logger.info(k['bias_rate'])
                    if k['bias_rate'] > -1.2 / 1000:
                        self.open_short()
                        return
                else:
                    if dp < s2:
                        if len(p1.query('bias2_rate > 0 ')) > 5:
                            return
                        if len(p1.query('bias2_rate > 0.01 ')) > 2:
                            return
                        logger.info(k['bias2_rate'])
                        bias_v = 1.5 / 1000
                        if 0 > k['bias2_rate'] > -bias_v:
                            self.open_short()
                            return
                    elif dp < use:
                        if self.dp2 > 0:
                            if len(p1.query('bias3_rate > 0 ')) > 5:
                                return
                            if len(p1.query('bias3_rate > 0.005')) > 2:
                                return
                            if k['bias3_rate'] > -1 / 1000:
                                self.open_short()
                                self.save_win_price(k['c'] - k['c'] * 3 / 100)
                                return

    def handle_win_out(self):
        super().handle_win_out()
        if self.has_order:
            k = self.k
            c = k['c']
            pm = self.get_price_move()
            back = 2 / 100
            if self.have_key('loss_price'):
                v = float(self.get_key('loss_price'))
                if self.is_now_long():
                    lp = c - c * back
                    if lp > v:
                        self.save_loss_price(lp)
                if self.is_now_short():
                    lp = c + c * back
                    if lp < v:
                        self.save_loss_price(lp)
            else:
                if abs(pm) / k['c'] > 3 / 100:
                    if self.is_now_long():
                        self.save_loss_price(k['c'] - k['c'] * 2 / 100)
                    if self.is_now_short():
                        self.save_loss_price(k['c'] + k['c'] * 2 / 100)

            # logger.info(f"{k['c']}  {k['bias']}")
            # logger.info(f"price move : {pm:.2f}")
            style = 1
            if style == 1:
                if abs(pm) / k['c'] > 6.8 / 100:
                    self.close()
                    return
                if abs(k['bias2_rate']) > 6 / 100:
                    self.close()
                    return
                self.ma_reverse_close()
            elif style == 2:
                self.ma_reverse_close()

    def handle_loss_out(self):
        # logger.info(self.get_run_ms() / 1000 / 60)
        super().handle_loss_out()
        k = self.k
        if self.has_order:
            if self.get_pure_win() < -1.3 / 100:
                self.ma_reverse_close()
                if self.has_order:
                    wave = abs((k['c'] - k['o'])) / k['o']
                    logger.info(f"{wave:.4f}")
                    if wave > 2.5 / 100:
                        self.close()
                        self.close_strategy()

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



# V4

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

        data = find_csv(s, '15m')
        wave_4h = nake.max_wave(data[-24:])
        i = '15m'
        if wave_4h > 5 / 100:
            i = '1m'
        else:
            i = '3,'
        self.i = i
        logger.info(f"use {i}")
        df = find_csv(s, '1m', limit=1000)
        # df = find_csv(s, '3m', limit=2000)
        df['ma'] = talib.EMA(df['c'], 73)
        df['ma2'] = talib.EMA(df['c'], 133)
        df['ma3'] = talib.EMA(df['c'], 166)
        df = macd.cal_macd(df)
        df = nake.nake_convert(df)
        # df15m = find_csv(s, '15m')
        df['bias'] = df['c'] - df['ma']
        df['bias2'] = df['c'] - df['ma2']
        df['bias3'] = df['c'] - df['ma3']
        df['bias_rate'] = df['bias'] / df['ma']
        df['bias2_rate'] = df['bias2'] / df['ma2']
        df['bias3_rate'] = df['bias3'] / df['ma3']

        self.df = df
        self.k = df.iloc[-1]
        self.gp = ema.gold_pos_v2(df, 'ma', 'ma2')
        self.dp = ema.dead_pos_v2(df, 'ma', 'ma2')
        self.gp2 = ema.gold_pos_v2(df, 'ma', 'ma3')
        self.dp2 = ema.dead_pos_v2(df, 'ma', 'ma3')
        # self.df15m = df15m

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
        use = 800
        s1 = 300
        s2 = 500
        bias_v = 1 / 1000
        min = 2
        if self.i == '1m':
            s1 = 200
            s2 = 300
            use = 500
        if self.i == '3m':
            s1 = 100
            s2 = 200
            use = 300

        if gp > min:
            if p1_min > p2_min:
                if gp < s1:
                    if gp > 20:
                        if len(p1[-20:].query('bias2_rate < 0 ')) > 5:
                            return
                        if len(p1[-20:].query('bias2_rate < -0.01 ')) > 2:
                            return
                    else:
                        bias_v = 9 / 1000
                    if k['bias_rate'] < bias_v:
                        self.open_long()
                        return
                else:
                    if gp < s2:
                        if len(p1.query('bias2_rate < 0 ')) > 5:
                            return
                        if len(p1.query('bias2_rate < -0.01 ')) > 2:
                            return
                        if k['bias2_rate'] < bias_v:
                            self.open_long()
                            return
                    elif gp < use:
                        if self.gp2 > 0:
                            # logger.info(k['bias3'])
                            if len(p1.query('bias3_rate < -0.004 ')) > 4:
                                logger.info("ma3 support fail")
                                return
                            if k['bias3_rate'] < 1 / 1000:
                                self.open_long()
                                self.save_win_price(k['c'] + k['c'] * 3 / 100)
                                return

        if dp > min:
            if p1_max < p2_max:
                if dp < s1:
                    if dp > 20:
                        if len(p1[-20:].query('bias2_rate > 0 ')) > 5:
                            return
                        if len(p1[-20:].query('bias2_rate > 0.01 ')) > 2:
                            return
                    else:
                        bias_v = 9 / 1000
                    logger.info(k['bias_rate'])
                    if k['bias_rate'] > -bias_v:
                        self.open_short()
                        return
                else:
                    if dp < s2:
                        if len(p1.query('bias2_rate > 0 ')) > 5:
                            return
                        if len(p1.query('bias2_rate > 0.01 ')) > 2:
                            return
                        logger.info(k['bias2_rate'])
                        bias_v = 1.5 / 1000
                        if 0 > k['bias2_rate'] > -bias_v:
                            self.open_short()
                            return
                    elif dp < use:
                        if self.dp2 > 0:
                            if len(p1.query('bias3_rate > 0 ')) > 5:
                                return
                            if len(p1.query('bias3_rate > 0.005')) > 2:
                                return
                            if k['bias3_rate'] > -1 / 1000:
                                self.open_short()
                                self.save_win_price(k['c'] - k['c'] * 3 / 100)
                                return

    def handle_win_out(self):
        super().handle_win_out()
        bias_out = 5 / 100
        if self.i == '1m':
            bias_out = 2.5 / 100
        if self.has_order:
            k = self.k
            c = k['c']
            pm = self.get_price_move()
            back = 2 / 100
            if self.have_key('loss_price'):
                v = float(self.get_key('loss_price'))
                if self.is_now_long():
                    lp = c - c * back
                    if lp > v:
                        self.save_loss_price(lp)
                if self.is_now_short():
                    lp = c + c * back
                    if lp < v:
                        self.save_loss_price(lp)
            else:
                if abs(pm) / k['c'] > bias_out:
                    if self.is_now_long():
                        self.save_loss_price(k['c'] - k['c'] * 2 / 100)
                    if self.is_now_short():
                        self.save_loss_price(k['c'] + k['c'] * 2 / 100)
            style = 1
            if style == 1:
                if abs(pm) / k['c'] > 6.8 / 100:
                    self.close()
                    return
                if abs(k['bias2_rate']) > bias_out:
                    self.close()
                    return
                self.ma_reverse_close()
            elif style == 2:
                self.ma_reverse_close()

    def handle_loss_out(self):
        # logger.info(self.get_run_ms() / 1000 / 60)
        super().handle_loss_out()
        k = self.k
        if self.has_order:
            if self.get_pure_win() < -1 / 100:
                self.ma_reverse_close()
                if self.has_order:
                    wave = abs((k['c'] - k['o'])) / k['o']
                    logger.info(f"{wave:.4f}")
                    if wave > 2.5 / 100:
                        self.close()
                        self.close_strategy()

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





# V5

```python
import talib
from loguru import logger

from db.fkline import find_csv
from findex import ema, nake
from strategy.BaseStrategy import BaseStrategy


def o_key(tag):
    return f"engine_one_{tag}"


class EngineOne(BaseStrategy):

    def __init__(self, _id, uid, s, margin, sid, mode, ext):
        super().__init__(_id, uid, s, margin, sid, mode, ext)

        df1m = find_csv(s, '1m', limit=1000)
        df3m = find_csv(s, '3m', limit=1000)
        df1m['ma'] = talib.EMA(df1m['c'], 73)
        df1m['ma2'] = talib.EMA(df1m['c'], 133)
        df1m['bias'] = df1m['c'] - df1m['ma']
        df1m['bias2'] = df1m['c'] - df1m['ma2']
        df1m['bias_rate'] = df1m['bias'] / df1m['ma']
        df1m['bias2_rate'] = df1m['bias2'] / df1m['ma']
        df3m['ma'] = talib.EMA(df3m['c'], 73)
        df3m['ma2'] = talib.EMA(df3m['c'], 133)
        df3m['bias'] = df3m['c'] - df3m['ma']
        df3m['bias2'] = df3m['c'] - df3m['ma2']
        df3m['bias_rate'] = df3m['bias'] / df3m['ma']
        df3m['bias2_rate'] = df3m['bias2'] / df3m['ma']

        self.gp1 = ema.gold_pos_v2(df1m, 'ma', 'ma2')
        self.dp1 = ema.dead_pos_v2(df1m, 'ma', 'ma2')
        self.gp2 = ema.gold_pos_v2(df3m, 'ma', 'ma2')
        self.dp2 = ema.dead_pos_v2(df3m, 'ma', 'ma2')
        self.df1m = df1m
        self.df3m = df3m
        # self.df15m = df15m

    def handle_in(self):
        k1m = self.df1m.iloc[-1]
        k3m = self.df3m.iloc[-1]
        if self.gp1 > 0:
            if self.gp1 < 100:
                if 0 < k1m['bias_rate'] < 1.5 / 1000:
                    self.open_long()
                    return
            else:
                if self.gp2 > 0:
                    if 0 < k3m['bias_rate'] < 1.5 / 1000:
                        self.open_long()
                        return
                else:
                    pass
        if self.dp1 > 0:
            if self.dp1 < 100:
                if 0 > k1m['bias_rate'] > -1.5 / 1000:
                    self.open_short()
                    return
            else:
                if self.dp2 > 0:
                    if 0 > k3m['bias_rate'] > -1.5 / 1000:
                        self.open_short()
                        return
                else:
                    pass

    def handle_win_out(self):
        super().handle_win_out()
        k1m = self.df1m.iloc[-1]
        k3m = self.df3m.iloc[-1]
        # logger.info(nake.max_wave(self.df3m[-30:]))
        # if nake.max_wave(self.df3m[-30:]) < 1/100:
        #     logger.info('into swing stage')
        if not self.has_order:
            return
        if self.get_pure_win() < 1 / 100:
            return

        if self.get_pure_win() > 7.5 / 100:
            self.close()
            return
        pm = self.get_price_move()
        c = k1m['c']
        back = 2 / 100
        if self.have_key('loss_price'):
            v = float(self.get_key('loss_price'))
            if self.is_now_long():
                lp = c - c * back
                if lp > v:
                    self.save_loss_price(lp)
            if self.is_now_short():
                lp = c + c * back
                if lp < v:
                    self.save_loss_price(lp)
        else:
            if abs(pm) / c > 3 / 100:
                if self.is_now_long():
                    self.save_loss_price(c - c * 2 / 100)
                if self.is_now_short():
                    self.save_loss_price(c + c * 2 / 100)

        if self.is_now_long():
            if self.gp1 > 0:
                if self.gp2 > 0:
                    logger.info("two gp cross")
                if self.gp1 > 300:
                    if k1m['bias2_rate'] > 2.7 / 100:
                        self.close()
                        return
                else:
                    if k1m['bias2_rate'] > 3 / 100:
                        self.close()
                        return
            else:
                logger.info("gp reverse ...")
                # double dead
                if self.dp2 > 0:
                    self.close()
                    return
        if self.is_now_short():
            if self.dp1 > 0:
                if self.dp1 > 300:
                    if k1m['bias2_rate'] < -2.7 / 100:
                        self.close()
                        return
                else:
                    if k1m['bias2_rate'] < -3 / 100:
                        self.close()
                        return
            else:
                if self.gp2 > 0:
                    self.close()
                    return

    def handle_loss_out(self):
        # logger.info(self.get_run_ms() / 1000 / 60)
        super().handle_loss_out()
        k3m = self.df3m.iloc[-1]
        df3m = self.df3m
        # 2h vs 2h
        d31 = df3m[-35:]
        d32 = df3m[-80:-35]
        if d31['h'].max() > d32['h'].max():
            logger.info('high high')
        if d31['l'].min() > d32['l'].min():
            logger.info('low  high')
        if self.has_order:
            if self.get_pure_win() < -1 / 100:
                if self.is_now_long():
                    if self.dp1 > 0:
                        if self.dp2 > 0:
                            self.close()
                            return
                if self.is_now_short():
                    if self.gp1 > 0:
                        if self.gp2 > 0:
                            self.close()
                            return
                if self.get_pure_win() < -3 / 100:
                    self.close()
                    return 
```

# V7

```python
import talib
from loguru import logger

from db.fkline import find_csv
from findex import ema, nake
from strategy.BaseStrategy import BaseStrategy
from util import dingutil as du


def o_key(tag):
    return f"engine_one_{tag}"


class EngineOne(BaseStrategy):

    def close_v2(self):
        self.close()
        # self.cool_down(240)
        self.cool_down(240)

    def __init__(self, _id, uid, s, margin, sid, mode, ext):
        super().__init__(_id, uid, s, margin, sid, mode, ext)

        df3m = find_csv(self.s, '3m', limit=500)
        df3m['ma'] = talib.EMA(df3m['c'], 73)
        df3m['ma2'] = talib.EMA(df3m['c'], 133)
        df3m['bias'] = df3m['c'] - df3m['ma']
        df3m['bias2'] = df3m['c'] - df3m['ma2']
        df3m['bias_rate'] = df3m['bias'] / df3m['ma']
        df3m['bias2_rate'] = df3m['bias2'] / df3m['ma2']
        df3m = nake.nake_convert(df3m)
        self.gp2 = ema.gold_pos_v2(df3m, 'ma', 'ma2')
        self.dp2 = ema.dead_pos_v2(df3m, 'ma', 'ma2')
        self.df3m = df3m

        df1h = find_csv(self.s, '1h', limit=1000)
        df1h['ma'] = talib.EMA(df1h['c'], 73)
        df1h['ma2'] = talib.EMA(df1h['c'], 133)
        df1h['bias'] = df1h['c'] - df1h['ma']
        df1h['bias_rate'] = df1h['bias'] / df1h['ma']

        self.df1h = df1h

    def handle_in(self):
        df1h = self.df1h
        gp = ema.gold_pos_v2(df1h, 'ma', 'ma2')
        dp = ema.dead_pos_v2(df1h, 'ma', 'ma2')
        k1h = df1h.iloc[-1]
        c = k1h['c']

        df3m = self.df3m
        k3m = self.df3m.iloc[-1]
        pre = df3m.iloc[-2]

        if gp > 10:
            logger.info('逆转多头')
            du.send_tag_time_uid('逆转为多头', 'xxxxxxxxx', 3600 * 8, 7)
        else:
            df15m = find_csv(self.s, '15m')

            if abs(c - df1h[-50:-1]['l'].min()) / c < 1 / 100:
                self.open_long()
                self.save_loss_price(c - c * 3.5 / 100)
                self.save_win_price(c + c * 5 / 100)
                return
            if self.dp2 > 0:
                temp = self.dp2 - 1
                if temp > 60:
                    temp = 60
                if temp > 1:
                    sub = df3m[-temp:]
                    if len(sub.query('bias2_rate > 0.015')) > 0:
                        logger.info("41")
                        return
                    if len(sub.query('bias2_rate > 0 ')) > 6:
                        logger.info("42")
                        return
                logger.info(k3m['bias_rate'])
                if k3m['bias_rate'] < 0:
                    if abs(k3m['bias_rate']) < 1.1 / 1000:
                        # if not pre['yy']:
                        self.open_short()
                        return
                            # 是否触发macd的背离
            if self.gp2 > 10:
                if df15m[-20:]['l'].min() < df15m[-50:-20]['l'].min():
                    logger.info('上涨无力')
                    return
                temp = self.gp2 - 1
                if temp > 60:
                    temp = 60
                if temp > 1:
                    sub = df3m[-temp:]
                    if len(sub.query('bias2_rate < -0.015')) > 0:
                        return
                    if len(sub.query('bias2_rate < 0 ')) > 6:
                        return
                if pre['yy'] and k3m['bias_rate'] > 0 and k3m['bias_rate'] < 3 / 1000:
                    logger.info(f'考虑多吗? {k3m["c"]} {k3m["ma"]}')
                    self.open_long()
                    self.save_loss_price(df3m[-60:]['l'].min() - c / 100)
                    return

    def handle_win_out(self):
        df1h = self.df1h
        gp = ema.gold_pos_v2(df1h, 'ma', 'ma2')
        dp = ema.dead_pos_v2(df1h, 'ma', 'ma2')
        super().handle_win_out()

        if self.has_order:
            if self.is_now_long():
                if self.get_pure_win() > 1 / 100:
                    self.close()
                    return

        df = self.df3m
        k3m = self.df3m.iloc[-1]
        v1 = 1.7 / 100
        wave = nake.max_wave(df[-100:])
        if wave > 12 / 100:
            v1 = 7 / 100
        elif wave > 8 / 100:
            v1 = 4 / 100
        elif wave > 6 / 100:
            v1 = 2.8 / 100

        logger.info(f"v1 {v1}")
        logger.info(k3m['bias_rate'])
        if not self.has_order:
            return
        if self.get_pure_win() < 1 / 100:
            return

        if self.get_pure_win() > 7.5 / 100:
            self.close_v2()
            return
        pm = self.get_price_move()
        c = k3m['c']
        back = 2 / 100
        if self.have_key('loss_price'):
            v = float(self.get_key('loss_price'))
            if self.is_now_long():
                lp = c - c * back
                if lp > v:
                    self.save_loss_price(lp)
            if self.is_now_short():
                lp = c + c * back
                if lp < v:
                    self.save_loss_price(lp)
        else:
            if abs(pm) / c > 3 / 100:
                if self.is_now_long():
                    self.save_loss_price(c - c * 2 / 100)
                if self.is_now_short():
                    self.save_loss_price(c + c * 2 / 100)

        if self.is_now_long():
            if dp > 0:
                v1 = 1.4 / 100
                if self.get_pure_win() > 2.6 / 100:
                    self.close()
                    return

            if k3m['bias_rate'] > v1:
                self.close_v2()
                return
        if self.is_now_short():
            if gp > 0:
                if self.get_pure_win() > 2.2 / 100:
                    self.close()
                    return

            if k3m['bias_rate'] < -v1:
                self.close_v2()
                return

    def handle_loss_out(self):
        super().handle_loss_out()
        k3m = self.df3m.iloc[-1]
        df3m = self.df3m
        d31 = df3m[-35:]
        d32 = df3m[-80:-35]
        c = k3m['c']
        if not self.have_key('loss_price'):
            if self.has_order:
                if self.is_now_short():
                    # 保证不能亏太多
                    self.save_loss_price(df3m[-50:]['h'].max() + c * 1 / 100)
                if self.is_now_long():
                    self.save_loss_price(df3m[-50:]['l'].min() - c / 100)

        if self.has_order:
            if self.get_pure_win() > -1 / 100:
                return

            if self.is_now_long():
                if c < df3m[-100:-10]['l'].min():
                    self.close_v2()
                    return

                if self.dp2 > 0:
                    self.close()
                    return

            if self.is_now_short():
                logger.info(df3m[-100:-10]['h'].max())
                if c > df3m[-100:-10]['h'].max():
                    self.close_v2()
                    return
                if self.gp2 > 0:
                    self.close()
                    return
            if self.get_pure_win() < -3.5 / 100:
                self.close_v2()
                return
```

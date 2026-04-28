Реалізація фільтрів

SMAFilter - інкрементальна сума O(1): якщо вікно повне - віднімаємо старий елемент, додаємо новий, повертаємо sum / len.

EMAFilter - рекурсія a·x + (1−a)·last, O(1) пам'яті. При a=0.1 ≈ SMA з вікном 19.

MedianFilter - np.median(deque). Нелінійний, повністю ігнорує поодинокі викиди.

class SMAFilter:
    def update(self, x):
        if len(self.q) == self.w:
            self.sum -= self.q[0]
        self.q.append(x); self.sum += x
        return self.sum / len(self.q)

class EMAFilter:
    def update(self, x):
        if self.last is None: self.last = x
        else: self.last = self.a * x + (1 - self.a) * self.last
        return self.last

class MedianFilter:
    def update(self, x):
        self.q.append(x)
        return np.median(self.q)

Експеримент 1 — Базовий (Baseline)


W_SMA=20 - a_EMA=0.1 - W_MED=21

<table> <thead><tr><th>Фільтр</th><th>RMSE X</th><th>RMSE Y</th></tr></thead> <tbody> <tr><td>SMA</td><td>0.4485</td><td>0.9667</td></tr> <tr><td>EMA</td><td>0.4323</td><td>0.8591</td></tr> <tr><td>Median</td><td>0.4853</td><td>0.9538</td></tr> </tbody> </table>

Lag на змійці (13–20 с): всі фільтри запізнюються ~200 мс (10 відліків при W=20) - помітно на поворотах синусоїди.

Медіана vs викиди: зелена лінія рівна, синя (SMA) має сплески - викид у 10 м дає 10/20 = 0.5 м приросту середнього, тоді як медіана його просто відкидає.

Експеримент 2 - Over-smoothing

W_SMA=100 - a_EMA=0.02 - W_MED=21

<table> <thead><tr><th>Фільтр</th><th>RMSE X</th><th>RMSE Y</th></tr></thead> <tbody> <tr><td>SMA</td><td>2.0580</td><td>2.4698</td></tr> <tr><td>EMA</td><td>1.8275</td><td>2.3252</td></tr> <tr><td>Median</td><td>0.4853</td><td>0.9538</td></tr> </tbody> </table>

Парадокс спектру: RMSE зріс у 4-5 разів. На графіку 4b лінії помилки SMA/EMA піднялись вище сірої (вхідний шум) у зоні 0–1 Гц.

Причина: Lag = W/2 / FS = 100/2 / 50 = 1 с. При швидкості 3 м/с це 3 м позиційної помилки -- більше ніж вхідний шум 0.8 м. Фільтр прибрав "тремтіння", але спотворив саму траєкторію . Більше згладжування != менша похибка

Експеримент 3 - Медіанний фільтр W=5

W_SMA=5 - a_EMA=0.33 - W_MED=5

<table> <thead><tr><th>Фільтр</th><th>RMSE X</th><th>RMSE Y</th></tr></thead> <tbody> <tr><td>SMA</td><td>0.5809</td><td>0.7097</td></tr> <tr><td>EMA</td><td>0.5948</td><td>0.7018</td></tr> <tr><td>Median</td><td>0.4077</td><td>0.4684</td></tr> </tbody> </table>

Median W=5 на 34% точніший за SMA W=5 при однаковому lag - завдяки стійкості до викидів "Широкі" викиди фільтр не усуває, але при OUTLIER_PROB=0.02 вони майже завжди поодинокі

Висновки

<table> <thead><tr><th>Критерій</th><th>SMA</th><th>EMA</th><th>Median</th></tr></thead> <tbody> <tr><td>Гауссовий шум</td><td>так</td><td>так</td><td>так</td></tr> <tr><td>Імпульсні викиди</td><td>ні</td><td>ні</td><td>так</td></tr> <tr><td>Lag</td><td>W/2</td><td>≈1/α/2</td><td>W/2</td></tr> <tr><td>Складність</td><td>O(1)</td><td>O(1)</td><td>O(W·logW)</td></tr> <tr><td>Пам'ять</td><td>O(W)</td><td>O(1)</td><td>O(W)</td></tr> </tbody> </table>

а) Видалення викидів - Median W=5–11 - єдиний правильний вибір

б) Плавна траєкторія - EMA α=0.2–0.3 або SMA W=5–10 - мінімальний lag

Практичний рецепт: каскад Median W=5 - EMA a=0.2 - стійкість до аномалій без значного lag
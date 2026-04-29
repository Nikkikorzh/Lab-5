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

Експеримент 1 - Базовий

<img width="1386" height="532" alt="image" src="https://github.com/user-attachments/assets/d02a2c46-f7b9-411b-928a-a18e02ae22de" />
<img width="1087" height="567" alt="image" src="https://github.com/user-attachments/assets/b740cec2-918e-41ca-8325-c92f8956df51" />
<img width="1047" height="289" alt="image" src="https://github.com/user-attachments/assets/83cad23e-0bae-469a-9766-2d39894a1483" />


W_SMA=20 - a_EMA=0.1 - W_MED=21

<table> <thead><tr><th>Фільтр</th><th>RMSE X</th><th>RMSE Y</th></tr></thead> <tbody> <tr><td>SMA</td><td>0.4485</td><td>0.9667</td></tr> <tr><td>EMA</td><td>0.4323</td><td>0.8591</td></tr> <tr><td>Median</td><td>0.4853</td><td>0.9538</td></tr> </tbody> </table>

Lag на змійці (13–20 с): всі фільтри запізнюються ~200 мс (10 відліків при W=20) - помітно на поворотах синусоїди.

Медіана vs викиди: зелена лінія рівна, синя (SMA) має сплески - викид у 10 м дає 10/20 = 0.5 м приросту середнього, тоді як медіана його просто відкидає.

Експеримент 2 - Over-smoothing

<img width="1032" height="399" alt="image" src="https://github.com/user-attachments/assets/385d3d66-6abd-442f-ad90-47d75ae5d6e4" />
<img width="1009" height="556" alt="image" src="https://github.com/user-attachments/assets/5c9aeb32-859f-4c51-ae45-1a936a1caa03" />
<img width="1025" height="291" alt="image" src="https://github.com/user-attachments/assets/87b8a291-c127-44ea-ac6c-cf25219ba818" />


W_SMA=100 - a_EMA=0.02 - W_MED=21

<table> <thead><tr><th>Фільтр</th><th>RMSE X</th><th>RMSE Y</th></tr></thead> <tbody> <tr><td>SMA</td><td>2.0580</td><td>2.4698</td></tr> <tr><td>EMA</td><td>1.8275</td><td>2.3252</td></tr> <tr><td>Median</td><td>0.4853</td><td>0.9538</td></tr> </tbody> </table>

Парадокс спектру: RMSE зріс у 4-5 разів. На графіку 4b лінії помилки SMA/EMA піднялись вище сірої (вхідний шум) у зоні 0-1 Гц.

Причина: Lag = W/2 / FS = 100/2 / 50 = 1 с. При швидкості 3 м/с це 3 м позиційної помилки - більше ніж вхідний шум 0.8 м. Фільтр прибрав "тремтіння", але спотворив саму траєкторію . Більше згладжування != менша похибка

Експеримент 3 - Медіанний фільтр W=5

<img width="1072" height="398" alt="image" src="https://github.com/user-attachments/assets/32f9178e-5f7e-4a2c-a2c2-d9cc52f55254" />
<img width="1046" height="546" alt="image" src="https://github.com/user-attachments/assets/ed3436b8-6b9e-4077-9bcf-b9057b956bd4" />
<img width="1087" height="286" alt="image" src="https://github.com/user-attachments/assets/888d8719-07e9-44ab-82df-ba47651bf9ec" />


W_SMA=5 - a_EMA=0.33 - W_MED=5

<table> <thead><tr><th>Фільтр</th><th>RMSE X</th><th>RMSE Y</th></tr></thead> <tbody> <tr><td>SMA</td><td>0.5809</td><td>0.7097</td></tr> <tr><td>EMA</td><td>0.5948</td><td>0.7018</td></tr> <tr><td>Median</td><td>0.4077</td><td>0.4684</td></tr> </tbody> </table>

Median W=5 на 34% точніший за SMA W=5 при однаковому lag - завдяки стійкості до викидів "Широкі" викиди фільтр не усуває, але при OUTLIER_PROB=0.02 вони майже завжди поодинокі

Висновки

<table> <thead><tr><th>Критерій</th><th>SMA</th><th>EMA</th><th>Median</th></tr></thead> <tbody> <tr><td>Гауссовий шум</td><td>так</td><td>так</td><td>так</td></tr> <tr><td>Імпульсні викиди</td><td>ні</td><td>ні</td><td>так</td></tr> <tr><td>Lag</td><td>W/2</td><td>≈1/α/2</td><td>W/2</td></tr> <tr><td>Складність</td><td>O(1)</td><td>O(1)</td><td>O(W·logW)</td></tr> <tr><td>Пам'ять</td><td>O(W)</td><td>O(1)</td><td>O(W)</td></tr> </tbody> </table>

а) Видалення викидів - Median W=5-11 - єдиний правильний вибір

б) Плавна траєкторія - EMA α=0.2-0.3 або SMA W=5–10 - мінімальний lag

Практичний рецепт: каскад Median W=5 - EMA a=0.2 - стійкість до аномалій без значного lag

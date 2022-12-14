# Содержание _0_

---

0. Содержание
1. Постановка задачи
2. Решение задачи
	1. Обучение на подмножестве оригинального датасета
3. Дистилляция
4. Сравнение моделей
5. Вывод
6. Список литературы

---

# Постановка задачи _1_

---

Задача состоит из четырёх основных этапов:
- Сравнение предобученной модели из `clip.load("ViT-B/32", device=device, jit=False)` с необученной моделью из конструктора `clip.model.CLIP(...)`;
- Обучение модели на датасете из 3000 вхождений и модели на подмножестве этого датасета (2600 вхождений);
- Сравнение всех созданных до сего момента моделей;
- Выбор метода дистилляции и произведение дистилляции.

---

# Решение задачи _2_

---

Помимо основных двух статей, ссылки на которые будут дальше в отчёте, есть *honorable mentions* — информация, которая очень помогла понять некоторые концепты, разобраться с неопределённостями и заполнить пробелы в знаниях, необходимых для решения данной задачи:
- *Issue* в гитхабе[^5];
- Статья о *Knowledge Distillation*[^4].

## Обучение на подмножестве оригинального датасета _2.1_

Из оригинального датасета были удалены все вхождения, соответствующие следующим фильтрам по названию файлов:
- \*black\*;
- \*advice\*;
- \*angry\*;
- \*annoying\*;
- \*jew\*;
- \*cat\*;
- \*anime\*;
- \*man\*;
- \*dog\*;
- \*idiot\*;
- \*owl\*;
- \*dad\*;
- \*minecraft\*;
- \*girl\*;
- \*boy\*.

---

# Дистилляция _3_

---

Не вдаваясь в глубокие технологические подробности, можно сказать, что дистилляция это попытка уменьшить размер модели и возможно увеличить её быстродействие, жертвуя сложностью модели и иногда её точностью. Стоит заметить, что не всегда дистилляция значит потерю качества модели.

Был выбран метод дистилляции, называемый *Knowledge Distillation*, так как он наиболее распространён в исследованиях дистилляции моделей (в том числе моделей *CLIP*), и соответсвенно больше подробной и понятной информации о том как проводить данный метод дистилляции, какие у него плюсы и минусы. 

Суть этого метода заключается в обучении более сложной модели **учителя** — иногда на большем количестве эпох — с одной из классических[^1] функций потерь.
Затем обучается модель ученика, но каждое вхождение датасета подаётся на вход не только обучающейся модели ученика, но и уже статичной обученной модели учителя. Затем потеря для модели ученика считается как функция от выходных данных модели как ученика, так и учителя. В данной конкретной реализации, основанной на статье[^3], функция выглядит следующим образом:

$$L_{KD}(x, y, W_s, W_t) = L_S(y, z^s) + λ_T * L_H(y, z^s, z^t)$$

$$z^s = f_s(x, y; W_s)$$

$$z^t = f_t(x, y; W_t)$$

- $L_S$ — первый компонент сложной функции потерь. Классическая потеря, в данной реализации — функция потери кросс энтропии;
- $L_H$ — "подсказка" учителя, которая в данной реализации вычисляется как дивергенция Кульбака-Лейблера между софтмаксами выходов моделей учителя и ученика;
- $\lambda_T$ — масштабный параметр для масштабирования вклада подсказки учителя в функцию потери ученика;
- $z^s$ — выход модели ученика;
- $z^t$ — выход модели учителя;
- $T$ — параметр температуры для подсказки учителя.

Формулы вычисления $L_S$ и $L_H$ выглядят следующим образом:

$$L_S(y, z^s) = −\frac{1}{N} \sum_{n=1}^{N}y_n * log(z^s_n)$$

$$L_H(y, z^s, z^t) = 2 * T^2 * D_{KL} ((\sigma'(\frac{z^s}{T}) || \sigma'(\frac{z^t}{T}))$$

- $\sigma'$ — функция *softmax*.
- $D_{KL}$ — дивергенция Кульбака-Лейблера

Программная реализация вышеописанных формул выглядит следующим образом:

![image](./imgs/img%20(5).png)

---

# Сравнение моделей _4_

---

Обозначения:
- 1e — дообученная одну эпоху на полном 3000 датасете модель из `clip.load`
- cs — дообученная пять эпох на урезанном 2600 датасете модель из `clip.load`
- stud_best — лучшая из 20 эпох дистиллированная модель ученика
- stud_last — последняя из 20 эпох дистиллированная модель ученика
- teach_best — лучшая из 40 эпох дистиллированная модель учителя
- teach_last — последняя из 40 эпох дистиллированная модель учителя

Сравнение моделей происходило с помощью следующей функции, высчитывающей точность (*precision*) модели:

![image](./imgs/img%20(6).png)

Аргументы в эту функцию подставлялись для каждой отдельной модели соответсвенно, а `appendix` — это текстовый параметр, который помогает легче в текстовом выводе отличить точность одной модели от точности другой.

Вывод вышеописанной функции для всех созданных моделей выглядит следующим образом:

![image](./imgs/img%20(12).png)

Также с помощью следующей функции:

![image](./imgs/img%20(10).png)

был посчитан BLEU score:

![image](./imgs/img%20(13).png)

И посчитаны значения *Word Diversity* для каждой модели:

![image](./imgs/img%20(11).png)

Стоит заметить, что модели *1e* и *cs* показывают намного лучшие результаты, так как они были не обучены с нуля, а дообучены на датасете из 3000 и 2600 вхождений в течение 1 и 5 эпох соответственно. Базоая для них модель — `clip.load("ViT-B/32", device=device, jit=False)`.

---

# Вывод _5_

---

## Переобучение

Судя по всему, имеет место быть переобучение как при классическом обучении модели, так и при дистилляции методом *Knowledge Distillation*, так как среди всех сорока эпох обучения учителя лучшей эпохой является нулевая; аналогично для дистилляции, создания ученика. В течение всего обучения обоих моделей потеря на тестовом наборе стабильно увеличивалась, а потеря на тренировочном наборе стабильно уменьшалась, что непрямым образом говорит о переобучении. Отсюда можно сделать вывод, что обучения с нуля модели *CLIP* на датасете из 3000 вхождений не является оптимальным. Можно предположить, что за 3000 вхождений модель не способна выделить особенные признаки, которые позволили бы ей обобщить её предсказательную способность на вхождения вне датасета, на котором модель училась — об этом говорит крайне низкая точность работы модели (*precision*) в ~3%.

## Заключение

Успешно удалость произвести дистилляцию модели методом *Knowledge Distillation*. Вес файла обученной модели уменьшился с ~338мб до ~192мб. Показатели точности не уменьшились, а даже увеличились у ученика относительно учителя, но не стоит из этого делать вывод, что дистилляция моделей может увеличить точность или всегда её сохранить. 

Данное поведение вероятно связано с тем, что модели ученика и учителя были обучены в течение 40 эпох на датасете из 3000 вхождений, что занимает в общей сложности ~40 часов, когда оригинальный *CLIP*, который можно получить с помощью `clip.load` обучается на таких объёмных датасетах, на которых 3000 вхождений это меньше сотой доли процента всего датасета.

Для сравнения, новый датасет, который собрали создатели *CLIP* содержит около 400 миллионов вхождений, обучается 32 эпохи с размером батча в 32768. Обучение подобных моделей заняло у авторов около 2-х недель при использовании 592 Nvidia Tesla V100 видеокарт [^2].

---

# Список литературы _6_

[^1]: Под классической функцией потерь имеется ввиду любая функция, задействующая только информацию внутри одной модели, для которой потеря и считается.
[^2]: [Оригинальная статья о *CLIP*](https://arxiv.org/pdf/2103.00020.pdf)
[^3]: [Статья, на которой основана реализация](https://arxiv.org/pdf/1910.12061.pdf)
[^4]: [Статья об экспериментах с *Knowledge Distillation*](https://tech.pic-collage.com/distillation-of-clip-model-and-other-experiments-f8394b7321ce)
[^5]: [*Issue* с информативной подсказкой о дистилляции](https://github.com/openai/CLIP/issues/72)
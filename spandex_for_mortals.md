# Разреженные матрицы для смертных

## Введение

>  Homo supermatricus - люди будущего - будут принимать ответственные решения с использованием суперматриц
>
> Томас Саати

Многие расчетные задачи сводятся к решению системы линейных уравнений

[перечислить задачи]

Задачи реального мира, как правило, оперируют большим набором переменных состояния (параметров) [привести примеры]. Растет и система уравнений. С курса линейной алгебры мы знаем, что решить такую систему можно, например, методом Гаусса. На практике, впрочем, обычно используются другие алгоритмы.

У физических систем есть особенность, что в их матрицах содержатся частные производные. Т.е., как правило, большая часть строк в такой матрице заполнена нулями. В реальных задачах, решаемых в частных производных, в такой матрице до 99.9% элементов равны нулю. Откуда можно сделать вывод, что можно серьезно сэкономить память, используя некий формат сжатого хранения ненулевых чисел в матрице.

## Целочисленные множества

При работе с разреженными данными часто приходится хранить уникальные индексы элементов. Во всеми нами любом С++ (или в чуть менее нами любимом C#) есть std::set (System.Collections.Generic.HashSet), которые как будто предназначены как раз для решения таких задач.
Однако все становится не столь радужно, когда приходится добавлять элементы в множество несколько раз за итерацию. Становится печально видеть, что вместо полезной нагрузки мы тратим время на вызовы хэш-функции и поиск элементов во множестве.

Предположим, что максимальное значение включаемого целого числа в такое множество известно заранее. Например, это число столбцов в разреженной матрице. В таком случае, можно имитировать множество следующим образом:

- EMPTY - целочисленная константа

- NIL - целочисленная константа

- Count - число элементов в множестве

- Capacity - размер множества

- First - первый элемент множества (NIL, если множество пустое)

- Next - вектор целых чисел длиной N. Если значение в векторе:

- - Не NIL и не EMPTY, то это значение следующего элемента в множестве
  - EMPTY, то такого элемента нет в множестве
  - NIL, то это последний элемент в множестве

Например, добавляем в множество длинной 10 поочередно значения 5, 1, 5, 2, 9. В результате получим:

```
Count = 4
Capacity = 10
First = 9
Next = EMPTY | 5 | 1 | EMPTY | EMPTY | NIL | EMPTY | EMPTY | EMPTY | 2
(индексы: 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9)
```

Преимущества:

- Типовые операции не требуют вызова хеш-функции
- Типовые операции (кроме случая, описанного в недостатках) не требуют поиска элемента в дереве значений
- Типовые операции не (де)аллоцируют память
- Как следствие предыдущих пунктов, типовые операции крайне быстры

Недостатки:

- Годится только для хранения целых чисел
- Требуется заранее знать максимальное значение хранимого элемента
- Требуется заранее выделить память под вектор Next
- Требуется O(n) действий для удаление элемента

## Структуры данных

Вспомним, что применить все значения X к матрице коэффициентов А равнозначно умножению матрицы А на матрицу Х. Известный нам алгоритм умножения матриц имеет алгоритмическую сложность O(n^3) и […] флопсов (т.е. математических операций). Для более сложного в реализации метода Штрассена […], что тоже много. Т.е. скажем для матриц со стороной 10000 получится 10^12 операций. Это неприемлемо много.

Получается, что для наших разреженных матриц кроме особого формата хранения требуется и модификация элементарных алгоритмов работы с матрицами.

**C**ompressed **S**parse **R**ows (сжатые разреженные строки) и **C**ompressed **S**parse **C**ols (сжатые разреженные столбцы). Формат CSC нередко более удобен при решении численных задач.

Идея сжатых разреженных форматов в том, что раз память компьютера линейна, то и хранить участвующие в расчеты соседние элементы надо рядом, в соседних ячейках памяти. Так мы с большой вероятностью можем уместить данные в кэш-память процессора и снизим число ошибок предсказания [ссылки на википедию].

Т.е., если в некой условной строке матрицы содержится ```[0 3 0 0 0 1 0]```, то в сжатом формате должно храниться последовательно только ```[3 1]```. Хорошо, но таким образом мы теряем информацию о номерах столбцов ненулевых элементов. Поэтому первоначальную строку следует представить в сжатом виде уже двумя последовательностями ```[3 1]``` и ```[1 5]```, где первая последовательность - значения ненулевых элементов строки, а вторая - номера их столбцов.

Теперь добавим в нашу матрицу еще одну строку ```[[0 3 0 0 0 1 0] ; [2 0 0 0 4 0 0]]```. Соответственно эти две строки можно представить так: ```[3 1 2 4]``` и ```[1 5 0 4]```. 

Появилось затруднение, связаннное с тем, что невозможно узнать, где заканчиваются ненулевые элементы одной строки и начинаются другой. Заведем для этого третий вектор ```[0 2 4]```, в котором каждый элемент (за исключением последнего) указывает на индекс начала строки в векторах ненулевых элементов и номеров столбцов (для CSR матриц и строк - для CSC матриц).

Последнее число в векторе Rows - число ненулевых элементов во всей матрице. А число ненулевых элементов в i-ой строке = ```R[i+1] - R[i]```

Мы столь подробно остановились на CSR формате по той причине, что все последующее изложение предполагает, что читатель понимает и может свободно оперировать разреженными данными в сжатом формате

[описание CSR формата]

Какие недостатки у этого формата хранения разреженных данных. Все хорошо ровно до тех пор, пока мы обрабатываем данные в матрице строго по очереди следования. Если же потребуется перебрать элементы матрицы по столбцам, то придется каждый раз искать требуемый элемент. Другая проблема возникнет, если нам надо обращаться к элементу на произвольной позиции. А[i,j] так просто уже не получится.

Проблема обращения к элементу на произвольной позиции не такая критичная, как может показаться. Дело в том, что, как правило, такая возможность невостребована. Кроме частного случая получения элемента на диагонали (т.е. у которого i  == j). Но можно модифицировать формат хранения, отдельно храня диагональ матрицы в полном векторе.

[описание измененного CSR формата]

Для сравнения покажем отличие такого формата от, к сожалению, нередко встречаемого SortedSet. Перечислить все элементы для CSR формата E(k), где k – число элементов в i-й строке. Для SortedSet это будет E(k \* logk). Т.е. существенно дольше (не говоря уж о дополнительных затратах на память)

## Умножение матриц

Теперь мы можем перейти к модификации алгоритма перемножения матриц.

Надо помнить, что нам надо не только изменить проход по матрице, но также и учитывать, что писать в результирующую матрицу мы напрямую не можем. Сначала надо построить «портрет» (его еще называют «профилем») матрицы. Т.е. узнать в каких позициях результирующей матрицы будут ненулевые элементы. Без прямого вычисления. Или хотя бы, сколько таких элементов будет на каждой строке. Дело в том, что используемый нами формат хранения матрицы строго фиксирует число элементов в каждой строке и изменить его можно только вставив в середину вектора новое значение и скорректировав все последующие элементы в массиве rows. Алгоритмическая сложность вставки в массив […]

Для перемножения матриц надо учитывать, что ```0 * x``` и ```x * 0``` равны нулю. Т.е. в результирующей матрице ненулевые элементы будут стоять при наличии ненулевых элементов на соответствующих позициях в *обеих* матрицах-множителях. Это простое правило позволит нам эффективно построить портрет матрицы X, которая будет результатом перемножнения матриц A (размерностью M на N) и B (размерностью N на K).

Приведем сначала известный алгоритм умножения полных матриц:

```ruby
for i = 0..M:
    for k = 0..K:
        for j = 0..N:
            X[i,k] += A[i,j] * B[j,k]
        end
    end
end
```



Поэтому разделим процесс умножения разреженных матриц на два этапа. Сначала сформируем портрет матрицы, а затем проведем, собственно вычисление.

[описание алгоритма]

Аналогичным образом модифицируются алгоритмы сложения матриц, умножения на вектор, умножения на число и другие. Оставим их читателю.

## Разложение Холецкого

Вернемся к первоначальной задаче. Вспомним, что все затевалось ради того, чтобы решать системы линейных уравнений. Представим, на время, что за каждой буквой в уравнении Ax = b стоят обычные (действительные) числа. Тогда, чтобы найти х надо посчитать b/A. Или, что тоже самое, A^(-1) \* b.

Можно также поступить и для матричного представления. Найти матрицу, обратную к A и умножить результат на b. Однако получить обратную матрицу не так просто. Это сложно и с вычислительной точки зрения и точки зрения надежности расчетов [привести пример, какие недостатки]

Поэтому для решения системы линейных уравнений воспользуемся алгоритмом разложения Холецкого.

Сначала покажем алгоритм для полных матриц

[холецкий для полных матриц]

Все просто, не так ли? Но нас интересуют разреженные матрицы, в которых мы не можем бесплатно обращаться к элементу на произвольной позиции. Для этого есть несколько трюков:

- необходимо иметь полный вектор (обычный массив длиной в число столбцов в матрице), куда за один проход записать все ненулевые элементы в строке. Затем можно многократно менять значения в этом векторе с алгоритмической сложностью O(1). После окончания вычисления возвращаем значения из вспомогательного массива в строку разреженной матрицы.

- необходимо поддерживать два портрета матрицы: построчный и постолбцовый. Таким образом можно быстро проходить матрицу в двух направлениях.

- матрица в разложении Холецкого получается симметричная. Нет никакого смысла хранить одну из половин, т.к. она дублируется. Хранить можно только верхний (или нижний, это непринциально) треугольник матрицы. Этим мы сокращаем вдвое не только потребление памяти, но и объем вычислений.

Теперь можно перейти к разложению Холецкого для разреженных матриц. Опять же разобьем алгоритм на два этапа: построение портрета результирующей матрицы и непосредственно вычисление.

[холецкий для разреженных матриц]

## Метод наименьших квадратов

[...]

## Система линейных уравнений

Мы можем решить систему линейных уравнений на разреженных матрицах. Внедрили в алгоритм трюки для сокращения потребления памяти и объема вычислений. Перемножение матриц для прямого прохода у нас тоже есть. Кажется, на этом все?

К счастью, нет. Тема работы с разреженными матрицами неисчерпывается так быстро.

Заметим, что обычно нам требуется не просто однократное решение системы линейных уравнений.  Надо вычислять одну и ту же систему уравнений снова и снова. Например, в методе ньютона, вычисляя подшагивания.

Облегчает задачу, что часто уточняя коэффициенты в уравнении, мы не меняем ее портрет. Или, по крайней мере, не меняем портрет фактора (результата разложения Холецкого). Таким образом, как правило, можно однократно сформировать портреты участвующих в расчетах матрицах до начала первой итерации, а затем в каждом алгоритме пропускать первый этап (этап построения портрета результирующей матрицы). Это значительно повышает эффективность расчетов

Кроме того, может так случиться (и случается, на самом деле, всегда), что данные в матрице распределены неравномерно. [..привести картинки…]

А значит можно выделить в разреженных матрицах блоки, которые можно представить как полные подматрицы. А библиотеки быстро выполняющие операции над полными матрицами существуют давно. Кроме такого, чисто механического ускорения, сокращается и размерность матрицы, что тоже крайне положительно влияет на скорость расчетов.

Однако сам алгоритм выделения достаточно сложен и трудоемок. [ссылки]. Вынесем его за скобки этой серии статей. Разберем частный случай.

Вспомним, что в самом начале мы говорили, что решаем системы, полученные из решения уравнений в частных производных. Нередко это частные производные от комплексного переменного. Т.е. у нас появляется гарантия, что ненулевые элементы будут идти попарно. А когда будет строится симметричная матрица, то как минимум полными подматрицами будут каждый блок 2 на 2 [пример]. Это все ускоряет. Однако надо скорректировать все алгоритмы с учетом использования комплексного переменного.

[…]

Про нормализацию

[…]

## Упорядочивания строк в матрице

Про перестановку строк. Это сложная задача. Она решается в […] metis. Но с удовлетворительным качеством ее можно решить используя алгоритм AMD. Алгоритм заключается в следующем

[рисунки на графах]

A =

   0.44561   0.92962   0.10582   0.87256   0.00000   0.00000

   0.00000   0.00000   0.00000   0.41421   0.00000   0.00000

   0.00000   0.00000   0.00000   0.00000   0.58921   0.00000

   0.00000   0.87043   0.06028   0.00000   0.00000   0.63730

   0.00000   0.47918   0.00000   0.42161   0.15305   0.00000

AtA =

   0.19857   0.41425   0.04715   0.38882   0.00000   0.00000

   0.41425   1.85146   0.15084   1.01318   0.07334   0.55473

   0.04715   0.15084   0.01483   0.09233   0.00000   0.03842

   0.38882   1.01318   0.09233   1.11069   0.06453   0.00000

   0.00000   0.07334   0.00000   0.06453   0.37059   0.00000

   0.00000   0.55473   0.03842   0.00000   0.00000   0.40615

## Обновление фактора Холецкого

И последнее, что можно сказать про разреженные матрицы для смертных. Это алгоритмы Update/Downdate ([cсылка на википедию cholrank1]). Представим, что у нас есть система линейных уравнений описывающих некую физическую систему. Вычислив ее раз, мы получим некие значения переменной X. Однако результаты нас не удовлетворяют, т.к. мы точно знаем, что одно (или несколько) значений Х не может выходить за конкретные физические пределы. Одним из возможных решений является добавить в исходную матрицу дополнительное («фиктивное») уравнение, которое ограничивает значение Х.

Однако это, если и поменяет портрет исходной матрицы (а если использовать весовые коэффициенты, то не изменит), то на портрет фактора холецкого уже не влияет. Однако даже с учетом неизменности портрета, можно предположить, что если мы добавляем только одну строку в исходную матрицу, то навряд ли изменится целтиком весь фактор. И, конечно, это всегда не так, меняется лишь малая часть. Можно частично многократно обновлять фактор. Это значительно уменьшает число операций.

[описание алгоритма для полных матриц]

[описание алгоритма для разреженных матриц]

## Заключение

В эту серию статей не попали темы, касающиеся собственно процесса поиска решения, учета ограничений, расчета производных, построение полного портрета матриц и т.д.

Ссылки:

Синяя книга

Писсанецки

Гил, Мюррей

Другая синяя книга

И т.д.
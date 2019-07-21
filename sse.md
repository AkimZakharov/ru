# Векторизация

Рассмотрим следующую программу, в которой складывают два массива:

```c++
#pragma GCC optimize("O3")
// ^ включает самый "агрессивный" уровень оптимизации
// то же самое, что добавить флаг "-O3" при компиляции из консоли
#include <iostream>

const int n = 1e6;
int a[n], b[n];

int main() {
    int s = 0;
    // сложим всю сумму в "где-то используемую" переменную,
    // иначе компилятор просто вырежет нужный нам кусок кода

    for (int t = 0; t < 10000; t++)
        for (int i = 0; i < n; i++)
            s += a[i] + b[i];

    std::cout << s << std::endl;

    return 0;
}
```

Если скомпилировать этот код под GCC без всяких дополнительных настроек и запустить, он отработает за 1.65 секунды.

Добавим теперь следующую магическую директиву в самое начало программы:

```c++
#pragma GCC target("avx2")
// ...остальное в точности как было
```

Скомпилировав и запустив при тех же условиях, программа завершается уже за 0.73 секунды. Это более чем в два раза быстрее, при том, что сам код и уровень оптимизации мы не меняли.

Чтобы понять, что здесь происходит, нам нужно сначала разъяснить некоторые особенности работы современных компьютеров.

## Complex Instruction Set Computing

Раньше, во времена, когда компьютеры назывались ЭВМ-ами и занимали целую комнату, увеличение производительности происходило в основном за счёт увеличения тактовой частоты. Тактовая частота условно равна количеству инструкций, выполняемому процессором за единицу времени. (На современных процессорах [это не так](http://ithare.com/infographics-operation-costs-in-cpu-clock-cycles/) — разные инструкции занимают разное время, которое ещё и может зависеть от разных обстоятельств.)

Помимо жесткого [физического ограничения](https://ru.wikipedia.org/wiki/%D0%A1%D0%BA%D0%BE%D1%80%D0%BE%D1%81%D1%82%D1%8C_%D1%81%D0%B2%D0%B5%D1%82%D0%B0) на максимально возможную тактовую частоту, такой такой подход в какой-то момент просто перестал быть экономически оправданным: прямое увеличение тактовой частоты приводит к в более чем линейному потреблению энергии и выделению тепла, которое к тому же нужно как-то выводить.

Поэтому вендоры, в погоне за более дешёвым [флопсом](https://en.wikipedia.org/wiki/FLOPS) за доллар, пошли по другому пути: стали добавлять более сложные инструкции, которые делают сразу много полезных действий за раз. Микросхема от добавления новых инструкций сильно усложняется, что становится критичным для многих других применениях. В связи с этим, все архитектуры стали делиться на два типа:

* [RISC](https://en.wikipedia.org/wiki/Reduced_instruction_set_computer) (англ. **reduced** instruction set computer), в которых длина кода (идентификатора) самой инструкции ограничена, а значит ограничено и само количество инструкций. Самые первые компьютеры относились к этому типу и могли не иметь даже отдельных инструкций для умножения и деления. Такие процессоры требует меньше транзисторов, и как следствие сами меньше, дешевле и потребляют меньше энергии. Самое популярное семейство архитектур называется [arm](https://en.wikipedia.org/wiki/ARM_architecture) и используется почти на всех современных мобильных устройствах.

* [CISC](https://en.wikipedia.org/wiki/Complex_instruction_set_computer) (англ. **complex** instruction set computer), к которому относят всё, что не RISC — в них длина команды не фиксирована, что позволяет поддерживать практически произовльное количество инструкций. Самоя популярное семейство архитектур называется [x86](https://en.wikipedia.org/wiki/X86) и используется почти на всех современных декстопах и серверах.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/2/2a/Mips32_addi.svg/370px-Mips32_addi.svg.png)

Новые инструкции стали добавлять постепенно, причём разные в зависимости от области применения.

* В общеприменимых CPU довольно быстро добавили инструкцию, которая принимает числа $(x, y)$ и загружает в регистр данные по адресу $a \cdot x + y$. Это полезно при индексации массивов — не нужно отдельно индекс считать.

* На графических сопроцессорах появилась отдельная инструкция, которую называют «saxpy» (сокращенно от выражения `s += a * x + y`), которая полезна, например, при перемножении матриц.

* В последние GPU от Nvidia добавили «tensor core» — отдельную схему, которая перемножает две матрицы $4 \times 4$ и прибавляет к третьей, как бы производя $4 \times 4 \times 4 = 64$ умножений и $4 \times 4 = 16$ сложений за раз, что сильно ускоряет алгоритмы [блочного матричного умножения](https://en.wikipedia.org/wiki/Strassen_algorithm).

В этой статье мы сфокусируемся на отдельным виде инструкций, которые позволяют выполнять одну и ту же операцию сразу на какой-то последовательности данных. Эта концепция называется [SIMD](https://en.wikipedia.org/wiki/SIMD)-параллелизмом (англ. *single instruction, multiple data*).

## Streaming SIMD Extensions

SSE — это обобщённое называние всех SIMD-инструкций для x86.

Работают они следующим образом. Помимо обычных регистров (самых близких к процессору ячеек памяти, с которыми он непосредственно работает), есть дополнительные, вмещающие не 64, а 128, 256 или даже 512 бит — в зависимости от поддерживаемой версии SSE. В эти регистры загружается последовательные блоки из памяти, над ним производится какая-то последовательность операций, и итоговый результат записывается обратно в память. Сами операции обычно разбивают эту булеву последовательность на блоки, например, по 32 бит, и логически работают уже с ними.

Довольно легко получается оптимизировать простые циклы, производящие какие-нибудь независимые друг от друга операции над векторами (массивами) — поэтому сам такой подход называют *векторизацией*.

Например, какое-нибудь сложение двух int-овых массивов удаётся таким образом соптимизировать в $\frac{512}{32} = 16$ раз, если процессор поддерживает AVX512, а операции [битсета](https://algorithmica.org/ru/bitset) — в 512 раз (реализация из STL, [по всей видимости](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/std/bitset#L77), SSE не использует).

Очень часто SSE используют для работы с действительными числами, и вы этой  ситуации возникает прямой trade-off между точностью вычислений и скоростью работы: например, вместо double можно использовать float, и тогда в один и тот же регистр поместится в два раза больше чисел. По этой причине в последнее время стали развиваться различные методы [квантизации](https://en.wikipedia.org/wiki/Quantization): перевода исходых данных в какой-то более дискретизированный формат на входе какой-нибудь процедуры (например, [матричного умножения](https://github.com/google/gemmlowp)) и восстановления в исходный формат на выходе.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/2/21/SIMD.svg/1200px-SIMD.svg.png)

Конкретный набор инструкций и размеры регистров зависит от вендора и поколения архитектуры. На данный момент (лето 2019 года) [большинство процессоров](https://www.cpubenchmark.net/market_share.html) архитектуры x86 производит Intel, поэтому мы сконцентрируемся именно на их наборе инструкций.

Поддержка SIMD-инструкций добавлялись постепенно, сохраняя обратную совместимость. Если третий пентиум в 1999-м году умел работать с регистрами размера 128, то в самых современных i7 есть 512-битные регистры. Автор не является специалистом в проектировании микропроцессоров, но предполагает, то регистры больше 64 байт (512 бит) появятся очень не скоро, потому что это уже больше размера [кэш-линии](https://en.wikipedia.org/wiki/CPU_cache)

![](https://i0.wp.com/www.urtech.ca/wp-content/uploads/2017/11/Intel-mmx-sse-sse2-avx-AVX-512.png)

Чтобы разработчикам не нужно было предоставлять отдельные оптимизированные бинарники под каждую конкретную архитектуру, информация о поддержке наборов инструкций процессором зашита в ассемблерную инструкцию `cpuid`, которую можно просто вызвать в рантайме и всё узнать: [например так](https://gist.github.com/hi2p-perim/7855506).

В GCC есть встроенная функция `__builtin_cpu_supports`, которая берёт строчку-название набора инструкций ("sse", "avx2", "avx512f" и т. п.) и возвращает целое число — ноль или какую-то степень двойки. Эта функция работает так: входная строка во время компиляции переводится в нужную степень двойки, которая в рантайме просто AND-ится с маской из cpuid и возвращается — всё ради эффективности.

Экономя время читателю: сервера CodeForces и большинство онлайн джаджей на момент написания статьи поддерживают AVX2, то есть умеют работать с 256-битными регистрами.

## C++ intrinsics

SSE это те же чистые ассемблерные инструкции. Языки с каким-либо более верхним [уровнем абстракции](https://en.wikipedia.org/wiki/Java_virtual_machine) напрямую работать с ними уже не могут. Однако не обязательно писать на чистом ассемблере, чтобы их использовать — разработчики компиляторов уже позаботились об этом за вас и сделали встроенные функции-обёртки, которые называют *интринзиками* (англ. *intrinsic* — «внутренний»).

Чтобы их подключить, нужно указать `include` на соответствующий заголовочный файл, а также сказать компилятору о том, что мы хотим использовать конкретный набор или наборы инструкций. В примере из начала статьи мы сделали именно это, указав `target("avx2")` — компилятор получил доступ к более широким регистрам и продвинутым инструкциям для них, и смог соптимизировать программу примерно в два раза (по умолчанию включены 128-битные `sse` и `sse2`, поэтому в 2, а не в $\frac{256}{32} = 8$).

По аналогии с `<bits/stdc++.h>`, в GCC такой же заголовочный файл `<x86intrin.h>`, включающий в себя сразу все SSE-интринзики. Шаблон любителя засоренных неймспейсов и избыточно долгой компиляции может начинаться так:

```c++
#pragma GCC optimize("O3")
#pragma GCC target("avx2")

#include <x86intrin.h>
#include <bits/stdc++.h>

using namespace std;
```

Простой цикл, в котором складывают два массива 64-битных действительных чисел, на SSE-интринзиках будет выглядеть так:

```c++
double a[100], b[100], c[100];

for (int i = 0; i < 100; i += 4) {
    // загрузим два отрезка по 256 бит в свои регистры
    __m256d x = _mm256_loadu_pd(&a[i]);
    __m256d y = _mm256_loadu_pd(&b[i]);
    // - 256 означает размер регистров
    // - d означает "double"
    // - pd озанчает "packed double"

    // просуммируем числа и положим результат в другой регистр:
    __m256d z = _mm256_add_pd(x, y);
    // запишем содержимое регистра в память:
    _mm256_storeu_pd(&c[i], z);
}
```

Конвенция именования интринзиков такая же, как самих инструкций, а она такая же, как в ассемблере — то есть максимально короткая и непонятная.

Большинство команд кодируются как `_mm<размерность>_<действие>_<тип>`. Например:

* `_mm_add_epi16` — складывает две группы 16-битных *extended packed integer*, проще говоря $\frac{128}{16} = 8$ `short`-ов (в инструкциях, где размер регистра не указан, он равен 128).

* `_mm256_acos_pd` — принимает один регистр, содержащий 4 `double`-ов, и возвращает их арк-косинусы.

* `_mm256_broadcast_sd` — бродкастит (копирует) `double` из памяти во все четыре слота в регистре.

* `_mm256_ceil_pd` — округляет `double` к ближайшему `int`-у вверх.

* `_mm256_cmpeq_epi32` — сравнивает запакованные `int`-ы и возвращает вектор-маску, в которой для полностью совпавших элементов будет по 32 единицы.

* `_mm256_blendv_ps` — по заданной маске берёт значения либо из первого массива, либо из второго. Часто применяется для замены `if`-а.

Комбинаторно получается огромное количество различных SSE-интринзиков. Полная документация по ним — [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/) — лежит в закладках браузера у каждого уважающего себя performance engineer-а.

**Выравнивание.** Отдельно стоит отметить одну делать: операции чтения и записи имеют по две версии: `load` / `loadu` и `store` / `storeu`. Буква «u» здесь означает «unaligned» (англ. *невыровненный*). Первая корректно работает только тогда, когда весь считываемый блок помещается на одну кэш-линию (если это не так, то в рантаеме вызвется segfault), в то время как unaligned версия работает всегда и везде. 

Это отличие имело очень большое значение на старых компьютерах — если не получалось «выровнять» память, то производительность могла резко упасть (в два и более раз), так как нарушался паттерн последовательного доступа. На современных компьютерах это не так существенно: unaligned версия будет медленне, но в пределах 5%. Вручную «выровнять» память для последовательного чтения через `load` в случае с массивами можно так:

```c++
alignas(32) int a;
// указатель на начало массива теперь будет кратен 32 байтам,
// то есть размеру sse-блока; теперь любое чтение будет внутри кэш-линии

for (int i = 0; i < n; i += 8) {
    __m256i x = _mm256_load_epi32(&a[i]);
    // ...
}
```

**Типизация.** Вообще, загружать и сохранять данные интринзиками и вообще использовать `__m`-типы на самом деле не обязательно — всё можно сделать и обычным reinterpret_cast-ом. Все данные хранятся в одном и том же формате, и разные типы нужны просто для проверки типов и избежания связанных ошибок.

Некоторые операции есть только для какого-то одного типа, например, тот же `_mm256_blendv_ps` не имеет аналога для 32-битных `int`-ов, однако будет работать с ними абсолютно так же. Поэтому to make compiler happy можно применять к ним преобразания типов, которые не будут стоить дополнительных инструкций в рантайме. Они все имеют такой формат: `_mm<размерность>_cast<откуда>_<куда>`.

## Трудности автовекторизации

В самом начале статьи мы приводили пример кода, в котором уже оптимизированный бинарник получается без каких-либо изменений, кроме подключения нужного таргета компиляции. Зачем тогда вообще программисту делать что-либо ещё?

Дело в том, что иногда — очень редко — программист всё-таки умнее компилятора, потому что знает про задачу чуть больше.

Рассмотрим этот же пример, убрав из него всё лишнее:

```c++
void sum(int a[], int b[], int c[], int n) {
    for (int i = 0)
        c[i] = a[i] + b[i];
}
```

Почему эту функцию нельзя заменить на векторизованный-вариант автоматически?

Во-первых, потому что это не всегда корректно. Предположим, что `a[]` и `c[]` пересекаются, причём так, что указатели на начало массивов отличаются на 1-2 позиции. Ну, мало ли — может, мы такой изощрённой свёрткой хотели посчитать последовательность Фибоначчи. Тогда в simd-блоках данные будут пересекаться, и наблюдаемое поведение будет совсем не то, которое мы хотели.

Во-вторых, мы ничего не знаем про выравнивание этих массивов, и можем потерять производительность здесь (компилятор скорее всего сгенерирует инструкции с `loadu`).

На самом деле, для достаточно больших циклов на высоких уровнях оптимизации компилятор вставит проверки на эти случаи и сгенерирует два разных варианта: через SSE и обычный. Но эти проверки выполняются в рантайме и стоят лишних инструкций.

Существуют [различные способы](https://software.intel.com/sites/default/files/m/4/8/8/2/a/31848-CompilerAutovectorizationGuide.pdf) намекнуть компилятору, что конкретно мы имели в виду, но в сложных случаях — когда внутри цикла используются `if`-ы или какие-нибудь внешние функции — проще спуститься до уровня интринзиков и написать всё самому.

## Нетривиальный пример

Пусть нам зачем-то понадобилось возвести $10^8$ чисел в какие-то степени.

В SSE весьма сложно делить `int`-ы (см. примечания ниже), поэтому будем считать всё по модулю $2^{32}$, то есть просто переполняя естественным образом `unsigned int`.

```c++
#pragma GCC optimize("O3")
#pragma GCC target("avx2")

#include <x86intrin.h>
#include <bits/stdc++.h>

using namespace std;

typedef unsigned long long ull;
typedef __m256i reg;

const int n = 1e8;
alignas(32) unsigned bases[n], results[n], powers[n];

void timeit(void (*f)()) {
    // запускает другую функцию и меряет время её исполнения
    clock_t start = clock();
    f();
    cout << double(clock() - start) / CLOCKS_PER_SEC << endl;

    for (int i = 0; i < 10; i++)
        cout << results[i] << " ";
    cout << endl;
}

int main() {
    for (int i = 0; i < n; i++) {
        bases[i] = rand();
        powers[i] = rand();
    }
    
    // timeit(binpow_simple);
    // timeit(binpow_sse);
    
    return 0;
}
```

Напишем стандартное итеративное бинарное возведение в степень:

```c++
void binpow_simple() {
    for (int i = 0; i < n; i++) {
        unsigned a = bases[i], p = powers[i];

        unsigned res = 1;
        while (p > 0) {
            if (p & 1)
                res = (res * a);
            a = (a * a);
            p >>= 1;
        }

        results[i] = res;
    }
}
```

Этот код работает за 9.47 секунды.

Теперь попробуем векторизованную версию:

```c++
void binpow_sse() {
    const reg ones = _mm256_set_epi32(1, 1, 1, 1, 1, 1, 1, 1);
    for (int i = 0; i < n; i += 8) {
        reg a = _mm256_load_si256((__m256i*) &bases[i]);
        reg p = _mm256_load_si256((__m256i*) &powers[i]);
        reg res = ones;
        
        // на самом деле, здесь никакого цикла не будет
        // -- компилятор это развернёт в 30 отдельных операций
        for (int i = 0; i < 32; i++) {
            // чтобы не писать лишний иф, посчитаем для каждого элемента его множитель:
            // это будет либо единица, либо a, в зависимости от значения нижнего бита p

            // маски элементов, которые нужно домножать на a:
            reg mask = _mm256_cmpeq_epi32(_mm256_and_si256(p, ones), ones);
            // эта операция смешивает два вектора по маске:
            reg mul = _mm256_blendv_epi8(ones, a, mask);

            // res *= mul:
            res = _mm256_mullo_epi32(res, mul);

            // a *= a:
            a = _mm256_mullo_epi32(a, a);

            // p >>= 1:
            p = _mm256_srli_epi32(p, 1);
        }

        _mm256_store_si256((__m256i*) &results[i], res);
    }
}
```

Она уже работает за 0.7 секунды — в 13.5 раз быстрее. При этом там и дальше есть, что оптимизировать.

## Разное

**С++ в ассемблер.**  Посмотреть на генерируемые инструкции можно так:

```
g++ -S program.cpp -o program.s
```

Это позволяет понять, векторизует ли уже компилятор код или нет. Во многих IDE есть удобные плагины, позволяющие выяснять это для конкретных функций.

**Распечатать вектор.** Для дебага помогает такой код:

```c++
template<typename T>
void print(T var) {
    unsigned *val = (unsigned*) &var;
    for (int i = 0; i < 4; i++)
        cout << bitset<32>(val[i]) << " ";
    cout << endl;
}
```

В данном случае он выводит 4 группы по 32 бита из 128-битного вектора.

**Деление.** В SSE нет операции деления  `int`-ов, но есть для `float`-ов и производных. Также нет взятия остатка от деления, что осложняет вычисления в комбинаторике.

Для деления 32-битных целых чисел их можно аккуратно скастовать к даблу, поделить так, и скастовать обратно — точности хватит, хоть это и будет медленно.

Умножение работает в несколько раз быстрее деления, и поэтому для ускорения деления на известную константу $d$ есть следующий [трюк](https://ridiculousfish.com/blog/posts/labor-of-division-episode-iii.html): заменить выражение $x / d$  на $x \cdot \frac{1}{d}$, и при этом $\frac{1}{d}$ посчитать во время компиляции. Это уже идеально работает с `float`-ами, а для `int`-ов нужно приблизить $\frac{1}{d}$, подобрав «магическое» число $m$ и степень двойки $s$, такое что `x / d == (x * m*) >> s` для всех `x`, то есть мы заменяем деление на умножение и битовый сдвиг.

Можно показать, что такая пара чисел всегда существует, и компилятор сам оптимизирует деление на константу подобным образом. Вот, например, сгенерированные инструкции для деления `unsigned long long` на $10^9 + 7$:

```nasm
movq    %rdi, %rax
movabsq	$-8543223828751151131, %rdx ; загружает магическую константу в регистр
mulq	%rdx                        ; делает умножение
movq	%rdx, %rax
shrq	$29, %rax                   ; делает битовый сдвиг результата
```

Здесь для умножения используется «mixed precision» инструкция `mulq`, которая берёт два 64-битных числа и записывает 128-битный результат их умножения в два 64-битных регистра (lo, hi).

Для деления `long`-ов на SSE такой способ пока что не работает: аналогичная инструкция добавилась только в AVX512.
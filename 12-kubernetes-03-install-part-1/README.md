# 12.3 Развертывание кластера на собственных серверах, лекция 1  

Известно, что проекту нужны база данных, система кеширования, а само приложение состоит из бекенда и фронтенда. Опишите, какие ресурсы нужны, если известно:

* База данных должна быть отказоустойчивой. Потребляет 4 ГБ ОЗУ в работе, 1 ядро. 3 копии.
* Кэш должен быть отказоустойчивый. Потребляет 4 ГБ ОЗУ в работе, 1 ядро. 3 копии.
* Фронтенд обрабатывает внешние запросы быстро, отдавая статику. Потребляет не более 50 МБ ОЗУ на каждый экземпляр, 0.2 ядра. 5 копий.
* Бекенд потребляет 600 МБ ОЗУ и по 1 ядру на копию. 10 копий.

Расчет параметров проекта:  

| Наименование  | CPU  | Кол-во | Всего  |
|---------------|:----:|:------:|:------:|
| БД            |  1   |   3    |   3    |
| Кэш           |  1   |   3    |   3    |
| Фронтэнд      | 0.2  |   5    |   1    |
| Бекэнд        |  1   |   10   |   10   |
| **Итого CPU** |      |        | **17** |

| Наименование  | ОЗУ/ГБ | Кол-во |   Всего   |
|---------------|:------:|:------:|:---------:|
| БД            |   4    |   3    |    12     |
| Кэш           |   4    |   3    |    12     |
| Фронтэнд      |  0.05  |   5    |   0.25    |
| Бекэнд        |  0.6   |   10   |     6     |
| **Итого ОЗУ** |        |        | **30.25** |

Думаю достаточно три воркер ноды. Округлим полученные цифры чтобы делились на три:  
CPU - 18/3 = 6  
ОЗУ - 33/3 = 11  
Т.е. получается три ноды по 6 CPU и 11 ГБ.
Добавим к полученным нодам запас, на случай выхода из строя одной из них. Т.е. +3 CPU и +6ГБ (опять округляем до целых).  
Итого, промежуточное значение, три ноды по 9 CPU и 17ГБ.  
Теперь добавим служебных ресурсов к воркер нодам. По 1-му ядру и 1-му ГБ. В проекте нет информации о дисках, поэтому возьмем
минимум 100ГБ на каждую ноду.  
Итого, три ноды по 10 CPU, 18ГБ ОЗУ и 100ГБ диск.  
Не забудем про контрол ноду. Для отказоустойчивости возьмем три ноды по 2 ядра, 2ГБ ОЗУ и 50ГБ диск.  
  
**Посчитаем итоговые цифры:**  
* CPU - 36  
* ОЗУ - 60ГБ  
* Диск - 450ГБ  
  
**3 воркер ноды по 10 CPU, 18ГБ ОЗУ и 100ГБ каждая + 3 контрол ноды по 2 ядра, 2ГБ ОЗУ и 50ГБ диск каждая.**
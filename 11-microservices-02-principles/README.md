# 11.02 Микросервисы: принципы  

Вы работаете в крупной компании, которая строит систему на основе микросервисной архитектуры. Вам как DevOps специалисту
необходимо выдвинуть предложение по организации инфраструктуры, для разработки и эксплуатации.  

## Задача 1: API Gateway  

Сравинительная таблица различных программых решений:  

|                         | Kong                    | Tyk.io    | APIGee                            | AWS Gateway | Azure Gateway | Express Gateway |
|-------------------------|-------------------------|-----------|-----------------------------------|-------------|---------------|-----------------|
| Сложность развертывания | Одна нода               | Одна нода | Несколько нод с различными ролями | Облачный    | Облачный      | Гибкий          |
| Open Source             | Да                      | Да        | Нет                               | Нет         | Нет           | Да              |
| Требования ДБ           | Cassandra или Postgres  | Redis     | Cassandra, Zookeeper и Postgres   | -           | -             | Redis           |
| Основная технология     | NGINX                   | GoLang    | Java                              | Закрытая    | Закрытая      | Express.js      |
| On Premise              | Да                      | Да        | Да                                | Нет         | Нет           | Да              |
| Комьюнити               | Большое                 | Среднее   | Нет                               | Нет         | Нет           | Небольшое       |
| SSL/TLS                 | Да                      | Да        | Да                                | Да          | Да            | Да              |
| Аутентификация          | Да                      | Да        | Да                                | Да          | Да            | Да              |
| Авторизация             | Да                      | Да        | Да                                | Да          | Да            | Да              |
| Маршрутизация запросов  | Да                      | Да        | Да                                | Да          | Да            | Да              |
| Ограничение скорости    | Да                      | Да        | Да                                | Да          | Да            | Да              |
| Логирование             | Да                      | Да        | Да                                | Да          | Да            | Да              |

Большинство решений, доступных сегодня, поддерживают аутентификацию, терминацию HTTPS, маршрутизация запросов на основе конфигураций и многое другое.
Выбор зависит от конкретных потребностей, а так же и от платформ, которые уже используются. Например факторы, которые следует учитывать:  
* **Проприетарный или opensource.** Например в компании уже развернута инфраструктура в AWS/Azure. Логичным кажется выбор AWS/Azure Gateway. Или же наоборот нужна гибкость opensource.
* **Архитектура.** Различные шлюзы API поддерживают различные системы баз данных. Опять же, есть определенная зависимость от знакомства организации с конкретными внешними базами данных и их зависимости от них.
* **Язык программирования.** Некоторые варианты, особенно с открытым исходным кодом, могут включать некоторую настройку. Поэтому, может иметь значение, имеют ли разработчики знания определенных языков, таких как Golang например. Это упростит доработку.

## Задача 2: Брокер сообщений  

Сравнительная таблица брокеров сообщений:  

|                       | RabbitMQ | Kafka | ActiveMQ | Qpid C++ | Redis |
|-----------------------|----------|-------|----------|----------|-------|
| Кластеризации         | +        | +     | +        | +        | +     |
| Хранение              | +        | +     | +        | +        | -     |
| Скорость              | +        | +     | -        | -        | +     |
| Поддержка форматов    | +        | +     | -        | -        | +     |
| Права доступа         | +        | +     | +        | +        | +     |
| Простота эксплуатации | +        | +     | +        | -        | +     |

Cтоит обратить внимание на программный стек. Если нужна относительно простая интеграция, то можно довольствоваться уже 
имеющимся в стеке брокером, без использования дополнительного. Например, если в системе есть Celery Task Queue 
поверх RabbitMQ, удобнее работать с RabbitMQ или Redis, а не с Kafka, который не поддерживается и потребует написания 
дополнительного кода.  
Чтобы выбрать необходимый инструмент, важно понимать все их плюсы и минусы, с учетом конкретного применения, ситуации и требований.
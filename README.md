# posts-feed-recsys
Выпускной проект курса [Karpov.Courses специализации Start ML](https://karpov.courses/ml-start)

## Описание проекта

Цель - разработка сервиса рекомендательной системы постов в условной социальной сети.

### Исходные данные
##### Пользователи сервиса
| Поле      | Описание                        |
|-----------|---------------------------------|
| age       | Возраст                         |
| city      | Город                           |
| country   | Страна                          |
| exp_group | Экспериментальная группа        |
| gender    | Пол                             |
| id        | Идентификатор                   |
| os        | Операционная система устройства |
| source    | Источник трафика                |
Количество зарегистрированных пользователей: ~163 тысячи

##### Новостные посты
| Поле  | Описание      |
|-------|---------------|
| id    | Идентификатор |
| text  | Текст         |
| topic | Тематика      |
Количество постов: ~7 тысяч

##### Логи действий пользователей с постами ленты
| Поле      | Описание                                 |
|-----------|------------------------------------------|
| timestamp | Время                                    |
| user_id   | id пользователя                          |
| post_id   | id поста                                 |
| action    | Совершённое действие - просмотр или лайк |
| target    | Поставлен ли лайк **после** просмотра.   |
Количество записей: ~77 миллионов

### Описание задачи

Необходимо создать готовый к интеграции веб-сервис, возвращающий по запросу зарегистрированного
пользователя персонализированную ленту новостей.

Кроме того, необходимо подготовить инфраструктуру для проведения A\B-тестирования для определения изменения поведения
пользователей при выдаче ленты разными алгоритмами.

##### Параметры запроса
| Поле  | Описание                  |
|-------|---------------------------|
| id    | id пользователя           |
| time  | Время запроса             |
| limit | Количество постов в ленте |

##### Параметры отклика (одного поста из ленты)
| Поле  | Описание    |
|-------|-------------|
| id    | id поста    |
| text  | Текст поста |
| topic | Тема поста  |

##### Метрика
Оценка качества обученных алгоритмов будет замеряться по метрике hitrate@5 - есть ли хотя бы один лайк от пользователя в показанной ему ленте.

##### Технические требования и принятые допущения
1. Время отклика сервиса на 1 запрос не должен превышать 0.5 секунд.
2. Сервис не должен занимать более 2 Гб памяти системы.
3. Сервис должен иметь минимум две модели - обученную методом классического ML и с помощью DL.
4. Набор пользователей и постов фиксирован.
5. Временные рамки подаваемых сервису запросов ограничены предельными значениями в логах действий пользователей.

## Описание решения

### Анализ данных
При EDA препятствием стал размер датасета логов пользователей - 77 миллионов строк.
С целью уменьшить потребляемую память из базы данных было выгружено 10 миллионов записей.

### Выбор алгоритма решения
Так как сервис должен иметь возможность давать рекомендации двумя различными моделями, 
необходимо сделать и обосновать выбор.
1. Рекомендационную модель на методе классического ML решено обучать с помощью CatBoostClassifier.
Данный выбор обоснован табличной сущностью данных и необходимостью ранжировать полученные моделью предсказания.
Так как целевым показателем для нас является получение like хотя бы у одного поста в выдаче (согласно метрике hitrate@5),
ранжировать предсказания будем по вероятностям, полученными в классификаторе.
2. Вторая модель представляет собой тот же CatboostClassifier с обучением на полученных с помощью DL эмбеддингами
текстов постов. Эмбеддинги текстов получены с помощью эмбеддингов слов, в свою очередь
полученные с помощью предобученной Distil-Bert модели с дальнейшими преобразованиями.

### Реализация сервиса
Сервис реализован с помощью FastAPI в виде endpoint "ручки":
1. По POST запросу принимается запрос от пользователя на выдачу ленты.
2. Полученный в запросе JSON обрабатывается, все необходимые признаки приводятся к требуемой для модели форме.
3. Модель делает предсказания для каждого поста и выбираются с наибольшей вероятностью получающие лайк.
4. Сервис возвращает отклик со списком рекомендованных постов.

### Итоговые метрики
1. Среднее значение hitrate@5 в _экспериментальной_ группе: **0.605**
2. Среднее значение hitrate@5 в _контрольной_ группе: **0.552**

### Выводы

Реализован веб-сервис, по POST запросу id пользователя возвращающий ленту постов.
Подготовлена инфроструктура для проведения A\B-тестов

### Пути улучшения полученного результата
Ввиду различных допущений и ограничений, а так же учебного характера проекта, укажем возможные пути улучшения сервиса:
1. Обученные модели довольно просты. Не исследованы классические RecSys подходы (коллаборативная фильтрация и т.д.).
2. Возможна более тонкая настройка гиперпараметров использовавшегося CatboostClassifier.
3. Возможно улучшение сервиса по сбору и мониторингу метрик - времени отклика под нагрузкой, разделение пользователей по группам, список постов в ленте и т.д.

## Структура репозитория

```buildoutcfg
|   README.md
|   request-examples.txt    <- Примеры запросов к сервису
|   requirements.txt    <- Модули Python, необходимые для корректной работы проекта
|   app.py    <- Актуальная версия приложения с функционалом A/B-тестирования
|   schema.py    <- Схема, определяющая формат ответов сервиса
|
├───Dev_notebooks
│       Разработка_классический_ML_v1.ipynb     <- Ноутбук с процессом разработки модели на классическом ML
│       Разработка_DL_v2.ipynb      <- Ноутбук с процессом разработки модели с помощью DL
|       
├───Models
│       model_berted    <- DL-модель (она же "эксперементальная")
│       model_classic   <- ML-модель (она же "контрольная")
|
├───Previous_app_versions
│       app_berted.py   <- Версия приложения, работающая только с эксперементальной моделью
│       app_classic.py  <- АВерсия приложения, работающая только с контрольной моделью
|       schema.py    <- Схема, определяющая формат ответов сервиса
```

## Инструкция для запуска

`git clone https://github.com/ratseoff/Recommendation_system.git` <br />
`cd ./Recommendation_system` <br />
`python.exe -m pip install --upgrade pip` <br />
`pip install -r requirements.txt` <br />
`python.exe -m uvicorn app:app` <br />

Сервис доступен по http://127.0.0.1:8000/post/recommendations/
Примеры запросов [здесь](request_examples.txt).

# AI Journey 2019

Соревнование исскусственных интеллектов, автоматически решающих тесты Единого Государственного Экзамена.

## Постановка задачи

Необходимо разработать алгоритм, который получает на вход содержимое экзаменационного билета в электронном виде, в качестве результата предоставляет правильные ответы на вопросы из задач. В рамках соревнования, решения тестируются на тестах ЕГЭ по предмету "русский язык".

### Формат данных

Экзаменационный билет передается решению в формате JSON. Объект экзаменационного билета содержит поле `tasks` со списком заданий.

Объекты задания состоят из следующих полей:
- `id`: Идентификатор задания.
- `text`: Текст задания. Возможно использование markdown-style форматирования. Внутри текста могут содержаться ссылки на прикрепленные файлы, например — графические иллюстрации к заданию.
- `attachments`: Набор прикрепленных файлов (с указанием id, mime-type).
- `meta`: Метаинформация. Произвольные пары ключ-значение. Предназначено для указания структурированных данные о вопросе. Пример: категория или подтип вопроса, источник вопроса, предмет экзамена, из которого пришел вопрос.
- `answer`: Описание формата, в котором необходимо дать ответ. Допускаются разные типы ответов, каждый из которых имеет свои дополнительные параметры и поля:
    - `choice`: выбор одного варианта из списка
    - `multiple_choice`: выбор подмножества вариантов из списка
    - `order`: расстановка вариантов в правильном порядке
    - `matching`: верное соотнесение объектов из двух множеств
    - `text`: ответ в виде произвольного текста
- `score`: Максимальное количество баллов за задание.

[Подробные пояснения с примерами из ЕГЭ по русскому языку](ege_data_format.md).

Результат работы алгоритма записывается в виде объекта с одним полем `answers` — словарем, в котором ключи соответствуют идентификатору задания `id`, а значения — ответами на задание в соответствующем формате.

### Пример

Пример экзаменационного билета:
```json
{
  "tasks": [
    {
      "id": "literature_3",
      "meta": {
        "language": "ru",
        "source": "ege_literature"
      },
      "text": "Сопоставьте имена и фамилии классических русских писателей.",
      "attachments": [],
      "answer": {
        "type": "matching",
        "left": [
          {"id": "A", "text": "Пушкин"},
          {"id": "B", "text": "Тургенев"},
          {"id": "C", "text": "Набоков"}
        ],
        "choices": [
          {"id": "1", "text": "Иван"},
          {"id": "2", "text": "Петр"},
          {"id": "3", "text": "Александр"},
          {"id": "4", "text": "Владимир"},
        ]
      }
    },
    ...    
  ]
}
```

Пример объекта с ответами:
```json
{
  "answers": {
    "literature_3": {"A": "3", "B": "1", "C": "4"},
    ...
  }
}
```

### Тестовые данные

Участникам предоставляется тестовый набор вариантов ЕГЭ, собранных из открытых источников.

[Каталог с данными `data/check`](data/check)



## Формат решений

В проверяющую систему необходимо отправить код алгоритма, запакованный в ZIP-архив. Решения запускаются в изолированном окружении при помощи [Docker](https://www.docker.com/). Время и ресурсы во время тестирования ограничены. Участнику нет необходимости разбираться с технологией Docker.

### Содержимое контейнера

В корне архива обязательно должен быть файл `metadata.json` следующего содержания:

```json
{
    "image": "sberbank/python",
    "entry_point": "python server.py"
}
```

Здесь `image` — поле с названием docker-образа, в котором будет запускаться решение, `entry_point` — команда, при помощи которой запускается решение. Для решения текущей директорией будет являться корень архива.

Для запуска решений можно использовать существующие окружения:

- `sberbank/python` — Python3 с установленным большим набором библиотек ([подробнее](images/sberbank-python))
- `gcc` - для запуска компилируемых C/C++ решений (подробнее здесь)
- `node` — для запуска JavaScript
- `openjdk` — для Java
- `mono` — для C#

Подойдет любой другой образ, доступный для загрузки из [DockerHub](http://dockerhub.com). При необходимости, Вы можете подготовить свой образ, добавить в него необходимое ПО и библиотеки (см. [инструкцию по созданию Docker-образов](https://docs.docker.com/engine/reference/builder/)); для использования его необходимо будет опубликовать на DockerHub.


### Программный интерфейс

Решение должно быть выполнено в виде HTTP-сервера, доступного по порту `8000`, который отвечает на два вида запросов:

#### `GET /ready`

На запрос необходимо ответить кодом `200 OK` в случае, если решение готово к работе. Любой другой код означает, что решение еще не готово. У алгоритма есть ограниченное время на подготовку к работе, за которое можно прочитать данные с диска, создать в оперативной памяти необходимые структуры данных, загрузить модели машинного обучения.

#### `POST /take_exam`

Запрос на решение экзаменационного билета. Тело запроса — JSON объект экзаменационного билета в описанном выше формате.

В качестве ответа необходимо вернуть JSON-объект с ответами на задания.  

Запрос и ответ должны иметь `Content-Type: application/json`. Рекомендуется использовать кодировку UTF-8.

В процессе тестирования решения, запрос `/take_exam` может выполняться последовательно множество раз. Гарантируется, что одновременно решение может получить не более одного запроса `/take_exam`. Т.е. до тех пор, пока решение не ответит на текущий экзаменационный билет, новых запросов поступать не будет. Время выполнения запроса ограничено.


### Ограничения

Контейнер с решением запускается в следующих условиях:

- решению доступны ресурсы
  - **16 Гб** оперативной памяти
  - 4 vCPU
- решение не имеет доступа к ресурсам интернета
- время на подготовку к работе: 120 секунд (после чего на запрос `/ready` необходимо отвечать кодом 200)
- время на один запрос `/take_exam`: 600 секунд
- решение должно принимать HTTP запросы с внешних машин (не только localhost/127.0.0.1)
- во время тестирования запросы производятся последовательно (не более 1 запроса одновременно)
- максимальный размер упакованного и распакованного архива с решением: **20 Гб**
- максимальный размер используемого Docker-образа: **20 Гб**

### Пример запуска решения

Запускаем решение [`base-python`](examples/base-python):

```bash
$ python3 server.py
 * Serving Flask app "server" (lazy loading)
 * Environment: production
 * Debug mode: off
```

При помощи Python 3 и библиотеки [`requests`](http://docs.python-requests.org/en/master/) делаем запросы.

```python
>>> import json
>>> import requests
>>> with open('data/check/test_00.json') as fin:
...    exam_ticket = json.load(fin)
>>> requests.get('http://localhost:8000/ready')
<Response [200]>
>>> resp = requests.post('http://localhost:8000/take_exam', json=exam_ticket)
>>> resp
<Response [200]>
>>> resp.json()
{'answers': {
  '1': ['3', '2', '4', '1', '5'],
  '2': 'стоять',
  ...
}}
```

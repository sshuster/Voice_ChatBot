# Voice ChatBot

Проект состоит из двух частей - бот и RESTful сервер для взаимодействия с ним.

Полный список всех необходимых для работы пакетов:
1. Для Python3.5: decorator, Flask v1.0.2, Flask-HTTPAuth v3.2.4, gensim, gevent v1.3.7, h5py, Keras v2.2.4, matplotlib, numpy, pocketsphinx, pydub, simpleaudio, [recurrentshop](https://github.com/datalogai/recurrentshop), requests, [seq2seq](https://github.com/farizrahman4u/seq2seq), tensorflow[-gpu].
2. Для Ubuntu: ffmpeg, libav-tools, make, git, scons, gcc, pkg-config, pulseaudio, libpulse-dev, portaudio19-dev, libglibmm-2.4-dev, libasound-dev, libao4, libao-dev, sonic, sox, swig.

Если вы используете Ubuntu 16.04 или выше, для установки всех пакетов можно воспользоваться `install_packages.sh` (проверено в Ubuntu 16.04). По умолчанию будет установлен [TensorFlow](https://www.tensorflow.org/install/) для CPU. Если у вас есть видеокарта nvidia, вы можете установить [TensorFlowGPU](https://www.tensorflow.org/install/gpu). Для этого необходимо при запуске `install_packages.sh` передать параметр `gpu`. Например:
```
./install_packages.sh gpu
```

Если вы не можете или не хотите воспользоваться скриптом для установки всех необходимых пакетов, нужно вручную установить RHVoice и CMUclmtk, используя инструкции в `Install RHVoice.txt` и `Install CMUclmtk.txt`. Так же необходимо скопировать файлы языковой, акустической модели и словаря для PocketSphinx из `temp/` в `/usr/local/lib/python3.5/dist-packages/pocketsphinx/model` (у вас путь к `python3.5` будет отличаться). Файлы языковой модели `prepared_questions.lm` и словаря `prepared_questions.dic` необходимо переименовать в `ru_bot.lm` и `ru_bot.dic` (либо заменить их название в `speech_to_text.py`).

# Бот

Основа бота - рекуррентная нейронная сеть, а именно модель AttentionSeq2Seq. В текущей реализации она состоит из 2 двунаправленных LSTM ячеек в кодировщике, слоя внимания и 2 LSTM ячеек в декодировщике. Использование модели внимания позволяет установить "мягкое" соответствие между входными и выходными последовательностями, что повышает качество и производительность. Размерность входа в последней конфигурации равна 500 и длина последовательности 26 (т.е. максимальная длина предложений в обучающей выборке). Слова переводятся в вектора с помощью кодировщика word2vec из бибилотеки gensim. Модель seq2seq реализована с помощью Keras и RecurrentShop. Обученная модель seq2seq (веса которой находятся в `data/net_final_weights.h5`) с параметрами, которые указаны в исходных файлах, имеет точность 98.5% (т.е. бот ответит на 1577 из 1601 вопросов правильно). 

Бот умеет работать в нескольких режимах:
1. Обучение модели seq2seq.
2. Работа с обученной моделью seq2seq в текстовом режиме.
3. Работа с обученной моделью seq2seq с озвучиванием ответов с помощью [RHVoice](https://github.com/Olga-Yakovleva/RHVoice).
4. Работа с обученной моделью seq2seq с распознаванием речи с помощью [PocketSphinx](https://github.com/cmusphinx/pocketsphinx).
5. Работа с обученной моделью seq2seq с озвучиванием ответов и распознаванием речи.

---

**1. Обучение модели seq2seq.**

Обучающая выборка состоит из 1600 пар вопрос %% ответ, взятых из различных русских пьес. Она хранится в файле `data/soruce_data.txt`. Каждая пара вопрос %% ответ пишется с новой строки, т.е. на одной строке только одна пара. 

Процесс обучения состоит из нескольких этапов:

1. Подготовка обучающей выборки.

Для подготовки обучающей выборки предназначен модуль `preprocessing.py`, состоящий из класса `Preparation`. Данный класс считывает обучающую выборку из файла, разделяет вопросы и ответы, удаляет неподдерживаемые символы и знаки препинания, преобразует полученные вопросы и ответы в последовательности фиксированного размера (с помощью слов-наполнителей `<PAD>`). Так же этот класс осуществляет подготовку вопросов к сети и обработку её ответов. Например:

Вход: `"Зачем нужен этот класс? %% Для подготовки данных"`

Выход: `[['<PAD>', ..., '<PAD>', '?', 'класс', 'этот', 'нужен', 'Зачем', '<GO>'], ['Для', 'подготовки', 'данных', '<EOS>', '<PAD>', ..., '<PAD>']]`

Обучающая выборка считывается из файла `data/soruce_data.txt`, преобразованные пары [вопрос,ответ] сохраняются в файл `data/prepared_data.pkl` (и дублируются в `data/prepared_data.txt` для возможности простого просмотра этих данных). Так же при этом строится гистограмма размеров вопросов и ответов, которая сохраняется в `data/features_hist.png`.

2. Перевод слов в вещественные вектора.

За этот этап отвечает модуль `coder_w2v.py`, состоящий из класса `CoderW2V`. Данный класс кодирует последовательности фиксированного размера (т.е. наши вопросы и ответы) в вещественные вектора. Использутся кодировщик word2vec из библиотеки gensim. В классе реализованы методы для кодирования сразу всех пар [вопрос,ответ] из обучающей выборки в вектора, а так же для кодирования вопроса к сети и декодирования её ответа. Например:

Вход: `[['<PAD>', ..., '<PAD>', '?', 'класс', 'этот', 'нужен', 'Зачем', '<GO>'], ['Для', 'кодирования', 'предложений', '<EOS>', '<PAD>', ..., '<PAD>']]`

Выход: `[[[0.43271607, 0.52814275, 0.6504923, ...], [0.43271607, 0.52814275, 0.6504923, ...], ...], [[0.5464854, 1.01612, 0.15063584, ...], [0.88263285, 0.62758327, 0.6659863, ...], ...]]` (т.е. каждое слово кодируется вектором с длинной 500 (это значение можно изменить, аргумент `size` в методе `words2vec()`))

Пары [вопрос,ответ] считываются из файла `data/prepared_data.pkl` (который был получен на предыдущем этапе), закодированные пары сохраняются в файл `data/encoded_data.npz`. Так же в процессе работы строится список всех используемых слов, т.е. словарь, который сохраняется в файле `data/w2v_vocabulary.txt`, осуществляется поиск "слов-соседей" (т.е. близких по смыслу, по мнению кодировщика word2vec), которые затем сохраняются в `data/w2v_neighborhood.txt`. Также сохраняется обученная модель word2vec в `data/w2v_model.bin`.

3. Обучение сети.

На этом этапе выполняется обучение модели seq2seq на уже подготовленных данных. За это отвечает модуль `training.py`, состоящий из класса `Training`. Данный класс осуществляет обучение сети, сохранение модели сети и весовых коэффициентов. 

Для обучение необходим файл `data/encoded_data.npz`, содержащий пары [вопрос,ответ], закодированные в вектора, которые были получены на предыдущем этапе. В процессе обучения после каждой 5-ой эпохи (это значение можно изменить) сохраняется промежуточный результат обучения сети в файл `data/net_[номер_итерации]_weights.h5`, а на последней итерации в файл `data/net_final_weights.h5` (итерация - один цикл обучения сети, некоторое число эпох, после которых происходит сохранение весов в файл и можно например оценить точность работы сети или вывести другие её параметры. В данном случае число эпох равно 5, а общее число итераций 150). Модель сети (а точнее значение аргументов для функции `SimpleSeq2Seq(...)`) сохраняется в файле `data/net_model.txt`.

4. Оценка качества обучения.

Это необязательный этап, можно оценивать обученность сети и вручную. За этот этап отвечает метод `assessment_training_accuracy()` класса `Prediction` из модуля `prediction.py`. Данный метод подаёт на вход сети каждый вопрос из обучающей выборки (которые берёт из файла `data/encoded_data.npz`) и сравнивает ответ сети с соответствующим ответом из обучающей выборки. В конце выводит неправильные ответы (если их меньше 50) и полученную точность работы сети в процентах.

5. Построение языковой модели и словаря для PocketSphinx.

Этот этап нужен в случае, если будет использоваться распознавание речи. На этом этапе осуществляется создание статической языковой модели и фонетического словаря для PocketSphinx на основе вопросов из обучающей выборки. Для этого используется функция `building_language_model()` (которая обращается к `text2wfreq, wfreq2vocab, text2idngram` и `idngram2lm` из CMUclmtk_v0.7) из модуля `preparing_speech_to_text.py`. Данный метод использует данные из файла `data/soruce_data.txt`, сохраняет языковую модель в файл `temp/prepared_questions.lm`, а словарь в `temp/prepared_questions.dic`. В конце работы языковая модель и словарь будут скопированы в `/usr/local/lib/python3.5/dist-packages/pocketsphinx/model` (если используется не Ubuntu, то данный путь необходимо исправить на актуальный для вашей ОС) с именами `ru_bot.lm` и `ru_bot.dic` (потребуется ввод пароля root-пользователя).

Данный режим (обучение модели seq2seq) реализован в функции `train()` модуля `bot.py`. Для запуска бота в режиме обучения можно выполнить `train.sh` либо запустить `bot.py` с параметром `train`. Например, так:
```
python3 bot.py train
```

Так же можно просто запустить `bot.py` и в предложенном меню выбрать режим 1.

---

**2. Работа с обученной моделью seq2seq в текстовом режиме.**

Для взаимодействия с обученной моделью seq2seq предназначена функция `predict()` модуля `bot.py`. Данная функция поддерживает несколько режимов работы. В текстовом режиме, т.е. когда пользователь вводит вопрос с клавиатуры и сеть отвечает текстом, используется только метод `predict()` класса `Prediction` из модуля `prediction.py`. Данный метод принимает строку с вопросом к сети и возвращает строку с ответом сети. Для работы необходимы: файл `data/net_model.txt` с параметрами модели сети, файл `data/net_final_weights.h5` с весами обученной сети и файл `data/w2v_model.bin` с обученной моделью word2vec.

Для запуска бота в данном режиме можно выполнить `run.sh` либо запустить `bot.py` с параметром `predict`. Например, так:
```
python3 bot.py predict
```

Так же можно просто запустить `bot.py` и в предложенном меню выбрать режим 2.

---

**3. Работа с обученной моделью seq2seq с озвучиванием ответов с помощью [RHVoice](https://github.com/Olga-Yakovleva/RHVoice).**

Данный режим отличается от предыдущего тем, что функции `predict()` модуля `bot.py` передаётся параметр `for_speech_synthesis = True`. Это означает, что взаимодействие с сетью будет проходить так же, как и в режиме 2, но ответ сети дополнительно будет озвучиваться. 

Озвучивание ответов, т.е. синтез речи, реализован в функции `tts()` модуля `text_to_speech.py`. Данная функция требует установленного RHVoice-client и с помощью аргументов командной строки передаёт ему необходимые параметры для синтеза речи (об установке RHVoice и примеры обращения к RHVoice-client можно посмотреть в `Install RHVoice.txt`). Функция принимает на вход строку, которую нужно преобразовать в речь, требуемый режим работы и, если требуется, имя .wav файла, в который будет сохранена синтезированная речь. Данная функция может работать в двух режимах: `into_file` - запись синтезированной речи в файл `temp/answer.wav` (либо тот, что указан в аргументах функции) с частотой дискретизации 32 кГц и глубиной 16 бит, моно; и `playback` - воспроизведение речи сразу после синтезирования (при этом синтезированная речь сохраняется в файл `temp/answer.wav` с параметрами, как в `into_file`, для упрощения воспроизведения).

Для запуска бота в данном режиме нужно запустить `bot.py` с параметрами `predict -ss`. Например, так:
```
python3 bot.py predict -ss
```

Так же можно просто запустить `bot.py` и в предложенном меню выбрать режим 3.

---

**4. Работа с обученной моделью seq2seq с распознаванием речи с помощью [PocketSphinx](https://github.com/cmusphinx/pocketsphinx).**

Для работы в этом режиме необходимо функции `predict()` модуля `bot.py` передать параметр `for_speech_recognition = True`. Это означает, что взаимодействие с сетью, а точнее ввод вопросов, будет осуществляться с помощью голоса.

Распознавание речи реализовано в методе `stt()` класса `SpeechRecognition` модуля `speech_to_text.py`. Данный класс использует PocketSphinx и языковую модель со словарём (`ru_bot.lm` и `ru_bot.dic`), которые были построены в режиме обучения сети. Метод `stt()` может работать в двух режимах: `from_file` - распознавание речи из .wav или .opus файла с частотой дискретизации >=16кГц, 16bit, моно (имя файла передаётся в качестве аргумента функции) и `from_microphone` - распознавание речи с микрофона. Режим работы задаётся при создании экземпляра класса `SpeechRecognition`, т.к. загрузка языковой модели занимает некоторое время (чем больше модель, тем дольше она загружается).

Для запуска бота в данном режиме нужно запустить `bot.py` с параметрами `predict -sr`. Например, так:
```
python3 bot.py predict -sr
```

Так же можно просто запустить `bot.py` и в предложенном меню выбрать режим 4.

---

**5. Работа с обученной моделью seq2seq с озвучиванием ответов и распознаванием речи.**

Это комбинация режимов 3 и 4.

Для работы в этом режиме необходимо функции `predict()` модуля `bot.py` передать параметры `for_speech_recognition = True` и `for_speech_synthesis = True`. Это означает, что ввод вопросов будет осуществляться с помощью голоса, а ответы сети будут озвучиваться. Описание используемых модулей можно найти в описании режимов 3 и 4.

Для запуска бота в данном режиме можно выполнить `run_with_tts_stt.sh` либо запустить `bot.py` с параметрами `predict -ss -sr`. Например, так:
```
python3 bot.py predict -sr -ss
```
или
```
python3 bot.py predict -ss -sr
```

Так же можно просто запустить `bot.py` и в предложенном меню выбрать режим 5.



# RESTful-сервер

Данный сервер предоставляет REST-api для взаимодействия с ботом. Сервер реализован с помощью [Flask](http://flask.pocoo.org/), а многопоточный режим (production-версия) с помощью [gevent.pywsgi.WSGIServer](http://flask.pocoo.org/docs/1.0/deploying/wsgi-standalone/). Также сервер имеет ограничение на размер принимаемых данных в теле запроса равное 16 Мб. Реализация находится в модуле `rest_server.py`.

Запустить WSGI сервер можно выполнив `run_rest_server.sh` (запуск без аргументов командной строки).

Сервер поддерживает аргументы командной строки, которые немного упрощают его запуск. Аргументы имеют следующую структуру: `[ключ(-и)] [адрес:порт]`.

Возможные ключи:
1. `-d` - запуск тестового Flask сервера (если ключ не указывать - будет запущен WSGI сервер)
2. `-s` - запуск сервера с поддержкой https (используется самоподписанный сертификат, получен с помощью openssl)

Допустимые варианты `адрес:порт`:
1. `host:port` - запуск на указанном `host` и `port`
2. `localaddr:port` - запуск с автоопределением адреса машины в локальной сети и указанным `port`
3. `host:0` или `localaddr:0` - если `port = 0`, то будет выбран любой доступный порт автоматически

Список возможных комбинаций аргументов командной строки и их описание: 
1. без аргументов - запуск WSGI сервера с автоопределением адреса машины в локальной сети и портом `5000`. Например: ```python3 rest_server.py```
2. `host:port` - запуск WSGI сервера на указанном `host` и `port`. Например: ```python3 rest_server.py 192.168.2.102:5000```
3. `-d` - запуск тестового Flask сервера на `127.0.0.1:5000`. Например: ```python3 rest_server.py -d```
4. `-d host:port` - запуск тестового Flask сервера на указанном `host` и `port`. Например: ```python3 rest_server.py -d 192.168.2.102:5000```
5. `-d localaddr:port` - запуск тестового Flask сервера с автоопределением адреса машины в локальной сети и портом `port`. Например: ```python3 rest_server.py -d localaddr:5000```
6. `-s` - запуск WSGI сервера с поддержкой https, автоопределением адреса машины в локальной сети и портом `5000`. Например: ```python3 rest_server.py -s```
7. `-s host:port` - запуск WSGI сервера с поддержкой https на указанном `host` и `port`. Например: ```python3 rest_server.py -s 192.168.2.102:5000```
8. `-s -d` - запуск тестового Flask сервера с поддержкой https на `127.0.0.1:5000`. Например: ```python3 rest_server.py -s -d```
9. `-s -d host:port` - запуск тестового Flask сервера с поддержкой https на указанном `host` и `port`. Например: ```python3 rest_server.py -s -d 192.168.2.102:5000```
10. `-s -d localaddr:port` - запуск тестового Flask сервера с поддержкой https, автоопределением адреса машины в локальной сети и портом `port`. Например: ```python3 rest_server.py -s -d localaddr:5000```

Сервер может сам выбрать доступный порт, для этого нужно указать в `host:port` или `localaddr:port` порт `0` (например: ```python3 rest_server.py -d localaddr:0```).

Всего поддерживается 5 запросов:
1. GET-запрос на `/chatbot/about`, вернёт информацию о проекте
2. GET-запрос на `/chatbot/questions`, вернёт список всех поддерживаемых вопросов
3. POST-запрос на `/chatbot/speech-to-text`, принимает .wav/.opus-файл и возвращает распознанную строку
4. POST-запрос на `/chatbot/text-to-speech`, принимает строку и возвращает .wav-файл с синтезированной речью
5. POST-запрос на `/chatbot/text-to-text`, принимает строку и возвращает ответ бота в виде строки

---

**Описание сервера**

1. Сервер имеет базовую http-авторизацию. Т.е. для получения доступа к серверу надо в каждом запросе добавить заголовок, содержащий логин:пароль, закодированный с помощью `base64` (логин: `testbot`, пароль: `test`). Пример на python:
```
import requests
import base64

auth = base64.b64encode('testbot:test'.encode())
headers = {'Authorization' : "Basic " + auth.decode()}
```

Выглядеть это будет так:
```
Authorization: Basic dGVzdGJvdDp0ZXN0
```

2. В запросе на распознавание речи (который под номером 3) сервер ожидает .wav или .opus файл (>=16кГц 16бит моно) с записанной речью, который так же передаётся в json с помощью кодировки `base64` (т.е. открывается .wav/.opus-файл, читается в массив байт, потом кодирутеся `base64`, полученный массив декодируется из байтовой формы в строку `utf-8` и помещается в json), в python это выглядит так:
```
# Формирование запроса
auth = base64.b64encode('testbot:test'.encode())
headers = {'Authorization' : "Basic " + auth.decode()}

with open('test.wav', 'rb') as audio:
    data = audio.read()
data = base64.b64encode(data)
data = {'wav' : data.decode()}

# Отправка запроса серверу
r = requests.post('http://' + addr + '/chatbot/speech-to-text', headers=headers, json=data)

# Разбор ответа
data = r.json()
data = data.get('text')
print(data)
```

3. В запросе на синтез речи (который под номером 4) сервер пришлёт в ответе json с .wav-файлом (16бит 32кГц моно) с синтезированной речью, который был закодирован так, как описано выше (что бы обратно его декодировать нужно из json получить нужную строку в массив байт, потом декодировать его с помощью `base64` и записать в файл или поток, что бы потом воспроизвести), пример на python:
```
# Формирование запроса
auth = base64.b64encode('testbot:test'.encode())
headers = {'Authorization' : "Basic " + auth.decode()}
data = {'text':'который час'}

# Отправка запроса серверу
r = requests.post('http://' + addr + '/chatbot/text-to-speech', headers=headers, json=data)

# Разбор ответа
data = r.json()
data = base64.b64decode(data.get('wav'))
with open('/home/vladislav/Проекты/Voice chat bot/temp/answer.wav', 'wb') as audio:
    audio.write(data)
```

---

**Передаваемые данные в каждом запросе**

Все передаваемые данные обёрнуты в json (в том числе и ответы с ошибками).
1. Сервер передаёт клиенту:
```
{
"about" : "тут будет информация о проекте"
}
```
2. Сервер передаёт клиенту:
```
{
"questions" : ["Вопрос 1",
               "Вопрос 2",
               "Вопрос 3"]
}
```
3. Клиент в теле запроса должен отправить:
```
{
"wav" : "UklGRuTkAABXQVZFZm10IBAAAAABAAEAAH..."
}
```
или
```
{
"opus" : "ZFZm10IBUklQVZFZm10IBARLASBAAEOpH..."
}
```
Сервер ему передаст:
```
{
"text" : "который час"
}
```
4. Клиент в теле запроса должен отправить:
```
{
"text" : "который час"
}
```
Сервер ему передаст:
```
{
"wav" : "UklGRuTkAABXQVZFZm10IBAAAAABAAEAAH..."
}
```
5. Клиент в теле запроса должен отправить:
```
{
"text" : "прощай"
}
```
Сервер ему передаст:
```
{
"text" : "это снова я"
}
```

---

**Примеры запросов**

1. GET-запрос на `/chatbot/about`

Пример запроса, который формирует `python-requests`:
```
GET /chatbot/about HTTP/1.1
Host: 192.168.2.83:5000
Connection: keep-alive
Accept-Encoding: gzip, deflate
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: python-requests/2.9.1
```

Пример запроса, который формирует curl (`curl -v -u testbot:test -i http://192.168.2.83:5000/chatbot/about`):
```
GET /chatbot/about HTTP/1.1
Host: 192.168.2.83:5000
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: curl/7.47.0
```

В обоих случаях сервер ответил:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 305
Date: Fri, 02 Nov 2018 15:13:21 GMT

{
"about" : "Информация о проекте."
}
```

---

2. GET-запрос на `/chatbot/questions`

Пример запроса, который формирует `python-requests`:
```
GET /chatbot/questions HTTP/1.1
Host: 192.168.2.83:5000
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: python-requests/2.9.1
Connection: keep-alive
Accept-Encoding: gzip, deflate
```

Пример запроса, который формирует curl (`curl -v -u testbot:test -i http://192.168.2.83:5000/chatbot/questions`):
```
GET /chatbot/questions HTTP/1.1
Host: 192.168.2.83:5000
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: curl/7.47.0
```

В обоих случаях сервер ответил:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1086
Date: Fri, 02 Nov 2018 15:43:06 GMT

{
"questions" : ["Что случилось?",
               "Срочно нужна твоя помощь.",
               "Ты уезжаешь?",
               ...]
}
```

---

3. POST-запрос на `/chatbot/speech-to-text`

Пример запроса, который формирует `python-requests`:
```
POST /chatbot/speech-to-text HTTP/1.1
Host: 192.168.2.83:5000
User-Agent: python-requests/2.9.1
Accept: */*
Content-Length: 10739
Connection: keep-alive
Content-Type: application/json
Authorization: Basic dGVzdGJvdDp0ZXN0
Accept-Encoding: gzip, deflate

{
"wav" : "UklGRuTkAABXQVZFZm10IBAAAAABAAEAAH..."
}
```

Пример запроса, который формирует curl (`curl -v -u testbot:test -i -H "Content-Type: application/json" -X POST -d '{"wav":"UklGRuTkAABXQVZFZm10IBAAAAABAAEAAH..."}' http://192.168.2.83:5000/chatbot/speech-to-text`):
```
POST /chatbot/speech-to-text HTTP/1.1
Host: 192.168.2.83:5000
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: curl/7.47.0
Accept: */*
Content-Type: application/json
Content-Length: 10739

{
"wav" : "UklGRuTkAABXQVZFZm10IBAAAAABAAEAAH..."
}
```

Сервер ответил:
```
HTTP/1.1 200 OK
Content-Length: 81
Date: Fri, 02 Nov 2018 15:57:13 GMT
Content-Type: application/json

{
"text" : "Распознные слова из аудиозаписи"
}
```

---

4. POST-запрос на `/chatbot/text-to-speech`

Пример запроса, который формирует `python-requests`:
```
POST /chatbot/text-to-speech HTTP/1.1
Host: 192.168.2.83:5000
Connection: keep-alive
Accept: */*
User-Agent: python-requests/2.9.1
Accept-Encoding: gzip, deflate
Content-Type: application/json
Content-Length: 73
Authorization: Basic dGVzdGJvdDp0ZXN0

{
"text" : "который час"
}
```

Пример запроса, который формирует curl (`curl -v -u testbot:test -i -H "Content-Type: application/json" -X POST -d '{"text":"который час"}' http://192.168.2.83:5000/chatbot/text-to-speech`):
```
POST /chatbot/text-to-speech HTTP/1.1
Host: 192.168.2.83:5000
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: curl/7.47.0
Accept: */*
Content-Type: application/json
Content-Length: 32

{
"text" : "который час"
}
```

Сервер ответил:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 78151
Date: Fri, 02 Nov 2018 16:36:02 GMT

{
"wav" : "UklGRuTkAABXQVZFZm10IBAAAAABAAEAAH..."
}
```

---

5. POST-запрос на `/chatbot/text-to-text`

Пример запроса, который формирует `python-requests`:
```
POST /chatbot/text-to-text HTTP/1.1
Host: 192.168.2.83:5000
Accept-Encoding: gzip, deflate
Content-Type: application/json
User-Agent: python-requests/2.9.1
Connection: keep-alive
Content-Length: 48
Accept: */*
Authorization: Basic dGVzdGJvdDp0ZXN0

{
"text" : "прощай"
}
```

Пример запроса, который формирует curl (`curl -v -u testbot:test -i -H "Content-Type: application/json" -X POST -d '{"text":"прощай"}' http://192.168.2.83:5000/chatbot/text-to-text`):
```
POST /chatbot/text-to-text HTTP/1.1
Host: 192.168.2.83:5000
Authorization: Basic dGVzdGJvdDp0ZXN0
User-Agent: curl/7.47.0
Accept: */*
Content-Type: application/json
Content-Length: 23

{
"text" : "прощай"
}
```

Сервер ответил:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 68
Date: Fri, 02 Nov 2018 16:41:22 GMT

{
"text" : "это снова я"
}
```

---

**Предполагаемый алгоритм работы с сервером**

1. Запросить список вопросов у сервера (запрос 2) и отобразить его
2. В зависимости от выбранного режима:
* Записать речь с микрофона клиента
* Отправить на сервер (запрос 3) и получить ответ с распознанным текстом
* Отобразить текст в поле ввода
* Отправить текст на сервер (запрос 5) и получить ответ бота
* Отправить ответ бота на сервер (запрос 4) и получить аудиофайл с синтезированной речью
* Воспроизвести аудиофайл
3. Если клиент хочет узнать информацию о данном проекте, послать запрос 1 на сервер и отобразить полученные данные

Если у вас возникнут вопросы или вы хотите сотрудничать, можете написать мне на почту: vladsklim@gmail.com

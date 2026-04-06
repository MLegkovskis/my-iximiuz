# Weather Bot for Telegram

## Introduction

К огромному сожалению с ресурсом для мониторинга качества воздуха aqicn.org случился РОСКОМНАДЗОР и теперь он не доступен на территории РФ.

Я переделал кодовую базу для работы с сайтом https://iqair.com/ и его API. В README используются мой действующий токен, однако Вам имеет смысл создать свой: 
1. Перейдите по ссылке https://dashboard.iqair.com/personal/api-keys и зарегистрируйтесь.
2. Перейдите в раздел __Air quality API__, нажмите __Get a plan__ 
3. В разделе __Community__ нажмите __Subscribe now__.

Полученный токен нужно использовать в качестве переменной окружения `IQAIR_TOKEN`.

Также обратите внимание, для сервиса aqicn.org мы указывали только наименование города, однако для сервиса iqair.com необходимо указывать страну, регион и город. 

Чтобы не было путаницы, Docker образы новой версии, работающей с iqair.com, будут начинаться со второй мажорной версии: `2.*.*`

## Description

Service runs 2 clients: 
- telegram client: telegram bot, which handle commands, render templates and receive air quality info;
- http client: rest-endpoint `/air`, which receives __raw__ data from www.iqair.com API.

## Build and Run Docker

Build:
```bash
docker build . -t marilee/weather-bot:0.0.1

```

Run:
```bash
docker run -it -e TG_TOKEN="8453599350:AAHy6IJ_UTriaEE0dbcUMJ7vi70Rbgf_dys" -e IQAIR_TOKEN="9f4a4fc0-df7f-4aca-b7f6-1809980397cf" -p 8080:8080 -p 8086:8086 marilee/weather-bot:0.0.1

docker run -it -e TG_TOKEN="8453599350:AAHy6IJ_UTriaEE0dbcUMJ7vi70Rbgf_dys" -e IQAIR_TOKEN="9f4a4fc0-df7f-4aca-b7f6-1809980397cf" -p 8080:8080 -p 8086:8086 marilee/weather-bot:0.0.1
```


## Configuration

Environment variables:
| Name | Type | Destription |
| ---  |  --- |      ---    |
| TG_URL | string | no need now |
| TG_TOKEN | string | telegram token from https://t.me/BotFather |
| TEMPLATE_PATH | string | path to message template |
| IQAIR_URL |string | API url (http://api.airvisual.com/v2/) |
| IqairToken |string| Obtain token here: https://dashboard.iqair.com/personal/api-keys |
| COUNTRY | string | state on country (Russia by default) |
| STATE | string | state of the country (Moscow by default) |
| CITY| string | city in the state (Moscow by default)  |
| HTTP_PORT | int    | port, where HTTP server launch | 
| LOG_LEVEL | string | info/warn/debug/trace, default:"info"|

Supported countries: 
```sh
curl --location 'http://api.airvisual.com/v2/countries?key=9f4a4fc0-df7f-4aca-b7f6-1809980397cf'
```

```
Russia
Norway
Belarus
Kazakhstan
Kyrgyzstan
...
```

Supported states: 
```sh
curl --location 'http://api.airvisual.com/v2/states?country=Russia&key=9f4a4fc0-df7f-4aca-b7f6-1809980397cf'
```

```
Moscow
North Ossetia
Novosibirsk
...
```

Supported cities: 
```sh
curl --location 'http://api.airvisual.com/v2/cities?state=Moscow&country=Russia&key=9f4a4fc0-df7f-4aca-b7f6-1809980397cf'
```

```
Moscow
```

# 
> TODO: 
1. More commands
2. CI/CD
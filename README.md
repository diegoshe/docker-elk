# Elastic stack (ELK) on Docker

[![Join the chat at https://gitter.im/deviantony/docker-elk](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/deviantony/docker-elk?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Elastic Stack version](https://img.shields.io/badge/ELK-7.6.2-blue.svg?style=flat)](https://github.com/deviantony/docker-elk/issues/483)
[![Build Status](https://api.travis-ci.org/deviantony/docker-elk.svg?branch=master)](https://travis-ci.org/deviantony/docker-elk)

Запустите последнюю версию [Elastic stack][elk-stack] с Docker и Docker Compose.

Это дает вам возможность анализировать любой набор данных, используя возможности поиска / агрегирования Elasticsearch и возможности визуализации Kibana.

> :information_source: Образы Docker, поддерживающие этот стек, включают [Stack Features][stack-features] (formerly X-Pack)
с [платными функциями][paid-features] включенными по умолчанию (см. [Как отключить платные функции](#how-to-disable-paid-features) чтобы отключить их). [Пробная лицензия][trial-license] действительна в течение 30 дней.

Основано на официальных образах Docker - Elastic:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/master/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/master/docker)
* [Kibana](https://github.com/elastic/kibana/tree/master/src/dev/build/tasks/os_packages/docker_generator)

Другие доступные варианты стека:

* [`searchguard`](https://github.com/deviantony/docker-elk/tree/searchguard): Search Guard support

## Содержание

1. [Требования](#requirements)
   * [Настройка хоста](#host-setup)
   * [SELinux](#selinux)
   * [Docker for Desktop](#docker-for-desktop)
     * [Windows](#windows)
     * [macOS](#macos)
2. [Использование](#usage)
   * [Выбор версии](#version-selection)
   * [Поднятие стека](#bringing-up-the-stack)
   * [Cleanup](#cleanup)
   * [Начальная настройка](#initial-setup)
     * [Настройка аутентификации пользователя](#setting-up-user-authentication)
     * [Внедрение данных](#injecting-data)
     * [Создание шаблона индекса Kibana по умолчанию](#default-kibana-index-pattern-creation)
3. [Конфигурации](#configuration)
   * [Как настроить Elasticsearch](#how-to-configure-elasticsearch)
   * [Как настроить Kibana](#how-to-configure-kibana)
   * [Как настроить Logstash](#how-to-configure-logstash)
   * [Как отключить платные функции](#how-to-disable-paid-features)
   * [Как масштабировать кластер Elasticsearch](#how-to-scale-out-the-elasticsearch-cluster)
4. [Расширения](#extensibility)
   * [Как добавить плагины](#how-to-add-plugins)
   * [Как включить предоставленные расширения](#how-to-enable-the-provided-extensions)
5. [JVM настройка](#jvm-tuning)
   * [Как указать объем памяти, используемой сервисом](#how-to-specify-the-amount-of-memory-used-by-a-service)
   * [Как включить удаленное соединение JMX с сервисом](#how-to-enable-a-remote-jmx-connection-to-a-service)
6. [Идем дальше](#going-further)
   * [Плагины и интеграции](#plugins-and-integrations)
   * [Swarm mode](#swarm-mode)

## Requirements

### Настройка хоста

* [Docker Engine](https://docs.docker.com/install/) version **17.05** or newer
* [Docker Compose](https://docs.docker.com/compose/install/) version **1.20.0** or newer
* 1.5 GB of RAM

> :information_source: особенно в Linux, убедитесь, что у вашего пользователя есть [необходимые разрешения][linux-postinstall] для
> взаимодействия с Docker daemon.

По умолчанию в стеке доступны следующие порты:
* 5000: Logstash TCP input
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

> :warning: Elasticsearch [bootstrap checks][booststap-checks] были специально отключены, чтобы упростить настройку
> Elastic stack в средах разработки. Для производственных установок мы рекомендуем пользователям настроить свой хост в соответствии с
> инструкции из документации Elasticsearch: [Важная конфигурация системы][es-sys-config].

### SELinux

В дистрибутивах, в которых SELinux включен «из коробки», вам нужно будет либо пересоздать контекст файлов, либо установить SELinux
в разрешающий режим для правильного запуска docker-elk. Например, на Redhat и CentOS, следующее будет
применять правильный контекст:

```console
$ chcon -R system_u:object_r:admin_home_t:s0 docker-elk/
```

### Docker for Desktop

#### Windows

Убедитесь, что функция [Shared Drives][win-shareddrives] включена для диска `C:`.

#### macOS

Стандартная конфигурация Docker для Mac позволяет монтировать файлы из `/Users/`, `/Volumes/`, `/private/`, и `/tmp`
исключительно. Убедитесь, что хранилище клонировано в одном из этих мест, или следуйте инструкциям
[документации][mac-mounts] чтобы добавить больше мест.

## Usage

### Выбор версии

Этот репозиторий пытается соответствовать последней версии Elastic stack. Ветвь "мастера" отслеживает
текущая основная версия (7.x).

Чтобы использовать другую версию основных компонентов Elastic, просто измените номер версии внутри файла `.env`. Если
Вы обновляете существующий стек, пожалуйста, внимательно прочитайте примечание в следующем разделе.

> :warning: Всегда обращайте внимание на [официальные инструкции по обновлению][upgrade] для каждого отдельного компонента, прежде чем
выполнить обновление стека.

Старые версии также поддерживаются в отдельных ветках:

* [`release-6.x`](https://github.com/deviantony/docker-elk/tree/release-6.x): 6.x series
* [`release-5.x`](https://github.com/deviantony/docker-elk/tree/release-5.x): 5.x series (End-Of-Life)

### Поднятие стека

Клонируйте этот репозиторий на хост Docker, который будет запускать стек, а затем запустите службы локально, используя Docker Compose:

```console
$ docker-compose up
```

Вы также можете запустить все сервисы в фоновом режиме, добавив флаг `-d` к вышеприведенной команде.

> :warning: Вы должны перестраивать образы стека с помощью `docker-compose build` всякий раз, когда вы переключаете ветку или обновляете
> версию уже существующего стека.

Если вы запускаете стек впервые, пожалуйста, внимательно прочитайте раздел ниже.

### Очистка

По умолчанию данные Elasticsearch сохраняются внутри тома.

Чтобы полностью закрыть стек и удалить все сохраненные данные, используйте следующую команду Docker Compose:

```console
$ docker-compose down -v
```

## Начальная настройка

### Настройка аутентификации пользователя

> :information_source: Обратитесь к [Как отключить платные функции] (# как отключить платные функции), чтобы отключить аутентификацию.

Стек предварительно настроен для следующего **привилегированного** пользователя начальной загрузки:

* user: *elastic*
* password: *changeme*

Хотя все компоненты стека работают с этим пользователем «из коробки», мы настоятельно рекомендуем использовать непривилегированный [встроенный
пользователи][builtin-users] вместо этого для повышения безопасности.

1. Инициализировать пароли для встроенных пользователей

```console
$ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```

Пароли для всех 6 встроенных пользователей будут генерироваться случайным образом. Примите к сведению.

2. Удалить пароль начальной загрузки (_optional_)

Удалите переменную окружения `ELASTIC_PASSWORD` из службы `elasticsearch` внутри файла Compose
(`docker-compose.yml`). Он используется только для инициализации хранилища ключей во время первоначального запуска Elasticsearch.

3. Замените имена пользователей и пароли в файлах конфигурации

Используйте пользователя `kibana` внутри файла конфигурации Kibana (`kibana/config/kibana.yml`) и пользователя `logstash_system`
внутри файла конфигурации Logstash (`logstash/config/logstash.yml`) вместо существующего пользователя `elastic`.

Замените пароль для пользователя `elastic` в pipeline файле Logstash (`logstash/pipeline/logstash.conf`).

> :information_source: Не используйте пользователя `logstash_system` внутри файла Logstash *pipeline*, он не имеет
> достаточные разрешения для создания индексов. Следуйте инструкциям в [Настройка безопасности в Logstash][ls-security]
> создайте пользователя с подходящими ролями.

См. так же раздел [Конфигурация](#configuration) ниже.

4. Перезапустите Kibana и Logstash, чтобы применить изменения

```console
$ docker-compose restart kibana logstash
```

> :information_source: Узнайте больше о безопасности стека Elastic на [Учебник: Начало работы с
> безопасностью][sec-tutorial].

### Внедрение данных

Дайте Kibana около минуты для инициализации, затем откройте веб-интерфейс Kibana, введя
[http://localhost:5601](http://localhost:5601) с помощью веб-браузера и используйте следующие учетные данные по умолчанию для входа в систему:

* user: *elastic*
* password: *\<your generated elastic password>*

Теперь, когда стек работает, вы можете пойти дальше и добавить некоторые записи в журнал. Поставленная конфигурация Logstash позволяет
отправлять контент через TCP:


```console
# Using BSD netcat (Debian, Ubuntu, MacOS system, ...)
$ cat /path/to/logfile.log | nc -q0 localhost 5000
```

```console
# Using GNU netcat (CentOS, Fedora, MacOS Homebrew, ...)
$ cat /path/to/logfile.log | nc -c localhost 5000
```

Вы также можете загрузить пример данных, предоставленных вашей установкой Kibana.

### Создание Kibana index pattern по умолчанию

Когда Kibana запускается в первый раз, он не настроен ни на один индексный шаблон.

#### Через веб-интерфейс Kibana

> :information_source: Вам необходимо ввести данные в Logstash, прежде чем вы сможете настроить шаблон индекса Logstash через
веб-интерфейс Kibana.

Перейдите к _Discover_ представлению Kibana с левой боковой панели. Вам будет предложено создать шаблон индекса. Войти
`logstash-*` для соответствия индексам Logstash, затем на следующей странице выберите `@timestamp` в качестве поля фильтра времени. В заключение,
нажмите  _Create index pattern_  и вернитесь в представление _Discover_, чтобы просмотреть записи журнала.

Обратитесь к [Connect Kibana with Elasticsearch][connect-kibana] и [Creating an index pattern][index-pattern]] для получения подробной информации
инструкции о конфигурации шаблона индекса.

#### В командной строке

Создайте шаблон индекса с помощью API Kibana:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.6.2' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

Созданный шаблон будет автоматически помечен как шаблон индекса по умолчанию, как только пользовательский интерфейс Kibana будет открыт в первый раз.

## Конфигурация

> :information_source: Конфигурация не загружается динамически, вам нужно будет перезапустить отдельные компоненты после
любое изменение конфигурации.

### Как настроить Elasticsearch

Конфигурация Elasticsearch хранится в [`elasticsearch/config/elasticsearch.yml`][config-es].

Вы также можете указать параметры, которые вы хотите переопределить, установив переменные среды внутри Compose file:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Пожалуйста, обратитесь к следующей странице документации для получения более подробной информации о том, как настроить Elasticsearch в Docker
контейнеры: [Install Elasticsearch with Docker][es-docker].

### Как настроить Kibana

Конфигурация Kibana по умолчанию хранится в [`kibana/config/kibana.yml`][config-kbn].

Также возможно отобразить весь каталог `config` вместо одного файла.

Пожалуйста, обратитесь к следующей странице документации для получения более подробной информации о том, как настроить Kibana в Docker.
контейнеры: [Запуск Kibana в Docker][kbn-docker].

### Как настроить Logstash

Конфигурация Logstash хранится в [`logstash/config/logstash.yml`][config-ls].

Также возможно отобразить весь каталог `config` вместо одного файла, однако вы должны знать, что
Logstash ожидает файл [`log4j2.properties`][log4j-props] для собственной регистрации.

Пожалуйста, обратитесь к следующей странице документации для получения более подробной информации о том, как настроить Logstash внутри Docker
контейнеры: [Настройка Logstash для Docker][ls-docker].

### Как отключить платные функции

Переключите значение Elasticsearch's `xpack.license.self_generated.type` из `trial` на `basic` (see [License
settings][trial-license]).

### Как масштабировать кластер Elasticsearch

Следуйте инструкциям Wiki: [Scaling out Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

## Расширение

### Как добавить плагины

Чтобы добавить плагины к любому компоненту ELK, вы должны:

1. Добавьте оператор `RUN` в соответствующий `Dockerfile` (например, `RUN logstash-plugin install logstash-filter-json`)
2. Добавьте соответствующую конфигурацию кода плагина в конфигурацию сервиса (например, Logstash input/output)
3. Перестройте изображения с помощью команды `docker-compose build`

### Как включить предоставленные расширения

Несколько расширений доступны в каталоге [`extensions`](extensions). Эти расширения предоставляют функции, которые
не являются частью стандартного стека Elastic, но могут использоваться для его обогащения дополнительными интеграциями.

Документация для этих расширений предоставляется внутри каждого отдельного подкаталога для каждого расширения. Немного
из них требуется ручное изменение конфигурации ELK по умолчанию.

## JVM tuning

### Как указать объем памяти, используемой сервисом

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

Скрипты запуска для Elasticsearch и Logstash могут добавлять дополнительные параметры JVM из значения переменных 
среды, позволяющие пользователю регулировать объем памяти, который может использоваться каждым компонентом:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

Чтобы приспособиться к средам с ограниченным объемом памяти (по умолчанию в Docker для Mac доступно только 2 ГБ), размер Heap Size
выделение ограничено по умолчанию до 256 МБ на службу в файле `docker-compose.yml`. Если вы хотите переопределить
конфигурацию JVM по умолчанию, отредактируйте соответствующие переменные окружения в файле `docker-compose.yml`.

Например, чтобы увеличить максимальный размер JVM Heap Size для Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### Как включить удаленное соединение JMX с сервисом

Что касается памяти Java Heap (см. Выше), вы можете указать параметры JVM, чтобы включить JMX и отобразить порт JMX на Docker.
хост.

Обновите переменную среды `{ES,LS}_JAVA_OPTS` следующим содержимым (я сопоставил службу JMX с портом
18080, вы можете изменить это). Не забудьте обновить опцию `-Djava.rmi.server.hostname` с IP-адресом вашего
хост докера (замените **DOCKER_HOST_IP**):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Идем дальше

### Плагины и интеграции

Смотрите следующие Wiki-страницы:

* [External applications](https://github.com/deviantony/docker-elk/wiki/External-applications)
* [Popular integrations](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)

### Swarm mode

Экспериментальная поддержка Docker [Swarm mode][swarm-mode] предоставляется в виде файла `docker-stack.yml`, который может
быть развернутым в существующем кластере Swarm с помощью следующей команды:

```console
$ docker stack deploy -c docker-stack.yml elk
```

Если все компоненты будут развернуты без каких-либо ошибок, следующая команда покажет 3 запущенные службы:

```console
$ docker stack services elk
```

> :information_source: Чтобы масштабировать Elasticsearch в режиме Swarm, настройте *zen* на использование DNS-имени `tasks.elasticsearch`
вместо `elasticsearch`.


[elk-stack]: https://www.elastic.co/elk-stack
[stack-features]: https://www.elastic.co/products/stack
[paid-features]: https://www.elastic.co/subscriptions
[trial-license]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html

[linux-postinstall]: https://docs.docker.com/install/linux/linux-postinstall/

[booststap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

[win-shareddrives]: https://docs.docker.com/docker-for-windows/#shared-drives
[mac-mounts]: https://docs.docker.com/docker-for-mac/osxfs/

[builtin-users]: https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
[ls-security]: https://www.elastic.co/guide/en/logstash/current/ls-security.html
[sec-tutorial]: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-getting-started.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html
[index-pattern]: https://www.elastic.co/guide/en/kibana/current/index-patterns.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html

[log4j-props]: https://github.com/elastic/logstash/tree/7.6/docker/data/logstash/config
[esuser]: https://github.com/elastic/elasticsearch/blob/7.6/distribution/docker/src/docker/Dockerfile#L23-L24

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html

[swarm-mode]: https://docs.docker.com/engine/swarm/

===============================================================================================================

### Запись Django логов в ELK

Для взаимодействия `Django` с сервисом `Logstash` установите дополнительный пакет `Python-logstash`.

```console
$ pip install python-logstash
```
Изменим настройки Django приложения, чтобы логи отправлялись в Logstash сервис.

```py
LOGGING = {
  'version': 1,
  'disable_existing_loggers': False,
  'formatters': {
      'simple': {
            'format': 'velname)s %(message)s'
        },
  },
  'handlers': {
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'simple'
        },
        'logstash': {
            'level': 'INFO',
            'class': 'logstash.TCPLogstashHandler',
            'host': 'logstash',
            'port': 5000, 
            'version': 1,
            'message_type': 'django',  # 'type' поле для logstash сообщения.
            'fqdn': False,
            'tags': ['django'], # список тег.
        },
  },
  'loggers': {
        'django.request': {
            'handlers': ['logstash'],
            'level': 'INFO',
            'propagate': True,
        },
        ...
    }
}
```

После этого приложение будет отправлять логи в Logstash. Пример использования:

```py
import logging

logger = logging.getLogger(__name__)

def test_view(request, arg1, arg):
    ...
    if is_error:
        # Отправляем лог сообщение
        logger.error('Something went wrong!')
```

### Запись Nginx логов в ELK

Для работы с nginx логами потребуется дополнительный сервис `Filebeat`.

`Filebeat` будет читать логи из файла и отправлять в `Logstash`. Пример конфигурации:

```yml
filebeat.inputs:
  type: log
  enabled: true
  paths:
      - /var/log/nginx/access.log
  fields:
    type: nginx
  fields_under_root: true
  scan_frequency: 5s

output.logstash:
  hosts: ["logstash:5000"]
  ```
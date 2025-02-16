# Запуск теста в сервисе Yandex Cloud Load Testing с помощью YC CLI

В данном примере объясняется, как выполнить запуск теста в сервисе Load Testing с помощью YC CLI.

_Примечание._ В приведенных ниже фрагментах кода предполагается, что параметр `folder-id` уже задан в конфигурации `yc`:

```bash
yc config set folder-id 'my_folder_id'
```

### Подготовьте агент тестирования

В данном сценарии предполагается, что необходимый набор агентов нагрузочного тестирования уже подготовлен.

```bash
export AGENT_ID='agent id'
```

Подробнее о создании агента см. в указанных ниже руководствах:
- [Как создать агента в YC Compute](https://cloud.yandex.ru/en/docs/load-testing/operations/create-agent).
- [Как создать внешнего агента](https://cloud.yandex.ru/en/docs/load-testing/tutorials/loadtesting-external-agent).

### Подготовьте YAML-файл с конфигурацией теста.

Загрузите конфигурацию теста из YAML-файла.

```bash
export TEST_CONFIG_FILE="sample/_config_requests_in_file.yaml"

export TEST_CONFIG_ID=$(yc loadtesting test-config create --from-yaml-file $TEST_CONFIG_FILE --format json | jq -r ".id")
```

Подробнее о конфигурационных файлах см. в [соответствующей документации](https://yandextank.readthedocs.io/en/latest/config_reference.html#).

### Подготовьте тестовые данные (нагрузку)

Загрузите тестовые данные.

```bash
export TEST_PAYLOAD_FILE_IN_CONFIG="requests.uri"
export TEST_PAYLOAD_FILE="sample/_requests.uri"
export S3_PAYLOAD_BUCKET="my_bucket"
export S3_PAYLOAD_FILENAME="my_requests.uri"

export YC_TOKEN=$(yc iam create-token)
curl -H "X-YaCloud-SubjectToken: $YC_TOKEN" --upload-file - "https://storage.yandexcloud.net/$S3_PAYLOAD_BUCKET/$S3_PAYLOAD_FILENAME" < $TEST_PAYLOAD_FILE
```

Подробнее о файлах с тестовыми данными см. в [соответствующей документации](https://cloud.yandex.ru/en/docs/load-testing/concepts/payload).

### Запустите тест

Убедитесь, что предыдущие шаги завершены, и запустите тест с помощью следующей команды:

```bash

yc loadtesting test create \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial \
    --configuration id=$TEST_CONFIG_ID,agent-id=$AGENT_ID,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME

```

### Запустите мультитест

С помощью YC можно также запустить мультитест. Мультитест — это тест, использующий несколько агентов одновременно, что позволяет обойти ограничения одного агента по нагрузке (например, по пропускной способности, CPU или другим ресурсам).


```bash
export AGENT_ID1='first agent id'
export AGENT_ID2='second agent id'

yc loadtesting test create \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial,kind=multi \
    --configuration id=$TEST_CONFIG_ID,agent-id=$AGENT_ID1,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --configuration id=$TEST_CONFIG_ID,agent-id=$AGENT_ID2,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME
```

### Запустите тест с использованием первого доступного агента

Иногда может возникнуть необходимость запуска теста на первом доступном агенте или на группе агентов. 
Для выбора нужного агента можно использовать опцию `agent-by-filter`.
В следующем примере мы создадим мультитест, в котором первая часть будет выполнена на первом доступном агенте, а вторая — на любом агенте с меткой `key=value1` или `key=value2`.

```bash
export ANY_AGENT_SELECTOR=""
export SPECIFIC_AGENT_SELECTOR="labels.key IN (value1, value1."

yc loadtesting test create \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial \
    --configuration id=$TEST_CONFIG_ID,agent-by-filter=$ANY_AGENT_SELECTOR,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --configuration id=$TEST_CONFIG_ID,agent-by-filter={$SPECIFIC_AGENT_SELECTOR},test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME
```

### Дождитесь завершения теста

Чтобы дождаться завершения теста, используйте команду `wait`:

```bash
export TEST_ID='test id to wait to finish'

yc loadtesting test wait $TEST_ID
``` 

Также можно использовать флаг `--wait` с командой `test create`.

```bash

yc loadtesting test create \
    --wait \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial \
    --configuration id=$TEST_CONFIG_ID,agent-id=$AGENT_ID,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME

```

### При необходимости остановите тесты

Следующий код остановит все запущенные тесты:

```bash
export TESTS_TO_STOP=$(yc loadtesting test list --filter "summary.status not in (CREATED, DONE, STOPPED, AUTOSTOPPED, FAILED)" --format json | jq -r "[.[].id] | join(\" \")")
echo $TESTS_TO_STOP | xargs yc loadtesting test stop
```

### При необходимости удалите тесты

Следующий код удалит все созданные за текущий день тесты со статусом `Failed` или `Created`:

```bash
export TODAY=$(date +'%Y-%m-%d')
export TESTS_TO_DELETE=$(yc loadtesting test list --filter "summary.status in (FAILED, CREATED) and summary.created_at >= $TODAY" --format json | jq -r "[.[].id] | join(\" \")")
echo $TESTS_TO_DELETE | xargs yc loadtesting test delete
```


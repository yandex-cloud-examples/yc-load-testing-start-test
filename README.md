# Start a test in Yandex Cloud Load Testing service using YC CLI.

In this example, it is shown how to start a test in Load Testing service using YC CLI tool.

NOTE: in snippets bellow, it is assumed that folder-id setting is already set in `yc`:

```bash
yc config set folder-id 'my_folder_id'
```

### 1. Prepare testing agent

Here we assume that you already have a set of suitable LT agents.

```bash
export AGENT_ID='agent id'
```

On how to create an agent, see following guides:
- [How to create an agent in YC Compute](https://cloud.yandex.ru/en/docs/load-testing/operations/create-agent).
- [How to create an external agent](https://cloud.yandex.ru/en/docs/load-testing/tutorials/loadtesting-external-agent).

### 2. Prepare test yaml config

Upload your test configuration defined in YAML file.

```bash
export TEST_CONFIG_FILE="sample/_config_requests_in_file.yaml"

export TEST_CONFIG_ID=$(yc loadtesting test-config create --from-yaml-file $TEST_CONFIG_FILE --format json | jq -r ".id")
```

For information about config files, see [related documentation](https://yandextank.readthedocs.io/en/latest/config_reference.html#).

### 3. Prepare test payload

Upload test data payload.

```bash
export TEST_PAYLOAD_FILE_IN_CONFIG="requests.uri"
export TEST_PAYLOAD_FILE="sample/_requests.uri"
export S3_PAYLOAD_BUCKET="my_bucket"
export S3_PAYLOAD_FILENAME="my_requests.uri"

export YC_TOKEN=$(yc iam create-token)
curl -H "X-YaCloud-SubjectToken: $YC_TOKEN" --upload-file - "https://storage.yandexcloud.net/$S3_PAYLOAD_BUCKET/$S3_PAYLOAD_FILENAME" < $TEST_PAYLOAD_FILE
```

For information about data payload files, see [related documentation](https://cloud.yandex.ru/en/docs/load-testing/concepts/payload).

### 4. Start test

Given that all previous steps are done, start test with following command:

```bash

yc loadtesting test create \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial \
    --configuration id=$TEST_CONFIG_ID,agent-id=$AGENT_ID,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME

```

### 4.1 Start a multitest

You can also start a multitest using YC. A multitest is a test that utilizes multiple agents simulteneously,
thus surpassing a limit for a single load generation agent (either a bandwidth, cpu, or other resources).

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

### 4.2 Start test using first available agent

Sometimes, you may want to run a test on the first available agent or on a subset of agents. 
You can specify the agent selection option with the `agent-by-filter` option.
In the following example, we create a multitest. The first part will run on the first available agent, and the second part will run on any agent with the label `key=value1` or `key=value2`

```bash
export ANY_AGENT_SELECTOR=""
export SPECIFIC_AGENT_SELECTOR="labels.key IN (value1, value2)"

yc loadtesting test create \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial \
    --configuration id=$TEST_CONFIG_ID,agent-by-filter=$ANY_AGENT_SELECTOR,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --configuration id=$TEST_CONFIG_ID,agent-by-filter={$SPECIFIC_AGENT_SELECTOR},test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME
```

### 4.3 Waiting for completion

You can wait for the test to finish using the command `wait`

```bash
export TEST_ID='test id to wait to finish'

yc loadtesting test wait $TEST_ID
``` 

Or using flag `--wait` with `test create` command.

```bash

yc loadtesting test create \
    --wait \
    --name "yc-examples-test" \
    --description "Test has been created using YC" \
    --labels source=gh,type=tutorial \
    --configuration id=$TEST_CONFIG_ID,agent-id=$AGENT_ID,test-data=$TEST_PAYLOAD_FILE_IN_CONFIG \
    --test-data name=$TEST_PAYLOAD_FILE_IN_CONFIG,s3bucket=$S3_PAYLOAD_BUCKET,s3file=$S3_PAYLOAD_FILENAME

```

### (Optional) 5. Stop tests

The code here will stop all running tests

```bash
export TESTS_TO_STOP=$(yc loadtesting test list --filter "summary.status not in (CREATED, DONE, STOPPED, AUTOSTOPPED, FAILED)" --format json | jq -r "[.[].id] | join(\" \")")
echo $TESTS_TO_STOP | xargs yc loadtesting test stop
```

### (Optional) 6. Delete tests

The code here will delete all tests created today which have `Failed` or `Created` statuses.

```bash
export TODAY=$(date +'%Y-%m-%d')
export TESTS_TO_DELETE=$(yc loadtesting test list --filter "summary.status in (FAILED, CREATED) and summary.created_at >= $TODAY" --format json | jq -r "[.[].id] | join(\" \")")
echo $TESTS_TO_DELETE | xargs yc loadtesting test delete
```


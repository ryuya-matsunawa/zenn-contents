---
title: "Cloud Run JobsをAirflowで動的に実行する"
emoji: "🏃‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Airflow", "CloudComposer", "CloudRunJobs", "CloudScheduler"]
published: true
publication_name: "wed_engineering"
---

WED株式会社でデータエンジニアをしている[ryuya-matsunawa](https://github.com/ryuya-matsunawa)です。

この記事では、Cloud Run JobsをAirflowで動的に実行する方法について紹介します。

## はじめに
WEDでは、データ転送にCloud Composerを使用しています。
コードベースで管理できるため運用は楽ですが、一方で並列タスクを実行すると、予期せぬエラーが発生することがあります。このようなエラーの場合、DAGそのものが強制終了されてしまうため、エラー検知が難しいです。
そこで、並列実行したいタスクをCloud Run Jobsで実行し、Airflowで動的に実行する方法を考えました。

## Airflowのみでタスクを並列実行する場合
AirflowのDAGでタスクを並列実行する場合、以下のように記述します。

```python
# importは省略

@task
def get_ids():
    return [1, 2, 3]

@task(max_active_tis_per_dag=2)
def execute_task(id: int):
    print(f"execute_task: {id}")

with DAG(
    "dag_name",
    default_args=default_args,
    description="description",
    schedule_interval="* 21 * * *",  # 毎日06:00 JSTに実行
    dagrun_timeout=timedelta(hours=3),
    catchup=False,
    concurrency=2, # 並列実行数
) as dag:
    ids = get_ids()
    execute_task.expand(id=ids)
```

このDAGは、`get_ids`で取得したIDを`execute_task`で実行しています。
並列実行数を制御しても、処理時間が長い場合には、エラーが発生してしまいます。
この`execute_task`をCloud Run Jobsで実行することで、エラーを発生させずに処理を続行できるようにします。

## Cloud Run JobsをAirflowで動的に実行する
Cloud Run JobsをAirflowで動的に実行するためには、以下の手順が必要です。

1. Cloud Run Jobsを作成する
2. AirflowのサービスアカウントにCloud Run Jobsの実行権限を付与する
3. AirflowのDAGでCloud Run Jobsを実行する

### Cloud Run Jobsを作成する
Cloud Run Jobs用のコンテナイメージを作成します。
今回は、以下のようなコンテナイメージを作成します。

```Dockerfile
FROM python:3.11.9-slim

RUN pip install poetry

WORKDIR /work

COPY task /work/task
COPY poetry.lock /work/
COPY pyproject.toml /work/
RUN poetry new task
RUN poetry install

WORKDIR /work/task
EXPOSE 80
CMD poetry run python main.py
```

poetryを使用してPythonのパッケージをインストールしています。
main.pyは、実際に実行したい処理を記述します。
また、環境変数用のファイルを作成しておきます。

```python
import os

class Env:
    """Class for Environment variables"""

    def __init__(self):
        self.cloud_run_task_count = int(os.getenv("CLOUD_RUN_TASK_COUNT", 1))
        self.cloud_run_task_index = int(os.getenv("CLOUD_RUN_TASK_INDEX", 0))
        self.ids = os.getenv("IDS", "1,2").split(",")
```
CLOUD_RUN_TASK_COUNTとCLOUD_RUN_TASK_INDEXは、Cloud Run Jobsのタスク数とタスクインデックスを指定するための環境変数です。IDSはAirflowから渡されるIDのリストです。
実際に渡す部分は後ほど説明します。

次に、Cloud Run JobsをTerrafomで作成するにあたって、Artifact Registryにリポジトリを作成します。

```hcl
resource "google_artifact_registry_repository" "sample_repository" {
  provider = google-beta
  project  = var.project_id
  location = local.default_region
  repository {
    format = "DOCKER"
    description = "sample repository"
  }
}
```

作成したArifacts Registryにコンテナイメージを登録します。

```shell
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/sample-job .
docker tag ${REGION}-docker.pkg.dev/${PROJECT_ID}/sample-job ${REGION}-docker.pkg.dev/${PROJECT_ID}/sample-job
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/sample-job
```
Makefileを作っておくと更新しやすくなるのでおすすめです。

次に、Cloud Run Jobsを作成します。

```hcl
resource "google_cloud_run_v2_job" "sample_job" {
  provider = google-beta
  project  = var.project_id
  location = local.default_region
  name     = "sample-job"
  template {
    containers {
      image = "${REGION}-docker.pkg.dev/${PROJECT_ID}/sample-job"
      command = ["poetry"]
      args = ["run", "python", "main.py"]
    }
    task_count  = 10
    parallelism = 5
  }
}
```
CPUやメモリ、環境変数などを設定することもできます。
詳しくは[公式ドキュメント](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/cloud_run_v2_job)を参照してください。

### AirflowのサービスアカウントにCloud Run Jobsの実行権限を付与する

AirflowのサービスアカウントにCloud Run Jobsの実行権限を付与します。
Terraformで作成したサービスアカウントにCloud Run Jobsの実行権限を付与します。

```hcl
resource "google_cloud_run_v2_job_iam_member" "service_account_cloud_run_job" {
  provider = google-beta
  project  = var.project_id
  role     = "roles/run.developer"
  member   = "serviceAccount:${
    google_service_account.service_account.email
  }"
}
```

### AirflowのDAGでCloud Run Jobsを実行する

DAGでCloud Run Jobsを実行するためには、`CloudRunExecuteJobsOperator`を使用します。
CloudRunExecuteJobsOperatorは、Cloud Run Jobsを実行するためのOperatorです。詳しくは[公式ドキュメント](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/_api/airflow/providers/google/cloud/operators/cloud_run/index.html#airflow.providers.google.cloud.operators.cloud_run.CloudRunExecuteJobsOperatorl)を参照してください。
今回は、`overrides`を使用して、Cloud Run Jobsの環境変数を上書きすることで、タスク数を動的に実行します。

```python
# 他のimportは省略
from airflow.operators.python import PythonOperator
from airflow.providers.google.cloud.operators.cloud_run import CloudRunExecuteJobsOperator

def get_ids(**kwargs):
    return [1, 2, 3]

def execute_cloud_run(**kwargs):
    ids = kwargs["task_instance"].xcom_pull(task_ids="get_ids")
    overrides = {
      "container_overrides": [
        {
          "env": [
            {"name": "IDS", "value": ",".join(map(str, ids))}
          ]
        }
      ],
      "task_count": max(len(id_list), 5),
    }
    return CloudRunExecuteJobsOperator(
        task_id="execute_cloud_run",
        region="us-central1",
        project_id="project_id",
        job_name="job_name",
        overrides=overrides,
    ).execute(context=kwargs)

with DAG(
    "dag_name",
    default_args=default_args,
    description="description",
    schedule_interval="* 21 * * *",  # 毎日06:00 JSTに実行
    dagrun_timeout=timedelta(hours=3),
    catchup=False,
    concurrency=2, # 並列実行数
) as dag:
    get_ids = PythonOperator(
        task_id="get_ids",
        python_callable=get_ids,
        provide_context=True,
    )

    task_execute_cloud_run = PythonOperator(
        task_id="execute_cloud_run",
        python_callable=execute_cloud_run,
        provide_context=True,
    )

    get_ids >> task_execute_cloud_run
```

overridesで環境変数を上書きすることで、Cloud Run Jobsのタスク数を動的に実行できます。
また、task_countを指定することで、並列実行数を制御できます。

## まとめ
Cloud Run JobsをAirflowで動的に実行する方法について紹介しました。
改善点としては、DAGのコード量が増えてしまうのでより良い方法があれば検討したいです。
Airflowのみで並列実行すること自体は可能なので、Cloud Run Jobsを使用するかどうかは状況によって変わると思います。
この記事が、Cloud Run JobsをAirflowで動的に実行する際の参考になれば幸いです。

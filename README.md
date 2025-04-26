# Отправка label в систему логирования и мониторинга из метаданных GitLab Runner (job_id, pipeline_id)

## Введение — какую проблему решаем

При использовании GitLab CI/CD с Kubernetes возникает необходимость видеть связку между логами и конкретными CI job'ами или pipeline'ами. Это особенно полезно для отладки и мониторинга. Однако по умолчанию логи из подов не содержат этих связующих метаданных.

В данной статье мы покажем, как можно передавать метки `job_id`, `pipeline_id`, `project_name` и другие из GitLab Runner в систему логирования VictoriaLogs с помощью Promtail и систему мониторинга VictoriaMetrics.

## Почему используем VictoriaLogs, а не Loki

Хотя Loki является популярным решением для логирования в Kubernetes, у него есть проблема с производительностью при большом количестве высококардинальных меток. Системы, вроде GitLab CI, где каждый job и pipeline имеет уникальные идентификаторы, создают большое количестве уникальных label'ов.

VictoriaLogs превосходит Loki благодаря простой и эффективной архитектуре: единый движок обеспечивает хранение, индексацию и поиск логов без сложных распределённых компонентов. Поддерживает полнотекстовый поиск по любым полям без настройки схемы, а колоночное хранение со сжатием экономит ресурсы и ускоряет аналитические запросы. VictoriaLogs использует принципы ClickHouse для высокой производительности.

## Регистрация GitLab Runner в gitlab.com

Перед установкой runner'а необходимо зарегистрировать его в GitLab:

1. Перейдите в репозиторий GitLab.
2. Откройте `Settings -> CI/CD -> Runners`.
3. Скопируйте registration token.

Этот токен понадобится для регистрации раннера в вашем Kubernetes-кластере.

## Запуск Kubernetes и установка GitLab Runner

Установка kubernetes через kind
```shell
kind create cluster --name test-gitlab-runner
kubectl cluster-info --context kind-test-gitlab-runner
```

Установка GitLab Runner через Helm:
```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update

helm upgrade --install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner --create-namespace \
  --values gitlab-runner-values.yaml
```

Пример `gitlab-runner-values.yaml`:
```yaml
gitlabUrl: "https://gitlab.com/"
runnerToken: "<YOUR_REGISTRATION_TOKEN>"
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        [runners.kubernetes.pod_labels]
          "job_name" = "${CI_JOB_NAME_SLUG}"
          "job_id" = "${CI_JOB_ID}"
          "project_name" = "${CI_PROJECT_NAME}"
          "project_id" = "${CI_PROJECT_ID}"
          "pipeline_id" = "${CI_PIPELINE_ID}"
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
service:
  enabled: true
rbac:
  create: true
```

## Настройка GitLab Runner
После установки Runner начнёт запускать поды с метками:

- `job_name`
- `job_id`
- `project_name`
- `project_id`
- `pipeline_id`

Просмотр сырых метрик GitLab Runner
```shell
kubectl port-forward -n gitlab-runner service/gitlab-runner 9252
```

## Установка victoria-metrics-k8s-stack

Добавим Helm репозиторий и установим VictoriaMetrics stack:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update

helm upgrade --install vm-stack vm/victoria-metrics-k8s-stack \
  --namespace vm-stack --create-namespace
```


Необходимо включить либо
```shell
    kube-state-metrics:
      enabled: true
      metricLabelsAllowlist:
        - pods=[*]
```
либо namespaces gitlab-runner
https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-state-metrics/values.yaml#L397
https://github.com/VictoriaMetrics/helm-charts/blob/master/charts/victoria-metrics-k8s-stack/values.yaml#L975C1-L975C19


После установки, Grafana будет доступна через port-forward.
```shell
kubectl port-forward -n vm-stack service/vm-stack-grafana 8080:80
```

В Grafana уже будут видны метки, переданные из GitLab Runner (`job_id`, `pipeline_id` и т.д.).

## Установка VictoriaLogs и Promtail

### Установка VictoriaLogs
```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update

helm install victorialogs vm/victoria-logs-single \
  --namespace victorialogs --create-namespace
```

### Настройка Promtail для захвата меток

Файл конфигурации Promtail `promtail-values.yaml`:
```yaml
tolerations:
  - operator: Exists
    effect: NoSchedule

config:
  clients:
    - url: http://victorialogs-victoria-logs-single-server.victorialogs.svc.cluster.local.:9428/insert/loki/api/v1/push?_msg_field=msg

  snippets:
    pipelineStages:
      - cri: {}
      - labeldrop:
          - filename
          - node_name

    extraRelabelConfigs:
      - action: replace
        source_labels:
          - __meta_kubernetes_pod_label_ci_pipeline_id
        target_label: ci_pipeline_id

      - action: replace
        source_labels:
          - __meta_kubernetes_pod_label_job_name
        target_label: job_name

      - action: replace
        source_labels:
          - __meta_kubernetes_pod_label_project_name
        target_label: project_name
```

### Установка Promtail
```bash
helm upgrade --install promtail grafana/promtail \
  --namespace promtail --create-namespace \
  -f promtail-values.yaml
```

## Просмотр как это выглядит в VictoriaLogs

1. Зайдите в UI VictoriaLogs (обычно на `/select/`) или подключитесь к Grafana.
2. Выполните запросы с фильтрацией по `job_id`, `pipeline_id`, например:
```logql
{ci_pipeline_id="12345678"}
```

Теперь вы можете фильтровать логи по job, pipeline и другим CI-метаданным, что значительно упрощает отладку и мониторинг процессов.

## Удаление kubernetes через kind
```shell
kind delete cluster --name test-gitlab-runner
```

## Заключение

Теперь вы можете легко отслеживать, какие логи/метрики относятся к какому pipeline и job'у. Использование VictoriaLogs позволяет справляться с большим объемом уникальных меток. Этот подход хорошо масштабируется и обеспечивает гибкую, но мощную систему обсервабилити для CI/CD процессов в Kubernetes.

С добавлением всего нескольких строк в конфигурацию вы получаете мощный инструмент для мониторинга и отладки CI/CD pipeline'ов.
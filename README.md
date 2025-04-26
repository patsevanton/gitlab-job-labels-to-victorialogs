# Отправка label в систему логирования и мониторинга из метаданных GitLab Runner (job_id, pipeline_id)

## Введение — какую проблему решаем

При использовании GitLab CI/CD с Kubernetes часто возникает необходимость видеть связку между логами и конкретными CI job'ами или pipeline'ами. Это особенно полезно для отладки и мониторинга. Однако по умолчанию логи из подов не содержат этих связующих метаданных.

В данной статье мы покажем, как можно передавать метки `job_id`, `pipeline_id`, `project_name` и другие из GitLab Runner в систему логирования Victorialogs с помощью Promtail и корректной настройки runner'а в Kubernetes.

## Почему используем Victorialogs, а не Loki

Хотя Loki является популярным решением для логирования в Kubernetes, у него есть проблема с производительностью при большом количестве высококардинальных меток. Системы, вроде GitLab CI, где каждый job и pipeline имеет уникальные идентификаторы, создают миллионы уникальных label'ов.

Victorialogs, в свою очередь, построен на базе VictoriaMetrics и лучше справляется с подобными сценариями — он эффективно обрабатывает высокую кардинальность меток, не теряя в производительности.

## Регистрация GitLab Runner в gitlab.com

Перед установкой runner'а необходимо зарегистрировать его в GitLab:

1. Перейдите в репозиторий GitLab.
2. Откройте `Settings -> CI/CD -> Runners`.
3. Скопируйте registration token.

Он понадобится на этапе установки.

## Быстрый старт: запуск Kubernetes и установка GitLab Runner

Для запуска Kubernetes можно использовать:

- minikube
- kind
- любой managed кластер (GKE, EKS, AKS)

Установка GitLab Runner через Helm:

```bash
gitlabUrl="https://gitlab.com/"
registrationToken="<ВАШ_ТОКЕН>"

helm repo add gitlab https://charts.gitlab.io
helm repo update

helm upgrade --install gitlab-runner gitlab/gitlab-runner \
  --namespace gitlab-runner --create-namespace \
  --set gitlabUrl=$gitlabUrl \
  --set runnerRegistrationToken=$registrationToken \
  --values values.yaml
```

Пример `values.yaml`:

```yaml
gitlabUrl: "https://gitlab.com/"
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

Эти метки будут прикреплены к каждому поду, и Promtail сможет использовать их для маркировки логов.

## Установка victoria-metrics-k8s-stack

Добавим Helm репозиторий и установим VictoriaMetrics stack:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update

helm upgrade --install vm-stack vm/victoria-metrics-k8s-stack \
  --namespace monitoring --create-namespace
```

После установки откройте Grafana (по умолчанию — портфорвард с `kubectl port-forward`), и вы увидите метки `job_id`, `pipeline_id` в Prometheus данных.

## Установка victorialogs и promtail

Victorialogs можно развернуть через Helm или манифесты. После этого устанавливаем Promtail:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install promtail grafana/promtail \
  --namespace logging --create-namespace \
  -f promtail-values.yaml
```

Пример `promtail-values.yaml`:

```yaml
tolerations:
  - operator: Exists
    effect: NoSchedule

config:
  clients:
    - url: http://victorialogs.corp/insert/loki/api/v1/push?_msg_field=msg

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

## Просмотр как это выглядит в victorialogs

После запуска всех компонентов и выполнения одного или нескольких job'ов в GitLab, вы можете зайти в UI Victorialogs и убедиться, что:

- логи видны;
- у логов присутствуют метки: `job_name`, `pipeline_id`, `project_name` и т.д.

Теперь вы можете фильтровать логи по job, pipeline и другим CI-метаданным, что значительно упрощает отладку и мониторинг процессов.

## Заключение

Мы рассмотрели, как с помощью GitLab Runner, Promtail и Victorialogs можно отправлять CI-метаданные в систему логирования. Это решение масштабируемо и устойчиво к высоким нагрузкам, в отличие от стандартного стека Loki.

С добавлением всего нескольких строк в конфигурацию вы получаете мощный инструмент для мониторинга и отладки CI/CD pipeline'ов.


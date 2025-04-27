# Отправка label в систему логирования и мониторинга из метаданных GitLab Runner (job_id, pipeline_id)

## Введение — какую проблему решаем

При использовании GitLab CI/CD с Kubernetes возникает необходимость видеть связку между логами и конкретными CI job'ами 
или pipeline'ами. Это особенно полезно для отладки и мониторинга. Однако по умолчанию логи из подов не содержат этих 
связующих метаданных.

В данной статье мы покажем, как можно передавать метки `job_id`, `pipeline_id`, `project_name` и другие из GitLab Runner 
в систему логирования VictoriaLogs с помощью Promtail и систему мониторинга VictoriaMetrics.

## Почему используем VictoriaLogs, а не Loki

- **Мгновенный полнотекстовый поиск** по любым полям логов (включая высококардинальные, такие как user_id или trace_id), 
без необходимости предварительной настройки схемы. В Loki подобные запросы требуют осторожного управления метками и 
могут приводить к резкому росту потребления памяти.

- **Экономия ресурсов**: колоночное хранение как ClickHouse с автоматическим сжатием сокращает объём данных на диске в 
[5–10](https://docs.victoriametrics.com/victorialogs/faq/#what-is-the-difference-between-victorialogs-and-grafana-loki) 
раз по сравнению с Loki, а также ускоряет аналитические запросы.

- **Простота запросов**: интуитивный язык LogsQL удобен для фильтрации, агрегации и анализа, 
тогда как LogQL в Loki часто требует сложных конструкций для базовых задач.

- **Производительность**: типовые запросы выполняются до 
[1000x](https://docs.victoriametrics.com/victorialogs/faq/#what-is-the-difference-between-victorialogs-and-grafana-loki) 
быстрее, чем в Loki, благодаря оптимизированным алгоритмам индексации и отсутствию накладных расходов распределённых систем.

## Регистрация GitLab Runner в gitlab.com

Перед установкой runner'а необходимо зарегистрировать его в GitLab:

1. Перейдите в репозиторий GitLab.
2. Откройте `Settings -> CI/CD -> Runners`.
3. Выключите общие раннеры
4. Скопируйте registration token.

Этот токен понадобится для регистрации раннера в вашем Kubernetes-кластере.

## Установка Kubernetes

Установка kubernetes через terraform
```shell
export YC_FOLDER_ID='ваш folder_id'
terraform init
terraform apply
```

## Установка GitLab Runner через Helm:
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

### Проверка, что у pod есть label
Запустите сборку на этом раннере и проверьте label
Должно быть примерно так
```shell
kubectl get pod -n gitlab-runner --show-labels
NAME                                                      READY   STATUS        RESTARTS   AGE   LABELS
gitlab-runner-5fd7489954-jx682                            1/1     Running       0          27m   app=gitlab-runner,chart=gitlab-runner-0.76.0,heritage=Helm,pod-template-hash=5fd7489954,release=gitlab-runner
runner-3lj4ihzt1-project-69309276-concurrent-0-shpvdimu   2/2     Terminating   0          21s   job.runner.gitlab.com/pod=runner-3lj4ihzt1-project-69309276-concurrent-0,job_id=9838839173,job_name=build-job,manager.runner.gitlab.com/id-short=3LJ4iHZt1,manager.runner.gitlab.com/name=gitlab-runner-5fd7489954-jx682,pipeline_id=1787794791,pod=runner-3lj4ihzt1-project-69309276-concurrent-0,project.runner.gitlab.com/id=69309276,project.runner.gitlab.com/name=gitlab-for-job-labels-to-victorialogs,project.runner.gitlab.com/namespace-id=1739251,project.runner.gitlab.com/namespace=anton_patsev,project.runner.gitlab.com/root-namespace=anton_patsev,project_id=69309276,project_name=gitlab-for-job-labels-to-victorialogs
```

## Установка victoria-metrics-k8s-stack

Добавим Helm репозиторий и установим VictoriaMetrics stack:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update

helm upgrade --install vmks vm/victoria-metrics-k8s-stack \
  --namespace vmks --create-namespace \
  --values vmks-values.yaml
```

В values указываем что kube-state-metrics, разрешая экспорт всех метрик ([*]), связанных с подами (pods).
```shell
kube-state-metrics:
  metricLabelsAllowlist:
    - pods=[*]
```

Подробности в values kube-state-metrics и victoria-metrics-k8s-stack:
https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-state-metrics/values.yaml#L397
https://github.com/VictoriaMetrics/helm-charts/blob/master/charts/victoria-metrics-k8s-stack/values.yaml#L975C1-L975C19


После установки, Grafana будет доступна по адресу http://grafana.apatsev.org.ru

Получение пароля grafana для admin юзера
```shell
kubectl get secret vmks-grafana -n vmks -o jsonpath='{.data.admin-password}' | base64 --decode
```

Вы можете увидеть метрики по pod
```shell
node_namespace_pod_container:container_memory_working_set_bytes{namespace="gitlab-runner", pod="runner-3lj4ihzt1-project-69309276-concurrent-0-0066ip3c"}
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
  --values promtail-values.yaml
```

## Просмотр как это выглядит в VictoriaLogs

1. Зайдите в UI VictoriaLogs (обычно на `/select/`) или подключитесь к Grafana.
2. Выполните запросы с фильтрацией по `job_id`, `pipeline_id`, например:
```logql
{ci_pipeline_id="12345678"}
```

Теперь вы можете фильтровать логи по job, pipeline и другим CI-метаданным, 
что значительно упрощает отладку и мониторинг процессов.

## Удаление kubernetes через terraform
```shell
terraform destroy
```

## Заключение

Теперь вы можете легко отслеживать, какие логи/метрики относятся к какому pipeline и job'у. Использование VictoriaLogs 
позволяет справляться с большим объемом уникальных меток. Этот подход хорошо масштабируется и обеспечивает гибкую, 
но мощную систему обсервабилити для CI/CD процессов в Kubernetes. С добавлением всего нескольких строк в конфигурацию 
вы получаете мощный инструмент для мониторинга и отладки CI/CD pipeline'ов.


# Отправка label в систему логирования и мониторинга из метаданных gitlab runner (job_id, pipeline_id)

Содержание
- Введение - какую проблему решаем
- Почему используем victorialogs, а не Loki (Причина большое кол-во высококардинальных метрик)
- Регистрация gitlab runner в gitlab.com (запоминаем token)
- Быстрый старт: запуск k8s, установка gitlab runner.
- Настройка gitlab runner
- Установка victoria-metrics-k8s-stack. Видим что в grafana, которая доступна после установки victoria-metrics-k8s-stack, появились label по job_id, pipeline_id
- Установка victorialogs и promtail. Настройка promtail чтобы он преобразовывал label pod в label в системе мониторинга
- Просмотр как это выглядит в victorialogs
- Заключение

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

## Часть 2: Настройка Promtail для захвата меток


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

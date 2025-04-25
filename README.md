
# Создание Kubernetes labels для GitLab Runner и отправка логов в Victorialogs через Promtail

## Введение

При интеграции GitLab CI/CD с Kubernetes и системой логирования Victorialogs через Promtail важно сохранять контекст выполнения заданий. Использование меток (labels) Kubernetes помогает маркировать pod'ы нужной информацией, а Promtail может эти метки извлекать и передавать в Victorialogs.

## Часть 1: Настройка GitLab Runner с метками

В конфигурации GitLab Runner, работающего в Kubernetes, вы можете задать метки для каждого создаваемого pod'а:

```toml
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

Эти переменные GitLab CI автоматически заменяются на соответствующие значения при создании pod'ов. Метки позволяют Promtail точно идентифицировать, какому проекту и пайплайну принадлежат логи.

## Часть 2: Настройка Promtail для захвата меток

Promtail собирает логи из pod'ов и может извлекать метки, заданные в Kubernetes, и передавать их в Victorialogs. Вот пример конфигурации:

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

Здесь Promtail:

1. Извлекает стандартные Kubernetes метки (`__meta_kubernetes_pod_label_*`).
2. Переносит их в собственные метки `ci_pipeline_id`, `job_name`, `project_name`.
3. Передаёт эти логи и метки в Victorialogs через указанный URL.

## Заключение

Таким образом, вы получаете связку: GitLab Runner → Kubernetes → Promtail → Victorialogs, где логи каждого задания содержат всю необходимую информацию о пайплайне и проекте. Это значительно упрощает отладку и мониторинг CI/CD процессов.

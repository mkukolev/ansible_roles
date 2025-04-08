# Ansible Playbook: Обновление fixVersion у задач Jira при событиях в Bitbucket

## Назначение

Этот Ansible Playbook используется для автоматического обновления поля `fixVersion` у задач в Jira, связанных с коммитами в определённых ветках Bitbucket. 
Обновление происходит при:

- Слиянии функционала из веткок разработки для `devel`, `master`, `main` и `release/*`.
- Создании новой релизной ветки `release/*` со связанными коммитам из задач Jira.

Автоматизация реализована через связку Bitbucket → AWX → Ansible → Jira.

## Архитектура решения

```
Bitbucket Webhook
        ↓
AWX Job Template (webhook payload)
        ↓
Ansible Playbook
        ↓
Jira API (обновление fixVersion у задач)
```

## Входные данные

Шаблон [Jira FixVersion Update](https://controller.aa.astra-team.ru/#/templates/job_template/3460/details) запускается при получении webhook от Bitbucket и получает JSON следующей структуры:

```json
{
  "changes": [
    {
      "ref": {
        "displayId": "release/<release_version>"
      }
    }
  ]
}
```

Из этого JSON извлекается имя ветки `release/<release_version>`, из которого получается имя версии. Например `1.2-upd2`.

## 🛠 Настройка

### 1. Bitbucket
- В разделе **Repository Settings → Webhooks** создайте новый webhook.
- Укажите URL, соответствующий webhook URL в AWX Job Template.
- Выберите события **Push** и **Create Branch**.
- Убедитесь, что JSON, отправляемый в webhook, содержит `ref.displayId`.

### 2. AWX
- Создан новый шаблон - [Jira FixVersion Update](https://controller.aa.astra-team.ru/#/templates/job_template/3460/details) :
  - Playbook: `update_fixversion.yml`
  - Включена настройка **Enable Webhook** (тип — GitHub).
  - Включен настройка **Prompt on Launch** для передачи `branch_name` и `webhook_payload`.
- В рамках проекта добавлен новый **Credentials** - [AA - Jira Automation Credential](https://controller.aa.astra-team.ru/#/credentials/1600/details):
  - Тип: **Machine** (для доступа к localhost)
  - Используются переменные:
    - `jira_username` — логин пользователя Jira
    - `jira_password` — пароль или API токен
- Все настройки объедины в проект - [Jira Automation](https://controller.aa.astra-team.ru/#/projects/3458/details)

## Примечания

- Плейбук предполагает, что названия веток содержат идентификаторы Jira-задач в формате `AA-1234`, `BB-5678` и т.д.
- Если `fixVersion` с нужным значением отсутствует в Jira, то задача завершится с ошибкой. Версию необходимо создавать заранее или реализовать создание версии через API.
- Код ответа `204 No Content` от Jira означает успешное обновление задачи.

## Результат

При любом мержевом событии в главные ветки Bitbucket или создании релизной ветки — все связанные задачи Jira автоматически получают нужную версию в поле `fixVersion`. 
Это позволяет поддерживать релизную дисциплину и упростить контроль за задачами, входящими в релиз.

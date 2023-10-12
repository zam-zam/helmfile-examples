# Описание

Пример использования helmfile для деплоя в несколько окружений

- Удобно структурировано по окружениям
- Легко добавлять новые окружения и приложения
- Секретные переменные хранятся в зашифрованном мире

# Инструкции

## Деплой окружений

### Как задеплоить окружение `client01/prod` в неймспейс `client01-prod`

```
helmfile -e client01/prod -n client01-prod apply
```

### Как задеплоить окружение `team/dev` в неймспейс `team-dev`

```
helmfile -e team/dev -n team-dev apply
```

## Как добавить новое окружение. На примере client02/test

1. Создать в папке envs следующую структуру папок:

```
client02
└── test
    ├── env.yaml
    ├── secrets
    │   └── _all.yaml
    └── values
        └── _all.yaml.gotmpl
```

> env.yaml - описание окружения
> 
> secrets - зашифрованные переменные приложений
> 
> values - переменные приложений

2. Описать параметры окружения и переменные приложений по примеру других окружений

3. Добавить окружение в `.helmfile/environments.yaml`

```
environments:
  client02/test:
    <<: *default
```

4. Добавить CICD для окружения в папку .gitlab по примеру других окружений. Если был создан отдельный файл, его необходимо добавить в .gitlab-ci.yml в секцию `include`

## Как добавить новое приложение. На примере приложения newapp

1. Описать приложение в `apps/newapp_group.yaml`

```
apps:
  newapp:
    repo: private
    chart: newapp
    version: 0.1.0-develop
```

2. Добавить файл с общими для всех окружения параметрами релиза приложения в `releases/newapp.yaml.gotmpl`

```
fullnameOverride: newapp
replicas: 1
resources:
  requests:
    cpu: 50m
    memory: 32Mi
  limits:
    cpu: 1
    memory: 1Gi
ingress:
  enabled: true
  paths:
    - /
  hosts:
    - newapp-{{ .Values.global.ingressDomain }}
```

## Как добавить приложение в окружение. На примере приложения newapp в окружение client02/test

1. Добавить установку приложения и его версию в специальный файл в папке окружения `envs/client02/test/env.yaml`

```
apps:
  newapp:
    installed: true
    version: 1.2.0
```

## Как добавить зашифрованные переменные приложений в окружение. На примере окружения client02/test

1. Установить локально gpg и [sops](https://github.com/mozilla/sops)

2. Импортировать gpg ключ, fingerprint которого указан в .sops.yaml

```
read -r -d '' GPG_PRIVATE_KEY <<EOF
<GPG PRIVATE KEY CONTENT HERE>
EOF
cat<<<"${GPG_PRIVATE_KEY}" | gpg --import
```

3. Открыть и отредактировать файл переменных утилитой sops

`sops envs/client02/test/secrets/_all.yaml`

# Как это работает

## Окружения

Окружения описаны в файле `.helmfile/environments.yaml`. Все окружения переиспользуют (с помощью yaml якоря и ссылки) единый шаблон, в котором в нужной последовательности подключаются переменные из папок окружений

1. Описания хелмовых релизов из `apps/*.*`: имя релиза, репа helm чарта, его название и версия
2. Настройки и глобальные переменные самого окружения из `envs/{{ .Environment.Name }}/*.*`. Здесь описаны, какие helm релизы должны быть установлены в окружение, зависимости от других helm релизов, а также глобальные переменные самого окружения, которые можно подключать в values релизов
3. Переменные для helm релизов `envs/{{ .Environment.Name }}/values/*.*`
4. Зашифрованные переменные для helm релизов `envs/{{ .Environment.Name }}/secrets/*.*`

## Helm репозитории

Описываются в `.helmfile/repositories.yaml`. Используются затем в описании helm релизов

## Helm релизы

Список релизов для каждого окружения динамически формируется из словаря `apps` прогонкой через шаблон `.helmfile/releases.yaml`

Например, такое описание apps

```yaml
apps:
  postgres:
    repo: stable
    chart: postgresql
    version: 8.4.0
    installed: true
```

превратится в такой список релизов

```yaml
releases:
- name: postgres
  labels:
    app: postgres
  chart: stable/postgresql
  version: 8.4.0
  missingFileHandler: Info
  values:
    - releases/postgres.yaml.gotmpl
    - releases/_override.yaml.gotmpl
  installed: true
  installed: false
```

### Переменные helm релиза

В `releases/postgres.yaml.gotmpl` описаны общие для всех окружений переменные хелмового релиза

С помощью файла `releases/_override.yaml.gotmpl` переменные хелмового релиза конкретного окружения оверрайдят общие. Так сделано, чтобы в окружении можно было задавать переменные в одном файле в таком формате

```yaml
store-backend:
  replicas: 2
  env:
    DB_ADDR: jdbc:postgresql://db.test.example.com:5432/store
    ENV: test

store-frontend:
  replicas: 2
  env:
    ENV: test
```

вместо того, чтобы создавать для каждого релиза отдельный файл (когда в одном окружении десяток приложений, то неудобно бегать по десяткам файлов)

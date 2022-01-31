# Как добавить новое окружение. На примере client02/test

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
> secrets - зашифрованные переменные приложений
> values - переменные приложений

2. Описать параметры окружения и переменные приложений по примеру других окружений

3. Добавить окружение в `.helmfile/environments.yaml`

```
environments:
  client02/test:
    <<: *default
```

4. Добавить CICD для окружения в папку .gitlab по примеру других окружений. Если был создан отдельный файл, его необходимо добавить в .gitlab-ci.yml в секцию `include`

# Как добавить новое приложение. На примере приложения newapp

1. Описать приложение в `apps/newapp_group.yaml`

```
apps:
  newapp:
    repo: helm-private-incubator
    chart: newapp
    version: 0.1.0-develop-SNAPSHOT
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

# Как добавить приложение в окружение. На примере приложения newapp в окружение client02/test

1. Добавить установку приложения и его версию в специальный файл в папке окружения `envs/client02/test/env.yaml`

```
apps:
  newapp:
    installed: true
    repo: helm-private-stable
    version: 1.2.0
```

# Как добавить зашифрованные переменные приложений в окружение. На примере окружения client02/test

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

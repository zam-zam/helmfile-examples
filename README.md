# Описание

Пример использования helmfile для деплоя в несколько окружений

- Удобно структурировано по окружениям
- Легко добавлять новые окружения и приложения
- Секретные переменные хранятся в зашифрованном мире

# Быстрый старт (Ubuntu 22.04)

> [!IMPORTANT]
>
> 
>
> Исходники **simple-python-web-app** https://github.com/zam-zam/simple-python-web-app
>
> Helm чарт для **simple-python-web-app** https://github.com/zzamtools/helm-charts/tree/master/charts/simple-python-web-app

https://github.com/zam-zam/helmfile-examples/assets/5456100/33e8b300-f3a2-4c67-9341-612842086811

Установим необходимые утилиты
```bash
sudo apt-get update
sudo apt-get install -y git jq
```

Запустим тестовый кубер через k0s
```bash
curl -sSLf https://get.k0s.sh | sudo K0S_VERSION='v1.28.2+k0s.0' sh
sudo k0s install controller --single
sudo k0s start
```

Проверяем, что кубер запустился (может занять какое-то время)
```bash
sudo k0s status
```

![](.readme/k0s_status.png?raw=true "K0S Status")

Установим kubectl, helm, helmfile
```bash
KUBECTL_VERSION='1.28.2'
HELM_VERSION='3.13.1'
HELMFILE_VERSION='0.157.0'
# Установить kubectl
sudo curl -L "https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl && sudo chmod +x /usr/local/bin/kubectl
# Установить helm
curl -L "https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3" | sudo DESIRED_VERSION="v${HELM_VERSION}" bash
# Установить helmfile
curl -L "https://github.com/helmfile/helmfile/releases/download/v${HELMFILE_VERSION}/helmfile_${HELMFILE_VERSION}_linux_amd64.tar.gz" | sudo tar xzvf - -C /usr/local/bin/ --no-same-owner helmfile
```

Иницализируем helmfile (он установит необходимы helm плагины)
```bash
helmfile init --force
```

Копируем kubeconfig, сформированный k0s, в дефолтный путь, чтобы kubectl/helm/helmfile могли его найти
```bash
mkdir ~/.kube
sudo k0s kubectl config view --raw > ~/.kube/config && chmod 600 ~/.kube/config
```

Копируем репу helmfile-examples
```bash
git clone https://github.com/zam-zam/helmfile-examples.git
cd helmfile-examples
```

Установим утилиту sops для работы с зашифрованными переменными
```bash
sudo curl -L https://github.com/getsops/sops/releases/download/v3.8.1/sops-v3.8.1.linux.amd64 -o /usr/local/bin/sops && sudo chmod +x /usr/local/bin/sops
```

Добавим ключ age, необходимый для расшифровки секретных переменных

> [!WARNING]
> Ни в коем случае не используйте этот приватный ключ для шифрования своих секретов. Установите утилиту [age](https://github.com/FiloSottile/age) и сгенерируйте свою пару ключей

```bash
mkdir -p ~/.config/sops/age
cat << EOF > ~/.config/sops/age/keys.txt
# created: 2023-10-13T13:40:14+03:00
# public key: age1famrcgy36ke00r3vwpxhmf547d9xxc8snsaen50hfsx9k0xe5sdsmstyzf
AGE-SECRET-KEY-1LL2L450GYAF4EE9EJMEDX25T40WVP8JAEYCMYMQPRDEH25S558VSWYJR35
EOF
```

Установим кластерные аддоны (ingress-nginx-controller)
```bash
helmfile -e clusters/k0s apply
```

Дождёмся, пока поды ingress-controller поднимутся и перейдут в состояние Running
```bash
kubectl -n ingress-nginx get po
```

![](.readme/ingress_controller_pod_status.png?raw=true "Ingress Controller Pod Status")


Деплоим клиентские окружения
```bash
helmfile -e client-a/prod -n client-a-prod apply
helmfile -e client-b/prod -n client-b-prod apply
helmfile -e client-c/prod -n client-c-prod apply
```

Проверяем количество задеплоенных подов в каждый неймспейс
```bash
kubectl get po -A | grep ^client
```

![](.readme/get_clients_pods.png?raw=true "Get Clients Pods")

**client-a-prod** и **client-c-prod** имеют по одному экземпляру приложения, а у **client-b-prod** крутятся 2 инстанса - это соответствует тому, что задано в параметрах клиентских окружений

**simple-python-web-app** в **client-a-prod** и **client-c-prod** наследуют количество реплик приложений, заданных для всех окружений в файле `releases/simple-python-web-app.yaml.gotmpl`, а для **client-b-prod** их количество переопределяется в настройках окружения

![](.readme/pods_replicas.png?raw=true "Pods Replicas")

Проверяем, что возвращают клиентские инстансы
```bash
curl -s 'http://127.0.0.1' -H 'Host: simple-python-web-app.client-a-prod.example.com' | jq
curl -s 'http://127.0.0.1' -H 'Host: simple-python-web-app.client-b-prod.example.com' | jq
curl -s 'http://127.0.0.1' -H 'Host: simple-python-web-app.client-c-prod.example.com' | jq
```

![](.readme/curl_pods.png?raw=true "Curl Pods")

Ответы инстансов приложения также соответствуют тому, что задано в параметрах клиентских окружений

В **client-a-prod** и **client-b-prod** запущено приложение **simple-python-web-app** версии **1.0.0**, т.к. это задано для всех окружений, а для client-c-prod версия приложения задана **1.2.1**

Переменные окружения для приложения в каждом окружении заданы свои

![](.readme/pods_versions_envs.png?raw=true "Pods Versions And Envs")

**client_secret** у каждого окружения также задан свой в секретных переменных

![](.readme/pods_secrets.png?raw=true "Pods Secrets")

# Инструкции

## Переменные helm чартов

В `releases/<release-name>.yaml.gotmpl` задаются общие для всех окружений параметры helm чарта релиза

В `envs/values/*.yaml[.gotmpl]` задаются параметры helm чарта, которые оверрайдят общие параметры

В `envs/secrets/*.yaml` в зашифрованном виде задаются параметры helm чарта, которые оверрайдят и общие параметры, и параметры окружения из `values/`

## Шифрование секретов

Секреты хранятся в самом репозитории в зашифрованном виде. Для шифрования используется [плагин helm secrets](https://github.com/jkroepke/helm-secrets) и утилита [sops](https://github.com/getsops/sops). Шифровать можно gpg ключом, [age](https://github.com/FiloSottile/age) и другими способами (см. манаул по **sops**)

В файле `.sops.yaml` указывается публичный ключ/fingerprint вашего приватного ключа

## Как изменить секрет (зашифрованную переменную)

1. Добавить приватный ключ
```bash
mkdir -p ~/.config/sops/age
cat << EOF > ~/.config/sops/age/keys.txt
# created: 2023-10-13T13:40:14+03:00
# public key: age1famrcgy36ke00r3vwpxhmf547d9xxc8snsaen50hfsx9k0xe5sdsmstyzf
AGE-SECRET-KEY-1LL2L450GYAF4EE9EJMEDX25T40WVP8JAEYCMYMQPRDEH25S558VSWYJR35
EOF
```

2. Отредактировать секреты нужного окружения
```bash
helm secrets edit envs/client-a/prod/secrets/_all.yaml
```

## Деплой окружений

### Как задеплоить окружение `client-a/prod` в неймспейс `client-a-prod`

```
helmfile -e client-a/prod -n client-a-prod apply
```

## Как добавить новое окружение. На примере client-d/prod

1. Создать в папке envs следующую структуру папок:

```
client-d
└── prod
    ├── env.yaml
    ├── secrets
    └── values
        └── _all.yaml.gotmpl
```

> env.yaml - описание окружения
> 
> secrets - зашифрованные переменные хелмовых релизов
> 
> values - переменные хелмовых релизов

2. Описать параметры окружения и переменные приложений по примеру других окружений

3. Добавить окружение в `.helmfile/environments.yaml`

```
environments:
  client-d/prod:
    <<: *default
```

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

Список релизов для каждого окружения динамически формируется из словаря **apps** прогонкой через шаблон `.helmfile/releases.yaml`. Это сделано для абстрагирования от формата самого helmfile с целью упрощения

Например, такое описание **apps**

```yaml
apps:
  simple-python-web-app:
    repo: zzamtools
    chart: simple-python-web-app
    version: 1.0.0
    installed: true
```

превратится в такой список релизов, который ожидает получить helmfile

```yaml
releases:
- name: simple-python-web-app
  labels:
    app: simple-python-web-app
  chart: zzamtools/simple-python-web-app
  version: 1.0.0
  missingFileHandler: Info
  values:
    - releases/simple-python-web-app.yaml.gotmpl
    - releases/_override.yaml.gotmpl
  installed: true
```

### Переменные helm релиза

В `releases/simple-python-web-app.yaml.gotmpl` описаны общие для всех окружений переменные хелмового релиза

С помощью файла `releases/_override.yaml.gotmpl` переменные хелмового релиза конкретного окружения оверрайдят общие. Так сделано, чтобы в окружении можно было задавать переменные в одном файле вместо того, чтобы создавать для каждого релиза отдельный файл (когда в одном окружении десяток приложений, то неудобно бегать по десяткам файлов)

```yaml
simple-python-web-app:
  env:
    CLIENT_ID: "Client A"

simple-nodejs-app:
  replicas: 2
  image:
    tag: 2.5.3
  env:
    MARKET_ID: "cl-aaa000fff"
  ingress:
    enabled: true
```

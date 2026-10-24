# Локальная инфраструктура для работы над проектами с автоматизацией

- [Локальная инфраструктура для работы над проектами с автоматизацией](#локальная-инфраструктура-для-работы-над-проектами-с-автоматизацией)
  - [Требования](#требования)
  - [Последовательность запуска и развертывания](#последовательность-запуска-и-развертывания)
  - [Установка и настройка инструментов](#установка-и-настройка-инструментов)
    - [brew](#brew)
    - [helm, kubectl, mkcert](#helm-kubectl-mkcert)
  - [Gitea](#gitea)
    - [Локальный домен](#локальный-домен)
    - [Поддержка сгенерированного SSL в ingress-nginx контроллере](#поддержка-сгенерированного-ssl-в-ingress-nginx-контроллере)
    - [Развертывание Gitea](#развертывание-gitea)
    - [Проверка и решение проблем](#проверка-и-решение-проблем)
    - [Настройка Gitea](#настройка-gitea)

## Требования

- WSL2 в режиме mirrored network с оболочкой ZSH + OhMyZsh и Windows Terminal
- Powertoys для управления Windows hosts файлом
- Docker Desktop для контейнеризации и k8s
- Настроенный кластер k8s с nginx-ingress контроллером
- kubectl, helm для управления k8s и деплойментом
- mkcert утилита в Windows и WSL для генерации локальных доверенных сертификатов SSL
- VSCode с набором рекомендованных расширений:
  - Containe Tools
  - Docker
  - Docker DX
  - GitLens
  - Go
  - Helm Intellisense
  - Kubernetes
  - Makefile Tools
  - markdownlint
  - YAML
- Установленный в WSL Go компилятор с настроенными ENV в .zshrc

## Последовательность запуска и развертывания

1. Запуск WSL2
2. Запуск Docker Desktop c ожиданием готовности кластера
3. Ожидание готовности неймспейса Gitea

Сворачивать в обратном порядке с ожиданием выключения Docker Desktop. WSL выключать в Powershell:

```poweshell
wsl --shutdown
```

## Установка и настройка инструментов

После настройки и установки окружения из WSL и Docker Desktop можно переходить к развертыванию Gitea. Для этого нам понадобится:

- Локальные SSL Wildcard сертификаты, которым доверяет как WSL, так и Windows
- Домен для hosts - в конфигурациях используется *.dev.local
- Helm, kubectl для контроля развертывания Gitea

### brew

Пакетный менеджер требуемый для установки Helm и mkcert. Установка:

```bash
# Зависимости
sudo apt install build-essential

# Сам Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Прописать в .zshrc
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv zsh)"

# Применение в оболочке
source ~/.zshrc
```

### helm, kubectl, mkcert

helm - инструмент более гибкого управления обьектами k8s. Позволяет обьединять множество разнотианых ресурсов в "чарт" и управлять им через файлы переменных. Необходим для управления комплексами сложных развертываний с множеством зависимостей - вольюмы, базы, хранилища ключей и т.д. Основные определения:

- Чарт - пакет по функционалу аналогичный Homebrew формулам или .deb пакетам содержащий все определения ресурсов k8s нужных для развертывания
- Репозиторий - хранилище чартов
- Релиз - текущий запущенный в k8s экземпляр чарта, сконфигурированный при помощи values.yaml

```bash
brew install helm
```

kubectl - инструмент работы с кластером k8s. Позволяет работать с обьектами напрямую, удалять, перезапускать развертывания, считывать логи, получать данные об обьектах в неймспейсе:

```bash
# Вместо v1.37.0 версия кластера из Docker Desktop
curl -LO https://dl.k8s.io/release/v1.37.0/bin/linux/amd64/kubectl

# Сделать файл исполняемым
chmod +x ./kubectl

# Переместить в каталог с бинарными утилитами
sudo mv ./kubectl /usr/local/bin/kubectl

# Проверка
kubectl version --client
```

mkcert - инструмент генерации доверенных SSL сертификатов с локальнысм ЦС (CA или Центр Сертификации). Упрощщает создание и работу когда требуется работа в локали с подписанными CA SSL сертификатами нужно установить в обоих системах:

Для WSL

```bash
# Зависимости
sudo apt install libnss3-tools

brew install mkcert
```

Для Windows требуется пакетные менеджеры Chokolately или Scoop

```powershell
choco install mkcert

scoop bucket add extras
scoop install mkcert
```

docker и docker compose - не требуют установки т.к. прокидываются в WSL после запуска Docker Desktop

## Gitea

Для работы в локали будем использовать развертывание Gitea для k8s. Дальнейшие работы с проектами подразумевают работу с репозиториями в ней, настройку CI/CD пайплайнов через встроенный раннер, и развертыванние проекта в k8s. Перед развертыванием Gitea небходимо следующее:

### Локальный домен

Откройте Powertoys и запустите Open Hosts File Editor и внесите запись git.dev.local с IP 127.0.0.1. Далее сгенерируйте сертификаты в Windows и сохраните в папке юзера или другой доступной директории:

```powershell
mkcert -inastall

mkcert -cert-file dev.local.pem -key-file dev.local-key.pem "*.dev.local" "dev.local"
```

Эта команда создает доверяемый Windows и его браузерами локальный CA и Wildcard сертификаты для любых поддоменов третьего уровня (* указывается для этого). Далее нужно получить путь до центра сертификации и сделать его доверенным для WSL, таким образом сделав доверенными и подписанные им ранее сгенерированные сертификаты:

```powershell
mkcert -CAROOT
```

Далее в WSL нужно подставить Windows путь до CA в переменную CAROOT в WSL. В WSL Windows каталоги доступны через каталог /mnt/Диск. Например если вы получили предыдущей командой значение `C:\Users\test_user\AppData\Local\mkcert` то в WSL прописывается следуюющее:

```bash
export CAROOT=/mnt/c/Users/test_user/AppData/Local/mkcert/

mkcert -install
```

### Поддержка сгенерированного SSL в ingress-nginx контроллере

Для установки ingress-nginx в k8s кластере используйте helm чарт:

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Для текущей среды стандартных параметров достаточно и values.yaml не требуется. Для работы HTTPS и обращений к домену git.dev.local необходимо добавить ранее сгенерированные сертификаты как обьект-секрет неймспейса gitea для того чтобы указать секрет в tls секции ingress в values.yaml и разрешать HTTPS. Предварительно скопируйте в WSL сертификаты из Windows или укажите путь до Windows директории:

```bash
# Создайте неймспейс
kubectl create namespace gitea

# Создайте секрет, в --cert и --key либо /mnt/c/... пути, либо файлы в директории, в которой запускается команда
kubectl -n gitea create secret tls dev-local-tls --cert=dev.local.pem --key=dev.local-key.pem
```

### Развертывание Gitea

Конфигурация развертывания values.yaml для Helm находится в infra/k8s/gitea_helm_example/gitea-values.yaml. Она содержит конфигурации сервисов, домена, параметры Gitea и дефолтные логопасы администратора:

```bash
# Чарты Gitea
helm repo add gitea-charts https://dl.gitea.com/charts/

# Обновление
helm repo update

# Релиз Gitea по файлу с переменными, файл должен быть в директории, в которой выполняется команда, либо указывайте путь
helm install gitea gitea-charts/gitea -n gitea -f gitea-values.yaml
```

### Проверка и решение проблем

Для проверки используется kubectl, запросите текущие обьекты неймспейса и получите их состояния:

```bash
kubectl -n gitea get all

NAME                         READY   STATUS    RESTARTS       AGE
pod/gitea-977c4fd69-6m7r2    1/1     Running   0              139m
pod/gitea-postgresql-0       1/1     Running   2 (173m ago)   13d
pod/gitea-runner-runner-0    2/2     Running   4 (173m ago)   13d
pod/gitea-valkey-primary-0   1/1     Running   2 (173m ago)   13d

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/gitea-http              ClusterIP      10.96.91.245    <none>        3000/TCP         13d
service/gitea-postgresql        ClusterIP      10.96.109.198   <none>        5432/TCP         13d
service/gitea-postgresql-hl     ClusterIP      None            <none>        5432/TCP         13d
service/gitea-ssh               LoadBalancer   10.96.90.4      172.18.0.6    2222:30022/TCP   13d
service/gitea-valkey-headless   ClusterIP      None            <none>        6379/TCP         13d
service/gitea-valkey-primary    ClusterIP      10.96.245.199   <none>        6379/TCP         13d

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gitea   1/1     1            1           13d

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/gitea-5f74b64ff8   0         0         0       13d
replicaset.apps/gitea-6f864c8fdc   0         0         0       13d
replicaset.apps/gitea-977c4fd69    1         1         1       139m

NAME                                    READY   AGE
statefulset.apps/gitea-postgresql       1/1     13d
statefulset.apps/gitea-runner-runner    1/1     13d
statefulset.apps/gitea-valkey-primary   1/1     13d
```

Если все поды в статусе Running значит можно идти в браузер по <https://git.dev.local> и настраивать его.

### Настройка Gitea

Для дальнейшей работы нужно зайти в Gitea под логопасами администратора, перейти в Панель управления->Идентификация и доступ->Пользователи и создать новый аккаунт от которого будете работать. Снимите флаг Ограниченный после создания аккаунта. Вернитесь в Панель управления и перейдите в Действия->Раннеры для настройки раннера. Раннер - программа запуска задач CI/CD в изолированной среде с отслеживанием результата. Он создается разово на каждую задачу в пайплайне и в отдельной среде. В нашем случае это будет раннер в k8s изолирующий джобы при помощи dind - Docker-In-Docker. Для настройки и запуска раннера нужно Создать новый раннер и скопировать токен регистрации. Далее создадим из этого токена секрет и при помощи Helm релизним раннер в неймспейс Gitea. Его values.yaml находится в infra/k8s/gitea_helm_example/runner-values.yaml:

```bash
kubectl -n gitea create secret generic runner-secret --from-literal=token='токен регистрации'

helm install gitea-runner gitea-charts/actions -n gitea -f runner-values.yaml
```

Проверьте статус запуска раннера аналогично Gitea, после чего в разделе где был создан раннер должна появиться карточка раннера со статусом Простаивает. После этого залогиньтесь под ранее созданным юзером и создайте репозиторий. Для более удобной работы с git используйте SSH клонирование репозитория в WSL или Windows (зависит от вашей основной рабочей среды). Предварительно добавьте сгенерированный SSH ключ в Настройки->Ключи SSH/GPG для вашего рабочего пользователя.

# test-nginx-app

Тестовое приложение для дипломного практикума в Yandex.Cloud — **Александра Бужор**.

Статическая страница, которую отдаёт nginx в Docker-контейнере. Сборка образа и деплой
в Kubernetes полностью автоматизированы через GitHub Actions.

- Инфраструктурный репозиторий: https://github.com/AlexBuQA/netology-diplom
- Container Registry: `cr.yandex/crpfdk2dgogep9irkq8g`
- Образ приложения: `cr.yandex/crpfdk2dgogep9irkq8g/test-nginx-app`
- Приложение в кластере: http://93.77.176.75/app
- Namespace в кластере: `abuzhor-nginx`

---

## Состав репозитория

| Файл | Назначение |
|------|-----------|
| `index.html` | статическая страница (в ней указана версия приложения) |
| `nginx.conf` | конфиг nginx: отдаёт страницу на 80 порту, есть `/healthz` |
| `Dockerfile` | образ на базе `nginx:1.27-alpine` |
| `.github/workflows/build.yaml` | **CI**: сборка и пуш образа при каждом коммите в `main` |
| `.github/workflows/deploy.yaml` | **CD**: сборка версии и деплой в кластер при создании тега `v*` |

---

## Как это работает

**CI (`build.yaml`).** Любой коммит в ветку `main` → GitHub Actions логинится в Yandex
Container Registry по JSON-ключу сервисного аккаунта `cicd-sa-diplom`, собирает Docker-образ
и пушит его с двумя тегами: `latest` и `<sha коммита>`.

**CD (`deploy.yaml`).** Создание тега вида `v1.1` → собирается образ с тегом `:v1.1`,
пушится в реестр, после чего workflow подключается к кластеру по kubeconfig и обновляет
Deployment командой `kubectl set image`. Затем ждёт завершения rollout.

---

## Секреты репозитория

Настраиваются один раз: **Settings → Secrets and variables → Actions → New repository secret**.

| Секрет | Значение | Где взять |
|--------|----------|-----------|
| `YC_SA_KEY` | JSON-ключ сервисного аккаунта CI/CD (весь файл целиком) | `yc iam key create --service-account-name cicd-sa-diplom --output key.json` → скопировать содержимое `key.json` |
| `YC_REGISTRY_ID` | `crp...` | `terraform output container_registry_id` в папке `infrastructure` |
| `KUBE_CONFIG` | kubeconfig кластера, закодированный в base64 | `base64 -w0 ~/.kube/config` (на macOS: `base64 ~/.kube/config \| tr -d '\n'`) |

> ⚠️ **Важно про `KUBE_CONFIG`.** В `~/.kube/config` адрес сервера должен быть внешним:
> `https://93.77.176.75:6443`, а не `https://127.0.0.1:6443`. Иначе runner GitHub
> не достучится до кластера. Проверьте перед кодированием в base64:
> ```bash
> grep server ~/.kube/config
> ```

---

## Проверка CI (сборка образа)

```bash
git commit --allow-empty -m "trigger CI"
git push
```

Вкладка **Actions** → workflow `Build and Push image` должен стать зелёным.
В консоли Yandex Cloud → Container Registry → реестр `crpfdk2dgogep9irkq8g` →
репозиторий `test-nginx-app` появится образ с тегом `latest`.

---

## Проверка CD (выпуск новой версии)

1. Откройте `index.html`, найдите строку «Версия приложения» и поменяйте номер,
   например `v1.0` → `v1.1`.

2. Закоммитьте изменение:
   ```bash
   git add index.html
   git commit -m "bump version to v1.1"
   git push
   ```

3. Создайте и запушьте тег:
   ```bash
   git tag v1.1 -m "Release v1.1"
   git push origin v1.1
   ```

4. Workflow `Build and Deploy on tag` соберёт образ `:v1.1`, запушит его и обновит
   Deployment в кластере.

5. Проверьте результат:
   ```bash
   kubectl -n abuzhor-nginx get pods
   kubectl -n abuzhor-nginx describe deployment test-nginx | grep -i image
   ```
   И откройте http://93.77.176.75/app — на странице должна появиться версия v1.1.

> ⚠️ Deployment должен **уже существовать** в кластере (создаётся манифестами из
> репозитория `netology-diplom`, папка `k8s-configs`). `kubectl set image` только
> обновляет образ у существующего Deployment, но не создаёт его с нуля.

---

## Локальная проверка образа (необязательно)

```bash
docker build -t test-nginx-app:local .
docker run --rm -p 8080:80 test-nginx-app:local
# откройте http://localhost:8080
```

---

## Частые проблемы

| Симптом | Причина и решение |
|---------|-------------------|
| CI падает на шаге `yc-cr-login` | В секрет `YC_SA_KEY` попал не весь JSON или лишние пробелы. Скопируйте `key.json` целиком, включая фигурные скобки. |
| CI падает на `docker push` с `denied` | У сервисного аккаунта `cicd-sa-diplom` нет роли `container-registry.images.pusher`, либо в `YC_REGISTRY_ID` опечатка. |
| CD падает на `kubectl` с ошибкой сертификата | В сертификате API нет внешнего IP мастера. Нужен параметр `supplementary_addresses_in_ssl_keys` в Kubespray. |
| CD падает с `connection refused` / таймаутом | В `KUBE_CONFIG` остался адрес `127.0.0.1`. Замените на `93.77.176.75` и перекодируйте в base64. |
| Поды в кластере в статусе `ImagePullBackOff` | Образ ещё не собрался (дождитесь зелёного CI) либо у реестра нет анонимного доступа на pull. |

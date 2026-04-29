# Лабораторная работа №2 — CI/CD пайплайн для ML на Docker + GitLab

Студент разворачивает у себя локальный GitLab через Docker, регистрирует runner, собирает свой ML-пайплайн в `.gitlab-ci.yml` и прогоняет его в CI.

> **Что в этой папке — шаблон, а не готовое решение.**
> ML-скрипты (`data_collection.py`, `data_preprocessing.py`, `model_training.py`, `model_testing.py`) пишете вы сами по требованиям модуля 2 (см. корневой `README.md`). Они опираются на скрипты из lab1, которые уже у вас есть.
>
> Из коробки даны: инфраструктура локального GitLab (`gitlab-compose.yml`) и каркас пайплайна (`.gitlab-ci.yml`, `Dockerfile`, `requirements.txt`).
> В `.gitlab-ci.yml` блоки `script:` намеренно оставлены в виде `# TODO` + `exit 1` — заполните их вызовами своих скриптов, иначе job'ы не пройдут. Структура (`stages`, `image`, `before_script`, `artifacts`, `needs`) уже расставлена.

---

## Требования к пайплайну (что сдаётся)

В вашем `.gitlab-ci.yml` должны быть **минимум 3 обязательных стейджа**:

| Stage | Что должно происходить | Артефакт (рекомендуется) |
|-------|------------------------|--------------------------|
| `prepare-dataset` | Сбор и/или предобработка данных | `data/` |
| `train` | Обучение модели, сериализация в файл | `models/` |
| `test` | Прогон модели на тестовых данных, метрики | `logs/` или stdout |

---

## Структура папки

```
lab2/
├── README.md                   ← этот файл
├── Dockerfile                  ← пример образа python:3.12-slim + зависимости
├── .dockerignore
├── .gitignore                  ← исключает data/, logs/, models/
├── .gitlab-ci.yml              ← пример пайплайна: prepare-dataset → train → test
├── gitlab-compose.yml          ← локальный GitLab + Runner (НЕ часть проекта-лабы!)
└── requirements.txt            ← минимальный набор зависимостей (расширьте)
```

---

## Локальный прогон без GitLab (опционально)

Когда напишете свои скрипты — можно проверить пайплайн локально через Docker, без поднятия GitLab:

```bash
cd lab2
docker build -t lab2 .
docker run --rm -v "$(pwd)/_run:/app/_run" -w /app/_run lab2 bash -c "
  python /app/data_collection.py &&
  python /app/data_preprocessing.py &&
  python /app/model_training.py &&
  python /app/model_testing.py
"
```

Если всё отрабатывает — переходите к запуску в локальном GitLab.

---

# Запуск в локальном GitLab — проверка CI

> Локальный GitLab нужен только чтобы убедиться, что ваш `.gitlab-ci.yml` корректно прогоняется в реальном CI-окружении. **Сдаёт работу студент в GitHub** (Шаг 8) — преподавателю нужен PR в `mlops_practice` со скриншотами зелёного pipeline из локального GitLab.

> **Нужен Docker.** Если ещё не установлен:
> [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Windows/macOS) или
> `sudo apt install docker.io docker-compose-v2` (Ubuntu/Debian).
> Проверить: `docker --version && docker compose version`.
>
> **Потребуется ~4 ГБ RAM** для GitLab. Закройте тяжёлые приложения.

## Шаг 1. Поднять локальный GitLab

Скопируйте `gitlab-compose.yml` в **отдельную папку** (НЕ внутрь проекта-лабы), например `~/gitlab-local/`:

```bash
mkdir -p ~/gitlab-local && cp gitlab-compose.yml ~/gitlab-local/
cd ~/gitlab-local
```

Запустите:

```bash
docker compose -f gitlab-compose.yml up -d
docker compose -f gitlab-compose.yml logs -f gitlab
```

Ждите ~5 минут до строки `gitlab Reconfigured!`. Потом Ctrl+C — это просто выйдет из логов, GitLab продолжит работу.

Откройте http://localhost:8929. Должен открыться экран входа.

![SCREENSHOT-01-login-page](screenshots/01-gitlab-login-page.png)

> Обратите внимание: поля **Username or primary email** и **Password** в центре. Логотип GitLab сверху.

## Шаг 2. Войти как root

- **Логин:** `root`
- **Пароль:** `ChangeMe-2026!` (задан в `gitlab-compose.yml`)

После входа вы окажетесь на главной странице GitLab.

![SCREENSHOT-02-after-login](screenshots/02-gitlab-main-page.png)

> Обратите внимание: левое меню (Home / Projects / Groups / …) и кнопка **Admin** + аватарка в правом верхнем углу — значит вы вошли как root.

## Шаг 3. Создать новый проект

Меню слева → **Projects** → **New project** → **Create blank project**.

- Project name: `lab2-mlops`
- Visibility: **Private** (или Internal — не Public)
- **Снимите галку** «Initialize repository with a README» — мы зальём свои файлы.

Нажмите **Create project**.

![SCREENSHOT-03-new-project-form](screenshots/03-new-project-form.png)

> Обратите внимание: поле **Project name** = `lab2-mlops`, **Visibility Level** = Private (точка возле Private), под секцией **Project Configuration** галка **Initialize repository with a README** должна быть **снята**.

После создания GitLab покажет страницу проекта с инструкциями по push кода. Запомните URL вида `http://localhost:8929/root/lab2-mlops.git`.

![SCREENSHOT-04-empty-project](screenshots/04-empty-project-page.png)

> Обратите внимание: зелёный баннер «Project 'lab2-mlops' was successfully created», заголовок репозитория и блок **Command line instructions** с готовыми git-командами — `git remote add origin …` находится чуть ниже в этой же секции.

## Шаг 4. Зарегистрировать GitLab Runner

Runner — это процесс, который выполняет ваши job'ы. Он уже запущен в Docker (см. `gitlab-compose.yml`), но его нужно «привязать» к проекту.

### 4.1. Получить registration token

В вашем проекте: **Settings** (внизу слева) → **CI/CD** → секция **Runners** → **New project runner**.

![SCREENSHOT-05-runners-section](screenshots/05-settings-cicd-runners.png)

> Обратите внимание: раздел **Runners** раскрыт, в правом верхнем углу секции — кнопка **Create project runner**. Жмём её.

Заполните форму:
- **Tags:** `docker` (одна метка, важно для соответствия с `.gitlab-ci.yml`)
- **Run untagged jobs:** ✓ включить
- остальное — по умолчанию

Нажмите **Create runner**. GitLab покажет токен вида `glrt-xxxxxxxxxx` — **скопируйте его**.

![SCREENSHOT-06-runner-token](screenshots/06-runner-token.png)

> Обратите внимание: в секции **Step 1** — команда `gitlab-runner register --url … --token glrt-…`. Скопируйте токен (`glrt-…`) — он понадобится в следующей команде.

### 4.2. Зарегистрировать runner-контейнер

В терминале (там же, где `gitlab-compose.yml`):

```bash
docker exec -it gitlab-runner gitlab-runner register \
  --non-interactive \
  --url http://gitlab:8929/ \
  --clone-url http://gitlab:8929/ \
  --token <ВАША_ТОКЕН_ИЗ_GITLAB> \
  --executor docker \
  --docker-image python:3.12-slim \
  --description "local-docker-runner" \
  --docker-network-mode gitlab-net
```

> Внутри docker-сети `gitlab-net` имя сервиса `gitlab` уже резолвится в нужный контейнер через docker-DNS — поэтому runner и спавненные им job-контейнеры обращаются к GitLab по `http://gitlab:8929/`. С хост-машины (браузер, `git push`) GitLab доступен по `http://localhost:8929/` через проброс порта.

Перезагрузите страницу Runners в GitLab — runner должен появиться со статусом «online» (зелёный кружок).

![SCREENSHOT-07-runner-online](screenshots/07-runner-online.png)

> Обратите внимание: в строке runner'а слева — зелёный кружок (статус **online**) и тег `docker`.

## Шаг 5. Подготовить ветку `lab2` в вашем форке `mlops_practice`

Вы работаете **в своём GitHub-форке** репозитория `mlops_practice` (где уже лежит `lab1/`). Там `origin` уже настроен на GitHub:

```bash
cd /путь/к/вашему/mlops_practice
git remote -v
# origin   git@github.com:<ваш-логин>/mlops_practice.git  (fetch/push)
```

Создайте отдельную ветку для lab2 и положите в неё свои файлы — все в подпапку `lab2/`:

```bash
git checkout -b lab2/docker-gitlab-ci
# ... в подпапке lab2/ — ваши .py-скрипты, .gitlab-ci.yml, Dockerfile, requirements.txt
git add lab2/
git commit -m "lab2: ML-пайплайн на Docker + GitLab CI"
```

> **Не коммитьте** в этот репозиторий `gitlab-compose.yml` — это инфраструктура локального GitLab, она живёт у вас в отдельной папке `~/gitlab-local/`. В `lab2/.gitignore` уже исключены артефакты прогона (`data/`, `logs/`, `models/`).

## Шаг 6. Привязать репозиторий к локальному GitLab и запустить pipeline

### Зачем здесь две разные команды push

У вас **один локальный репозиторий** (`mlops_practice`) и **два разных удалённых**:

| Куда | Зачем | Что должно туда уехать |
|------|-------|------------------------|
| **GitHub** (`origin`) — ваш форк `mlops_practice` | сдать преподавателю | весь репо: `lab1/`, `lab2/`, `lab3/`, … |
| **локальный GitLab** (`gitlab`) — проект `lab2-mlops` | прогнать CI | **только** содержимое `lab2/`, причём в КОРНЕ удалённого репо |

Два разных «контракта» — поэтому и команды push разные:

* В **GitHub** уходит всё как есть: `git push origin <ветка>` — обычный push, без хитростей.
* В **локальный GitLab** нужно отправить только подпапку `lab2/`, *подняв* её содержимое в корень удалённого репо. Иначе там окажется `lab2/.gitlab-ci.yml`, GitLab его не найдёт (он ищет `.gitlab-ci.yml` в корне) и pipeline не запустится. Для этого есть готовая команда — `git subtree push --prefix=lab2 gitlab main`.

`git subtree push` берёт коммиты, которые трогали `lab2/`, переписывает их так, будто `lab2/` всегда был корнем, и пушит в указанный remote. Ваш локальный репо при этом **не меняется** — переписывание происходит «в воздухе», только для отправки.

> Это ровно тот же приём, которым в open-source выкладывают папку из монорепо в самостоятельный публичный репозиторий.

### 6.1. Добавить второй remote

В той же папке вашего форка `mlops_practice` добавьте `gitlab` рядом с GitHub'овским `origin`:

```bash
git remote add gitlab http://root@localhost:8929/root/lab2-mlops.git
git remote -v
# origin   git@github.com:<ваш-логин>/mlops_practice.git           (fetch/push)
# gitlab   http://root@localhost:8929/root/lab2-mlops.git          (fetch/push)
```

### 6.2. Push содержимого `lab2/` в локальный GitLab

```bash
git subtree push --prefix=lab2 gitlab main
# Username: root
# Password: ChangeMe-2026!
```

Что произойдёт:
1. Git переберёт коммиты, в которых менялись файлы внутри `lab2/`.
2. Создаст «синтетическую» историю, где каждый коммит — то же самое, но с `lab2/` в качестве корня.
3. Эту синтетическую историю запушит в `gitlab/main` (default-ветка проекта `lab2-mlops`).

В UI лок. GitLab вы увидите, что в проекте `lab2-mlops` лежат **прямо в корне**: `.gitlab-ci.yml`, `Dockerfile`, `requirements.txt`, ваши `.py` — без подпапки `lab2/`.

GitLab сразу запустит pipeline. Перейдите в проект → **Build** → **Pipelines**.

> **Удобный alias** для повторных правок (один раз настроить):
>
> ```bash
> git config alias.lab2-push '!git push gitlab "$(git subtree split --prefix=lab2 HEAD)":main --force'
> ```
>
> Дальше любая итерация — просто `git lab2-push`. Это нужно потому что чистый `git subtree push` иногда падает с `non-fast-forward`, если вы делали `git rebase` или `git commit --amend`. Алиас всегда форсит — для одноразового тестового проекта в локальном GitLab это безопасно.

## Шаг 7. Проверить результаты pipeline

Дождитесь завершения всех стейджей (~2-3 минуты на первом прогоне из-за `pip install`). Все должны быть зелёными:

![SCREENSHOT-08-pipeline-success](screenshots/08-pipeline-all-green.png)

> Обратите внимание: статус **Passed** (зелёный) и в колонке **Stages** — три зелёных кружка (`prepare-dataset`, `train`, `test`).

Кликните на job `test` → справа **Job artifacts** → **Browse** — увидите файлы артефактов (`logs/`, `models/`):

![SCREENSHOT-09-test-job-log](screenshots/09-test-job-log.png)

> Обратите внимание: в логе job `test` — строки с метриками (`Accuracy`, `Precision`, `Recall`, `F1 Score`, `ROC AUC`) и финальная строка `Job succeeded`.

![SCREENSHOT-10-artifacts-browse](screenshots/10-artifacts-browse.png)

> Обратите внимание: вверху бейдж `passed`, ниже — список файлов в `logs/`: `evaluation_report.txt` и `testing.log`. Каждый можно скачать кнопкой справа.

## Шаг 8. Push в GitHub — сдать на проверку

Когда pipeline в локальном GitLab зелёный, **отправьте ветку в GitHub-форк** для финальной сдачи преподавателю — это уже обычный push без subtree:

```bash
git push origin lab2/docker-gitlab-ci
```

Здесь нам нужно отправить **весь репозиторий целиком** (с `lab1/`, `lab2/`, `lab3/`…), как обычно — поэтому никакого `--prefix` не нужно.

Откройте PR из ветки `lab2/docker-gitlab-ci` в свой форк или в upstream `mlops_practice` (по правилам преподавателя). В описание PR приложите скриншоты успешного pipeline из локального GitLab (`08-pipeline-all-green.png` и т.п.) — это и есть подтверждение, что CI отработал.

> **Что НЕ должно попасть в GitHub:** `gitlab-compose.yml`, ваш `runner registration token` (`glrt-...`), пароль `ChangeMe-2026!`. Это про ваше тестовое окружение, преподавателю оно не нужно.

---

## Каркас `.gitlab-ci.yml` — что в каком stage

| Stage | Что | Артефакты (пример) |
|-------|-----|--------------------|
| `prepare-dataset` | Сбор + предобработка данных | `data/` |
| `train` | Обучение модели | `models/` |
| `test` | Метрики и отчёт | `logs/` |

Пути короткие — без префикса `lab2/`, потому что после `git subtree push --prefix=lab2 ...` подпапка `lab2/` в локальном GitLab разворачивается в корень проекта.

`needs:` между job'ами выстраивает DAG, поэтому `train` дожидается `prepare-dataset`, `test` — обоих.

---

## Если что-то пошло не так

| Проблема | Решение |
|----------|---------|
| GitLab не открывается на 8929 | Подождите ещё минуту, GitLab инициализируется до 5 мин на первом старте |
| Runner offline в UI | Проверьте `docker logs gitlab-runner` |
| `git subtree push` падает с `Updates were rejected (non-fast-forward)` | Используйте alias из Шага 6.2 (`git lab2-push`) — он делает force push в `gitlab/main`. Для одноразового тестового проекта это безопасно |
| `git: 'subtree' is not a git command` | На некоторых минимальных сборках git нужен отдельный пакет: `sudo apt install git-subtree` (Debian/Ubuntu); в Git for Windows и macOS он входит из коробки |
| Pipeline не запустился, файлы в lab2-mlops лежат в подпапке `lab2/` | Вы запушили обычным `git push gitlab HEAD:main` вместо `git subtree push --prefix=lab2 ...`. Удалите проект lab2-mlops в GitLab, создайте заново и пушьте через subtree |
| Job упал с `Could not resolve host: gitlab` | При регистрации runner забыли `--docker-network-mode gitlab-net`, перерегистрируйте |
| Job упал на `pip install` (сетевая ошибка) | Может тормозить сеть к pypi; перезапустите job (Retry) |
| Push спрашивает пароль и не принимает | Используйте логин `root` и пароль `ChangeMe-2026!` (можно поменять в Settings → Profile) |

---

## Сворачивание окружения

```bash
cd ~/gitlab-local
docker compose -f gitlab-compose.yml down          # выключить
docker compose -f gitlab-compose.yml down -v       # выключить и удалить ВСЕ данные GitLab
```

---
---

<https://github.com/Lakr233/GitLab-License-Generator>
Установка Gitlab-EE и его Апдейт.
### Интро.
>[!info]
Т.к. установка производится в закрытый контур, нам необходимо скачать deb пакет с ресурса по ссылке ниже
Из России он недоступен, используем VPN.

>[!tip]
>В этом примере:
>Устанавливаем GitLab версии 16.3.0 и в дальнейшем производим апдейт до актуальной.
>ОС на целевой машине Ubuntu 22 jammy.
>IP целевой машины 10.10.11.205, пользователь ubuntu с возможностью sudo.
```bash
https://packages.gitlab.com/gitlab/gitlab-ee
```
### Установка.

Находим версию для наших операционной системы и архитектуры:
Скачиваем пакет по ссылке на странице:
```
wget --content-disposition https://packages.gitlab.com/gitlab/gitlab-ee/packages/ubuntu/jammy/gitlab-ee_16.3.0-ee.0_amd64.deb/download.deb
```
Необходимо скопировать установочный пакет на целевую машину:
Устанавливаем пакет:
```
ubuntu@gitlab-ee:~$ sudo EXTERNAL_URL="http://gitlab.alfastrah.ru" dpkg -i gitlab-ee_16.3.0-ee.0_amd64.deb
```
Пароль от root пользователя находится в файле:
```
/etc/gitlab/initial_root_password
```
### Активация.
Для активации нам будет нужен архив с кодом - GitLab-License-Generator.tgz
Распаковываем и запускаем ./make.sh
В результате будут созданы 3 файла:
```
license_key
license_key.pub
result.gitlab-license
```
Содержимым файла license\_key.pub необходимо заменить файл
```
/opt/gitlab/embedded/service/gitlab-rails/.license_encryption_key.pub
```
После замены - провести реконфигурацию и рестарт GitLab:
```
ubuntu@gitlab-ee:~$ sudo gitlab-ctl reconfigure
ubuntu@gitlab-ee:~$ sudo gitlab-ctl restart
```
Далее необходимо вставить содержимое result.gitlab-license через web-интерфейс и активировать лицензию нажав кнопку Add License:
```
Admin Area -> Settings -> General -> Add License -> Enter License Key
```
Файл  license\_key.pub необходимо сохранить, он понадобиться для реактивации GitLab после обновления.
### Обновление.
Для обновления нужно скачать актуальный deb-пакет с и скопировать на целевую машину.
```
https://packages.gitlab.com/gitlab/gitlab-ee
```
И запустить установку:
```
sudo dpkg -i gitlab-ee_16.4.1-ee.0_amd64.deb
```
После установки новой версии нужно дождаться завершения Background Migrations:
```
Admin Area -> Monitoring -> Background Migrations
```
Далее провести реактивацию заменой содержимого файла
```
/opt/gitlab/embedded/service/gitlab-rails/.license_encryption_key.pub
```
на сохранённый файл license\_key.pub

И произвести реконфигурацию и рестарт:
```
ubuntu@gitlab-ee:~$ sudo gitlab-ctl reconfigure
ubuntu@gitlab-ee:~$ sudo gitlab-ctl restart
```
После этого лицензия будет снова доступна:
2\. Настройка approvers при merge request обязательная.

Если у вас gitlab ultimate, после всех манипуляции, то есть функция codeowners. Тоже настроить можно.
Плюс можно в настройках mr выставить мин кол-во approvers
```bash
variables:
  # This variable for docker/build.yml to find abuse of CI_TMPL_ENVIRONMENT_NAME
  # Default value which equals to $CI_CONFIG_PATH is useless when check-MR-approve.yml is used in project CI/CD settings
  PROJECT_GITLAB_CI_YML_PATH: ".gitlab-ci.yml"

include:
  - project: $CI_PROJECT_PATH
    file: '.gitlab-ci.yml'
    ref: $CI_COMMIT_SHA

variables:
  ENABLE_OWNERS: false

check MR approve into dev:
  rules:
    - if: $ENABLE_OWNERS == "false"
      when: never
    - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "dev"
  before_script:
    - NEED_VOTES=${CI_MR_APPROVE_NEED_VOTES_INTO_DEV:-2}
  extends:
    - .check-mr-approve

check MR approve into main/master:
  rules:
    - if: $ENABLE_OWNERS == "false"
      when: never
    - if: $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main" || $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "master"
  before_script:
    - NEED_VOTES=${CI_MR_APPROVE_NEED_VOTES_INTO_MAIN:-2}
  extends:
    - .check-mr-approve

.check-mr-approve:
  # all options must be explicitly defined to protect them from malicious overriding
  allow_failure: false
  when: on_success
  stage: check approve
  image: harbor.wildberries.ru/devops/docker/alpine-bash:3.17
  tags:
    - linux-docker-executor
  variables:
    CI_API_V4_URL: $CI_API_V4_URL
    CI_MR_APPROVE_TOKEN: $CI_MR_APPROVE_TOKEN
    CI_MERGE_REQUEST_PROJECT_ID: $CI_MERGE_REQUEST_PROJECT_ID
    CI_MERGE_REQUEST_IID: $CI_MERGE_REQUEST_IID
    CI_OWNERS: $CI_OWNERS
    VOTES: 0
  script:
    - |
      if [[ "$CI_MR_APPROVE_TOKEN" == '$CI_MR_APPROVE_TOKEN' ]]; then
        echo "${ERR}CI_MR_APPROVE_TOKEN must be defined. You need to generate project access token with guest role and read_api permission scope."
        echo "To generate token visit $CI_SERVER_URL/$CI_PROJECT_PATH/-/settings/access_tokens"
        exit 1;
      fi
    - |
      if [[ "$CI_OWNERS" == '$CI_OWNERS' ]]; then
        echo -en "\033[41m\033[30m CI_OWNERS должна быть определена в Variables. Добавьте в нее gitlab юзернеймы апруверов \033[0m"
        exit 1;
      fi
    - owners=($(echo "$CI_OWNERS" | tr ' ' '\n'))
    - |
      if [[ ${#owners[@]} < $NEED_VOTES ]]; then
        echo -en "\033[41m\033[30m Назначено ${#owners[@]} апруверов из ${NEED_VOTES} необходимых. Добавьте апруверов в CI_OWNERS \033[0m"
        exit 1;
      fi
    - |
      if [[ ! MR_JSON_RESPONSE=$(curl --no-progress-meter --fail --request GET --header "PRIVATE-TOKEN: $CI_MR_APPROVE_TOKEN" ${CI_API_V4_URL}/projects/${CI_MERGE_REQUEST_PROJECT_ID}/merge_requests/$CI_MERGE_REQUEST_IID/approvals) ]]; then
        echo "${ERR}Failed to fetch data from Gitlab API. Check token in variable CI_MR_APPROVE_TOKEN have read_api permission scope. Also check Gitlab API availbility."
        exit 1
      fi
    - >
      count=$(curl --no-progress-meter --header "PRIVATE-TOKEN: $CI_MR_APPROVE_TOKEN"
      ${CI_API_V4_URL}/projects/${CI_MERGE_REQUEST_PROJECT_ID}/merge_requests/$CI_MERGE_REQUEST_IID/approvals | jq '.approved_by | length ')
    - >
      approvers=$(curl --no-progress-meter --header "PRIVATE-TOKEN: $CI_MR_APPROVE_TOKEN"
      ${CI_API_V4_URL}/projects/${CI_MERGE_REQUEST_PROJECT_ID}/merge_requests/$CI_MERGE_REQUEST_IID/approvals | jq '.approved_by[].user.username ')
    - approvers=$(echo "$approvers" | tr -d '"')
    - array=($(echo "$approvers" | tr ' ' '\n'))
    - |
      for i in ${array[@]}; do
        if [[ "$CI_OWNERS" == *"$i"* ]]; then
          VOTES=$((VOTES+1))
        fi
      done
    - echo -e "NEED APPROVES BY ${CI_OWNERS} \nAPPROVED BY ${approvers} \n"
    - |
      if [[ $VOTES < $NEED_VOTES ]]; then
        echo -e "\033[41m\033[30m У вас ${VOTES} апрувов из ${NEED_VOTES} необходимых. \033[0m \n"
        echo -e "\033[41m\033[30m Необходимы апрувы от ${CI_OWNERS} \033[0m"
        exit 1;
      fi
    - echo -e "\033[42m\033[30m APPROVES OK \033[0m"
```

За OWNERS логику отвечает джоба check MR approve into <branch\_name>.
Данная джоба проверяют через API гитлаба апрувы от заданных овнеров.
Если их количество больше порогового, то джоба завершается успешно, иначе с ошибкой, что должно блокировать возможность мержа.

Блокировка достигается с помощью опции в проекте, которая требует чтобы пайплайн завершился успешно чтобы можно было вмержить MR.
Для вмерживания Developer должен иметь ограниченные права на вмерживание MR в настройках проекта, подробнее в инструкции подключения ниже.

Если вы не хотите, например, давать разработчикам возможность мержить в main/master, но при этом хотите оставить возможность Maintainer'ам мержить по кнопке в UI, то необходимо поставить необходимое количество лайков в 0 с помощью переменной CI\_MR\_APPROVE\_NEED\_VOTES\_INTO\_MAIN.
Тогда джоба проверки всегда будет завершаться успешно не блокируя кнопку мержа.

Как подключить
Необходимо строго следовать данной инструкции. При отклонениях от неё есть потенциальный риск некорректной работы.

1\. Сгенерировать group/project access token
Необходимо создать access token для возможности джобы чтобы обращаться к API гитлаба. Групповой токен можно использовать, если хотите подключить систему апрувов к группе проектов без необходимости генерировать токен в каждом из них.

Перейти в настройки группы/проекта: Settings -> Access tokens
Дать имя токену (не важно какое)
Expiration date на ваше усмотрение
Роль - Reporter
Scopes - read\_api
Нажать create project access token
Сохранить сгенерированный токен для 3 шага
2\. Запретить мержить когда пайплайн завершается с ошибкой
Перейти в Settings -> Merge requests -> Merge checks. Поставить галку в графе Pipelines must succeed.

3\. Настроить CI/CD переменные группы/проекта

Перейти в настройки группы/проекта: Settings -> CI/CD -> Variables
Создать переменные. Из обязательных: CI\_MR\_APPROVE\_TOKEN, CI\_OWNERS
CI\_OWNERS - строка, содержащая gitlab %username% апруверов (посмотреть можно в профиле gitlab). Апруверов должно быть >= порогового значения для апрува.
CI\_MR\_APPROVE\_TOKEN - токен, сгенерированный на 1 шаге, обязателен для работы системы. Для безопасности необходимо активировать пункт Mask variable.
CI\_MR\_APPROVE\_NEED\_VOTES\_INTO\_DEV - числовое значение, задаёт пороговое значение для апрува MR в dev. По умолчанию 2.
CI\_MR\_APPROVE\_NEED\_VOTES\_INTO\_MAIN - числовое значение, задаёт пороговое значение для апрува MR в main/master. По умолчанию 2.

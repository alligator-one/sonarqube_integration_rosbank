
**Проверка кода в SonarQube**

Проверка кода в SonarQube осуществляется после создания Pull request в Bitbucket или после запуска задачи Jenkins (JENKINS_SERVER/job/abpm/job/Experiment/job/ExamplePipeline/job/Manual/). 

Результаты проверки доступны на Dashboard  в SonarQube.  

Название отчета в SonarQube: SMALL::<имя репозитория или имя приложения>::<имя модуля>::<Номер PullRequest>.

После проверки коммиттеру отправляется email со ссылками на результаты анализа.

**Устройство pipeline Jenkins**

Pipeline включает в себя следующие разделы:

*Checkout cicd*  - получение исходного кода pipeline, очистка jenkins workspace

*Checkout source to check* - получение исходного кода Java для проверки в SonarQube

*Build and run docker* - подготовка исходного кода для анализа, сборка и запуск докера, отправка файлов на анализа

*clean workspace*  - очистка jenkins workspace


**GenericTrigger**

В этом разделе происходит обработка webhook из Bitbuckt, из полученного Json парсятся следующие переменные:

*Repo* - Название репозитория

*ownerName* - Название проекта

*Branch* - Название ветки

*PullRequest*- Номер pullrequest

*Email* - Email коммитера

**environment**

Переменные окружения, необходимые для работы pipeline.

*BITBUCKET_BASE_URL*	- URL репозитория

*URL_SONAR* - URL сканера для SonarQube

*SONAR_SCANNER* - Название и версия сканера SonarQube

*DOCKER_REGISTRY*- URL репозитория образов Docker

*DEFAULT_CICD* - Путь по умолчанию для настроек CI-CD

*CICD_BRANCH* - Ветка, в которой хранится pipeline

*EXCLUSIONS* - Исключаемые из анализа типы файлов

*DEFAULT_JAVAVERSION*	- Версия Java по умолчанию

*DEFAULT_GRADLE* - Путь по умолчанию для build.gradle

*NOT_FIND_MODULES* - Каталоги, исключаемые из поиска build.gradle

**Настройка вебхука в Bitbucket**
Настройка осуществляется на уровне проекта. в разделе *Post Webhooks*.

В поле *Title* задается название вебхкука.

В поле URL задается *URL* вебхука: JENKINS_SERVER/generic-webhook-trigger/invoke?token=<токен>

В поле *Exclude repositories* задаются репозитории, которые не должны сканироваться при pull request.

В поле *Committers to Ignore* задаются логины коммитеров, которые будут игнорироваться при создании pull request.

**Настройка поддержки вебхука в Jenkins**

Необходимо включить *Generic Webhook Trigger* в разделе Build Triggers. В поле *Token* необходимо указать тот же токен, который был указан в токене вебхука в Bitbucket.


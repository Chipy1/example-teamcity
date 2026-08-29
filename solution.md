# Решение домашнего задания «Teamcity»

## Как это было устроено

Задание по идее должно было жить в Yandex Cloud: три виртуалки, возня с флоатинг-ипшниками, гарант тает на глазах. Я честно посмотрел на эту перспективу и подумал что оно не надо, когда комп на Arch Linux, и всю возню с виртуалками можно заменить локальным решением.
(Я несколько часов траблшутил почему не подключаюсь к виртуалкам на yandex cloud, и сдался. С ключами возился. Возился вообще много с чем чтобы получить доступ. Не выходило, никак.)

Все три героя (TeamCity server, агент и Nexus) живут в одной docker-сети `teamcity-net` и обращаются друг к другу по именам сервисов. Никаких залипающих публичных IP, всё локально.

| Компонент | Образ | Адрес |
|---|---|---|
| Nexus | `sonatype/nexus3:3.78.2` | `http://localhost:8081` (внутри сети: `http://nexus:8081`) |
| TeamCity Server | `jetbrains/teamcity-server:latest` | `http://localhost:8111` (внутри сети: `http://teamcity-server:8111`) |
| TeamCity Agent | `jetbrains/teamcity-agent:latest` | `SERVER_URL=http://teamcity-server:8111` |

Стенд целиком: `teamcity/docker-compose.yml`.

Пара фактов:

- TeamCity Server 2026.1.3 (build 222742), данные в volume `teamcity-server-data`.
- Форк живой: `https://github.com/Chipy1/example-teamcity.git`.
- Локальная рабочая копия лежит в `new/`, там же `settings.xml` для Maven.
- Авторизация на сервере: администратор `alex`. Сервер выдаёт токен суперпользователя в логах:
  `docker logs teamcity-server | grep "Super user authentication token"`.

## Подготовка (локально вместо облака)

1. **TeamCity server** поднят в контейнере, прошёл мастер первичной настройки, завёлся администратор.
2. **Агент** - контейнер с `SERVER_URL`, авторизован вручную через браузер.
3. **Форк** `example-teamcity` склонирован в `new/` и подчищен до исходного состояния.
4. **Nexus** поднят свежим, в нём `maven-releases` и `maven-snapshots`.
5. **VCS root** создан в TeamCity, URL указывает на форк. (Собственно, тот самый VCS root, из-за которого мы ещё всплакнём в п. 13.)

## Основная часть

### 1. Проект на основе форка

Создан проект `Example`, внутри него build configuration `Build` (`Example_Build`) c Maven runner.

### 2–3. Autodetect, шаги, первая сборка master

Maven runner получил `pom.xml` и цель `clean test`. Первая сборка по `master` - SUCCESS, `Tests passed: 5`. Запахло победой.

### 4. Условия сборки по веткам

Задача была: `master` → `mvn clean deploy`, любая другая ветка → `mvn clean test`. Для этого два Maven runner'а с условиями на признак «это дефолтная ветка»:

| Runner | Условие | Цель |
|---|---|---|
| `Maven deploy (master)` | `teamcity.build.branch.is_default` = `true` | `clean deploy` |
| `Maven test (other branches)` | `teamcity.build.branch.is_default` = `false` | `clean test` |

В нотации TeamCity это выглядит так:

```xml
<runner id="RUNNER_6" name="Maven deploy (master)" type="Maven2">
  <conditions>
    <equals name="teamcity.build.branch.is_default" value="true" />
  </conditions>
  ...
</runner>
<runner id="RUNNER_7" name="Maven test (other branches)" type="Maven2">
  <conditions>
    <equals name="teamcity.build.branch.is_default" value="false" />
  </conditions>
  ...
</runner>
```

Первое время условия были прописаны как `equals ... value=false` с двумя элементами `condition` в одном блоке - и Maven упорно не понимал, что от него хотят. Починилось переписыванием на `does-not-equal` и дальше на корректный подход «два шага с родным условием». Звучит проще, чем было.

### 5. settings.xml в TeamCity

`teamcity/settings.xml` (дублируется в корень репо как `new/settings.xml`) несёт креды для загрузки в Nexus:

```xml
<server>
  <id>nexus</id>
  <username>admin</username>
  <password>admin123</password>
</server>
```

В шаге `Maven deploy (master)` прописан `userSettingsPath=settings.xml` - Maven берёт именно этот файл и авторизуется на Nexus под `<id>nexus</id>`.

### 6. Правки pom.xml

В `pom.xml` прописан наш локальный Nexus:

```xml
<distributionManagement>
  <repository>
    <id>nexus</id>
    <url>http://nexus:8081/repository/maven-releases</url>
  </repository>
</distributionManagement>
```

`nexus` - имя docker-сервиса, агент добирается до него внутри сети `teamcity-net` без всяких внешних адресов. Плюс подключён `maven-shade-plugin`, чтобы собирался исполняемый `jar` с mainClass `plaindoll.HelloPlayer` - он пригодится на шагах 16-17.

### 7. Сборка по master и артефакт в Nexus

По `master` стартует `clean deploy`: Maven с `settings.xml` молча заходит на Nexus под `admin/admin123` и публикует `org.netology:plaindoll:0.0.2` в `maven-releases`.

Проверка curl'ом:

```bash
curl -u admin:admin123 \
  "http://localhost:8081/service/rest/v1/search?repository=maven-releases"
```

Артефакт на месте.

### 8. Миграция build configuration в репозиторий

Включены Versioned Settings: конфигурация проекта начинает жить в VCS.

```
.teamcity/
└── Example/
    ├── project-config.xml
    ├── buildTypes/
    │   └── Example_Build.xml
    └── vcsRoots/
        └── Example_GitHub.xml
```

Дальше любая правка в TeamCity сама коммитится в ветку. Об этом нам напоминала серия навязчивых коммитов `Build feature added`, `Build feature replaced`, `VCS root changed`.

### 9–12. Ветка feature/add_reply

От `master` отпочковалась `feature/add_reply`, в `src/main/java/plaindoll/Welcomer.java` появился новый метод:

```java
public String sayReply() {
    return "The hunter waits for a reply.";
}
```

А в `src/test/java/plaindoll/WelcomerTest.java` - тест:

```java
@Test
public void welcomerSaysReply() {
    assertThat(welcomer.sayReply(), containsString("hunter"));
}
```

Слово `hunter` есть и в реплике, и в assert'е - задание закрыто «в лоб». Пушим в `origin/feature/add_reply`. По `branchFilter +:*` TeamCity видит новую ветку и собирает её.

### 13. Автосборка по не-default ветке

Сборка по `feature/add_reply` (build `#203`): сработал шаг `Maven test (other branches)` (`clean test`), тесты - зелёные.

**Боль.** VCS root поначалу упорно молчал: веток не видел, фич не РАЗумел. Оказалось, параметр спецификации веток был записан под неверным ключом - `branchSpec` вместо правильного `teamcity:branchSpec`. Из-за этого TeamCity хранил правило, но не применял его, и в `repositoryState` был одинокий `master` без продолжения. После сохранения через UI параметр принял правильное имя - и волшебным образом в состоянии репозитория появились обе ветки: `refs/heads/feature/add_reply` и `refs/heads/master`. Мораль: имя параметра - это 90% победы.

### 14. Merge в master

`feature/add_reply` торжественно вливается в `master` через merge-коммит:

```bash
git checkout master
git merge feature/add_reply
git push origin master
```

### 15. Нет артефакта по master

После merge автосборка не запустилась - VCS-trigger в этой схеме не настроен, сборку по ветке я запускал вручную (см. «Известные нюансы», п. 2). Поэтому факт «без правил артефактов мастер ничего не публикует» проверен на `#202` - это мастер-сборка, которая прошла ещё до merge (18:50 UTC): правило артефактов не задано, и в табе артефактов честный `files count="0"`. Следующий мастер-прогон (`#204`, 23:08 UTC) состоялся уже после добавления правил - о нём в п. 17 ниже.

### 16. Сборка jar в артефакты

Общими настройками build configuration добавлено правило:

```
target/*.jar
```

Хранится как `<option name="artifactRules" value="target/*.jar" />` в `.teamcity/Example/buildTypes/Example_Build.xml` (TeamCity 2026). Приятный бонус: TeamCity сам закоммитил это изменение в репозиторий (коммит `1d1b9d4`) - версионные настройки работают как часы.

### 17. Повторная сборка master

Сборка `#204` по `master`: `clean deploy` SUCCESS. В артефактах лежат `plaindoll-0.0.2.jar` (shaded, с mainClass) и `original-plaindoll-0.0.2.jar`. Тот же jar синхронно появился и в Nexus.

### 18–19. Конфигурации в репозитории

Вся конфигурация (`project-config.xml`, `buildTypes/*.xml`, `vcsRoots/*.xml`) уже в VCS, и TeamCity сам подтягивает и публикует правки. Финальная сверка трёх файлов между `data/teamcity_server/datadir/config/projects/Example/` и `.teamcity/Example/` в `master` дала пустой `diff` - один в один, никакой магии.

## Итог

- Стенд: `teamcity-server`, `teamcity-agent`, `nexus` в сети `teamcity-net`.
- Проект `Example`, build configuration `Example_Build`.
- Ветки `master` и `feature/add_reply`, сборки по обеим - SUCCESS.
- Артефакт `0.0.2` спокойно лежит в Nexus `maven-releases`.
- Вся конфигурация TeamCity живёт в VCS и управляется REST API.

## Аа... нюансы

1. **Nexus не любит перезапись release-версий.** Если понадобится класть ту же `0.0.2` повторно - либо включаем `allow redeploy` в настройках репозитория, либо поднимаем версию в `pom.xml`.
2. **Настоящий автозапуск.** В текущей схеме сборку по нужной ветке я запускал REST-запросом, а branchFilter `+:*` отвечал лишь за то, чтобы ветка была видна. Для полностью безрукого CI добавился бы обычный VCS-trigger в конфигурацию.

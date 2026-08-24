---
name: api-contract-automation
description: Рабочая инструкция для автоматизации контрактных API-тестов в msg-tests по OpenAPI/Jira/Wiki/msg-docs и существующему стилю проекта.
---

# Автоматизация контрактных API-тестов

Используй этот skill, когда нужно автоматизировать контрактные тест-кейсы API в `msg-tests`.

Главная цель: тест должен проверять контракт ответа и ключевое поведение эндпоинта через понятные expected-объекты, а не через длинные цепочки `assertEquals` по отдельным JSON-строкам.

## 1. Входной контекст

Перед кодом собери факты:

1. Прочитай актуальные инструкции:
   - корневой `AGENTS.md`;
   - `msg-tests/AGENTS.md`;
   - `tasks/AGENTS.md`, если работа идет из задачи.
2. Обнови рабочую ветку `msg-tests`, если пользователь это просит или задача явно продолжает общую ветку.
3. Изучи источники контракта:
   - OpenAPI в `msg-docs`;
   - Jira-задачу/эпик;
   - Wiki/Confluence, если пользователь указал страницу;
   - уже подготовленный `.md` с тест-кейсами, если он есть.
4. Найди соседние тесты в том же API-пакете и повторяй их стиль.

Если Jira/Wiki недоступны, не блокируйся бесконечно: явно опирайся на доступные `msg-docs`/локальные тест-кейсы и скажи, какой источник не удалось открыть.

## 2. Где размещать тесты

Выбирай пакет по API и предметной области:

- `src/test/java/ru/wsoft/tests/api/adm/...` для `admin-console-api`/`acapi`;
- `src/test/java/ru/wsoft/tests/api/lk/...` для `lk-api`;
- `src/test/java/ru/wsoft/tests/api/broker/...` для `broker-api`;
- аналогично существующей структуре проекта.

Если эндпоинты делятся по каналам или типам сущности, создавай отдельные подпакеты и классы, например:

- `template/general/sms/SmsTemplateContractTests`;
- `template/general/call/CallTemplateContractTests`;
- `template/general/push/PushTemplateContractTests`.

Не создавай общий abstract/base test-класс только ради устранения дублирования, если такой стиль не принят в соседних тестах или пользователь явно против. В `msg-tests` предпочтительнее самостоятельные тестовые классы, которые наследуются только от принятого базового класса (`BaseApiTests`, `BaseBrokerApiTests` и т.п.).

## 3. Контрактные DTO

Сначала проверь, есть ли готовые contract-классы в `ru.wsoft.tests.framework.json.contracts`.

Если DTO уже есть:

- используй их через `JsonContractWorker<T>`;
- не дублируй классы в тестовом пакете.

Если DTO нет:

- обычно не изменяй `framework`, потому что это ограничено `msg-tests/AGENTS.md`;
- если пользователь явно разрешил исключение, добавь DTO в `ru.wsoft.tests.framework.json.contracts.<api>.<domain>`, разбивая по смысловым директориям.

DTO должны быть простыми:

- Jackson-аннотации `@JsonProperty` для snake_case;
- `@NoArgsConstructor` для маппинга;
- constructor для expected-объектов;
- getters/setters, если DTO следует стилю существующих контрактов;
- `@JsonIgnoreProperties(ignoreUnknown = true)`, если тест фиксирует только часть большого API-ответа.

Для каналов можно использовать общий DTO с общими полями и channel-specific наследников:

```java
public class SmsTemplateGetContract extends TemplateGetContract {
    public SmsTemplateGetContract() {
        super();
    }

    public SmsTemplateGetContract(/* common fields */) {
        super(/* common fields */);
    }
}
```

## 4. Подготовка данных

Используй существующие API фреймворка:

- `APIHandler` для HTTP-запросов;
- `CreateMethodsAcapi`/`DeleteMethodsAcapi` для предусловий и очистки;
- builder-модели из `ru.wsoft.tests.framework.fast.models`;
- `SimpleSqlQueriesWorker` или `msg` только для получения id/проверки созданных сущностей, когда нет удобного API.

Не пиши сырые JSON-тела вручную, если уже есть builder-модель. Правильный порядок:

1. Собрать модель через builder.
2. Сериализовать через `Mapper.toJson(...)` или `JsonMapperHolder.MAPPER`.
3. Если модель временно не поддерживает новый блок контракта, аккуратно добавить недостающие поля через `ObjectNode`.

Пример временного расширения body без изменения builder-модели:

```java
ObjectNode body = (ObjectNode) JsonMapperHolder.MAPPER.readTree(Mapper.toJson(new SmsTemplate.Builder()
        .withPartner(PARTNER_NAME)
        .withPartnerUser(PARTNER_USER_NAME)
        .withName(name)
        .withMessage(message)
        .build()));
body.set("imsi_settings", JsonMapperHolder.MAPPER.valueToTree(new TemplateCheckSettingsGetContract(true, "block", null, null)));
```

Не добавляй поля в request body просто потому, что они есть в GET-контракте. Например, если API не ожидает `is_active` при создании/редактировании шаблона, не вставляй `is_active` в `templateBody`.

## 5. Структура тестового класса

Держи класс самостоятельным и читаемым:

- константы имен сущностей сверху;
- `APIHandler`, `CreateMethods...`, `DeleteMethods...` как поля;
- `@BeforeEach` создает предусловия;
- `@AfterEach` удаляет созданные сущности;
- тестовые методы идут по контракту эндпоинтов;
- helper-методы остаются только если используются несколько раз или повышают читаемость.

Single-use helper'ы после реализации инлайнить в тест или ближайший метод.

Исключение: helper может остаться, если он выражает предметную операцию и используется в серии однотипных тестов (`getTemplate`, `templateBody`, `fullTemplateContract`, `assertTemplateEquals`).

## 6. Что проверять

Позитивные контрактные тесты обычно покрывают:

- GET list:
  - ответ является массивом или объектом с массивом, как описано в OpenAPI;
  - созданная сущность найдена по `id`;
  - DTO ответа совпадает с expected DTO.
- GET by id:
  - ответ является объектом;
  - DTO ответа совпадает с expected DTO.
- POST:
  - сущность создается;
  - последующий GET возвращает expected DTO.
- PUT:
  - сущность обновляется;
  - последующий GET возвращает expected DTO;
  - не ожидай изменения полей, которые не отправлялись в request body.
- DELETE:
  - удаление успешно;
  - последующий GET возвращает ожидаемую ошибку.

Негативные тесты добавляй по контракту и риску:

- некорректный path id (`/invalid`);
- несуществующий id, если поведение зафиксировано;
- некорректное enum/поле в body;
- некорректный query parameter.

## 7. Как сравнивать контракт

Предпочтительный стиль:

```java
assertThat(actual)
        .usingRecursiveComparison()
        .isEqualTo(expected);
```

Если API возвращает системно заполняемые поля, которые не являются предметом проверки, игнорируй их явно и узко:

```java
assertThat(actual)
        .usingRecursiveComparison()
        .ignoringFieldsMatchingRegexes("parameters\\[\\d+\\]\\.(name|description|owner|inputMode)")
        .isEqualTo(expected);
```

Не заменяй контрактный тест длинным набором `assertEquals` по каждому полю. Такой тест хуже читается и хуже масштабируется при изменении DTO.

## 8. JsonContractWorker

Для маппинга ответа используй `JsonContractWorker<T>`:

```java
private static final JsonContractWorker<ServiceGetContract> SERVICE_CONTRACT_WORKER =
        new JsonContractWorker<>(ServiceGetContract.class);
```

Для массива:

```java
ServiceGetContract service = SERVICE_CONTRACT_WORKER.getContractFromJsonArray(services, serviceId);
```

Для объекта:

```java
ServiceGetContract service = SERVICE_CONTRACT_WORKER.getContractFromJson(serviceNode);
```

Для поиска по массиву не пиши ручные циклы, если `JsonWorker`/`JsonContractWorker` уже закрывает задачу.

## 9. Работа с OpenAPI

При чтении OpenAPI выпиши для себя:

- путь и метод;
- status codes;
- обязательные поля request body;
- поля response body;
- enum-значения;
- query/path параметры;
- различия list response и full response;
- read-only поля;
- поля, которые существуют только для отдельных каналов/типов.

Не смешивай GET-контракт и POST/PUT body. GET может возвращать `status`, `partner`, `partner_user`, read-only поля, но это не значит, что их нужно отправлять в POST/PUT.

## 10. Проверка и финализация

После изменений:

1. Запусти `compileTestJava`, если пользователь не запретил.
2. Не запускай full `build`, если пользователь явно просил использовать только compile.
3. Убери служебные изменения `.gradle/*`, если они появились после Gradle.
4. Проверь:
   - `git status --short --branch`;
   - `git diff --stat`;
   - `git diff --check`.
5. Commit/push делай только когда пользователь попросил.
6. В commit добавляй только файлы задачи, не трогай unrelated/untracked файлы.

## 11. Практический шаблон expected-проверки

```java
@Test
@DisplayName("Контракт: создание и получение шаблона")
void createAndGetTemplateContractTest() throws Exception {
    createTemplate(TEMPLATE_NAME, TEMPLATE_MESSAGE);
    int templateId = templateId(TEMPLATE_NAME);

    TemplateGetContract actualTemplate = getTemplate(templateId);
    TemplateGetContract expectedTemplate = fullTemplateContract(templateId, TEMPLATE_NAME, TEMPLATE_MESSAGE, true);

    assertTemplateEquals(actualTemplate, expectedTemplate);
}
```

Хороший контрактный тест читается так:

- что создали;
- какой endpoint вызвали;
- какой expected contract получен;
- каким одним сравнением доказали соответствие.

# Инструкция к репозиторию msg-tests

Данный репозиторий представляет собой проект для автоматизации тестирования системы.

## Структура проекта
- `src/test/resources` - содержит файлы ресурсов. Пользоваться будешь не часто, но будешь.
- `src/test/java/ru/wsoft/tests/framework` - сердце проекта. Необходимо детально изучить, но никогда ничего не изменять. Для тебя все внешние методы модулей фреймворка - API по управлению и проверкам.
- `src/test/java/ru/wsoft/tests/api/adm` - автоматизированные тест-кейсы на компонент `admin-console-api`.
- `src/test/java/ru/wsoft/tests/api/broker` - автоматизированные тест-кейсы на главный функционал системы - рассылки.
- `src/test/java/ru/wsoft/tests/api/cdp` - автоматизированные тест-кейсы на компонент `cdp-api`.
- `src/test/java/ru/wsoft/tests/api/imsi` - автоматизированные тест-кейсы на компонент `imsi-api`.
- `src/test/java/ru/wsoft/tests/api/lk` - автоматизированные тест-кейсы на компонент `lk-api`.
- `src/test/java/ru/wsoft/tests/ui/adm` - автоматизированные тест-кейсы на компонент `admin-console-ui`.
- `src/test/java/ru/wsoft/tests/ui/cc` - автоматизированные тест-кейсы на компонент `call-center-ui`.
- `src/test/java/ru/wsoft/tests/ui/cdp` - автоматизированные тест-кейсы на компонент `cdp-ui`.
- `src/test/java/ru/wsoft/tests/ui/lk` - автоматизированные тест-кейсы на компонент `lk-api`.

## Общие положения
1. Использовать `framework` как внешний API - тебя не должен интересовать способ проверки каким-либо методом фреймворка. Но код изучи обязательно.
2. Придерживаться стандарта кода при автоматизации тест-кейсов различных групп. Прежде всего - посмотреть на способ написания тестов в соседних классах. 
3. Для данного репозиторий храни свои вложенные локальные `MEMORY.md` и `CHANGES.md` для записи памяти по репозиторию, написанию кода и изменениям.
4. После изменений в `msg-tests` обновлять `msg-tests/CHANGES.md`, если он есть, и корневой `ivozisov/CHANGES.md` краткой сводкой: что добавлено/изменено, какие проверки запускались, commit hash после commit.

## Ограничения
1. Ни в коем случае и никогда не изменять/добавлять/удалять ни один метод или класс из `framework`.

## Пример написания тестов
1. Автоматизация тестов на рассылку:
```java
@Epic("broker-api")
@Feature("call")
@Story("Черные списки")
@Link("https://jira.wsoft.ru/browse/MSG-7317")
@Tag("blacklists")
@Execution(ExecutionMode.SAME_THREAD)
public class BlackList extends BaseBrokerApiTests {
    private final ThreadLocal<CreateMethodsAcapi> create = ThreadLocal
            .withInitial(CreateMethodsAcapi::new);
    private final ThreadLocal<DeleteMethodsAcapi> delete = ThreadLocal
            .withInitial(DeleteMethodsAcapi::new);

    private static ThreadLocal<APIHandler> broker = new ThreadLocal<>();

    static Partner partner;
    static PartnerUser partnerUser;
    static Account account;
    static SenderAddress originator;
    static SenderAddress viberOriginator;
    static String recipient="790";

    @BeforeAll
     static void setUp(){
        CreateMethodsAcapi create = new CreateMethodsAcapi();

        partner = new Partner.Builder()
                .withName("BLACKLISTS_CALL")
                .withPrepaid(false)
                .withStatus(true)
                .withTransports(new PartnerTransport[]{
                        new PartnerTransport.Builder()
                                .withName(Transport.CALL).build(),
                        new PartnerTransport.Builder()
                                .withName(Transport.VIBER).build()
                })
                .build();
        create.createPartner(partner);

        partnerUser = new PartnerUser.Builder()
                .withName("BLACKLISTS_CALL")
                .withPartner(partner.getName())
                .withEmail("blacklistcall@call.com")
                .withPassword("blacklist123")
                .withIsCheckPass(false)
                .withShowMessagesText(true)
                .withStatus(true)
                .withTimezoneId(-1)
                .withRoleName("Admin")
                .build();
        create.createPartnerUser(partnerUser);

        String serviceProvider = """
                {
                      "name": "BLACKLISTS_CALL",
                      "status": 1,
                      "parameters": [
                          {
                              "name": "MAIN_GATEWAY",
                              "protocol_parameter_id": 3,
                              "protocol_id": 2,
                              "protocol_name": "SIP",
                              "value": null
                          },
                          {
                              "name": "RESERVE_GATEWAY",
                              "protocol_parameter_id": 4,
                              "protocol_id": 2,
                              "protocol_name": "SIP",
                              "value": null
                          }
                      ],
                      "transport_ids": [
                          2
                      ],
                      "call_protocol_id": 2
                  }
                """;
        create.createServiceProvider("BLACKLISTS_CALL", serviceProvider);

        account = new Account.Builder()
                .withName("BLACKLISTS_CALL")
                .withLogin("blistcallmessage")
                .withPassword("blistcallmessage")
                .withPartner(partner.getName())
                .withHttp(true, true, "http://" + CallbackServer.getCurrentIPv4Address() + ":8484/",
                        true)
                .withStatus(true)
                .build();
        create.createAccount(account);

        originator = new SenderAddress.Builder()
                .withPartner(partner.getName())
                .withTransport(Transport.CALL)
                .withOriginator("1000")
                .withStatus(true)
                .build();
        create.createSenderAddress(originator);

        viberOriginator = new SenderAddress.Builder()
                .withPartner(partner.getName())
                .withTransport(Transport.VIBER)
                .withOriginator("1001")
                .withStatus(true)
                .build();
        create.createSenderAddress(viberOriginator);

        broker.set(new APIHandler("broker", "blistcallmessage", "blistcallmessage"));
    }

    @Test
    @DisplayName("Отправка с ЧС без получателя")
    @Description("""
            Создать Черный список. Не добавлять получателя. Совершить отправку с указанием ЧС на любой номер получателя.
            -> Черный список не заблокировал отправку.
            """)
    @TmsLink("https://testlink.wsoft.ru/linkto.php?tprojectPrefix=MSG&item=testcase&id=MSG-8186")
    void blacklistCheckNoRecipient() {
        Blacklist blacklist = new Blacklist.Builder()
                .withName("BLACKLIST_CALL_NO_RECIPIENT")
                .withPartner(partner.getName())
                .withPartnerUser(partnerUser.getName())
                .withCheckDuplicatesPolicy(Blacklist.Builder.CheckDuplicatesPolicy.NOT_CHECK)
                .build();

        create.get()
                .createBlackList(blacklist);

        try {
            Allure.step("""
                    Совершить отправку с указанием ЧС на любой номер получателя.
                    -> Не ожидаем ошибку 30007 "Recipient is in blacklist".
                    """, () -> {
                String id = gen.genNumber(true);
                String body = String.format("""
                                {
                                    "blacklist-ids": ["%s"],
                                    "priority": "2",
                                    "messages": [
                                        {
                                            "recipient": "%s",
                                            "message-id": "%s",
                                            "call": {
                                                "originator": "%s",
                                                "content": {
                                                    "text": "LIMITING_NUMBER"
                                                }
                                            }
                                        }
                                    ]
                                }
                                """, msg.getBlacklistId(blacklist.getName()),
                        recipient, id, originator.getOriginator());

                broker.get()
                        .post("/broker-api/send", body)
                        .code(200);

                await()
                        .atMost(10, TimeUnit.SECONDS)
                        .during(5, TimeUnit.SECONDS)
                        .pollInterval(1, TimeUnit.SECONDS)
                        .untilAsserted(
                                () ->
                                        Allure.step("Проверяем, что не получена ошибка \"Recipient is in blacklist\"",()-> {
                                            assertNotNull(CallbackServer.getCallBackByIncompleteKey(new CallbackKey(id, "Call")));
                                            assertNull(CallbackServer.getCallBack(new CallbackKey
                                                            (id, "Call", "Rejected", "Recipient is in black list", null)),
                                                    logExistedNoError(id));
                                        })
                        );
            });
        } finally {
            delete.get()
                    .deleteClientBlackList(blacklist.getName());
        }
    }

    @AfterAll
    static void deleteAll(){
        new DeleteMethodsAcapi()
                .deleteSenderAddress(viberOriginator.getOriginator(),viberOriginator.getPartnerId(),viberOriginator.getTransportId())
                .deleteSenderAddress(originator.getOriginator(),originator.getPartnerId(),originator.getTransportId())
                .deleteBrokerAccount(account.getName(),account.getPartnerId())
                .deletePartnerUser(partnerUser.getName())
                .deletePartner(partner.getName());
    }
}
```

2. Автоматизация тестов API(Контрактное тестирование):
```java
@Epic("admin-console-api")
@Feature("Настройки")
@Story("Сервисы")
@Tag("MSG-6973")
@Execution(ExecutionMode.SAME_THREAD)
class ServiceContractTests extends BaseApiTests {

    private final APIHandler api = new APIHandler("acapi");
    private final CreateMethodsAcapi create = new CreateMethodsAcapi();
    private final DeleteMethodsAcapi delete = new DeleteMethodsAcapi();
    private final SimpleSqlQueriesWorker quariesWorker = new SimpleSqlQueriesWorker();
    private final JsonWorker jsonWorker = new JsonWorker();
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @DisplayName("Контракт: получение списка сервисов")
    @Description("""
            1. Создать несколько сервисов.
            2. Запросить список сервисов.
            3. Проверить, что получен массив сервисов.
            4. Проверить, что полученные данные соотвествуют данным при создании.
            """)
    @Test
    void getServicesContractTest() {
        int serviceId1=0, serviceId2=0, serviceId3=0;
        final JsonContractWorker<ServiceGetContract> jsonContractWorker=new JsonContractWorker<>(ServiceGetContract.class);

        try {
            Allure.step("Предварительное создание сервисов", () -> {
                create
                        .createService("name1", "type1", "desc1", true)
                        .createService("name2", "type2", "desc2", false)
                        .createService("name3", "type3", "desc3", true)

                        .updateService(quariesWorker.getServiceId("name1", "type1"),
                                new Service.Builder()
                                        .withParameters(new ServiceParameter[]{
                                                new ServiceParameter(1, "value1"),
                                                new ServiceParameter(2, "value2"),
                                                new ServiceParameter(3, "value3")
                                        })
                                        .build())
                        .updateService(quariesWorker.getServiceId("name2", "type2"),
                                new Service.Builder()
                                        .withParameters(new ServiceParameter[]{
                                                new ServiceParameter(4, "value4"),
                                                new ServiceParameter(5, "value5"),
                                                new ServiceParameter(6, "value6")
                                        })
                                        .build())
                        .updateService(quariesWorker.getServiceId("name2", "type2"),
                                new Service.Builder()
                                        .withParameters(new ServiceParameter[]{
                                                new ServiceParameter(7, "value7"),
                                                new ServiceParameter(8, "value8"),
                                                new ServiceParameter(9, "value9")
                                        })
                                        .build());
            });

            serviceId1 = quariesWorker.getServiceId("name1", "type1");
            serviceId2 = quariesWorker.getServiceId("name2", "type2");
            serviceId3 = quariesWorker.getServiceId("name3", "type3");

            Allure.step("Проверка контрактов", () -> {
                Response response = api
                        .get("/acapi/services")
                        .code(200)
                        .getResponse();

                JsonNode jNode = MAPPER.readTree(response.asString());
                assertTrue(jsonWorker.isArray(jNode), "Результат не является массивом");

                ServiceGetContract serviceContract1 = jsonContractWorker.getContractFromJsonArray(jNode, 1);
                assertEquals("name1", serviceContract1.getName(), "Параметр \"name\" не соответствует ожидаемому значению");
                assertEquals("type1", serviceContract1.getType(), "Параметр \"type\" не соответствует ожидаемому значению");
                assertEquals("desc1", serviceContract1.getDescription(), "Параметр \"description\" не соответствует ожидаемому значению");
                assertTrue(serviceContract1.isActive(), "Параметр \"is_active\" не соответствует ожидаемому значению");
                assertEquals(3, serviceContract1.getParameters().length, "Кол-во значений вложенного массива \"parameters\" не соответствует ожидаемому значению");
                assertEquals(1, serviceContract1.getParameters()[0].getId(), "Параметр \"parameters.1.id\" не соответствует ожидаемому значению");
                assertEquals("value1", serviceContract1.getParameters()[0].getValue(), "Параметр \"parameters.1.value\" не соответствует ожидаемому значению");
            });
        } finally {
            delete
                    .deleteService(serviceId1)
                    .deleteService(serviceId2)
                    .deleteService(serviceId3)
            ;
        }
    }
}
```

3. Автоматизация тестов на интерфейс(UI):
```java
@Epic("ADMIN UI")
@Feature("Звонки")
@Story("Библиотека звуков")
public class SoundsLibrary extends BaseConsoleTest {
    String section = "Звонки", subSection = "Библиотека звуков";

    private final ThreadLocal<CreateMethodsAcapi> createMethodsAcapi =
            ThreadLocal.withInitial(CreateMethodsAcapi::new);

    @DisplayName("Создание")
    @Test
    void createFile() {
        String filename = "create.mp3";
        File file = new File("src/test/resources/soundFiles/" + filename);

        ui
                .subSectionClick(section, subSection)
                .deleteIfExists(filename)

                .buttonClick("+")
                .inputSet("Файл", file)
                .inputSet("Клиент", "Wings")
                .inputSet("Пользователь", "wingsUser")
                .buttonClick("s");

        cache
                .cacheService("WCS:group=Services,instance-type=Cache,name=cache-service1")
                .openCache()
                .keySet("files")
                .stringContains(msg.getClientID("Wings") + "file" + filename);

        ui
                .tableHrefCellClick(filename)
                .inputContains("Файл", filename)
                .inputContains("Клиент", "Wings")
                .inputContains("Пользователь", "wingsUser")

                .buttonClick("d").confirm();
    }
}
```
В последнем примере обрати внимание на `cache`. Для некоторых сущностей системы, которые попадают в КЭШ, критически необходимо проверять её попадание и свойства в КЭШе.
Вот еще один, более наглядный пример проверки КЭШа:
```java
@DisplayName("Создание")
    @Description("""
            Создать шаблон, заполнив всевозможные параметры.
            Проверить, что данные корректно отображаются при открытии шаблона.
            Проверить, что данные в кэше заполнены корректно.
            """)
    @TmsLink("https://testlink.wsoft.ru/linkto.php?tprojectPrefix=MSG&item=testcase&id=MSG-9183")
    @Test
    void createTemplate() {
        final String templateName = "PUSH_TEMPLATE_ALL_FIELDS_TEST";

        Partner partner = new Partner.Builder()
                .withName(templateName)
                .withStatus(true)
                .withTransports(new PartnerTransport[]{
                        new PartnerTransport.Builder().withName(Transport.PUSH).build()
                })
                .withPrepaid(false)
                .build();
        create.get().createPartner(partner);

        PartnerUser partnerUser = new PartnerUser.Builder()
                .withName(templateName)
                .withEmail("pushtemplateallfields@test.com")
                .withPassword("pushtemplateallfields")
                .withRoleName("Admin")
                .withPartner(partner.getName())
                .withShowMessagesText(true)
                .withTimezoneId(-1)
                .withIsCheckPass(false)
                .build();
        create.get().createPartnerUser(partnerUser);

        PushApplication pushApplication = new PushApplication.Builder()
                .withName("PUSH_TEMP_ALL_FIELDS_APP")
                .withPartners(new String[]{partner.getName()})
                .withLogin("PUSH_TEMP_ALL_FIELDS_APP")
                .withPassword("PUSH_TEMP_ALL_FIELDS_APP")
                .withCheckSign(false)
                .withDevicesPolicy("ALL")
                .withPushSignPolicy("NONE")
                .withFcm(new FCM.Builder()
                        .withProtocol("XMPP")
                        .withServerType("SIMULATOR")
                        .withConnectionCount(5)
                        .withIp("127.0.0.1")
                        .withPort("5462")
                        .build())
                .build();
        create.get().createPushApplication(pushApplication);

        try {
            Allure.step("Создание шаблона", () -> {
                ui
                        .subSectionClick(section, subSection)
                        .deleteIfExists(templateName)
                        .buttonClick("+")

                        .inputSet("Название", templateName)
                        .inputSet("Публичность", "Публичный")
                        .inputSet("Клиент", partner.getName())
                        .inputSet("Пользователь", partnerUser.getName())
                        .inputSet("Заголовок", "Header")
                        .inputSet("Сообщение", "Message")
                        .inputSet("Контент", "Content")

                        .inputSet("Ссылка по нажатию на уведомление", "http://hreftotap")
                        .buttonClick("Добавить параметр")
                        .inputSet("Название параметра", "parameter name")
                        .inputSet("Значение параметра", "parameter value")

                        .buttonClick("Добавить кнопку")
                        .inputSet("Текст кнопки", "button text")
                        .inputSet("Идентификатор", "button id")
                        .inputSet("Назначение", "Перейти по ссылке")
                        .inputSet("Обрабатываемое значение", "button value")

                        .radioButtonOn("Средство отображения уведомления")
                        .inputSet("Идентификатор группы", "group id")
                        .radioButtonOn("Значок на иконке", 2)
                        .byIds().setById("icon_set_in", "3").back()

                        .uploads()
                        .inputButtonUpload("Баннер", new File("src/test/resources/files/push/templates/banner.jpg"))
                        .inputButtonUpload("Логотип", new File("src/test/resources/files/push/templates/logo.jpg"))
                        .back()
                        .inputSet("Наименование группы", "group name")

                        .inputSet("Подзаголовок", "subtitle")
                        .uploads()
                        .inputButtonUpload("Изображение", new File("src/test/resources/files/push/templates/logo.jpg"))

                        .inputButtonUpload("Ссылка на баннер", new File("src/test/resources/files/push/templates/banner.jpg"))
                        .inputButtonUpload("Ссылка на логотип", new File("src/test/resources/files/push/templates/logo.jpg"))
                        .back()

                        .inputSet("Приложения", pushApplication.getName())

                        .inputSet("Выбор по дате создания", "Последнее созданное")

                        .inputSet("Приоритет", "High")
                        .inputSet("Способ отправки", "Native Push и Web Push")

                        //Дожидаемся загрузки изображений
                        .waiting(15)
                        .buttonClick("s")
                        .tableRowExists(templateName);
            });

            Allure.step("Проверка отображения данных", () -> {
                ui
                        .tableHrefCellClick(templateName)

                        .inputContains("Название", templateName)
                        .inputContains("Публичность", "Публичный")
                        .inputContains("Клиент", partner.getName())
                        .inputContains("Пользователь", partnerUser.getName())
                        .inputContains("Заголовок", "Header")
                        .inputContains("Сообщение", "Message")
                        .inputContains("Контент", "Content")

                        .inputContains("Ссылка по нажатию на уведомление", "http://hreftotap")

                        .inputContains("Текст кнопки", "button text")
                        .inputContains("Идентификатор", "button id")
                        .inputContains("Назначение", "Перейти по ссылке")
                        .inputContains("Обрабатываемое значение", "button value")

                        .isRadioButtonOn("Средство отображения уведомления")
                        .inputContains("Идентификатор группы", "group id")
                        .isRadioButtonOn("Значок на иконке", 2)
                        .byIds().equalsById("icon_set_in", "3").back()

                        .inputContains("Баннер", "upload")
                        .inputContains("Логотип", "upload")
                        .inputContains("Наименование группы", "group name")

                        .inputContains("Подзаголовок", "subtitle")
                        .inputContains("Изображение", "upload")

                        .inputContains("Ссылка на баннер", "upload")
                        .inputContains("Ссылка на логотип", "upload")

                        .datatagContains(pushApplication.getName())

                        .inputContains("Выбор по дате создания", "Последнее созданное")

                        .inputContains("Приоритет", "High")
                        .inputContains("Способ отправки", "Native Push и Web Push");
            });

            Allure.step("Проверка заполнения кэша", () -> {
                cache
                        .cacheService("WCS:group=Services,instance-type=Cache,name=cache-service1")
                        .openCache()
                        .get("templates", msg.getTemplateId("Push", templateName))
                        .xmlContains("push-details/id", msg.getTemplateId("Push", templateName))
                        .xmlContains("push-details/message", "Message")
                        .xmlContains("push-details/applications", pushApplication.getName())
                        .xmlContains("push-details/sub_by_date", "2")
                        .xmlContains("push-details/title", "Header")
                        .xmlContains("push-details/content", "Content")
                        .xmlContains("push-details/responsible-for-view", "sdk")
                        .xmlContains("push-details/badge-rule", "set_to_3")
                        .xmlContains("push-details/group-id", "group id")
                        .xmlContains("push-details/tap-link", "http://hreftotap")
                        .xmlContains("push-details/buttons/button/id", getButtonId(templateName, "button id"))
                        .xmlContains("push-details/buttons/button/text", "button text")
                        .xmlContains("push-details/buttons/button/appointment", "link")
                        .xmlContains("push-details/buttons/button/appointment-value", "button value")
                        .xmlContains("push-details/android-push-details/group-name", "group name")
                        .xmlContains("push-details/android-push-details/images/image/type", "banner")
                        .xmlContains("push-details/android-push-details/images/image/external-url", "upload")
                        .xmlContains("push-details/android-push-details/images/image/upload-files-id")
                        .xmlContains("push-details/ios-push-details/subtitle", "subtitle")
                        .xmlContains("push-details/ios-push-details/images/image/type", "banner")
                        .xmlContains("push-details/ios-push-details/images/image/upload-files-id")
                        .xmlContains("push-details/ios-push-details/images/image/external-url", "upload")
                        .xmlContains("push-details/web-push-details/images/image/type", "logo")
                        .xmlContains("push-details/web-push-details/images/image/upload-files-id")
                        .xmlContains("push-details/web-push-details/images/image/external-url", "upload");
            });
        } finally {
            delete.get()
                    .deleteTemplate(Transport.PUSH, templateName)
                    .deletePushApp(pushApplication.getName())
                    .deletePartnerUser(partnerUser.getName())
                    .deletePartner(partner.getName());
        }
    }
```
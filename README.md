# SDK для интеграции функционала mPOS в мобильное приложение с проведением платежей через провайдера платежей [Assist](https://www.assist.ru/)

## Начальные требования
Для получения возможности проведения оплаты через SDK необходимо обратиться в службу поддержки компании Ассист (support@assist.ru) для заведения магазина и пользователя в системе Ассист с необходимыми разрешениями.
Полученные идентификатор магазина вместе с логином и паролем пользователя используются при вызове методов оплаты в SDK.
Для приёма банковских карт понадобится считыватель карт, который также должен быть запрошен в компании Ассист.

### Требуемые разрешения:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
```

### Требуемые зависимости:
```.gradle
//
// Проект logback https://github.com/tony19/logback-android
//
implementation "org.slf4j:slf4j-api:1.7.30"
implementation "com.github.tony19:logback-android:1.3.0-3"

//
// Gson
//
implementation "com.google.code.gson:gson:2.8.6"

//
// Google Firebase
//
implementation platform("com.google.firebase:firebase-bom:25.12.0")
implementation 'com.google.firebase:firebase-core'
implementation 'com.google.firebase:firebase-analytics'
implementation 'com.google.firebase:firebase-perf'
implementation 'com.google.firebase:firebase-crashlytics'
```

### Пример файла конфигурации logback.xml (располагается в папке assets модуля приложения)
```xml
<configuration>
    <appender name="logcat" class="ch.qos.logback.classic.android.LogcatAppender">
        <tagEncoder>
            <pattern>%logger{12}</pattern>
        </tagEncoder>
        <encoder>
            <pattern>[%-10thread] %msg</pattern>
        </encoder>
    </appender>

    <root level="TRACE">
        <appender-ref ref="logcat" />
    </root>
</configuration>
```

### Порядок интеграции SDK в приложение

**Инициализируем магазин и пользователя**
```java
AssistMerchant assistMerchant = new AssistMerchant();
assistMerchant.setId("merchant_id");

AssistUser assistUser = new AssistUser();
assistUser.setLogin("user_login");
assistUser.setPassword("user_password");
```

**Инициализируем платёжное ядро SDK**
```java
AssistPaymentEngine engine = AssistSDK.getPaymentEngine(Context context);
// Тестовый или боевой сервер Ассист в зависимости от настроек магазина и пользователя
engine.setServer("https://payments.t.paysecure.ru");
//engine.setServer("https://payments.paysecure.ru");
engine.setMerchant(assistMerchant);
engine.setUser(assistUser);
// Слушатель результата платежа
engine.setPaymentListener(new PaymentListener());
```

**Инициализируем данные платежа**
```java
AssistPaymentData paymentData = new AssistPaymentData(); 
// Номер заказа
paymentData.setOrderNumber(String.valueOf(System.currentTimeMillis()));
// Позиции заказа
int numOfItems = 2;
for (int i = 0; i < numOfItems; i++) {
    AssistChequeItem item = new AssistChequeItem();
    item.setId(String.valueOf(i));
    item.setName("Item_" + i);
    item.setQuantity("1");
    item.setPrice("1." + i);
    //item.setAmount("1." + i); // Если amount не указан, он рассчитывается автоматически как price*quantity
    item.setTax(AssistPaymentTax.novat);
    item.setFPMode("4");
    item.setUncode("12345");
    //item.setEancode("");
    //item.setGs1code("");
    //item.setFurcode("");
    //item.setEgaiscode("");
    //item.setSubjtype("");
    items.add(item);
    paymentData.addChequeItem(item);
}
// Сумма заказа
paymentData.setOrderAmount(AssistChequeItem.getItemsTotal(items));
// Валюта заказа (только "RUB")
paymentData.setOrderCurrency("RUB");
// Телефон покупателя (необязательно)
paymentData.setMobilePhone("+71234567890");
// Адрес расчёта (необязательно)
paymentData.setPaymentAddress("Где-то далеко");
// Место расчёта (необязательно)
paymentData.setPaymentPlace("Чуть ближе");
// ФИО кассира (необязательно)
paymentData.setCashier("Иванов И.И.");
// Способ фискализации оплаты
paymentData.setFiscalDocumentGenerator(AssistPaymentData.FiscalDocumentGenerator.ASSIST);
```
Описание и возможные значения полей позиции чека можно найти тут:  
https://docs.assist.ru/pages/viewpage.action?pageId=5768298  
Внимание: позиции в SDK поддерживают только те поля, которые представлены в примере выше.

**Инициализируем экземпляр класса для работы со считывателем банковских карт**
```java
final AssistCardReader cardReader = AssistSDK.getCartReader(Context context);
// Слушатель состояния подключения считывателя
cardReader.setConnectionListener(new AssistCardReader.ConnectionListener() {
    @Override
    public void onConnected(AssistCardReader.ConnectionType connectionType) {
        // Считываетль подключился, стартуем платёж
        engine.payWithCardReader(someActivity, paymentData, cardReader);
    }

    @Override
    public void onDisconnected(AssistCardReader.ConnectionType connectionType) {
        // Считыватель отключен
    }

    @Override
    public void onError(AssistCardReader.ConnectionType connectionType) {
        // Ошибка подключения считывателя
    }
});

// Слушатель необходимых действий при обработке карты считывателем
cardReader.setActionListener(new AssistCardReader.ActionListener() {
    @Override
    public void onSelectApp(String[] apps) {
        // Выбор приложения на карте
        // Можно не реализовать
    }

    @Override
    public void onProvideCardholderSignature(String amount) {
        // Графическая подпись покупателя в виде Base64 строки с максимальной длиной 4000 символов и специальным префиксом
        // signatureBase64 = "data:image/png;base64," + Base64.encodeToString(signaturePngAsByteArray, Base64.NO_WRAP);
        // cardReader.onCustomerSignature(signatureBase64);
    }
});

// Слушатель сообщений для пользователя, выдаваемых считывателем карт
cardReader.setMessageListener(new AssistCardReader.MessageListener() {
    @Override
    public void onUserMessage(String s) {
        // Можно вывести на экран для слежения за ходом платежа
        // tvInfo.setText(s);
    }

    @Override
    public void onLogMessage(String s, boolean b) {
        // Служебные сообщения считывателя карт о ходе платежа
        // Могут отсутствовать
    }
});

// Пинпад должен быть уже сопряжён по bluetooth
//BluetoothAdapter.getDefaultAdapter().getBondedDevices(); // Возвращает список девайсов, откуда берём адрес и название пинпада
// Bluepad-50 можно узнать по name, начинающемуся с букв "PP" 
// Подключаемся к пинпаду
cardReader.connectBluetooth(address, name);
```

**Пример слушателя результата платежа**
```java
class PaymentListener implements PaymentEngineListener {
    @Override
    public void onPaymentFinished(long order_id) {
        // Получение экземпляра хранилища заказов
        AssistOrderInfoStorage storage = engine.getOrderInfoStorage();
        // Получение заказа из хранилища по внутреннмему идентификатору
        AssistOrderInfo orderInfo = storage.getOrderInfo(order_id);
        Log.d("TAG", "Завершено со статусом: " + orderInfo.getOrderStateAsString());
        // Оплаченным является только заказ со статусом APPROVED!
        // Для способа оплаты СБП необходимо отобразить QR-код
        showQrToUser(orderInfo.getQrCodePlain()); // Ваш метод, который преобразует текст в QR-код
    }

    @Override
    public void onPaymentFinishedWithError(long order_id, String errorMessage) {
        // Платеж проведен, но возникли ошибки, например не удалось получить фискальный документ
        // Требуется перезапросить статус заказа
        // Получение экземпляра хранилища заказов
        AssistOrderInfoStorage storage = engine.getOrderInfoStorage();
        // Получение заказа из хранилища по внутреннмему идентификатору
        AssistOrderInfo orderInfo = storage.getOrderInfo(order_id);
        Log.d("TAG", "Завершено со статусом: " + orderInfo.getOrderStateAsString());
        Log.d("TAG", "Ошибка: " + errorMessage);
    }

    @Override
    public void onPaymentFinishedWithNetError(long order_id, String errorMessage) {
        // Платеж проведен, но возникли проблемы с сетью
        // Требуется перезапросить статус заказа
        // Получение экземпляра хранилища заказов
        AssistOrderInfoStorage storage = engine.getOrderInfoStorage();
        // Получение заказа из хранилища по внутреннмему идентификатору
        AssistOrderInfo orderInfo = storage.getOrderInfo(order_id);
        Log.d("TAG", "Завершено со статусом: " + orderInfo.getOrderStateAsString());
        Log.d("TAG", "Сетевая ошибка: " + errorMessage);
    }

    @Override
    public void onPaymentFault(String faultInfo) {
        // Ошибка платежа. Необходимо обратиться в тех. поддержку
    }

    @Override
    public void onPaymentTerminated() {
        // Платеж прерван. Например на считывателе карт нажали кнопку отказ от платежа
    }

    @Override
    public void onRegistrationError(String errorMessage) {
        // Ошибка регистрации приложения в системе Ассист. Необходимо обращение в тех. поддержку
    }

    @Override
    public void onNetworkInaccessible(String errorMessage) {
        // Отсутствует подключение к интернету.  Проведение платежа невозможно
    }
}
```

**Запуск платежа наличными**
```java
engine.payCash(someActivity, paymentData, new PaymentListener());
```

**Запуск платежа СБП**
```java
engine.payFastTransfer(someActivity, paymentData, new PaymentListener());
// Далее необходимо отобразить QR-код для оплаты (см. PaymentListener)
// После оплаты банковским приложением клиента проверяем статус заказа (см. ниже)
```

**Запрос статуса заказа**
```java
engine.setPaymentStateListener(new StateListener());
engine.getPaymentState(someActivity, order_id);

class StateListener implements PaymentStateListener {
    @Override
    public void onComplete(long order_id) {
        // Получение экземпляра хранилища заказов
        AssistOrderInfoStorage storage = engine.getOrderInfoStorage();
        // Получение заказа из хранилища по внутреннмему идентификатору
        AssistOrderInfo orderInfo = storage.getOrderInfo(order_id);
        Log.d("TAG", "Завершено со статусом: " + orderInfo.getOrderStateAsString());
        // Оплаченным является только заказ со статусом APPROVED!
    }

    @Override
    public void onError(long order_id, String errorMessage) {
        // Ошибка получения статуса платежа
    }

    @Override
    public void onRegistrationError(String errorMessage) {
        // Ошибка регистрации приложения в системе Ассист. Необходимо обращение в тех. поддержку
    }

    @Override
    public void onNetworkInaccessible(String errorMessage) {
        // Отсутствует подключение к интернету. Невозможно запросить статус заказа
    }
}
```
### Отмена заказа

**Инициализация данных для отмены**

```java
AssistCancelData cd = new AssistCancelData();
cd.setOrderId(order_id);
// Отмена может быть полная (на все позиции) и частичная (не на все позиции)
cd.setItems(engine.getOrderNotCanceledItems(order_id));
cd.setProvider(FiscalizationProvider.SERVER);
//Необязательные параметры
//cd.setCashier("");
//cd.setPaymentAddress("");
//cd.setPaymentPlace("");
```

**Инициализация слушателя результата отмены**

```java
engine.setPaymentCancelListener(new CancelListener());
class CancelListener implements PaymentCancelListener {
    @Override
    public void onCanceled(long id) {
        AssistOrderInfo orderInfo = engine.getOrderInfo(order_id);
        Log.d("TAG", "Завершено со статусом: " + orderInfo.getOrderStateAsString());
        // Полностью отменённые заказы должны иметь статус CANCELED
        // Частично отменённые - PARTIAL_CANCELED
    }

    @Override
    public void onCancelError(long order_id, String errorMessage) {
        // Ошибка произошла во время отмены
        AssistOrderInfo orderInfo = engine.getOrderInfo(order_id);
        Log.d("TAG", "Завершено со статусом: " + orderInfo.getOrderStateAsString());
        Log.d("TAG", "Ошибка: " + errorMessage);
    }

    @Override
    public void onError(long order_id, String errorMessage) {
        // Ошибка отмены заказа. Попробуйте обновить статус заказа, в случае неудачи обратитесь в тех. поддержку
    }

    @Override
    public void onRegistrationError(String errorMessage) {
        // Ошибка регистрации приложения в системе Ассист. Необходимо обращение в тех. поддержку
    }

    @Override
    public void onNetworkInaccessible(String errorMessage) {
        // Отсутствует подключение к интернету. Проведение платежа невозможно
    }
}
```

**Отмена заказа по карте**

```java
// На полную сумму
engine.cancelPayment(someActivity, cancelData, cardReader)
// Частичная, если требуется отменить не все позиции чека
engine.cancelPaymentPartially(someActivity, cancelData, cardReader);
```
AssistCardReader необходимо инициализировать также, как при оплате

**Отмена заказа наличными**

```java
// На полную сумму
engine.cancelCash(someActivity, cancelData)
// Частичная, если требуется отменить не все позиции чека
engine.cancelCashPartially(someActivity, cancelData)
```

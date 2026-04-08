# Мобильное тестирование — обзор

## Обзор

Мобильное тестирование — это проверка качества приложений, работающих на мобильных устройствах (смартфоны,
планшеты). С учётом того, что более 60% мирового веб-трафика приходится на мобильные устройства, тестирование
мобильных приложений является критически важной компетенцией для QA-инженера.

Данный раздел даёт обзорное понимание мобильного тестирования: типы мобильных приложений, архитектура Appium,
ключевые инструменты и вызовы, с которыми сталкиваются тестировщики. Материал достаточен для уверенного
прохождения интервью на позиции, где мобильное тестирование не является основной специализацией.

---

## Типы мобильных приложений

### Mobile Web

Веб-приложение, открываемое в мобильном браузере (Chrome, Safari). Не устанавливается на устройство.

**Характеристики:**
- Обычный сайт, адаптированный под мобильные экраны (responsive design)
- Тестируется через мобильный браузер
- Не имеет доступа к нативным функциям устройства (камера, GPS, push-уведомления)
- Обновления доставляются мгновенно (без публикации в сторах)
- Автоматизация через Selenium / Appium с мобильным браузером

**Пример:** мобильная версия сайта банка, интернет-магазин.

### Native App

Приложение, разработанное под конкретную платформу (Android или iOS) с использованием нативных SDK.

**Характеристики:**
- Устанавливается через App Store / Google Play
- Полный доступ к функциям устройства (камера, Bluetooth, датчики)
- Высокая производительность
- Разные кодовые базы для Android (Kotlin/Java) и iOS (Swift/Objective-C)
- Автоматизация через Appium, Espresso (Android), XCUITest (iOS)

**Пример:** Instagram, Telegram, мобильный банк.

### Hybrid App

Комбинация native и web. Внешне выглядит как нативное приложение, но внутри содержит WebView с веб-контентом.

**Характеристики:**
- Устанавливается через стор
- Основной контент отображается в WebView (HTML/CSS/JS)
- Есть доступ к некоторым нативным функциям (через мосты типа Cordova, Capacitor)
- Единая кодовая база для обеих платформ
- Производительность ниже нативных приложений
- Автоматизация требует переключения контекстов (NATIVE_APP / WEBVIEW)

**Пример:** приложения на Ionic, Cordova, React Native (частично).

### Сравнительная таблица

| Критерий | Mobile Web | Native App | Hybrid App |
|---|---|---|---|
| Установка | Не нужна | Через стор | Через стор |
| Доступ к устройству | Ограничен | Полный | Частичный |
| Производительность | Средняя | Высокая | Средняя |
| Обновление | Мгновенное | Через стор | Через стор |
| Кросс-платформенность | Да | Нет | Да |
| Работа offline | Нет | Да | Частично |
| Автоматизация | Selenium/Appium | Appium/Espresso/XCUITest | Appium |

---

## Appium — архитектура

Appium — это open-source фреймворк для автоматизации мобильных приложений (native, hybrid, mobile web)
на Android и iOS. Использует WebDriver-протокол, что позволяет писать тесты на любом языке.

### Принцип работы

```
Тестовый код (Java/Python/JS)
        |
        | HTTP (WebDriver Protocol)
        v
   Appium Server (Node.js)
        |
        | Использует соответствующий драйвер
        v
   ┌────────────┐     ┌─────────────┐
   │  UiAutomator2  │     │  XCUITest     │
   │  (Android)     │     │  (iOS)        │
   └──────┬─────┘     └──────┬──────┘
          │                   │
          v                   v
   Android Device/        iOS Device/
   Emulator               Simulator
```

### Ключевые компоненты

| Компонент | Описание |
|---|---|
| **Appium Server** | Node.js-сервер, принимающий WebDriver-запросы и перенаправляющий их на драйверы |
| **Appium Client** | Клиентская библиотека (java-client, python-client и др.) |
| **UiAutomator2 Driver** | Драйвер для Android, использует Google UiAutomator2 |
| **XCUITest Driver** | Драйвер для iOS, использует Apple XCUITest framework |
| **Appium Inspector** | GUI-инструмент для инспектирования элементов и создания локаторов |

### Философия Appium

1. **Не требует модификации приложения** — тестируется реальный APK/IPA
2. **Единый API** — один и тот же WebDriver-протокол для Android и iOS
3. **Любой язык** — Java, Python, JavaScript, C#, Ruby
4. **Open-source** — бесплатный и поддерживаемый сообществом

---

## Desired Capabilities

Capabilities в Appium определяют параметры тестовой сессии: платформу, устройство, приложение.

### Android

```java
UiAutomator2Options options = new UiAutomator2Options();

// Обязательные capabilities
options.setPlatformName("Android");
options.setDeviceName("Pixel_6_API_34");        // имя эмулятора или устройства
options.setApp("/path/to/app.apk");              // путь к APK-файлу
options.setAutomationName("UiAutomator2");       // драйвер автоматизации

// Дополнительные
options.setAppPackage("com.example.myapp");      // пакет приложения
options.setAppActivity(".MainActivity");          // стартовая Activity
options.setNoReset(true);                         // не сбрасывать состояние между тестами
options.setFullReset(false);                      // не переустанавливать приложение
options.setNewCommandTimeout(Duration.ofSeconds(60));

AndroidDriver driver = new AndroidDriver(
    new URL("http://127.0.0.1:4723"), options
);
```

### iOS

```java
XCUITestOptions options = new XCUITestOptions();

// Обязательные capabilities
options.setPlatformName("iOS");
options.setDeviceName("iPhone 15");
options.setPlatformVersion("17.0");
options.setApp("/path/to/app.ipa");
options.setAutomationName("XCUITest");

// Дополнительные
options.setBundleId("com.example.myapp");        // Bundle ID приложения
options.setNoReset(true);
options.setWdaLaunchTimeout(Duration.ofSeconds(120));

IOSDriver driver = new IOSDriver(
    new URL("http://127.0.0.1:4723"), options
);
```

### Mobile Web (Chrome на Android)

```java
UiAutomator2Options options = new UiAutomator2Options();
options.setPlatformName("Android");
options.setDeviceName("Pixel_6_API_34");
options.setAutomationName("UiAutomator2");
options.withBrowserName("Chrome");               // тестирование в мобильном Chrome

AndroidDriver driver = new AndroidDriver(
    new URL("http://127.0.0.1:4723"), options
);
driver.get("https://example.com");
```

---

## Базовая настройка Appium

### Предварительные требования

**Для Android:**
- Java JDK 17+
- Android SDK (через Android Studio)
- Переменные окружения: `ANDROID_HOME`, `JAVA_HOME`
- Создан эмулятор (AVD) или подключено реальное устройство с USB-debugging

**Для iOS (только macOS):**
- Xcode (последняя версия)
- Xcode Command Line Tools
- Carthage или CocoaPods
- Simctl для управления симуляторами

### Установка Appium

```bash
# Установка Appium Server (Node.js >= 18)
npm install -g appium

# Установка драйверов
appium driver install uiautomator2
appium driver install xcuitest

# Проверка окружения
appium driver doctor uiautomator2
appium driver doctor xcuitest

# Запуск сервера
appium server --port 4723
```

### Зависимости Maven (Java-клиент)

```xml
<dependency>
    <groupId>io.appium</groupId>
    <artifactId>java-client</artifactId>
    <version>9.3.0</version>
</dependency>
```

---

## Мобильно-специфичные взаимодействия

Мобильные устройства предполагают жесты, которых нет в десктопных браузерах. Appium предоставляет API
для их имитации.

### Tap (касание)

```java
// Простое касание по элементу
WebElement element = driver.findElement(AppiumBy.id("com.example:id/button"));
element.click();

// Касание по координатам (PointerInput API — W3C Actions)
PointerInput finger = new PointerInput(PointerInput.Kind.TOUCH, "finger");
Sequence tap = new Sequence(finger, 0)
    .addAction(finger.createPointerMove(Duration.ZERO, PointerInput.Origin.viewport(), 200, 400))
    .addAction(finger.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
    .addAction(new Pause(finger, Duration.ofMillis(100)))
    .addAction(finger.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));

driver.perform(List.of(tap));
```

### Swipe (свайп)

```java
/**
 * Свайп вверх для скролла страницы вниз.
 * Координаты рассчитываются относительно размера экрана.
 */
public void swipeUp(AppiumDriver driver) {
    Dimension size = driver.manage().window().getSize();
    int startX = size.width / 2;
    int startY = (int) (size.height * 0.8);    // 80% от верха
    int endY = (int) (size.height * 0.2);      // 20% от верха

    PointerInput finger = new PointerInput(PointerInput.Kind.TOUCH, "finger");
    Sequence swipe = new Sequence(finger, 0)
        .addAction(finger.createPointerMove(Duration.ZERO, PointerInput.Origin.viewport(), startX, startY))
        .addAction(finger.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
        .addAction(finger.createPointerMove(Duration.ofMillis(600), PointerInput.Origin.viewport(), startX, endY))
        .addAction(finger.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));

    driver.perform(List.of(swipe));
}
```

### Scroll (скролл к элементу — Android)

```java
// UiAutomator2 — встроенный скролл до элемента
driver.findElement(AppiumBy.androidUIAutomator(
    "new UiScrollable(new UiSelector().scrollable(true))" +
    ".scrollIntoView(new UiSelector().text(\"Target Element\"))"
));
```

### Long Press (долгое нажатие)

```java
/**
 * Долгое нажатие на элемент (например, для вызова контекстного меню).
 */
public void longPress(WebElement element) {
    PointerInput finger = new PointerInput(PointerInput.Kind.TOUCH, "finger");
    Sequence longPress = new Sequence(finger, 0)
        .addAction(finger.createPointerMove(
            Duration.ZERO, PointerInput.Origin.element(element), 0, 0))
        .addAction(finger.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
        .addAction(new Pause(finger, Duration.ofSeconds(2)))  // удержание 2 секунды
        .addAction(finger.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));

    ((AppiumDriver) driver).perform(List.of(longPress));
}
```

### Pinch / Zoom (масштабирование)

```java
/**
 * Zoom in — разведение двух пальцев.
 * Используется два PointerInput для имитации двух пальцев.
 */
public void zoomIn(AppiumDriver driver, int centerX, int centerY) {
    PointerInput finger1 = new PointerInput(PointerInput.Kind.TOUCH, "finger1");
    PointerInput finger2 = new PointerInput(PointerInput.Kind.TOUCH, "finger2");

    // Палец 1 двигается вверх от центра
    Sequence seq1 = new Sequence(finger1, 0)
        .addAction(finger1.createPointerMove(Duration.ZERO, PointerInput.Origin.viewport(), centerX, centerY))
        .addAction(finger1.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
        .addAction(finger1.createPointerMove(Duration.ofMillis(500), PointerInput.Origin.viewport(), centerX, centerY - 200))
        .addAction(finger1.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));

    // Палец 2 двигается вниз от центра
    Sequence seq2 = new Sequence(finger2, 0)
        .addAction(finger2.createPointerMove(Duration.ZERO, PointerInput.Origin.viewport(), centerX, centerY))
        .addAction(finger2.createPointerDown(PointerInput.MouseButton.LEFT.asArg()))
        .addAction(finger2.createPointerMove(Duration.ofMillis(500), PointerInput.Origin.viewport(), centerX, centerY + 200))
        .addAction(finger2.createPointerUp(PointerInput.MouseButton.LEFT.asArg()));

    driver.perform(List.of(seq1, seq2));
}
```

---

## Challenges мобильного тестирования

### 1. Device Fragmentation (фрагментация устройств)

- Тысячи моделей Android-устройств с разными экранами, процессорами и версиями ОС
- Разрешения экранов: от 480x800 до 3200x1440
- Версии Android: от 10 до 15+ (каждая с особенностями)
- iOS: меньше моделей, но различия между iPhone SE, iPhone 15 Pro Max существенны

### 2. Производительность

- Ограниченные ресурсы (CPU, RAM, батарея)
- Приложение может работать иначе на слабых устройствах
- Перегрев устройства при длительных тестах
- Фоновые процессы влияют на стабильность тестов

### 3. Сетевые условия

- Переключение между Wi-Fi, 4G, 5G
- Потеря связи (offline-режим)
- Низкая скорость соединения (2G/3G)
- Роуминг и смена базовых станций

### 4. Прерывания

- Входящие звонки и SMS во время работы приложения
- Push-уведомления от других приложений
- Низкий заряд батареи — режим энергосбережения
- Поворот экрана (portrait/landscape)

### 5. Особенности платформ

| Android | iOS |
|---|---|
| Аппаратная кнопка "Назад" | Жесты навигации |
| Разные лаунчеры | Единообразный интерфейс |
| Widget-ы на рабочем столе | Ограниченные виджеты |
| Свободная установка APK (sideload) | Только App Store (или TestFlight) |
| Split-screen | Stage Manager (iPad) |

---

## Эмуляторы vs реальные устройства

| Критерий | Эмулятор/Симулятор | Реальное устройство |
|---|---|---|
| Стоимость | Бесплатно | Дорого (покупка парка устройств) |
| Скорость запуска | Быстрая | Зависит от устройства |
| Производительность тестов | Стабильная | Более реалистичная |
| Аппаратные функции | Ограничены (камера, GPS — эмуляция) | Полный доступ |
| Сетевые условия | Эмуляция | Реальные |
| CI-интеграция | Простая (запуск на сервере) | Требует device farm |
| Точность тестирования | Средняя | Высокая |
| Touch/жесты | Эмуляция | Реальные ощущения (ручное тестирование) |

### Рекомендуемая стратегия

- **Разработка и отладка** — эмуляторы (быстро, удобно)
- **Регрессионное тестирование** — эмуляторы + 2-3 реальных устройства
- **Релизное тестирование** — реальные устройства (top-5 популярных моделей)
- **Мануальное тестирование** — только реальные устройства

---

## Обзор инструментов мобильного тестирования

### Appium

| Аспект | Описание |
|---|---|
| Тип | Кросс-платформенный фреймворк |
| Платформы | Android, iOS, Windows |
| Языки | Java, Python, JS, C#, Ruby |
| Протокол | W3C WebDriver |
| Приложения | Native, Hybrid, Mobile Web |
| Стоимость | Бесплатный (open-source) |
| Плюсы | Универсальность, большое сообщество, единый API |
| Минусы | Медленнее нативных инструментов, сложная настройка |

### Espresso (Android)

| Аспект | Описание |
|---|---|
| Тип | Нативный фреймворк от Google |
| Платформы | Только Android |
| Языки | Kotlin, Java |
| Особенность | Работает внутри процесса приложения |
| Стоимость | Бесплатный |
| Плюсы | Очень быстрый, стабильный, автоматическая синхронизация с UI |
| Минусы | Требует доступ к исходному коду, только Android |

### XCUITest (iOS)

| Аспект | Описание |
|---|---|
| Тип | Нативный фреймворк от Apple |
| Платформы | Только iOS |
| Языки | Swift, Objective-C |
| Особенность | Интеграция с Xcode |
| Стоимость | Бесплатный |
| Плюсы | Быстрый, стабильный, нативный доступ к accessibility API |
| Минусы | Только iOS, требует macOS для запуска |

### Сравнительная таблица инструментов

| Критерий | Appium | Espresso | XCUITest |
|---|---|---|---|
| Кросс-платформенность | Да | Нет | Нет |
| Скорость | Средняя | Высокая | Высокая |
| Доступ к коду | Не нужен | Нужен | Нужен |
| Тип тестирования | Black-box | White/Gray-box | White/Gray-box |
| Кривая обучения | Средняя | Низкая (для Android-разработчиков) | Низкая (для iOS-разработчиков) |
| CI-интеграция | Любая | Firebase Test Lab | Xcode Cloud |

### Облачные device-farm

- **Firebase Test Lab** — облачная ферма от Google для Android (и частично iOS)
- **AWS Device Farm** — реальные устройства в облаке Amazon
- **BrowserStack App Live** — мобильные устройства для ручного и автоматизированного тестирования
- **SauceLabs** — эмуляторы и реальные устройства

---

## Связь с тестированием

- **Тест-стратегия** — выбор типа мобильного тестирования зависит от архитектуры приложения
- **Тестовая пирамида** — unit (Espresso/XCUITest) -> integration -> E2E (Appium)
- **Accessibility-тестирование** — критически важно на мобильных устройствах (VoiceOver, TalkBack)
- **Тестирование производительности** — мобильные приложения критичны к потреблению батареи и памяти
- **Тестирование обновлений** — миграция данных при обновлении версии приложения
- **Регрессионное тестирование** — автотесты на Appium покрывают основные сценарии
- **Exploratory testing** — на мобильных устройствах особенно эффективно ввиду разнообразия жестов и прерываний

---

## Типичные ошибки

1. **Тестирование только на эмуляторах** — пропускаются проблемы с реальными устройствами (производительность, жесты, камера)
2. **Игнорирование различных размеров экрана** — приложение может выглядеть иначе на маленьких экранах
3. **Не тестируют offline-режим** — приложение должно корректно обрабатывать отсутствие сети
4. **Жёсткие координаты в жестах** — координаты зависят от разрешения экрана, нужно использовать относительные значения
5. **Отсутствие тестирования прерываний** — входящий звонок может вызвать потерю данных
6. **Не учитывают время запуска Appium-сессии** — первый старт занимает 30-60 секунд, это нормально
7. **Путаница между `noReset` и `fullReset`** — `noReset=true` сохраняет данные; `fullReset=true` полностью переустанавливает приложение
8. **Тестирование только portrait-ориентации** — landscape может сломать layout

---

## Вопросы на интервью

### Уровень Junior

- Какие типы мобильных приложений вы знаете? В чём их различия?
- Что такое Appium? Какие приложения можно тестировать с его помощью?
- В чём разница между эмулятором и симулятором?
- Что такое desired capabilities? Приведите примеры для Android.
- Какие жесты специфичны для мобильных устройств?

### Уровень Middle

- Опишите архитектуру Appium. Как тестовый код взаимодействует с устройством?
- Как тестировать hybrid-приложение с помощью Appium? Что такое переключение контекстов?
- Сравните Appium и Espresso. Когда какой инструмент использовать?
- Какие проблемы создаёт фрагментация Android-устройств? Как с этим справляться?
- Как организовать параллельный запуск мобильных тестов?
- Что такое Appium Inspector и зачем он нужен?

### Уровень Senior

- Как построить стратегию мобильного тестирования для проекта с native-приложениями на Android и iOS?
- Как организовать инфраструктуру мобильного тестирования: свой device farm vs облако?
- Какие метрики производительности мобильного приложения вы бы отслеживали?
- Как тестировать push-уведомления в автоматическом режиме?
- Как организовать CI/CD для мобильных автотестов с учётом долгого времени сборки?
- Как обеспечить стабильность мобильных автотестов с учётом особенностей платформ?

---

## Практические задания

### Задание 1: Настройка окружения

1. Установите Appium Server и UiAutomator2 Driver
2. Создайте Android-эмулятор через Android Studio (Pixel 6, API 34)
3. Запустите Appium Server и проверьте подключение через Appium Inspector
4. Откройте калькулятор (или любое предустановленное приложение) и инспектируйте его элементы

### Задание 2: Первый мобильный тест

Напишите тест на Java + Appium, который:
1. Открывает мобильный Chrome на эмуляторе
2. Переходит на `https://the-internet.herokuapp.com`
3. Кликает на ссылку "Form Authentication"
4. Выполняет логин
5. Проверяет успешную авторизацию

### Задание 3: Мобильные жесты

Напишите вспомогательный класс `MobileGestureHelper` с методами:
- `swipeUp()` — свайп вверх
- `swipeDown()` — свайп вниз
- `swipeLeft()` — свайп влево
- `swipeRight()` — свайп вправо
- `longPress(WebElement element)` — долгое нажатие
- `doubleTap(WebElement element)` — двойное касание

Все координаты должны рассчитываться относительно размера экрана.

### Задание 4: Анализ стратегии

Вам дано приложение интернет-магазина, которое существует в трёх вариантах:
- Mobile Web (responsive)
- Android Native App
- iOS Native App

Составьте таблицу:
- Какие инструменты автоматизации использовать для каждого варианта
- Какие тесты можно переиспользовать между вариантами
- Какие тесты специфичны для каждого варианта
- Предложите распределение тестов по тестовой пирамиде

---

## Дополнительные ресурсы

- [Appium — Official Documentation](https://appium.io/docs/en/latest/)
- [Appium Java Client — GitHub](https://github.com/appium/java-client)
- [Appium Inspector — GitHub](https://github.com/appium/appium-inspector)
- [Android Developer — Testing](https://developer.android.com/training/testing)
- [Espresso — Official Guide](https://developer.android.com/training/testing/espresso)
- [XCUITest — Apple Documentation](https://developer.apple.com/documentation/xctest/user_interface_tests)
- [Firebase Test Lab](https://firebase.google.com/docs/test-lab)
- [Awesome Appium](https://github.com/SrinivasanTarget/awesome-appium) — коллекция ресурсов по Appium

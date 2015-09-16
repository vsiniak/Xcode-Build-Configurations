# Xcode Build Configurations

## Введение

Во время разработки нужно собирать одно и то же приложение для разных целей: для разработчика (для тестирования и дебага во время написания кода), для QA, для демо (для клиента), для публикации. При этом версии приложения для разных целей незначительно отличаются: как минимум, подписываются разными provisioning profile & code signing identity.

## Оглавление

* [Зачем?](#Зачем)
* [Решение для чего?](#Решение-для-чего)
* [Настройка конфигураций](#Настройка-конфигураций)
* [Типичные примеры](#Типичные-примеры)

## Зачем?

Очевидно, что вручную менять настройки/код перед билдом в зависимости от того, для кого собирается билд, плохо.

* Чем больше шагов у процесса сборки билда (поменять URL, включить удалённую сборку логов), тем больше вероятность пропустить/сделать неправильно какой-то из них. За пару месяцев она превратится в 100%.
* Чем больше шагов, тем дольше их выполнять. Одна дополнительная минута превратится в часы за пару месяцев.
* Повышается зависимость проекта от человека: только избранный разработчик будет знать все необходимые шаги.
* Невозможно [автоматизировать процесс сборки билдов](https://ru.wikipedia.org/wiki/Непрерывная_интеграция).

## Решение для чего?

Чаще всего версии одного и того же приложения для разных пользователей (программист, QA, клиент) имеют следующие отличия.

* Настройки компилятора и линковщика (provisioning profile & code signing identity, флаги компиляции).
* Выполнение разных скриптов в момент сборки (запуск утилиты, инкрементация номера билда).
* Незначительные изменения в коде:
    * разный URL сервера для debug/release;
    * инициализация удалённого сбора логов только в release;
    * разное логирование: локально в debug, удалённо в release и др.

Подготовить настройку сборки билдов с подобными отличиями можно, используя Xcode Build Configurations. В случаях, когда нужны б**о**льшие отличия (когда общий код должен входить и в библиотеку, и в демо-приложение, когда нужно собрать приложение и прогнать тесты и т.д.), нужно создавать отдельные target'ы внутри проекта и отдельные проекты внутри workspace.

## Настройка конфигураций

### Создание

Обычно в процессе разработки приложения достаточно использования трёх конфигураций.

* Debug -- конфигурация для сборки билда для тестирования и дебага разработчиком "по шнурку".
* Adhoc -- конфигурация для QA и демо-билдов.
* Release -- конфигурация для публикации в App Store.

Первая и последняя созданы Xcode по умолчанию. Создание конфигурации Adhoc:

1. Открыть настройки проекта, раздел Info.
2. Нажать иконку + и выбрать конфигурацию-основу: для Adhoc больше всего подходит Release. <img src="/sample/step1.png">
3. Указать соответствующий Preprocessor Macros (`ADHOC=1`) в настройках компилятора (настройки проекта, раздел Build Settings, секция Preprocessing). <img src="/sample/step2.png">

После создания конфигурации нужно настроить build scheme на её использование. Создание и настройка build scheme:

1. Кликнуть по build scheme и выбрать пункт Manage Schemes. <img src="/sample/step3.png">
2. Нажать иконку + и создать build scheme. <img src="/sample/step4.png">
3. Новая build scheme будет использоваться для сборки билдов всей командой разработчиков, поэтому нужно поставить галку в колоне Shared. <img src="/sample/step5.png">
4. Кликнуть по Edit и настроить build scheme на использование конфигурации Adhoc для действия Archive. <img src="/sample/step6.png">

### Применение

Для каждой конфигурации можно настроить следующее:

* Настройки компиляции и линковки. Настройка Preprocessor Macros выше -- один из примеров. <img src="/sample/using1.png">
* Выполнение специфичных скриптов во время сборки. <img src="/sample/using2.png">
* Специфичная логика в коде.

```objc
- (UIColor *)mainColor
{
    #if DEBUG
        return [UIColor whiteColor];
    #elif ADHOC
        return [UIColor greenColor];
    #else
        return [UIColor redColor];
    #endif
}
```

## Типичные примеры

* Настройка provisioning profile & code signing identity.
* Задание разных адресов back-end'а для разных версий билда.

```objc
#if DEBUG
   static NSString *const APIEnpoint = @"dev.api.example.com";
#elif ADHOC
   static NSString *const APIEnpoint = @"staging.api.example.com";
#else
   static NSString *const APIEnpoint = @"api.example.com";
#endif

NSURL *serverURL = [NSURL URLWithString:APIEndpoint];
```

* Подключение [Fabric](fabric.io) только для Adhoc/Release билдов.

**AppDelegate.m**

```objc
#if !DEBUG
    #import <Fabric/Fabric.h>
#endif
```

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
#if !DEBUG
    [Fabric with:@[[Crashlytics sharedInstance]]];
#endif
    return YES;
}
```

**Run Script Phase**

```bash
if [[ "$CONFIGURATION" != "Debug" ]]; then
    ./Frameworks/Fabric.framework/run APP_ID APP_SECRET
fi
```
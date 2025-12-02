+++
date = '2025-12-01T19:38:22+03:00'
draft = false
title = 'Baseline Profile + Remote Config - рабочее решение для разных окружений'
comments = true
+++

Привет. Это небольшая заметка о том, с чем я столкнулся при работе с baseline profile.

Коротко: baseline профили помогают повышать скорость выполнения кода примерно на 30% уже с первого запуска.
Но статья не про то, как их настраивать по документации, а про другую боль - что делать, если приложение активно использует Remote Config, а baseline гоняется на релизной сборке.

### **Контекст задачи.**
Допустим, у нас есть проект, где уже настроен baseline, и внутри приложения есть Remote Config, который позволяет гибко менять поведение фич.
Baseline profile у нас гоняется на релизной версии, а значит Remote Config там тоже будет релизный, с боевым окружением.
Remote Config - может быть любой от своей реализации до Firebase Remote Config. У вас многомодульная архитектура.

Отсюда проблема:
как в baseline сборке запускать условно "дебажный" Remote Config, чтобы baseline генерировался в нужном окружении и при этом не ломать релиз.

На практике сразу вылезло несколько ограничений:
- тесты baseline/benchmark работают в отдельном процессе
- есть ограничения DI во время запуска baseline
- нужно не допустить утечки тестового кода в настоящую релизную сборку

### **Проблема №1. Почему не получилось просто заинжектить всё через DI**
Первая мысль была простая: в тестах BaselineProfileGenerator мы сможем заинжектить DI и разрулить окружение Remote Config.
То есть в тесте мы получаем доступ к DI и переключаем Remote Config из теста напрямую.
На практике оказалось, что это невозможно по самой природе запуска baseline-тестов:

- тесты действительно запускаются и прогоняются в отдельном процессе
- когда приложение может быть уже закрыто, вручную (а тесты продолжают жить своей жизнью) что говорит о отдельном процесс
- DI, который у нас определен в приложении в app, мы не можем получить из другого процесса/из модуля macrobenchmark

То есть вариант "в тесте берем RemoteConfigProvider из DI и дергаем его напрямую" просто не работает. Из macrobenchmark так же мы не можем получить модуль app.

### **Проблема №2. Варианты переключения окружения в тестах.**
Раз DI не подходит, нужно искать способ менять Remote Config снаружи - не залезая напрямую в DI, а дергая приложение извне.
Сразу на ум приходят базовые компоненты Android-приложения:

1) Activity
Есть нюансы. Мы не сможем динамически менять параметры в середине сценария, нам придется каждый раз открывать Activity. Это неудобно и не очень красиво для автоматизации.

2) ContentProvider
Можно использовать как точку входа, но он срабатывает слишком рано и не дает удобного управляемого протокола.

3) BroadcastReceiver
Как раз этот вариант показался самым гибким:
- мы можем точечно менять конфиги
- есть управляемый контракт через action и extras
- легко дергается из тестов (через adb)

В итоге выбор пал на BroadcastReceiver.

### **Как через BroadcastReceiver поменять Remote Config.**
Я создал WbConfigReceiver, который принимает команду от теста, внутри дергает Remote Config и переключает нужный параметр, плюс при необходимости перезапускает приложение.

Псевдокод для теста:
```kotlin
@RunWith(AndroidJUnit4::class)
class RandomBaselineTest {

    private val packageName = "com.example.randomapp"

    @Test
    fun applyRemoteConfigAndRun() {
        sendAdbBroadcast("feature_x_enabled", "true")
        killApp()
        runFlow()
    }

    private fun sendAdbBroadcast(name: String, value: String) {
        device.executeShellCommand(
            "am broadcast " +
                    "-a com.example.randomapp.SET_WB_CONFIG " +
                    "--es name $name " +
                    "--es value $value"
        )
    }
}
```

И сам ресивер примерно такой: 
```kotlin
class WbConfigReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context?, intent: Intent?) {
        if (intent?.action != "com.example.randomapp.SET_WB_CONFIG") return

        val name = intent.getStringExtra("name") ?: return
        val value = intent.getStringExtra("value") ?: return

        WbApi().setConfig(name, value)
    }
}
```

В целом это работало хорошо, пока не появилась следующая проблема.

### **Проблема №3. Этот WbConfigReceiver нельзя выпускать в релиз**
По безопасности и здравому смыслу такой механизм нельзя оставлять доступным в релизной сборке.
Решение: вынести этот код в отдельный билд-тип, который используется только для baseline и не попадает в настоящий release.
Билд-тип вида nonMinifiedRelease (суть в том, что это специальный вариант для baseline/benchmark).
Он присутствует только в вариантах для baseline и не попадает в продактовый release.


### **Проблема №4. Как определить окружение baseline внутри DI**
Допустим, у нас в DI есть простое разветвление для релиза и дебага:
```kotlin
if (BuildConfig.DEBUG) {
    enableDebugRemoteConfig()
} else {
    enableRealRemoteConfig()
}
```
И мы хотим добавить к этому baseline. Напрашивается что-то вроде:
```kotlin
if (BuildConfig.DEBUG || BuildConfig.IS_BASELINE_RELEASE) {
    enableDebugRemoteConfig()
} else {
    enableRealRemoteConfig()
}
```
Теоретически это звучит логично, но дальше возникает вопрос:
- а как именно задавать и определять IS_BASELINE_RELEASE, чтобы понимать, когда у нас baseline, а когда обычный релиз.

Можно было бы попробовать "зашить" переменную так:
```kotlin
buildTypes {
    release {
        buildConfigField "Boolean", "IS_BASELINE_RELEASE", "true"
    }

    debug {
        buildConfigField "Boolean", "IS_BASELINE_RELEASE", "false"
    }
}
```
И в теории это бы работало, но дальше в дело вмешивается CI и automaticGenerationDuringBuild.

### **automaticGenerationDuringBuild и почему простого флага мало**
Локально, если руками запустить baseline командой из Gradle:
```kotlin
./gradlew generateReleaseBaselineProfile
```
все будет отрабатывать корректно.

Но если у вас есть окружение, где релизы собирает CI, там сценарий обычно такой:

- запускается сборка релиза
- при включенном automaticGenerationDuringBuild baseline профили генерируются автоматически во время билда проекта, без ручного запуска macrobenchmark-тестов

automaticGenerationDuringBuild - это флаг, который включает или выключает автоматическую генерацию baseline профилей во время сборки проекта.

И вот тут возникает проблема:
при билде релиза у нас запускается автоматическая генерация baseline. И нужно два окружения. 

Из-за automaticGenerationDuringBuild у нас при вызове runRelease будет что-то типо:
 1) Запуск generateReleaseBaselineProfile
 2) Запуск assembleRelease
 3) Готовая апк с вшитой оптимизацией baseline.

Но как нам определить и установить что в 1 пункте должно быть окружение дебаг для RemoteConfig а для 2 пункта окружение релиз для RemoteConfig.

1) SharedPreferences
Первая идея - хранить флаг в SharedPreferences и переключать его когда мы будем запускать тест, а читать его в DI при переключение RemoteConfig.
Это будет работать в каких-то сценариях, но выглядит костыльно и ненадежно.

2) CI переменные
Вторая идея - брать флаг из CI переменных.
Но переменная будет одинаковой и при запуске baseline, и при обычном релизе, потому что собирается один и тот же билд.
Определить изнутри приложения, baseline это или просто release, таким способом не получается.

### **Итоговое решение №4: отдельный BuildConfig модуль и флаг**
В итоге рабочее решение получилось таким:

1) В DI добавили отдельный BuildConfigProvider:
```kotlin
interface BuildConfigProvider {
    val isBaselineBuild: Boolean
}
```

2) В модуле DI создаем реализацию, которая читает флаг из BuildConfig:
```kotlin
val buildConfigModule = module {
    single<BuildConfigProvider> {
        object : BuildConfigProvider {
            override val isBaselineBuild: Boolean
                get() = BuildConfig.IS_BASELINE_BUILD
        }
    }
}
```
3) Например, для типа сборки release плагин создает типы сборки benchmarkRelease и nonMinifiedRelease. 
Эти типы сборки автоматически настраиваются для конкретного варианта использования и, как правило, не требуют настройки. 
В Gradle через onVariants помечаем нужный билд-тип (тот самый для baseline, nonMinifiedRelease):
```kotlin
androidComponents {
    onVariants(selector().withBuildType("nonMinifiedRelease")) { variant ->
        variant.buildConfigFields.put(
            "IS_BASELINE_BUILD",
            variant.buildConfigField("Boolean", true)
        )
    }
}
```
Остальные варианты сборки (обычный release, debug и т.д.) получают false.

4) В DI и коде теперь можно получить:
```kotlin
if (buildConfigProvider.isBaselineBuild) {
    enableLocalRemoteConfig() // окружение для baseline
} else {
    enableRealRemoteConfig()  // обычный прод
}
```

Получается в итоге:
- для baseline профильной сборки флаг isBaselineBuild будет true
- для обычного релиза - false
- это работает и локально, и на CI, и при automaticGenerationDuringBuild


### Как я проверял, что все действительно работает
Так как речь идет о релизных сборках, просто так залезть внутрь и посмотреть значения не получится у нас release сборка. Лог не будет работать. 
SharedPreference тоже. Debug включить тоже нет.

Поэтому я проверял поведение в два шага.

Шаг первый. Временный метод, который пишет состояние в файл. Я добавил маленький вспомогательный метод, который логировал текущее состояние в файл:
- флаг `isBaselineBuild`
- значение нужного Remote Config параметра

Условно:

```kotlin
fun logBaselineStateToFile(
    context: Context,
    isBaselineBuild: Boolean,
    remoteConfigValue: String
) {
    val file = File(context.filesDir, "baseline_state.txt")
    file.writeText(
        "isBaselineBuild=$isBaselineBuild; remoteConfig=$remoteConfigValue"
    )
}
```
Этот метод вызывался в момент инициализации Remote Config и в di. Можно вызывать где вам виднее для логирования.
Дальше я просто смотрел файл через adb:
```kotlin
adb shell run-as com.example.randomapp cat files/baseline_state.txt
```
Так можно было убедиться, что:
- для baseline сборки isBaselineBuild=true
- Remote Config действительно подменен на нужное окружение

Шаг второй. Проверка для BroadcastReceiver. Есть ли он в релизе.
Соберите релизную сборку и установить приложение. Если ресивер есть в манифесте, он будет виден в dumpsys package.
```kotlin
adb shell dumpsys package com.example.randomapp | grep WbConfigReceiver
```
Если строк нет - ресивера в зарегистрированных компонентах нет.
Если есть - значит где-то все-таки попал в манифест.
Так можно проверить именно релизный apk, который ты собрал.

### Изначально задача звучала просто: "переключить Remote Config в baseline".

Но под капотом оказались:
- особенности Android-процессов в MacroBenchmark
- ограничения DI
- особенности nonMinifiedRelease и вариантов сборок
- различия между release, baseline-release 

В итоге это вылилось в отдельную небольшую доработку архитектуры вокруг одного флага и билд-типа, которую вряд ли получится придумать с первого раза, не пройдя через пару неудачных вариантов.

Если у вас похожий кейс с baseline и Remote Config - возможно, эта схема с BuildConfigProvider и отдельным билд-типом с флагом сэкономит вам время и нервы.

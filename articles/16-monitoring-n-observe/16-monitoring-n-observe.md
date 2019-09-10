.NET – *управляемая среда выполнения*. Это означает, что в ней представлены высокоуровневые функции, которые управляют вашей программой за вас (из [Introduction to the Common Language Runtime (CLR), 2007 г.](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/intro-to-clr.md#fundamental-features-of-the-clr)):
<blockquote>
Среда выполнения предусматривает множество функций, поэтому их удобно разделить по следующим категориям:
<ol>
    <li><b>Основные функции</b>, которые влияют на устройство других. К ним относятся:<ol>
	<li>сборка мусора;</li>
	<li>обеспечение безопасности доступа к памяти и безопасности системы типов;</li>
	<li>высокоуровневая поддержка языков программирования.</li>
      </ol></li>
    <li><b>Дополнительные функции</b>– работают на базе основных. Многие полезные программы обходятся без них. К таким функциям относятся: <ol>
	<li>изолирование приложений с помощью AppDomains;</li>
	<li>защита приложений и изолирование в песочнице.</li>
      </ol></li>
    <li><b>Другие функции</b> – нужны всем средам выполнения, но при этом они не используют основные функции CLR. Такие функции отражают стремление создать полноценную среду программирования. К ним относятся:
      <ol>
	<li>управление версиями;</li>
	<li>отладка/профилирование;</li>
	<li>обеспечение взаимодействия.</li> </ol> </li></ol></blockquote>

Видно, что хотя отладка и профилирование не являются основными или дополнительными функциями, они находятся в списке из-за ‘*стремления создать полноценную среду программирования*’.

![](https://habrastorage.org/webt/e9/m2/nu/e9m2nuieof2txx60tcs9uypcq7k.jpeg)
<cut/>

**Оставшаяся часть поста описывает, какие функции [мониторинга](https://en.wikipedia.org/wiki/Application_performance_management), [обеспечения наблюдаемости](https://en.wikipedia.org/wiki/Observability) и [интроспекции](https://en.wikipedia.org/wiki/Virtual_machine_introspection) существуют в Core CLR, почему они полезны и каким образом среда предоставляет их**

## Диагностика

Для начала взглянем на диагностическую информацию, которой нас обеспечивает CLR. Традиционно для этого используется отслеживание событий для Windows (ETW).
Событий, о которых CLR предоставляет информацию, достаточно много. Они связаны со:

- сбором мусора (GC);
- JIT-компиляцией;
- модулями и доменами приложений;
- работой с тредами и конфликтами при блокировках;
- а также многим другим.

Например, здесь возникает [событие во время загрузки в AppDomain](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/corhost.cpp#https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/corhost.cpp), здесь [событие связано с выбросом исключения](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/exceptionhandling.cpp#L203), а здесь с [циклом выделения памяти сборщиком мусора](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/gctoclreventsink.cpp#L139-L144).

### Perf View

Если вы хотите увидеть события в системе трассировки (ETW), связанные с вашими .NET приложениями, я рекомендую использовать великолепный [инструмент PerfView](https://github.com/Microsoft/perfview) и начать с этих [обучающих видео](https://channel9.msdn.com/Series/PerfView-Tutorial) или этой презентации [PerfView: The Ultimate .NET Performance Tool](https://www.slideshare.net/InfoQ/perfview-the-ultimate-net-performance-tool). PerfView получил широкое признание за предоставляемую бесценную информацию. Например, инженеры Microsoft регулярно используют его для [анализа производительности](https://github.com/dotnet/corefx/issues/28834).

### Общая инфраструктура

Если вдруг непонятно из названия, трассировка событий в ETW доступна только под Windows, что не очень вписывается в кроссплатформенный мир .NET Core. Можно использовать [PerfView для анализа производительности под Linux](https://github.com/dotnet/coreclr/blob/release/2.1/Documentation/project-docs/linux-performance-tracing.md) (с помощью LTTng). Однако этот инструмент с командной строкой, под названием PerfCollect, только собирает данные. Возможности анализа и богатый пользовательский интерфейс (в том числе [flamegraphs](https://github.com/Microsoft/perfview/pull/502)) в настоящий момент доступны только в решениях для Windows.

Но если вы всё-таки хотите проанализировать производительность .NET под Linux, есть и другие подходы:

- [Getting Stacks for LTTng Events with .NET Core on Linux](https://blogs.microsoft.co.il/sasha/2018/02/06/getting-stacks-for-lttng-events-with-net-core-on-linux/)
- [Linux performance problem](https://github.com/dotnet/coreclr/issues/18465)

Вторая ссылка сверху ведёт на обсуждение новой **инфраструктуры EventPipe**, над которой работают в .NET Core (помимо EventSources & EventListeners). Цели её разработки можно посмотреть в документе [Cross-Platform Performance Monitoring Design](https://github.com/dotnet/designs/blob/master/accepted/cross-platform-performance-monitoring.md). На высоком уровне эта инфраструктура позволит создать единое место, куда CLR будет отсылать события, связанные с диагностикой и производительностью. Затем эти события будут перенаправляться к одному или более логерам, которые, например, могут включать ETW, LTTng и BPF. Необходимый логер будет определяться, в зависимости от ОС или платформы, на которой запущена CLR. Подробное объяснение плюсов и минусов различных технологий логирования см. в [.NET Cross-Plat Performance and Eventing Design](https://github.com/dotnet/coreclr/blob/master/Documentation/coding-guidelines/cross-platform-performance-and-eventing.md).

Ход работы по EventPipes отслеживается в рамках [проекта Performance Monitoring](https://github.com/dotnet/coreclr/projects/5) и связанных с ним [‘EventPipe’ Issues](https://github.com/dotnet/coreclr/search?q=EventPipe&type=Issues).

### Планы на будущее

Наконец, существуют планы по созданию контроллера профилирования производительности [Performance Profiling Controller](https://github.com/dotnet/designs/blob/master/accepted/performance-profiling-controller.md), перед которым стоят следующие задачи:

Контроллер должен управлять инфраструктурой профилирования и представлять данные о производительности, созданные компонентами .NET, отвечающими за диагностику рабочих характеристик, в простом и кроссплатформенном виде.

Согласно замыслу контроллер должен обеспечивать [следующие функциональные возможности через HTTP-сервер](https://github.com/dotnet/designs/blob/master/accepted/performance-profiling-controller.md#functionality-exposed-through-controller), получая все необходимые данные из инфраструктуры EventPipes:

**REST APIs**

- Принцип 1: простое профилирование: профилировать среду выполнения в течение периода времени X и возвращать трассировку.
- Принцип 1: продвинутое профилирование: начинать отслеживание (вместе с конфигурацией)
- Принцип 1: продвинутое профилирование: завершать отслеживание (ответом на этот вызов будет сама трассировка).
- Принцип 2: получать статистику, связанную со всеми счётчиками EventCounters или определённым EventCounter.

**HTML-страницы, просматриваемые через браузер**

- Принцип 1: текстовое представление всех стеков управляемого кода в процессе.
  - Создаёт моментальные снимки запущенных процессов для использования в качестве простого диагностического отчёта.
- Принцип 2: отображение текущего состояния (возможно с историей) счётчиков EventCounters.
  - Обеспечивает обзор существующих счётчиков и их значений.
  - НЕРЕШЁННАЯ ПРОБЛЕМА: не думаю, что существуют нужные публичные API, чтобы подсчитывать EventCounters.

Я очень хочу увидеть, что в итоге получится с контроллером профилирования производительности (КПП?). Думаю, если его встроят в CLR, он принесёт много пользы .NET. Такой функционал есть [в других средах выполнения](https://github.com/golang/go/wiki/Performance).

## Профилирование

Ещё одно эффективное средство, которое есть в CLR – API профилирования. Его (в основном) используют сторонние инструменты для подключения к среде выполнения на низком уровне. Подробнее про API можно узнать [из этого обзора](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/bb384493(v=vs.100)), но на высоком уровне с его помощью можно выполнять обратные вызовы, которые активируются, если:

- происходят события, связанные со сборщиком мусора;
- выбрасываются исключения;
- загружаются/выгружаются сборки;
- [и многое другое](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-4.0/ms230818(v=vs.100)).

*Изображение со страницы [BOTR Profiling API – Overview](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/profiling.md#profiling-api--overview)*

Кроме того, у него есть другие эффективные функции. Во-первых, вы можете установить обработчики, которые вызываются каждый раз, когда выполняется метод .NET, будь то в самой среде или из пользовательского кода. Эти обратные вызовы известны как обработчики Enter/Leave. Вот здесь есть [хороший пример](https://github.com/Microsoft/clr-samples/tree/master/ProfilingAPI/ReJITEnterLeaveHooks), как их использовать. Однако для этого нужно понять [конвенции вызовов для разных ОС и архитектур центрального процессора](https://github.com/dotnet/coreclr/issues/19023), что [не всегда просто](https://github.com/dotnet/coreclr/issues/18977). Также не забывайте, что API профилирования это COM-компонент, доступ к которому можно получить только из кода C/C++, но не из C#/F#/VB.NET.

Во-вторых, профилировщик может переписать IL-код любого .NET-метода перед JIT-компиляцией с помощью [SetILFunctionBody() API](https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerfunctioncontrol-setilfunctionbody-method). Этот API действительно эффективен. Он лежит в основе многих [инструментов APM .NET](https://stackify.com/application-performance-management-tools/). Подробнее о его использовании можно узнать из моего поста [How to mock sealed classes and static methods](https://mattwarren.org/2014/08/14/how-to-mock-sealed-classes-and-static-methods/) и сопутствующего кода.

### ICorProfiler API

Оказывается, чтобы API профилирования работал, в среде выполнения должны быть всяческие ухищрения. Просто посмотрите на обсуждение на странице [Allow rejit on attach](https://github.com/dotnet/coreclr/pull/19054) (подробную информацию о ReJIT см. в [ReJIT: A How-To Guide](https://blogs.msdn.microsoft.com/davbr/2011/10/12/rejit-a-how-to-guide/)).

Полное определение всех интерфейсов и обратных вызовов API профилирования можно найти в [\vm\inc\corprof.idl](https://github.com/dotnet/coreclr/blob/master/src/inc/corprof.idl) (см. [Interface description language](https://en.wikipedia.org/wiki/Interface_description_language)). Оно разбивается на 2 логические части. Одна часть – это интерфейс **Профилировщик -> Среда выполнения (EE)**, известный как `ICorProfilerInfo`:

```
// Объявление класса, который реализует интерфейсы ICorProfilerInfo*, позволяющие 
// профилировщику взаимодействовать со средой выполнения. Таким образом, библиотека DLL профилировщика получает
// доступ к частным структурам данных среды выполнения и другим вещам, которые никогда не должны
// экспортироваться за пределы среды.
```

Это реализуется в следующих файлах:

- [\vm\proftoeeinterfaceimpl.h](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/proftoeeinterfaceimpl.h)
- [\vm\proftoeeinterfaceimpl.inl](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/proftoeeinterfaceimpl.inl)
- [\vm\proftoeeinterfaceimpl.cpp](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/proftoeeinterfaceimpl.cpp)

Другая основная часть – обратные вызовы Среда выполнения -> Профилировщик, которые группируются под интерфейсом `ICorProfilerCallback`:

```
// Этот модуль использует обёртки вокруг вызовов
// интерфейсов ICorProfilerCallaback* профилировщика. Если коду в среде нужно вызвать
// профилировщик, он должен пройти через EEToProfInterfaceImpl.
```

Эти обратные вызовы реализуются в следующих файлах:

- [vm\eetoprofinterfaceimpl.h](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.h)
- [vm\eetoprofinterfaceimpl.inl](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.inl)
- [vm\eetoprofinterfaceimpl.cpp](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.cpp)
- [vm\eetoprofinterfacewrapper.inl](https://github.com/dotnet/coreclr/blob/release/2.1/src/vm/eetoprofinterfaceimpl.inl)

Наконец, стоит заметить, что API профилирования могут не работать на всех ОС и архитектурах, на которых работает .NET Core. Вот один из примеров: [ELT call stub issues on Linux](https://github.com/dotnet/coreclr/issues/18977). Подробную информацию см. в [Status of CoreCLR Profiler APIs](https://github.com/dotnet/coreclr/blob/release/2.1/Documentation/project-docs/profiling-api-status.md).

### Profiling v. Debugging

В качестве небольшого отступления нужно сказать, что профилирование и отладка всё-таки немного пересекаются. Поэтому полезно понимать, что предоставляют различные API в контексте .NET Runtime (взято из [CLR Debugging vs. CLR Profiling](https://blogs.msdn.microsoft.com/jmstall/2004/10/22/clr-debugging-vs-clr-profiling/)).

**Разница между отладкой и профилированием в CLR**

| Отладка |	Профилирование |
| ------- |-------------|
| Предназначена для поиска проблем с корректностью кода | Предназначено для диагностики и поиска проблем с производительностью |
| Может иметь очень высокий уровень вмешательства | Как правило, имеет низкий уровень вмешательства. Хотя профилировщик поддерживает изменение IL-кода или установку обработчиков enter/leave, всё это предназначено для инструментирования, а не для радикального изменения кода. |
| Основная задача – полный контроль цели. Это включает инспекцию, контроль выполнения (например команда set-next-statement) и внесение модификаций (функция Edit-and-Continue). | Основная задача – инспекция цели. Для этого предусмотрено инструментирование (изменение IL-кода, установка обработчиков enter/leave) |
| Обширный API и толстая модель объекта, заполненная абстракциями. | Небольшой API. Абстракций мало или отсутствуют.|
| Высокий уровень интерактивности: действия отладчика контролируются пользователем (или алгоритмом). Фактически редакторы и отладчики часто интегрированы (IDE). |Интерактивность отсутствует: данные обычно собираются без вмешательства пользователя, после чего анализируются. |
| Небольшое количество критических изменений, если нужна обратная совместимость. Мы думаем, что миграция с версии 1.1 на версию 2.0 отладчика, будет простой или не очень сложной задачей. | Большое количество критических изменений, если нужна обратная совместимость. Мы думаем, что миграция с версии 1.1 на версию 2.0 отладчика, будет тяжёлой задачей, идентичной его полному переписыванию. |

## Отладка

Разработчики по-разному понимают, что такое отладка. Например, я спросил в твиттере «как вы отлаживаете .NET программы” и получил [множество](https://mobile.twitter.com/matthewwarren/status/1030444463385178113) [разных ответов](https://mobile.twitter.com/matthewwarren/status/1030580487969038344). При этом ответы действительно содержали хороший список инструментов и методов, поэтому я рекомендую их посмотреть. Спасибо, #LazyWeb!

Я думаю, что лучше всего всю суть отладки отражает это сообщение:

<oembed>https://twitter.com/fortes/status/399339918213652480</oembed>

CLR предусматривает обширный список возможностей, связанных с отладкой. Однако зачем нужны эти средства? Как минимум три причины указаны в этом великолепном посте [Why is managed debugging different than native-debugging?](https://blogs.msdn.microsoft.com/jmstall/2004/10/10/why-is-managed-debugging-different-than-native-debugging/):

1. Отладку неуправляемого кода можно абстрагировать на аппаратном уровне, но отладку управляемого кода необходимо абстрагировать на уровне IL-кода.
2. Для отладки управляемого кода требуется много информации, которая недоступна до выполнения.
3. Отладчик управляемого кода должен координировать действия со сборщиком мусора (GC)

Поэтому для удобства использования CLR должен предоставлять API отладки высокого уровня, известный как `ICorDebug`. Он показан на рисунке ниже, отображающем общий сценарий отладки (источник: BOTR):

### ICorDebug API

Принцип реализации и описание разных компонентов взято из [CLR Debugging, a brief introduction](https://github.com/Microsoft/clrmd/blob/master/Documentation/GettingStarted.md#clr-debugging-a-brief-introduction):

> Вся поддержка отладки в .Net реализуется поверх dll-библиотеки, которую мы называем The Dac. Этот файл (обычно под названием `mscordacwks.dll`) является структурным элементом как для нашего публичного API отладки (`ICorDebug`), так и двух частных API отладки: SOS-Dac API и IXCLR.
> В идеальном мире все бы использовали `ICorDebug`, наш публичный API. Однако в `ICorDebug` не хватает множества функций, которые нужны разработчикам инструментов.  Эта проблема, которую мы пытаемся исправить, где можем. Однако эти улучшения присутствуют только в CLR v.next, но не в более ранних версиях CLR. Фактически поддержка отладки по аварийному дампу появилась в `ICorDebug` API только с выходом CLR v4. Все, кто используют аварийные дампы для отладки в CLR v2 не смогут применить `ICorDebug` совсем.

*(Дополнительную информацию см. в SOS & ICorDebug)*

На самом деле `ICorDebug` API делится более чем на 70 интерфейсов. Я не буду приводить их все, но покажу, по каким категориям их можно разбить. Подробную информацию см. в Partition of ICorDebug, где этот список был опубликован.

- **Верхний уровень**: ICorDebug + ICorDebug2 – интерфейсы верхнего уровня, которые великолепно служат в качестве коллекции объектов ICorDebugProcess.
- **Обратные вызовы**: События отладки управляемого кода отсылаются через методы к объекту обратного вызова, реализуемого отладчиком.
- **Процесс**: Этот набор интерфейсов представляет работающий код и включает API, связанные с обработкой событий.
- **Инспекция кода / типов**: В основном работает со статическими PE-образами, но есть и удобные методы для реальных данных.
- **Контроль выполнения**: Возможность наблюдать за выполнением треда. На практике это означает возможность задавать точки останова (F9) и делать пошаговое прохождение кода (F11 вход в код, F10 обход кода, S+F11 выход их кода). Функция контроля выполнения ICorDebug работает только в управляемом коде.
- **Треды + стеки вызовов**: Стеки вызовов являются основой для функций инспекции, реализуемых отладчиком. Работа со стеком вызовов осуществляется с помощью следующих интерфейсов. ICorDebug поддерживает отладку только управляемого кода и, соответственно, можно отслеживать стек только управляемого кода.
- **Инспекция объектов**: Инспекция объектов – часть API, которая позволяет видеть значения переменных в отлаживаемом коде. Для каждого интерфейса я привожу метод MVP, который, как мне кажется, должен кратко описывать цель этого интерфейса.

Как и с API профилирования, уровни поддержки API отладки отличаются в зависимости от ОС и архитектуры процессора. Например, на август 2018 всё ещё нет решения для Linux ARM по диагностике и отладке управляемого кода. Подробную информацию о поддержке Linux можно посмотреть в посте Debugging .NET Core on Linux with LLDB и репозитории Diagnostics от Microsoft, которая стремится сделать отладку .NET программ под Linux проще.

Наконец, если хотите посмотреть, как `ICorDebug` API выглядят в C#, взгляните на обёртки в библиотеке CLRMD, включая все доступные обратные вызовы (подробнее о CLRMD будет рассказано далее в этом посте).

### SOS и DAC

Компонент доступа к данным (DAC) подробно рассматривается на [странице BOTR](https://github.com/dotnet/coreclr/blob/master/Documentation/botr/dac-notes.md). По сути, он обеспечивает внепроцессный доступ к структурам данных CLR, чтобы информацию внутри них можно было прочитать из другого процесса. Таким образом, отладчик (через `ICorDebug`) или [расширение ‘Son of Strike’ (SOS)](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension) может получить доступ к запущенному экземпляру CLR или дампу памяти и найти, например:

- все запущенные треды;
- объекты в управляемой куче;
- полную информацию о методе, в том числе машинный код;
- текущую трассировку стека.

**Небольшое отступление**: если хотите узнать, откуда взялись эти странные названия и получить небольшой урок истории .NET, посмотрите [этот ответ на Stack Overflow](https://stackoverflow.com/questions/21361602/what-the-ee-means-in-sos/21363245#21363245).

Полный список команд [SOS впечатляет](https://stackoverflow.com/questions/21361602/what-the-ee-means-in-sos/21363245#21363245). Если использовать его вместе с WinDBG, то можно узнать, что происходит внутри вашей программы и CLR на очень низком уровне. Чтобы увидеть, как всё реализовано, давайте посмотрим на команду `!HeapStat`, которая выводит описание размеров различных куч, которые использует .NET GC:

(Изображение взято из SOS: Upcoming release has a few new commands – HeapStat)

Вот поток кода, который показывает, как SOS и DAC работают вместе:
- **SOS** Полная команда `!HeapStat` ([ссылка](https://github.com/dotnet/coreclr/blob/release/2.1/src/ToolBox/SOS/Strike/strike.cpp#L4605-L4782))
- **SOS** Код в команде `!HeapStat`, который работает с Workstation GC (ссылка)
- **SOS**  Функция `GCHeapUsageStats(..)`, которая выполняет самую тяжёлую часть работы ([ссылка](https://github.com/dotnet/coreclr/blob/release/2.1/src/ToolBox/SOS/Strike/eeheap.cpp#L768-L850))
- **Shared** Структура данных `DacpGcHeapDetails`, которая содержит указатели на основные данные в куче GC, такие как сегменты, битовые маски и отдельные поколения ([ссылка](https://github.com/dotnet/coreclr/blob/release/2.1/src/inc/dacprivate.h#L690-L722)).
- **DAC** Функция `GetGCHeapStaticData`, которая заполняет структуру `DacpGcHeapDetails` ([ссылка](https://github.com/dotnet/coreclr/blob/release/2.1/src/inc/dacprivate.h#L690-L722))
- **Shared** Структура данных `DacpHeapSegmentData`, которая содержит информацию об отдельном сегменте кучи GC ([ссылка](https://github.com/dotnet/coreclr/blob/release/2.1/src/inc/dacprivate.h#L738-L771))
- **DAC** `GetHeapSegmentData(..)`, которая заполняет структуру `DacpHeapSegmentData` ([ссылка](https://github.com/dotnet/coreclr/blob/release/2.1/src/debug/daccess/request.cpp#L2829-L2868))

### Сторонние отладчики

Поскольку Microsoft опубликовала API отладки, сторонние разработчики смогли использовать интерфейсы `ICorDebug`. Вот список тех, которые мне удалось найти:

- [Debugger for .NET Core runtime](https://github.com/Samsung/netcoredbg) от [Samsung](https://github.com/Samsung)
    - Отладчик позволяет использовать интерфейс адаптера отладки GDB/MI или VSCode, чтобы исправлять ошибки в приложениях .NET из-под среды выполнения .NET Core.
    - *Вероятно*, написан как часть проекта по портированию [.NET Core в их Tizen OS](https://developer.tizen.org/blog/celebrating-.net-core-2.0-looking-forward-tizen-4.0)
- [dnSpy](https://github.com/0xd4d/dnSpy) – отладчик .NET и редактор сборок
    - [Очень мощный инструмент](https://github.com/0xd4d/dnSpy#features-see-below-for-more-detail). Это отладчик, редактор сборок, шестнадцатеричный редактор, декомпилятор и многое другое.
- [MDbg.exe (.NET Framework Command-Line Debugger)](https://docs.microsoft.com/en-us/dotnet/framework/tools/mdbg-exe)
    - Доступен в виде [NuGet-пакета](https://www.nuget.org/packages/Microsoft.Samples.Debugging.MdbgEngine). Его также можно скачать с  [репозитория GitHub](https://github.com/SymbolSource/Microsoft.Samples.Debugging/tree/master/src) или сайта Microsoft.
    - Однако в данный момент MDBG, кажется, не поддерживает работу с .NET Core. Подробную информацию см. в [Port MDBG to CoreCLR](https://github.com/dotnet/coreclr/issues/1145) и [ETA for porting mdbg to coreclr](https://github.com/dotnet/coreclr/issues/8999).
- [JetBrains ‘Rider’](https://blog.jetbrains.com/dotnet/2017/02/23/rider-eap-18-coreclr-debugging-back-windows/) позволяет проводить отладку .NET Core на Windows.
    - С этим инструментом наблюдались [некоторые проблемы](https://blog.jetbrains.com/dotnet/2017/02/23/rider-eap-18-coreclr-debugging-back-windows/) в связи с лицензированием
    - Более подробную информацию можно найти в [этой ветке HackerNews](https://news.ycombinator.com/item?id=17323911)

### Дампы памяти

Последнее, о чём мы поговорим, это дампы памяти, которые можно получить из работающей системы и проанализировать вне её. Среда выполнения .NET всегда отлично поддерживала [создание дампов памяти под Windows](https://news.ycombinator.com/item?id=17323911). А теперь, когда .NET Core стала кроссплатформенной, появились инструменты, выполняющие ту же задачу на других ОС.

При использовании дампов памяти иногда сложно получить правильные, совпадающие версии файлов SOS и DAC. К счастью, Microsoft недавно выпустила dotnet symbol CLI-инструмент, который:
может скачивать все необходимые для отладки файлы (наборы символов, модули, SOS и DAC файлы для определённого модуля coreclr) для любого определённого дампа ядра, минидампа или файлов любой поддерживаемой платформы, в том числе в формате ELF, MachO, Windows DLL, PDB и портативный PDB.
Наконец, если вы хотя-бы чуть-чуть занимаетесь анализом дампов памяти, рекомендую взглянуть на великолепную библиотеку CLR MD, которую Microsoft выпустила несколько лет назад. Я уже писал про её функции. Вкратце, с помощью библиотеки можно работать с дампами памяти через интуитивно понятный C# API, содержащий классы, которые обеспечивают доступ к ClrHeap, GC Roots, CLR Threads, Stack Frames и многому другому. Фактически CLR MD может реализовывать большинство (если не все) из SOS-команд.
Узнать о том, как она работает можно из этого поста:
Управляемая библиотека ClrMD – это обёртка вокруг API отладки, предназначенных только для внутреннего применения в CLR. Несмотря на то что эти API очень эффективны для диагностики, мы не поддерживаем их в виде публичных, задокументированных релизов, поскольку их использование сложно и тесно связано с другими особенностями реализации CLR. ClrMD решает эту проблему, предоставляя лёгкую в использовании, управляемую обёртку вокруг этих API отладки низкого уровня.
Сделав эти API доступными в официально поддерживаемой библиотеке, Microsoft дала разработчикам возможность создать широкий диапазон инструментов на базе CLRMD.
________________________________________
В заключение можно сказать, что среда выполнения .NET предоставляет широкий набор возможностей для диагностики, отладки и профилирования, которые позволяют глубоко понять, что происходит внутри CLR.
________________________________________
Обсудите этот пост на HackerNews, /r/programmingили /r/csharp
________________________________________
Дополнительные материалы
Здесь я добавил дополнительные ссылки на материалы по темам, затронутым в этом посте.
Общие
•	Monitoring and Observability
•	Monitoring and Observability — What’s the Difference and Why Does It Matter?
События ETW и PerfView:
•	ETW - Monitor Anything, Anytime, Anywhere (pdf) от Dina Goldshtein
•	Make ETW Great Again (pdf)
•	Logging Keystrokes with Event Tracing for Windows (ETW)
•	PerfView создан на базе Microsoft.Diagnostics.Tracing.TraceEvent. Это означает, что вы легко можете написать код, который будет собирать события ETW. Можно посмотреть вот этот пример: ‘Observe JIT Events’ sample
•	Более подробную информацию можно найти в TraceEvent Library Programmers Guide
•	Performance Tracing on Windows
•	CoreClr Event Logging Design
•	Bringing .NET application performance analysis to Linux (введение в блоге .NET)
•	Bringing .NET application performance analysis to Linux (более подробный пост в блоге LTTng)
API профилирования
•	Если собираетесь использовать API профилирования, почитайте статьи в блоге Дэвида Бромэна про API профилирования в CLR. Это хорошая отправная точка.
•	BOTR - Profiling – объясняет, какие возможности даёт API профилирования, что с ним можно делать и как использовать.
•	BOTR - Profilability – описание того, что нужно сделать в самом CLR, чтобы профилирование стало возможным.
•	Интересная презентация The .NET Profiling API (pdf)
•	Thought(s) on managed code injection and interception
•	CLR 4.0 advancements in diagnostics
•	Profiling: How to get GC Metrics in-process
Отладка
•	Опять же, если вы хотите эффективно использовать API отладки, рекомендую прочитать блог Майка Сталла об API отладки в .NET, в том числе:
o	How do Managed Breakpoints work?
o	Debugging any .Net language
o	How can I use ICorDebug?
o	You can’t debug yourself
o	Tool to get snapshot of managed callstacks
•	BOTR Data Access Component (DAC) Notes
•	What’s New in CLR 4.5 Debugging API?
•	Writing a .Net Debugger, Part 2, Part 3 and Part 4
•	Writing an automatic debugger in 15 minutes (yes, a debugger!)
•	Статья add SOS DumpAsync command
•	Question: what remaining SOS commands need to be ported to Linux/OS X
Дампы памяти:
•	Creating and analyzing minidumps in .NET production applications
•	Creating Smaller, But Still Usable, Dumps of .NET Applications и More on - MiniDumper: Getting the Right Memory Pages for .NET Analysis
•	Minidumper – A Better Way to Create Managed Memory Dumps
•	ClrDump is a set of tools that allow to produce small minidumps of managed applications


[![](https://hsto.org/getpro/habr/branding/7c7/f79/073/7c7f790735362496510443d19aacb34d.png)](https://habr.com/ru/company/clrium/blog/465081/)

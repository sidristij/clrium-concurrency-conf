# Инструменты для изучения внутреннего устройства .NET

Заглянуть «под капот» кода или посмотреть на внутреннее устройство CLR можно с помощью множества инструментов.
Этот пост родился из [твита](https://twitter.com/matthewwarren/status/973940550473797633), и я должен поблагодарить всех, кто помог составить список подходящих инструментов. Если я пропустил какие-то из них, напишите в комментариях.

## Необходимые упоминания

Во-первых, я должен упомянуть, что [хороший отладчик уже присутствует в Visual Studio](https://msdn.microsoft.com/en-us/library/sc65sadd.aspx?f=255&MSPPError=-2147217396) и [VSCode](https://code.visualstudio.com/docs/editor/debugging). Также существует множество хороших (коммерческих) [профилировщиков .NET](https://stackoverflow.com/questions/3927/what-are-some-good-net-profilers) и [инструментов мониторинга приложений](https://www.quora.com/What-is-the-best-NET-Application-Server-Monitoring-Tool), на которые стоит взглянуть. Например, недавно я попробовал поработать с [Codetrack](http://www.getcodetrack.com/) и был впечатлён его возможностями.

Однако оставшийся пост посвящён **инструментам для выполнения отдельных задач**, которые позволят **лучше понять**, что происходит. Все инструменты имеют **открытый исходный код**.

### [PerfView](https://github.com/Microsoft/perfview) от [Вэнса Моррисона](https://blogs.msdn.microsoft.com/vancem/)

PerfView – великолепный инструмент, который я использую уже несколько лет. Он работает на основе [трассировки событий Windows](https://msdn.microsoft.com/en-us/library/windows/desktop/bb968803(v=vs.85).aspx?f=255&MSPPError=-2147217396) (ETW) и позволяет лучше понять, что происходит внутри CLR, а также даёт возможность получить профиль использования памяти и центрального процессора. Чтобы освоить инструмент, придётся впитать много информации, например с помощью [обучающих видео](https://channel9.msdn.com/Series/PerfView-Tutorial), но это стоит потраченного времени и усилий.

Инструмент настолько полезен, что сами инженеры Microsoft используют его, а многие из недавних [улучшений производительности в MSBuild](https://blogs.msdn.microsoft.com/dotnet/2018/02/02/net-core-2-1-roadmap/#user-content-build-time-performance) появились после [анализа узких мест](https://github.com/Microsoft/msbuild/search?q=PerfView&type=Issues) с помощью PerfView.
Инструмент создан на базе библиотеки [Microsoft.Diagnostics.Tracing.TraceEvent library](https://www.nuget.org/packages/Microsoft.Diagnostics.Tracing.TraceEvent/), которую можно использовать для создания собственных инструментов. Кроме того, поскольку исходный код библиотеки является открытым, в ней благодаря сообществу появилось множество полезных функций, например [графики flame-graphs](https://github.com/Microsoft/perfview/pull/502):

![](./PerfView Flamegraphs.png)

### [SharpLab](https://sharplab.io/) от [Андрея Щёкина](https://twitter.com/ashmind)

SharpLab появился как инструмент для проверки IL-кода, генерируемого компилятором Roslyn, и со временем превратился [в нечто большее](https://github.com/ashmind/SharpLab):

> SharpLab – интерактивная среда для запуска кода .NET, в которой отображаются промежуточные шаги и результаты компиляции кода. Некоторые функции языка – всего лишь обёртки для других, например using() становится try/catch. С помощью SharpLab вы увидите код, как его видит компилятор и лучше поймёте суть языков .NET.

Инструмент поддерживает C#, Visual Basic и F#, но самыми интересными функциями в нём являются Decompilation/Disassembly:
Функции декомпиляции/дизассемблирования можно использовать для:

  1. C#
  2. Visual Basic
  3. IL
  4. JIT Asm (нативный Asm Code)

Вы правильно поняли: инструмент выводит [код ассемблера](https://sharplab.io/#v2:EYLgZgpghgLgrgJwgZwLQBEJinANjASQDsYIFsBjCAgWwAdcIaITYBLAeyIBoYQpkNAD4ABAAwACEQEYA3AFgAUCIDMUgEwSAwhIDeSiYalqRAFgkBZABQBKPQaOOAblAQTSyGBIC8EgKwAdGIKio6OMgCcVh4wNiGOAL5KCUA==), который .NET JIT генерирует из вашего кода C#:

![](./PerfView Sample.png)

### [Object Layout Inspector](https://github.com/SergeyTeplyakov/ObjectLayoutInspector) от [Сергея Теплякова](https://twitter.com/STeplyakov)

С помощью этого инструмента вы сможете проанализировать структуру .NET объектов в памяти, т.е. как JITter расположил поля, принадлежащие вашему классу или структуре. Это полезно при написании высокопроизводительного кода. Кроме того, приятно иметь инструмент, который сделает сложную работу за нас.

Официальной документации, которая бы описывала структуру полей, не существует, поскольку авторы CLR оставили за собой право изменить её в будущем. Но знания о структуре могут быть полезны, если вы работаете над быстродействующим приложением.
Как можно изучить структуру? Можно посмотреть на необработанную память в Visual Studio или использовать команду `!dumpobj` в [SOS Debugging Extension](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension). Оба подхода требуют много усилий, поэтому мы создадим инструмент, который будет выводить структуру объекта во время выполнения.

Согласно примеру в репозитории GitHub, если вы используете `TypeLayout.Print<NotAlignedStruct>()` с подобным кодом:

```csharp
public struct NotAlignedStruct
{
    public byte m_byte1;
    public int m_int;

    public byte m_byte2;
    public short m_short;
}
```

появится следующий вывод, который точно покажет, как CLR расположит struct в памяти на основании правил оптимизации и заполнения байтами.

```txt
Size: 12. Paddings: 4 (%33 of empty space)
|================================|
|     0: Byte m_byte1 (1 byte)   |
|--------------------------------|
|   1-3: padding (3 bytes)       |
|--------------------------------|
|   4-7: Int32 m_int (4 bytes)   |
|--------------------------------|
|     8: Byte m_byte2 (1 byte)   |
|--------------------------------|
|     9: padding (1 byte)        |
|--------------------------------|
| 10-11: Int16 m_short (2 bytes) |
|================================|
```

### [The Ultimate .NET Experiment (TUNE)](http://tooslowexception.com/the-ultimate-net-experiment-project/) от [Конрада Кокосы](https://twitter.com/konradkokosa)

Как сказано [на странице GitHub](https://github.com/kkokosa/Tune), TUNE – многообещающий инструмент. Он поможет изучить внутреннее устройство .NET и способы повышения производительности с помощью экспериментов с кодом C#.

Подробную информацию о нём можно узнать из [этого поста](http://tooslowexception.com/the-ultimate-net-experiment-project/), но на высоком уровне он функционирует [следующим образом](https://github.com/kkokosa/Tune):

  - напишите работающий пример кода на C#, который содержит хотя бы один класс с публичным методом, принимающим один строковый параметр. Код запускается с помощью кнопки Run. Вы можете включить любое количество методов и классов. Но помните, что первый публичный метод из первого публичного класса будет выполнен с использованием первого параметра из окна ввода под кодом;
  - нажмите кнопку Run, чтобы скомпилировать и выполнить код. Кроме того, он будет компилирован в IL-код и код ассемблера в соответствующих вкладках;
  - пока Tune работает (в том числе во время выполнения кода), инструмент строит график, отображающий данные сборщика мусора. Он содержит информацию о размерах поколений и сеансах сбора мусора (представлены в виде вертикальных линий с числом внизу, которое указывает на то, в каком поколении выполняется сборка мусора).

Выглядит это следующим образом:

![](TUNE Screenshot.png)

## Инструменты на базе CLR Memory Diagnostics (ClrMD)

Наконец, давайте взглянем на определённую категорию инструментов. С момента выхода .NET разработчики всегда могли использовать [WinDBG](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/getting-started-with-windbg) и [SOS Debugging Extension](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension), чтобы посмотреть, что происходит в среде выполнения .NET. Однако, это не самые простые инструменты для первого знакомства и, как сказано в следующем твите, не всегда самые продуктивные:

```twitter
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Very true. While hardcore windbg/sos skills are impressive, teaching them to novices without outlining the easier, less threatening alternatives is harmful. <a href="https://t.co/jQanX9LVtz">https://t.co/jQanX9LVtz</a></p>&mdash; Omer Raviv (@omerraviv) <a href="https://twitter.com/omerraviv/status/973923339906486272?ref_src=twsrc%5Etfw">March 14, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
```

К счастью, Microsoft сделал доступной библиотеку [ClrMD](https://mattwarren.org/2016/09/06/Analysing-.NET-Memory-Dumps-with-CLR-MD/) (также известную, как [Microsoft.Diagnostics.Runtime](https://www.nuget.org/packages/Microsoft.Diagnostics.Runtime)), и теперь любой может создать инструмент для анализа дампов памяти программ .NET. Подробную информацию можно прочитать [в официальном блоге](https://blogs.msdn.microsoft.com/dotnet/2013/05/01/net-crash-dump-and-live-process-inspection/). Я также рекомендую взглянуть на [ClrMD.Extensions](https://github.com/JeffCyr/ClrMD.Extensions), которые *“… обеспечивают интеграцию с LINPad и делают использование ClrMD ещё проще”*.

Я хотел собрать список всех существующих инструментов и призвал на помощь [твиттер](https://twitter.com/matthewwarren/status/973940550473797633). Напоминалочка самому себе: осторожнее с твитами. Менеджер, ответственный за WinDBG, может прочитать их и расстроиться!

```https://twitter.com/matthewwarren/status/973940550473797633```

Большинство этих инструментов работают на базе ClrMD, потому что так проще всего. Но при желании можно использовать [COM-интерфейсы напрямую](https://twitter.com/goldshtn/status/973941389791809540). Также нужно заметить, что любой инструмент на базе ClrMD **не является кроссплатформенным**, поскольку сама ClrMD предназначена только для Windows. Описание кроссплатформенных вариантов можно найти в [Analyzing a .NET Core Core Dump on Linux](http://blogs.microsoft.co.il/sasha/2017/02/26/analyzing-a-net-core-core-dump-on-linux/).

Наконец, чтобы как-то соблюсти баланс, недавно появилась [улучшенная версия WinDBG](https://blogs.msdn.microsoft.com/windbg/2017/08/28/new-windbg-available-in-preview/), которой сразу же попытались добавить функциональность:

- [Extending the new WinDbg, Part 1 – Buttons and commands](http://labs.criteo.com/2017/09/extending-new-windbg-part-1-buttons-commands/)
- [Extending the new WinDbg, Part 2 – Tool windows and command output](http://labs.criteo.com/2018/01/extending-new-windbg-part-2-tool-windows-command-output/)
- [Extending the new WinDbg, Part 3 – Embedding a C# interpreter](http://labs.criteo.com/2018/05/extending-new-windbg-part-3-embedding-c-interpreter/)
- [WinDBG extension + UI tool extensions](https://github.com/chrisnas/DebuggingExtensions) и ещё [здесь](https://github.com/kevingosse/windbg-extensions)
- [NetExt](https://github.com/rodneyviana/netext) – приложение WinDBG, которое [облегчает отладку в .NET](https://blogs.msdn.microsoft.com/rodneyviana/2015/03/10/getting-started-with-netext/) по сравнению с текущими опциями: sos или psscor. Также см. [эту статью InfoQ](https://www.infoq.com/news/2013/11/netext).

**После всех этих слов переходим к списку:**

- [SuperDump](https://www.slideshare.net/ChristophNeumller/large-scale-crash-dump-analysis-with-superdump) ([GitHub](https://github.com/Dynatrace/superdump))
  - Средство для автоматического анализа аварийного дампа ([презентация](https://www.slideshare.net/ChristophNeumller/large-scale-crash-dump-analysis-with-superdump))
- [msos](https://github.com/goldshtn/msos/wiki) ([GitHub](https://github.com/goldshtn/msos))
  - Среда с интерфейсом командной строки типа WinDbg для выполнения SOS-команд при отсутствии SOS.
- [MemoScope.Net](https://github.com/fremag/MemoScope.Net/wiki) ([GitHub](https://github.com/fremag/MemoScope.Net))
  - Инструмент для анализа памяти процесса в .NET. Можно сделать дамп памяти приложения в файл и прочитать его позже.
  - Файл содержит все данные (объекты) и информацию о тредах (состояние, стек, стек вызовов). MemoScope.Net проанализирует данные и поможет найти утечки памяти и взаимные блокировки.
- [dnSpy](https://github.com/0xd4d/dnSpy#dnspy) ([GitHub](https://github.com/0xd4d/dnSpy))
  - Отладчик и редактор сборок .NET
  - Его можно использовать для редактирования и отладки сборок, даже если у вас нет исходного кода.
- [MemAnalyzer](https://aloiskraus.wordpress.com/2017/08/17/memanalyzer-v2-5-released/) ([GitHub](https://github.com/Alois-xx/MemAnalyzer))
  - Инструмент анализа памяти для управляемого кода. Присутствует интерфейс командной строки.
  - Подобно `!DumpHeap` в Windbg может определить, какие объекты занимают больше всего места в куче без необходимости устанавливать отладчик.
- [DumpMiner](https://mycodingplace.wordpress.com/2016/11/24/dumpminer-ui-tool-for-playing-with-clrmd/) ([GitHub](https://github.com/dudikeleti/DumpMiner))
  - Инструмент с графическим интерфейсом для работы с ClrMD. Больше возможностей [появится в будущем](https://twitter.com/dudi_ke/status/973930633935409153).
- [Trace CLI](http://devops.lol/tracecli-a-production-debugging-and-tracing-tool/) ([GitHub](https://github.com/ruurdk/TraceCLI/))
  - Инструмент для отладки и отслеживания во время эксплуатации
- [Shed](https://github.com/enkomio/shed) ([GitHub](https://github.com/enkomio/shed))
  - Shed – приложение, которое анализирует выполнение программы в .NET. Его можно использовать для анализа вредоносного ПО, чтобы получить данные о том, какая информация сохраняется, при запуске такого ПО. Shed может:
    - извлекать все объекты, хранящиеся в управляемой куче;
    - выводить строки, хранящиеся в памяти;
    - создавать моментальный снимок кучи в формате JSON для последующей обработки;
    - делать дамп всех модулей, загруженных в память.

Вы можете найти множество других инструментов, [которые используют ClrMD](https://github.com/search?p=2&q=CLRMD&type=Repositories&utf8=✓). Сделать её доступной было хорошей идеей Microsoft.

## Другие инструменты

Стоит упомянуть и другие инструменты:

-  DebugDiag
  - Инструмент DebugDiag создан для устранения таких проблем, как зависания, низкая производительность, утечки памяти или её фрагментация, а также отказы процессов, выполняющихся в пользовательском режиме (теперь с интеграцией CLRMD).
- SOSEX (возможно больше не разрабатывается)
  - ...расширение для отладки управляемого кода, которое снижает моё недовольство SOS.
- VMMap от Sysinternals
  - VMMap – средство для анализа виртуальной и физической памяти процессов.
  - Я применял его, чтобы проанализировать использование памяти в CLR
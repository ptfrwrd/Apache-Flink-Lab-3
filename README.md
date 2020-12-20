# 1 RideCleansingExercise

Задание:  отфильтровать поток данных записей поездок на такси, чтобы сохранить только поездки, которые начинаются и заканчиваются в пределах Нью-Йорка. Полученный поток нужно распечатать до стандартного выхода.
<br>

После получения параметров (данных), нужно установить среду выполнения (env). Используется метод контекста StreamExecutionEnvironment, в котором выполняется потоковая программа, getExecutionEnvironment. Он создает среду выполнения, которая представляет контекст, в котором программа в настоящее время выполняется:

```val env = StreamExecutionEnvironment.getExecutionEnvironment```

Далее устанавливается параллелизм для операций, выполняемых через env-среду.

```env.setParallelism(parallelism) ```

Получаем поток данных поездок на такси:
Откроем DataStream путём добавления источника данных с информацией о настраиваемом типе:

```val rides = env.addSource(rideSourceOrTest(new TaxiRideSource(input, maxDelay, speed)))```

Выберем поездки со стартом и финишем в Нью-Йорке. Используем пакет GeoUtils из репозитория и метод из него isInNYC:

```val filteredRides = rides.filter(r => GeoUtils.isInNYC(r.startLon, r.startLat) && GeoUtils.isInNYC(r.endLon, r.endLat))```

И осуществляем запуск выполнения программы следующей командой, где jobName="Taxi Ride Cleansing":

```env.execute("Taxi Ride Cleansing")```

# 2 ExpiringStateExercise
Задание: добавить в TaxiRides информацию о тарифах.
<br>
В отличие от предыдущего задания, добавляется еще одна настройка среды, устанавливающая временную характеристику для всех потоков. Она нужна для определения системного времени, которое зависит от времени порядка и операций, которые зависят от времени (например, временных окон):

```env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)```

Открываем два DataStream как в задании один, создаем ConnectedStreams путем соединения выходов DataStream (возможных) разных типов друг с другом:

```val processed = rides.connect(fares)```

Теперь, с помощью функции process, применяем данную функцию ProcessFunction к входному потоку, тем самым создавая преобразованный выходной поток:

```val processed = rides.connect(fares).process(new EnrichmentFunction)```
Класс EnrichmentFunction расширяется KeyedCoProcessFunction, который содержит две функции для обработки: processElement1 и processElement2.  

- KeyedCoProcessFunction - использование ProcessFunction для реализации соединения двух потоков данных. методы processElement1 и processElement2 для обработки каждого элемента первого и второго потоков данных соответственно. Типы данных и типы вывода двух потоков данных могут отличаться друг от друга. 
- Функции серии ProcessFunction позволяют получить временную метку водяного знака в потоке данных или перемещаться вперед и назад во времени. Они являются API самого низкого уровня в системе Flink и предоставляют более детализированные разрешения на операции для потока данных. 

Они обрабатывают поездки и тарифы соответственно. Сначала определяются переменные состояния rideState и fareState с помощью RuntimeContext, который содержит информацию о контексте, в котором выполняются функции. Каждый параллельный экземпляр функции будет иметь контекст, через который он может получить доступ к статической контекстной информации (такой как текущий параллелизм) и другим конструкциям, таким как accumulators и broadcast переменные:

```lazy val rideState: ValueState[TaxiRide] = getRuntimeContext.getState( new ValueStateDescriptor[TaxiRide]("saved ride", classOf[TaxiRide]))```

Принцип работы пользовательских функций аналогичен, поэтому рассмотрим только processElement1. Получаем fareState.value, далее проверяется на пустое значение (null). В случае, если оно не пустое, то создаётся пара fare-ride.  Иначе, как только появится watermark (сообщают системе о прогрессе во времени события), можно перестать ждать соответствующего тарифа.
Также в классе EnrichmentFunction реализована функция onTimer, являющаяся функцией обратного вызова:

```override def onTimer(timestamp: Long, ctx: KeyedCoProcessFunction[Long, TaxiRide, TaxiFare, (TaxiRide, TaxiFare)]#OnTimerContext, out: Collector[(TaxiRide, TaxiFare)])```

Основную логику использования таймера можно представить следующим образом:
1.	Регистрация будущей временной метки t с помощью Context в методе processElement. 
2.	В методе onTimer реализуется некоторая логика. В момент t метод onTimer вызывается автоматически.
Из контекста мы можем получить TimerService, который представляет собой интерфейс для доступа к отметкам времени и таймерам. Часть кода, где используются эти функции находится в processElement1 и processElement2.  Регистрируем таймер с помощью Context.timerService.registerEventTimeTimer, нужно только передать метку времени.  Также производится удаление ранее зарегистрированных таймеров через Context.timerService.deleteProcessingTimeTimer и Context.timerService.deleteEventTimeTimer. 

# 3 HourlyTipsExercise
Задание: нужно сначала подсчитать общее количество чаевых, собираемых каждым водителем, час за часом, а затем из этого потока найти максимальное количество чаевых за каждый час.
<br>
В этом задании добавляется работа с окнами. Как и в предыдущем задании устанавливается временная характеристика для всех потоков. Окна разбивают поток на некоторые «корзины» конечного размера, к которым далее применяются какие-то вычисления. Есть окна с ключами и без, в данном задании происходит работа с первыми. Каждое окно имеет функцию, в контексте задания она находится в классе WrapWithWindowInfo:

```class WrapWithWindowInfo() extends ProcessWindowFunction[(Long, Float), (Long, Long, Float), Long, TimeWindow] { override def process(key: Long, context: Context, elements: Iterable[(Long, Float)], out: Collector[(Long, Long, Float)]): Unit = { val sumOfTips = elements.iterator.next()._2  out.collect((context.window.getEnd(), key, sumOfTips)) }```
    
Класс рсширяется ProcessWindowFunction, который получает Iterable (содержит все элементы окна) и объект Context с доступом к информации о времени и состоянии. 
Подсчёт общего количества поездок в час для каждого водителя:

```val hourlyTips = fares.map((f: TaxiFare) => (f.driverId, f.tip)).keyBy(_._1).timeWindow(Time.hours(1)).reduce((f1: (Long, Float), f2: (Long, Float)) => { (f1._1, f1._2 + f2._2) },new WrapWithWindowInfo())```

   

Подсчёт максимального размера чаевых за каждый час для каждого водителя:

```val hourlyMax = hourlyTips.timeWindowAll(Time.hours(1)).maxBy(2)```

# 4 RidesAndFaresExercise
Задание: обогатить TaxiRides информацией о тарифах. 
<br>
Задание аналогично ExpiringStateExercise, разница в классе, который расширяет. Здесь такой класс:

```class EnrichmentFunction extends RichCoFlatMapFunction[TaxiRide, TaxiFare, (TaxiRide, TaxiFare)]```

В отличие от ExpiringStateExercise, где было расширение с помощью KeyedCoProcessFunction, здесь – RichCoFlatMapFunction. В чём их разница?
RichCoFlatMapFunction позволяет объединить два потока параллельно. Задача разбивается на несколько параллельных экземпляров для выполнения, каждый из которых обрабатывает подмножество входных данных задачи. Внутренняя же составляющая функций не изменяется.


---
layout: post
title:  "Scala дорога Offheap"
date:   2015-09-16 09:43:01
categories: programming scala offheap
---

Поскольку **Java** а в последнее время и **Scala** очень широко используются для разработки высоко нагруженных систем, перед разработчиками возникает вопрос, как добиться высокой эффективности при обработке действительно больших наборов данных. **Garbage Collection (Сборка Мусора)** - стандартная парадигма управления памятью для JVM. Благодаря ей мы избавлены от необходимости заниматься такими вещами как аллокация и (де)аллокация памяти для наших объектов. Однако как известно, всему есть цена, а в случае с **GC**  - это проблема с задержками на больших хипах. 

К счастью в **java** сообществе, уже давно найдены пути обхода и один из них это уход из хипа в нативную память, в тех местах приложения, где это необходимо. 

В этой статье, на примере, мы посмотрим эффективность этого подхода в реализациях **Java/Scala**, а также узнаем, что такого может **Scala** чего **Java** не может.     

В качестве отправной точки, мы будем использовать пример про 50 000 000  биржевых сделок из доклада Никиты Сальникова-Тарновского, как раз по этой тематике.

Немного терминов:


* **Heap** - хип, область памяти внутри JVM, куда попадают все наши объекты и за которыми беспристрастно следит GC. 

* **GC (Garbage Collector)**  - сборщик мусора, прибирает за нами наш хип. 

* **OffHeap** - область памяти за пределами хипа и не подвластная GC.

Все, теперь мы можем переходить непосредственно к нашему примеру. 

Представим ситуацию на бирже, нам нужно создавать сделки, а затем их обрабатывать и получать некий результат. Кол-во этих сделок велико - предположим 50 000 0000 сделок в день.
Для этого нам понадобится некий класс представляющий собой сделку:

```scala
case class Trade(ticket: Int = 0, amount: Int = 0, price: Int = 0, buy: Boolean = false)
```
 
и интерфейс определяющий операции которые мы бы хотели совершать:

```scala
trait StockExchange { 
    def order(ticket: Int, amount: Int, price: Int, buy: Boolean): Unit          
    def dayBalance: Double 
} 
```

Интерфейс весьма прост, метод `order` создает сделку, а `dayBalance` возвращает нам дневной результат.

Настало время испытаний, напомню, что количество совершаемых сделок для наших испытаний равно 50 000 000.

## Испытание 1. Коллекция объектов.

Первое, что приходит на ум, при такой постановке, а буду ка я складывать наши сделки в некую коллекцию, а затем в цикле пробегусь по ним и посчитаю баланс.
При таком подходе наш код может выглядеть вот так:

```scala
class ListBufferStockExchange extends StockExchange {
    val orders = ListBuffer[Trade]()

    override def order(ticket: Int, amount: Int, price: Int, buy: Boolean): Unit = orders += new Trade(ticket, amount, price, buy)

    override def dayBalance: Double = {
        @tailrec
        def calculate(acc: Double, orders: ListBuffer[Trade]): Double =
        {
            if(orders.isEmpty) acc
            else calculate(acc + orders.head.amount * orders.head.price * (if(orders.head.buy)  1 else -1), orders.tail)
        }
        calculate(0.0, orders)
    }
}
```

Тестирование у нас будет по фен Шую и для этого, мы воспользуемся **JMH (Java Microbenchmark Harness)**. Для **sbt** существует плагин, позволяющий запускать бенч марки непосредственно из под нее.
Пример для нашего первого испытания:

```scala
@Benchmark
@BenchmarkMode(Array(Mode.AverageTime))
@OutputTimeUnit(TimeUnit.MILLISECONDS)
def listBufferTrade: Double = {
    val exchange: StockExchange = new ListBufferStockExchange()
    var i: Int = 0
    while (i < StockExchange.TRADES_PER_DAY) {
        exchange.order(i, i, i, (i & 1) == 0)
        i += 1
    }
    exchange.dayBalance
}
```
Здесь я остановлюсь подробнее, а дальше буду показывать только результаты, т.к. код самого бенч марка отличается только в части создаваемого экземпляра класса для тестов. 
С помощью трех аннотаций `@Benchmark, @BenchmarkMode(Array(Mode.AverageTime)), @OutputTimeUnit(TimeUnit.MILLISECONDS)` мы определяем, что это бенч марк, что нас интересует среднее время выполнения, и что единицами измерения будут милисекунды. 
Далее непосредственно сам метод, который будет создавать экземпляр тестируемого класса, совершать 50 000 000 сделок и запрашивать баланс. Время выполнения этого процесса мы и будем мерять.

Запускаем бенч марки ... к сожалению, я не дождался результатов выполнения для данной реализации, получив `java.lang.OutOfMemoryError: GC overhead limit exceeded`. Также хочу отметить, что использование других коллекций, как специфичных для scala, так и нет, не привело к улучшению результата. Все тот-же `GC overhead limit exceeded`. 

Что касается кода написанного на *Java*, а я напомню, что мы для каждого примера тестируем две версии, то версия с использованием ArrayList показывает средний результат в 6674.566 мс. 

Подводя итоги для первого испытания, *Scala* версия отказывается работать, java версия работает, но долго, результат более 6,5 секунд. Отсюда становится ясно, что работать с большим кол-вом объектов не эффективно, приложение простаивает ощутимое кол-во времени, ожидая пока GC сделает свою работу.

## Испытание 2. Массив примитивов. 

Раз уж с объектами работать так медленно, попробуем от них отказаться и остановим свой выбор на массиве примитивов. Поскольку в JVM они хранятся вне хипа, то мы должны получить ощутимый прирост производительности.

```
class ArrayIntStockExchange extends StockExchange{

    private var recordCount = 0

    private val flyweight = ArrayTrade

    private val orders = Array.fill(StockExchange.TRADES_PER_DAY * flyweight.getObjectSize)(0)

    override def order(ticket: Int, amount: Int, price: Int, buy: Boolean) =
    {
        val trade  = get(recordCount)
        trade.setTicket(ticket)
        trade.setAmount(amount)
        trade.setPrice(price)
        trade.setBuy(buy)
        recordCount += 1
    }

    def dayBalance: Double = {
        var balance: Double = 0
        var i: Int = 0
        while (i < recordCount) {
            val trade = get(i)
            balance += trade.getAmount * trade.getPrice * (if (trade.isBuy) 1 else -1)
            i += 1
        }
        balance
    }

    private def get(index: Int) = {
        val offset: Int = index * flyweight.getObjectSize
        flyweight.setObjectOffset(offset)
        flyweight
    }

    private object ArrayTrade {
        private var offset: Int = 0

        private final val ticketOffset: Int = offset

        private final val amountOffset: Int = {
            offset += 1
            offset
        }
        private final val priceOffset: Int = {
            offset += 1
            offset
        }
        private final val buyOffset: Int = {
            offset += 1
            offset
        }
        private final val objectSize: Int = {
            offset += 1
            offset
        }
        private var objectOffset: Int = 0

        def getObjectSize: Int = objectSize

        def setObjectOffset(objOffset: Int) = objectOffset = objOffset

        def setTicket(ticket: Int) = orders(objectOffset + ticketOffset) = ticket

        def getPrice = orders(objectOffset + priceOffset)

        def setPrice(price: Int) = orders(objectOffset + priceOffset) = price

        def getAmount = orders(objectOffset + amountOffset)
        
        def setAmount(quantity: Int) = orders(objectOffset + amountOffset) = quantity

        def isBuy: Boolean = orders(objectOffset + buyOffset) == 1

        def setBuy(buy: Boolean) = orders(objectOffset + buyOffset) = (if (buy) 1 else 0)
    }
}
```

Результаты выполнения теста для этого класса впринципе одинаковые для *Scala/Java* версий - *700.837 мс Scala VS 720.825 мс Java*, однако мы видим рост производительности практически на порядок и это действительно круто. Правда кода стало гораздо больше и он стал сложнее. Но не будем останавливаться и пойдем дальше, посмотрим можем ли это работать еще быстрее.

## Испытание 3. Байт буфер.

В **Java** существует два типа `ByteBuffer` - это *direct* и просто *ByteBuffer*, но нас на самом деле нас больше интересует *direct*, т.к. именно он живет вне хипа.

```scala
class ByteBufferStockExchange(direct: Boolean) extends StockExchange {

    private var recordsCount: Int = 0
    private val flyweight = BufferTrade

    private val buffer: ByteBuffer = {
        if (direct) ByteBuffer.allocateDirect(StockExchange.TRADES_PER_DAY * flyweight.getObjectSize)
        else ByteBuffer.allocate(StockExchange.TRADES_PER_DAY * flyweight.getObjectSize)
    }

    buffer.order(ByteOrder.nativeOrder())

    def order(ticket: Int, amount: Int, price: Int, buy: Boolean) {
        val trade  = get(recordsCount)
        trade.setTicket(ticket, buffer)
        trade.setAmount(amount, buffer)
        trade.setPrice(price, buffer)
        trade.setBuy(buy, buffer)
        recordsCount += 1
    }

    def dayBalance: Double = {
        var balance: Double = 0
        var i: Int = 0
        while (i < recordsCount) {
            val trade = get(i)
            balance += trade.getAmount(buffer) * trade.getPrice(buffer) * (if (trade.isBuy(buffer)) 1 else -1)
            i += 1
        }
        balance
    }

    private def get(index: Int) = {
        val offset: Int = index * flyweight.getObjectSize
        flyweight.setObjectOffset(offset)
        flyweight
    }

    def destroy = buffer.clear
}

object BufferTrade {
    private var offset: Int = 0
    private final val ticketOffset: Int = {
        offset + 0
    }
    private final val amountOffset: Int = {
        offset += 4
        offset
    }
    private final val priceOffset: Int = {
        offset += 4
        offset
    }
    private final val buyOffset: Int = {
        offset += 4
        offset
    }
    private final val objectSize: Int = {
        offset += 1
        offset
    }
    private var objectOffset: Int = 0

    def getObjectSize: Int = {
        objectSize
    }

    def setObjectOffset(objOffset: Int) = {
        objectOffset = objOffset
    }

    def setTicket(ticket: Int, buffer: ByteBuffer) = {
        buffer.putInt(objectOffset + ticketOffset, ticket)
    }

    def getPrice(buffer: ByteBuffer): Int = {
        buffer.getInt(objectOffset + priceOffset)
    }

    def setPrice(price: Int, buffer: ByteBuffer) = {
        buffer.putInt(objectOffset + priceOffset, price)
    }

    def getAmount(buffer: ByteBuffer): Int = {
        buffer.getInt(objectOffset + amountOffset)
    }

    def setAmount(quantity: Int, buffer: ByteBuffer) = {
        buffer.putInt(objectOffset + amountOffset, quantity)
    }

    def isBuy(buffer: ByteBuffer): Boolean = {
        buffer.get(objectOffset + buyOffset) == 1
    }

    def setBuy(buy: Boolean, buffer: ByteBuffer) = {
        buffer.put(objectOffset + buyOffset, (if (buy) 1 else 0))
    }
}
```
Код в принципе идентичен тому, что мы писали для массива примитивов, разница лишь в аллокации буфера, расчете офсетов и финальным его уничтожением (освобождением памяти).

По результатам *Scala/Java* версии работают одинаково *645.134 мс Scala VS 633.291 мс Java*. Однако, как мы видим *ByteBuffer direct* все-таки чуточку быстрее нежели *ArrayInt*. Не прямой ByteBuffer работает практически в два раза медленнее *1519.910 мс Scala VS 1250.257 мс Java*.

## Испытание 4. sun.misc.Unsafe.

В наших предыдущих примерах, как на массив примитивов, так и на байт буфер, мы хоть и попадали вне хипа, но управлением памятью как-таковым напрямую - не занимались. Ни в *Java* ни в *Scala* нет легального способа в ручную управлять памятью, однако есть класс Unsafe, предназначенный исключительно для внутреннего использования и название которого говорит само за себя. Именно он позволяет работать с памятью вне хипа, правда при этом и накладывает соответствующую ответственность. В последнее время в *Java* среде было много дискуссий относительно этого класса и его использования, особенно на фоне заявлений из Oracle о том, что они собираются исключить его из поставки *9 JDK*, но не смотря на это большинство *Java* приложений, где нужна высокая производительность на больших объемах данных, так или иначе его используют.

```scala
class UnsafeStockExchange extends StockExchange {
    import UnsafeTrade._

    def order(ticket: Int, amount: Int, price: Int, buy: Boolean) =  synchronized {
        recordsCount += 1
        val trade = get(recordsCount)
        trade.setTicket(ticket)
        trade.setAmount(amount)
        trade.setPrice(price)
        trade.setBuy(buy)
    }


    def dayBalance() = synchronized {
        var balance = 0
        var i = 0
        while(i < recordsCount){
            val trade = get(i)
            balance += trade.getAmount() * trade.getPrice() * (if(trade.isBuy) 1 else -1)
            i += 1
        }
        balance
    }

    private val address = unsafe.allocateMemory(StockExchange.TRADES_PER_DAY * UnsafeTrade.getObjectSize())
    private val flyweight = UnsafeTrade

    private def get(index: Int) = {
        val offset: Long = address + (index * UnsafeTrade.getObjectSize())
        flyweight.setObjectOffset(offset)
        flyweight
    }

    def destroy() = unsafe.freeMemory(address)

    private var recordsCount = 0
}

object UnsafeTrade {
    val unsafe = SunMisc.UNSAFE

    private var objectOffset = 0L

    private var offset = 0L

    private val ticketOffset = {offset += 0L; offset}

    private val amountOffset = {offset += 4L; offset}

    private val priceOffset = {offset += 4L; offset}

    private val buyOffset = {offset += 4L; offset}

    private val objectSize = {offset += 1L; offset}

    def getObjectSize() = objectSize

    def setObjectOffset(offset: Long) = UnsafeTrade.objectOffset = offset

    def setTicket(ticket: Int) = unsafe.putInt(UnsafeTrade.objectOffset + UnsafeTrade.ticketOffset, ticket)

    def getPrice() = unsafe.getInt(UnsafeTrade.objectOffset + UnsafeTrade.priceOffset)

    def setPrice(price: Int) = unsafe.putInt(UnsafeTrade.objectOffset + UnsafeTrade.priceOffset, price)

    def getAmount() = unsafe.getInt(UnsafeTrade.objectOffset + UnsafeTrade.amountOffset)

    def setAmount(quantity: Int)  = unsafe.putInt(UnsafeTrade.objectOffset + UnsafeTrade.amountOffset, quantity)

    def isBuy() = unsafe.getByte(UnsafeTrade.objectOffset + UnsafeTrade.buyOffset) == 1

    def setBuy(buy: Boolean) = unsafe.putByte(UnsafeTrade.objectOffset + UnsafeTrade.buyOffset, (if(buy) 1.toByte else 0.toByte))
}
```

Результаты *293.96 мс - Scala VS 300.945 мс - Java*. Как мы видим *Unsafe* быстрее *ByteBuffer* более чем в два раза и является бесспорным лидером в гонке за быстродействие.

Сводная таблица результатов всех испытаний.

| Вид испытания | Средний результат мс (Scala) | Средний результат мс (Java)  |
| ----------------------------|---------------------------------------|--------------|
|   ListBufferStockExchange   |   GC limit overhead exception   |   6674.566   |
|   ArrayInt   |   700.837   |   720.85   |
|   ByteBuffer(direct)   |   645.134   |   633.291   |
|   Unsafe   |   293.96   |   300.945   |

Как мы видим из таблицы, разницы в результатах между *Java* и *Scala* версиями кода практически нет. Смело можно писать на обоих языках. 

Но есть одно но, писать такой код не удобно, ошибиться просто, а в случае с *Unsafe*, еще и не безопасно. Так что же такого может предложить *Scala*, чего нет в *Java*. Ответ - макросы. Чем и воспользовался Денис Шабалин из EPFL, создав проект *scala-offheap*.

Благодаря ему, с помощью простых аннотаций, мы можем размещать наши объекты вне хипа и *безопасно* работать с ними. Пример кода для нашего случая:

```scala

@data class OffHeapTrade(ticket: Int, amount: Int, price: Int, buy: Boolean)
{
    def balance = amount * price * (if(buy) 1 else -1)
}

class OffHeapScalaStockExchange extends StockExchange {

    implicit val alloc = malloc

    private var pointer = 0

    val orders = EmbedArray.uninit[OffHeapTrade](StockExchange.TRADES_PER_DAY)

    override def order(ticket: Int, amount: Int, price: Int, buy: Boolean): Unit = {
        orders(pointer) = OffHeapTrade(ticket, amount, price, buy)
        pointer += 1
    }

    override def dayBalance: Double = {
        var result = 0.0
        var i = 0
        while(i < orders.size){
            val order = orders(i)
            result += order.balance
            i += 1
        }
        alloc.free(orders.addr)
        result
    }
}
```

Все что здесь есть необычного, это аннотация @data у класса, implicit аллокатор памяти malloc ну и специальная версия массива EmbedArray. Все, мы вне хипа.

Средний результат выполнения нашего теста **351.633 мс**! Никакого ручного выделения памяти, никаких *flyweight* паттернов, код выглядит также, будто бы мы работаем с обычной коллекцией объектов в хипе. 

Впечатляет, неправда ли.
---
layout: page
title: Scala中的Actor和Akka中的Actor
comments: true
---

---

本文旨在介绍Scala中的Actor。在Scala-2.11中中，scala.actors包已经被废弃，被替代成akka.actor。

scala.actors包中的Actor在本文第1部分（Actor和并发）介绍，内容选译自[《Programming in Scala》第30章](http://www.artima.com/pins1ed/actors-and-concurrency.html)

akka.actor包中的Actor在本文第2部分（scala.actors到akka.actor的迁移）介绍，内容选译自[迁移指南](http://docs.scala-lang.org/overviews/core/actors-migration-guide.html)和[akka的文档](http://doc.akka.io/docs/akka/snapshot/scala/actors.html)。

---

## 1.Actor和并发

有时候在设计一个程序时，指定事物独立并行地发生是有用的。Java包括了对并发的支持，虽然这种支持是足够的，但随着程序变得更大更复杂，保证并发的正确性就变得十分困难。scala通过添加actor扩充了Java对并发的原生支持。actor提供了一个并发模型，它更易用，并且可以帮助你避免很多在使用Java原生并发模型时的困难。这一章会为你展示如何使用scala的actor库，并提供一个延伸的示例：将单线程电路模拟代码转换成多线程版本。

### 1.1.并行的问题

Java平台的内置线程模型以共享数据和锁为基础。每个对象与一个逻辑上的监控器相关联，这个监控器被用于控制对数据的多线程访问。为了使用这个模型，你要决定什么数据会被多个线程共享，并将访问到共享数据，或者控制对共享数据的访问的代码用synchronized作标识。Java运行时使用一个锁机制来保证一次只有一个线程进入一个被同步的代码段，代码段中的内容都被同一个锁守护，从而使你可以精心安排对共享数据的多线程访问。

不幸的是，程序员发现使用共享数据和锁模型来可靠地创建健壮的多线程应用时很困难的，尤其当应用变得更大更复杂时。问题在于在程序中的每个点上，你必须推出你所修改或访问的数据有哪些可能会被其他线程修改或访问，以及有什么锁是被持有的。对于每个方法调用，你必须推出它将试图获取哪些锁，并使你自己信服在获取这些锁后不会产生死锁。将问题组合起来，你推出来的锁在编译时是不固定的，因为程序在运行时是自由创建新锁的。

使事情变得更糟糕的是，对多线程代码的测试是不可靠的。因为线程是不确定的，你可能成功测试了一个程序一千次，但它仍然在客户的机器上第一次运行就出错。使用共享数据和锁，你必须只通过推论来使程序变得正确。

此外，你也不能通过“过同步”来解决问题。它只是同步所有东西，这跟什么都没同步一样都是有问题的。问题在于新的锁操作除去了竞争条件的可能性，但同时增加了死锁的可能性。一个正确使用锁的程序必须同时没有竞争条件和死锁，所以你不能在任一方面做得太过分都不能保证程序是安全的。

Java 5引入了java.util.concurrent，一个并发构件库，它为并发编程提供了高层次的抽象。相对于使用Java低层次的同步机制来进行抽象，使用并发构件使得多线程编程更少发生错误。然而，并发构件同样基于共享数据和锁，因此没有解决使用该模型时的基本难点。

scala的actor库通过提供一个可替换的、无共享的、程序员更容易进行推论的消息传递模型来处理那些基本问题。actor在设计并发软件时会是第一选择，因为它们能帮你避免死锁和竞争条件，这两者是你使用共享数据和锁模型时容易陷入的问题。

### 1.2.actor和消息传递

一个actor是一个类似线程的实体，它拥有一个消息盒来接收消息。要实现一个actor，你创建scala.actors.Actor的子类并实现act方法。下面是一个例子，这个actor没有对它的消息盒做什么。它只是将一个消息打印五遍然后退出。

```scala
import scala.actors._

object SillyActor extends Actor {
  def act() { 
    for (i <- 1 to 5) {
      println("I'm acting!")
      Thread.sleep(1000)
    }
  }
}
```

你通过调用start方法来启动一个actor，与启动一个Java线程类似：

```shell
scala> SillyActor.start()
I'm acting!
res4: scala.actors.Actor = SillyActor$@1945696

scala> I'm acting!
I'm acting!
I'm acting!
I'm acting!
```

注意“I'm acting!”输出是跟scala shell的输出交错的。这是因为SillyActor对象与运行shell的线程是独立运行的。Actor对象也彼此独立。例如，下面这第二个actor：

```scala
import scala.actors._

object SeriousActor extends Actor {
  def act() { 
    for (i <- 1 to 5) {
      println("To be or not to be.")
      Thread.sleep(1000)
    }
  }
}
```

你可以同一时刻运行两个actor，如：
```shell
scala> SillyActor.start(); SeriousActor.start()
res3: scala.actors.Actor = seriousActor$@1689405

scala> To be or not to be.
I'm acting!
To be or not to be.
I'm acting!
To be or not to be.
I'm acting!
To be or not to be.
I'm acting!
To be or not to be.
I'm acting!
```

你也可以使用单例对象scala.actors.Actor中的actor方法来创建一个actor：

```shell
scala> import scala.actors.Actor._

scala> val seriousActor2 = actor {
for (i <- 1 to 5)
println("That is the question.")
Thread.sleep(1000)
}

scala> That is the question.
That is the question.
That is the question.
That is the question.
That is the question.
```

上面的val定义创建了一个actor，它会执行在actor方法体中定义的动作。这个actor会在它被定义的时后就立刻启动。因此没有必要单独调用start方法。

你可以创建多个actor，它们会独立运行。它们如何一起工作呢？它们如何不使用共享内存和锁来通信呢？Actor通过给彼此发送消息来通信。你通过!方法来发送消息，如：

```shell
scala> SillyActor ! "hi there"
```

这个例子中没有任何事情发生，因为SillyActor太忙了，无法处理它的消息，于是“hi there”消息就保留在它的消息盒中处于未读状态。下面的代码展示了一个新的更友好的actor，它等待消息盒收到的每条消息并将收到的打印出来。它通过调用receive方法、并将一个partial函数传给receive方法来接收一条消息。

```scala
val echoActor = actor {
  while (true) {
    receive {
      case msg =>
        println("received message: "+ msg)
    }
  }
}
```

当一个actor发送一条消息，它没有锁住，当它收到一条消息，它也没有中断。被发送的消息在接收者actor的消息盒里等待直到那个actor调用了receive。你可以从下面的输出看到这个行为：

```shell
scala> echoActor ! "hi there"
received message: hi there

scala> echoActor ! 15

scala> received message: 15
```

正如[原文第15.7节](http://www.artima.com/pins1ed/case-classes-and-pattern-matching.html#sec:partial-functions)所述，一个partial函数不是一个完整函数，也就是说，它可能不会定义所有输入值。除了一个接收一个参数的apply方法之外，一个partial函数还提供了一个isDefinedAt方法，它也接收一个参数。如果partial函数可以处理传进来的值，那么isDefinedAt方法将返回true。这样的值是可以安全地传给apply方法的。如果你传给apply方法的值让isDefinedAt方法返回false，那么apply将会抛出一个异常。

一个actor会处理的消息必须符合传给receive的partial函数中任一case条件。对于消息盒中的每一条消息，receive将会先调用传进去的partial函数中的isDefinedAt方法来确定函数中是否有一个case可以匹配并处理消息。receive方法将选择消息盒中第一条isDefinedAt方法返回true的消息，将其传给partial函数的apply方法。然后apply方法会处理消息。例如，echoActor的apply方法将会打印接收的消息。如果消息盒包含的消息对于isDefinedAt都返回false，那么actor会锁住直至下一个符合条件的消息到达。

例如，下面是一个只处理Int类型的actor：

```shell
scala> val intActor = actor {
receive {
case x: Int => // I only want Ints
println("Got an Int: "+ x)
}
}
intActor: scala.actors.Actor = 
scala.actors.Actor$$anon$1@34ba6b
```

如果你发送一个String或者Double，intActor将会忽略这条消息：

```shell
scala> intActor ! "hello"
scala> intActor ! Math.Pi
```

但如果发送的是Int，你就会得到一个应答：

```shell
scala> intActor ! 12
Got an Int: 12
```

### 1.3.将本地线程视为actor

actor子系统为自己管理一个或多个本地线程。只要你使用你定义的一个显式的actor，你就不需要考虑太多它们是怎么映射到线程的。

子系统还支持另一个方面：每个本地线程也可以当作一个actor来使用。然而，你不能直接使用Thread.current，因为它没有必需的方法。相反，如果你想将当前线程看作一个actor，你应该使用Actor.self。

这个功能对于从交互式shell上测试actor是尤其有用的。下面是一个例子：

```shell
scala> import scala.actors.Actor._
import scala.actors.Actor._

scala> self ! "hello"

scala> self.receive { case x => x }
res6: Any = hello
```

receive方法的返回值是传递给方法的partial函数的计算结果。在这个例子中，partial函数返回消息本身，因此接收的消息最终由交互式shell打印出来。

如果你使用这个技术，使用receive的一个变种receiveWithin是更好的。你可以指定一个超时阈值（单位是毫秒）。如果你在交互式shell中使用receive，那么receive将会锁住shell直至下一个消息到达。在self.receive的例子中，这意味着永远等下去！相反，使用receiveWithin并设置某个超时阈值：

```shell
scala> self.receiveWithin(1000) { case x => x } // wait a sec!
res7: Any = TIMEOUT
```

### 1.4.回复消息
在receive方法体中，处理消息部分可以使用`reply(msg)`方法来回复消息给消息发送方。

## 2.scala.actors到akka.actor的迁移

### 2.1.初始化
在Akka中，actor只能通过接口ActorRef来访问。ActorRef的实例可以从两个地方获取：一个是ActorDSL单例对象的actor方法；另一个是一个ActorRefFactory实例的actorOf方法。

下面是构造器调用初始化的转换。

```scala
val myActor = new MyActor(arg1, arg2)
myActor.start()
```

应该被替换为：

```scala
ActorDSL.actor(new MyActor(arg1, arg2))
```

或者使用ActorSystem：

```scala
import akka.actor.ActorSystem

// ActorSystem is a heavy object: create only one per application
val system = ActorSystem("mySystem")
val myActor = system.actorOf(Props[MyActor], "myactor2")
```

对于ActorSystem创建actor的方式，有几点需要说明。

- Props
 
其中，Props是一个配置类，指定了创建actor的选项。可以将它考虑为一个不可变的、可共享的“食谱”，它说明了如何创建一个actor，并包含了相关的部署信息（例如，要使用哪一个调度器）。下面是创建一个Props实例的一些例子。

```scala
import akka.actor.Props

val props1 = Props[MyActor]
val props2 = Props(new ActorWithArgs("arg")) // careful, see below
val props3 = Props(classOf[ActorWithArgs], "arg")
```

第二个变量展示了如何传递构造器参数给已创建的Actor，但是它应该只被用于actor的外面，如下面所述。

最后一行展示了无视上下文也可以传递构造器参数的一种可能。Props对象在初始化期间，会验证一个匹配的构造器是否存在，如果没有或者有多个匹配的构造器被找到。

```scala
// NOT RECOMMENDED within another actor:
// encourages to close over enclosing class
val props7 = Props(new MyActor)
```

不推荐在另一个actor中用这个方法，因为它encourages to close over the enclosing scope，这会引起不可序列化的Props和可能的竞争条件（破坏了actor的封装）。

- actorOf方法

这个方法返回一个ActorRef实例。这是一个对actor实例的操作接口，也是与actor交互的唯一途径。ActorRef是不可变的，它与它所代表的Actor是一一对应的。ActorRef也是可以序列化的，并且是网络感知的。这意味着你可以将其序列化并在网络上发送，在一台远程主机上使用它，并且在网络中，它将仍然代表着在源节点上的同一个Actor。

方法的第二个参数是可选的，但是你应该倾向于给你的actor命名，因为这是在写日志消息时会使用到的，可以标识actor。名字必须不能为空或者以`$`开头，但是它可以包括URL编码字符（例如一个空格的编码字符为%20）。actor之间是有一种继承关系的，在一个actor对象中创建的另一个actor称为这个actor的子actor。同一个actor的不同子actor不能使用同一个名称标识，否则会抛出异常`InvalidActorNameException`。Actor在被创建时就会自动启动。

### 2.2.Actor的转化

scala中的Actor类与AMK(Actor Migration Kit，里面包含了scala的Actor的一个扩展)中的ActWithStash类几乎相同，后者多提供了一些方法对应akka的Actor中的方法。因此，代码需要修改如下：

```scala
class MyActor extends Actor -> class MyActor extends ActWithStash
```

另外，在ActWithStash中的receive方法不能被用于act的方法体中。因此，需要将所有receive调用添加上类型参数：

```scala
receive { case x: Int => "Number" } ->
receive[String] { case x: Int => "Number" }
```

另外，用户必须在act方法前添加上override关键字，并创建出空的receive方法。act方法需要被覆盖是因为它在ActWithStash中的实现模仿了Akka的消息处理循环。修改后代码如下：

```scala
class MyActor extends ActWithStash {
  // dummy receive method (not used for now)
  def receive = {case _ => }
  override def act() {
  // old code with methods receive changed to react.
  }
}
```

远程actor无法使用ActWithStash out of the box。方法`register('name, this)`要替换为`registerActorRef('name, self)`。


### 2.3.移除act()方法

这里讲述如何从ActWithStash中移除act方法，以及如何修改ActWithStash中的方法以使其像Akka里一样。建议一次只修改一个actor。在scala中，一个actor的行为通过实现act方法来定义。逻辑上，一个actor是一个并发进程，它执行了act方法的方法体，然后结束。在Akka中，actor的行为通过使用一个全局消息处理器来定义，它一条接一条地处理actor的消息盒中的消息。消息处理器是一个partial函数，由receive方法返回，并应用于每一条消息。

因为在ActWithStash中的Akka方法的行为取决于现有act方法是否被移除，所以我们必须先移除掉act方法。然后再给出从scala.actors.Actor的方法转换到Akka的规则。

- 规则一

```scala
def act() {
  // initialization code here
  loop {
    react { ... }
  }
}
```

要替换为：

```scala
override def preStart() {
  // initialization code here
}
def act() {
  loop {
    react{ ... }
  }
}
```

- 规则二

```scala
def act() = {
  loop {
    react {
    // body
    }
  }
}
```

要替换为：

```scala
def receive = {
  // body
}
```

接着是其他方法的替换（只有部分）：

```shell
exit()/exit(reason) -> context.stop(self)

receiver -> self

reply(msg) -> sender ! msg
```

### 2.4.迁移到Akka

现在用户可以操作Akka的actor了。需要把库改为Akka的库。将scala-actors.jar和scala-actors-migration.jar移除，并加入akka-actor.jar和typesafe-config.jar。然后修改import代码：

```shell
scala.actors._ -> akka.actor._
scala.actors.migration.ActWithStash -> akka.actor.ActorDSL._
scala.actors.migration.pattern.ask -> akka.pattern.ask
scala.actors.migration.Timeout -> akka.util.Timeout
```

然后ActWithStash中定义的receive需要添加override关键字。另外，scala的actor中的stash(x)需要一个参数，在Akka中，不需要该参数，stash(x) -> stash()。


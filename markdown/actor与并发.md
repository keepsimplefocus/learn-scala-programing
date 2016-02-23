Actor 是 Scala 基于消息传递的并发模型，虽然自 Scala-2.10 其默认并发模型的地位已被 Akka 取代，但这种与传统 Java、C++完全不一样的并发模型依旧值得学习。

##如何使用 Actor
###扩展 Actor
先来看看第一种用法，下面是一个简单例子及部分说明

```
//< 扩展超类 Actor
class ActorItem extends Actor {
  //< 重载 act 方法
  def act(): Unit = {
    //< receive 从消息队列 mailbox 中去一条消息并处理
    receive { case msg => println(msg) }
  }
}

object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = new ActorItem
    //< 启动
    actorItem.start()

    //< 向 item 发送消息
    actorItem ! "actor test1"
    actorItem ! "actor test2"
  }
}
```

```
输出：
actor test1
```

这种用法在实际中并不常用，需要：

1. 扩展超类 Actor
2. 重载 act 方法
3. 调用扩展类对象 start 方法

###使用 scala.actors.Actor.actor 方法
第二种方式是实际中常用并且是 Scala 社区推荐的，例子如下：

```
object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = actor {
      receive { case msg => println(msg) }
    }

    //< 向 item 发送消息
    actorItem ! "actor test1"
    actorItem ! "actor test2"
  }
}
```

```
输出：
actor test1
```

这里需要特别注意的是，actor 其实是scala.actors.Actor的 actor 方法，并不是 scala 语言内建的。
这种使用方法更加方便，与第一种扩展超类 Actor 有以下几点不同：

1. 使用 Actor.actor 方法（返回类型为Actor）而不是扩展 Actor 并重载 act 方法
2. 构造完成即启动，不需要调用 start方法（当然你调用了也不会有什么问题）

###使用 react

除了可以使用 receive 从消息队列 mailbox 中取出消息并处理，react 同样可以。receive 和 react 的区别与联系将在下文中说明。先来看看怎么用，其实只要把上面两段代码的 receive 替换成 react 即可：

```
class ActorItem extends Actor {
  def act(): Unit = {
    react { case msg => println(msg) }
  }
}

object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = new ActorItem
    actorItem.start()

    actorItem ! "actor test1"
    actorItem ! "actor test2"
  }
}
```

```
输出：
actor test1
```

---

```
object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = actor {
      react { case msg => println(msg) }
    }

    actorItem ! "actor test1"
    actorItem ! "actor test2"
  }
}
```

```
输出：
actor test1
```

###持续处理消息
如果你仔细观察，就会发现上面的每个示例中，都向 actor 发送了"actor test1"和"actor test2"两条消息，但最终只打印了"actor test1"这一条消息。这是因为，不管是 receive 还是 react，都只从 mailbox 中取一条消息进行处理，处理完之后不会再取一条处理。如果想要持续从 maibox 中取消息并处理，也有两种方式。

**方式一**：使用 loop。适用于扩展 Actor 和 actor 方法两种方式

```
class ActorItem extends Actor {
  def act(): Unit = {
    loop {
      react { case msg => println(msg) }
    }
  }
}

object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = new ActorItem
    actorItem.start()

    actorItem ! "actor test1"
    actorItem ! "actor test2"
  }
}
```

```
输出：
actor test1
actor test2
```

**方式二**：在 receive 处理中调用receive；在 react 处理中调用 react。仅适用于 actor 方法这种方法

```
class ActorItem extends Actor {
  def act(): Unit = {
    react {
      case msg => {
        println(msg)
        act()
      }
    }
  }
}

object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = new ActorItem
    actorItem.start()

    actorItem ! "actor test1"
    actorItem ! "actor test2"
  }
}
```

##Actor是如何工作的

每个actor对象都有一个 mailbox，可以简单的认为是一个队列，用来存放发送给这个actor的消息。
当 actor 发送消息时，它并不会阻塞，而当 actor 接收消息时，它也不会被打断。发送的消息在接收 actor 的 mailbox 中等待处理，直到 actor 调用 receive 方法。

receive 具体是怎么工作的呢？来看看它的源码：

```
def receive[R](f: PartialFunction[Any, R]): R = {
  var done = false
  while (!done) {
    //< 从 mailbox 中取出一条消息
    val qel = mailbox.extractFirst((m: Any, replyTo: OutputChannel[Any]) => {
      senders = replyTo :: senders
      //< 与偏函数进行匹配，匹配失败返回 null
      val matches = f.isDefinedAt(m)
      senders = senders.tail
      matches
    })
    if (null eq qel) {
      //< 如果当前mailbox里面没有可以处理的消息，调用suspendActor，该方法会调用wait
      waitingFor = f.isDefinedAt  
      isSuspended = true 
      suspendActor()  
    } else {
      //< 执行到这里就说明成功从 mailbox 中获得匹配的消息
      received = Some(qel.msg)
      senders = qel.session :: senders
      done = true
    }
  }

  //< 成功获得消息后，调用 f.apply 来执行对应的操作
  val result = f(received.get)
  received = None
  senders = senders.tail
  result
}
```

一图胜千言，下图为 receive 模型工作流程

![](https://github.com/keepsimplefocus/learn-scala-programing/blob/master/pic/actor_receive.jpg)


###与线程的关系
Actor 的线程模型可以这样理解：在一个进程中，所有的 actor 共享一个线程池，总的线程个数可以配置，也可以根据 CPU 个数决定。

当一个 actor 启动后，Scala 分配一个线程给它使用，如果使用 receive 模型，这个线程就一直为该 Actor 所有。
如果使用 react 模型，react 找到并处理消息后并不返回，它的返回类型为 Nothing，Scala 执行完 react 方法后，抛出异常，调用 act 也就是间接调用 react 的线程会捕获这个异常，忘掉这个 actor，该线程就可以被其他actor 使用。
所以，如果能用 react 就尽量使用 react，可以节省线程。

##良好的 Actor 风格
###只通过消息与 actor 通信
举个例子，一个 GoodActor可能会在发往 BadActor 的消息中包含一个指向自己的引用，来表明作为消息源的自己。如果 BadActor 调用了 GoodActor 的某个任意的方法而不是通过 "!" 发送消息的话，问题就来了。被调用的方法可能读到 GoodActor 的私有实例数据，而这些数据可能是由另一个线程写进去。结果是，你需要确保 BadActor 线程对这些实例数据的读取和 GoodActor 线程对这些数据的写入是同步在一个锁上的。一旦绕开了 actor 之间的消息传递机制，就回到了共享数据和锁模型中。

###优选不可变的消息
由于 Scala 的 actor 模型提供了在每个 actor 的 act 方法中的单线程环境，不需要担心在这个方法的实现中使用的对象是否是线程安全的。

确保消息对象是线程安全的最佳途径是在消息中使用不可变对象。任何只有 val 字段且这些字段只引用到不可变对象的类的实例都是不可变的。

如果你发现自己有一个可变的对象，想继续使用它，同时也想用消息发送给另一个 actor，此时应该考虑制作并发送它的一个副本，比如利用 clone 方法。

###让消息自包含
向某个 actor 发送消息，如果你想得到这个 actor 的回复，可以在消息中包含自身。示例如下：

```
class ActorItem extends Actor {
  def act(): Unit = {
    react {
      case (name: String, actor: Actor) => {
        println( name )
        actor ! "Done"
      }
    }
  }
}

object Test {
  def main (args: Array[String]): Unit = {
    val actorItem = new ActorItem
    actorItem.start()

    actorItem ! ("scala", self)

    receive {
      case msg => println( msg ) 
    }
  }
}
```

```
输出：
scala
Done
```

###使用样本类
在上例中，若把```(name: String, actor: Actor)```定义成类，代码可读性会大大提高

```
case class Info(name: String, actor: Actor)

class ActorItem extends Actor {
  def act(): Unit = {
    react {
      case Info => {
        println( Info.name )
        actor ! "Done"
      }
    }
  }
}
```


##参考
1. 《Scala 编程》
2. http://developer.51cto.com/art/200908/141150.htm
3. http://git.bookislife.com/post/2015/jgsk-scala-06-actor/



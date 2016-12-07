# Circuit Breaker

Circuit Breaker模式会处理一些需要一定时间来重连远程服务和远端资源的错误。该模式可以提高一个应用的稳定性和弹性。

## 问题

在类似于云的分布式环境中，当一个应用需要执行一些访问远程资源或者是远端服务的时候，是很容易碰到一些偶然的错误的，比如说，网络连接速度很慢，超时，或者是资源的过量使用，或者临时资源不再可用等等。这一类的错误通常来说会在短暂的时间内，自动恢复过来。一个健壮的云应用也该能够通过一些策略能够处理这类错误，比如使用**重试**模式。

然而，也有一些情况，错误是出于一些意想不到的事件，这类事件很难预期，而且需要消耗很多时间来修正。这些错误在严重性上也从丢失部分连接到整个服务的失败。在这些情况下，让应用继续重试或者执行操作就已经没有意义了。相对的，应用应该迅速接收服务的失败，而尝试根据错误类型来采取对应的措施。

另外，如果一个服务非常繁忙，系统的部分错误可能会导致雪崩效应。举例来说，一个操作其他服务的操作可以配置超时时间的，如果服务再一段时间无法应答，调用方可以返回错误信息的。然而这个策略可能导致大量针对这个服务的请求阻塞直至Timeout时间到了。这些阻塞的请求可能会持有系统关键的资源，比如内存，线程，数据库连接等等信息。因此，引用的资源也可能会被耗尽，造成系统内其他不相关部分得失败。在这些情况下，最好的方法是令这些配置超时的操作立刻失败，并且只有当服务可能成功的时候才去调用。当然，配置较短的超时时间也能改善这一问题，但是超时时间不能配置太短，那样服务的调用反而会因为大量的超时而失败。

## 解决方案

Circuit-Breaker模式可以防止应用重复的尝试调用容易失败的操作，当Circuit-Breaker模式判断错会会持续的时候，它会令操作不再持续等待去浪费CPU资源。当然，Circuit-Breaker模式也令应用本身可以发现错误有没有被修复。如果发生的问题已经被修复了，应用可以重新尝试去调用服务。

> Circuit-Breaker模式的目的和Retry模式的目的是不同的。Retry模式令应用不断的重试调用，直到最后成功。而Circuit-Breaker模式是阻止应用继续尝试无意义的请求。应用可以同时使用两种模式。然而，重试逻辑应用对于所有的Circuit-Breaker返回的异常十分敏感，这样可以在Circuit-Breaker发现错误短时间无法修复的情况下直接不再继续重试。

Circuit-Breaker的作用就好似可能失败操作的代理。代理会监控最近发生的错误，然后依据这一信息来决定是否允许操作的继续执行，或者直接立刻返回异常信息。

Circuit-Breaker可以按照如下的状态来模仿一个断路器来实现：

* **关闭**：应用的请求已经路由到了这个操作。代理应该维护最近一段时间的错误信息，如果调用操作失败，那么大力增加这个错误信息的数量。如果这个错误数量超过给定时间的阈值，代理进入到**打开**状态。这个时候，代理启动一个超时的Timer，当Timer过期了，代理则进入**半开**状态。

> 超时Timer的目的是为了给系统一段时间来自我修复之前碰到的问题。

* **打开**：令调用可能失败的操作立刻失败，所有的调用直接抛异常给应用。
* **半开**：只有一定数量的应用请求可以进行操作的调用。如果这些请求成功了，那么就假定之前发成的错误已经被系统自动修复了，而Circuit-Breaker转换成**关闭**状态（同时重置错误计数器）。如果任何请求失败了，那么Circuit-Breaker会假定错误仍然在存在，Circuit-Breaker会重新转换成**打开**状态，并重启超时Timer给系统更多的时间来自我修复错误。
> **半开**状态可以很有效的阻止一个可以恢复的服务被大量的请求所淹没。而处于恢复中的服务，可能也能够承载一定数量的请求，直到完全恢复才能恢复全部的吞吐量。但是，突然大量的错误也可能会令恢复中的服务重新crash掉。

参考如下状态变换图：

![](cache-aside-pattern.png)

需要注意的是，上图中，关闭状态所用的错误计数器是基于时间的。它会以一定的时间间隔来重置。这也能够在常见错误的情况下不让Circuit-Breaker模式进入打开状态。而错误计数阈值才会令Circuit-Breaker进入到打开状态，只有当指定时间间隔内，错误计数达到阈值才能令Circuit-Breaker进入到打开状态。半开状态所使用的成功计数器则会记录成功的调用次数。Circuit-Breaker如果在之后出现了连续的成功的调用，那么Circuit-Breaker就会进入关闭状态。如果任何调用的失败了，那么Circuit-Breaker也会重新进入到打开状态，成功计数器也会重置，直到下次重新进入到半开状态。

> 通常系统的外部恢复，很多时候都是通过重启失败的组件或者修复网络连接来完成的。

实现Circuit-Braker模式可以增加系统的稳定性和弹性，当系统从错误恢复的时候，可以尽可能所有失败对系统性能的影响。Circuit-Breaker模式可以通过拒绝外部调用来保证服务的响应时间，而不是等待操作的超时（或者持续阻塞）。如果Circuit-Breaker在每一次状态改变的时候启动一些事件的话，这个状态的改变也可以用来监视Circuit-Breaker保护模块的健康状态，或者是对监控Circuit-Breaker的管理员发出警告，Circuit-Breaker已经进入了打开状态。

Circuit-Breaker模式可以很好的定制并适配很多可能的错误。举例来说，开发者可以应用一个增长的超时Timer，也可以直接令Circuit-Breaker在处于打开状态几秒，如果错误在之后还没有解决，就超时几分钟等等。在有些场景下，打开状态的Circuit-Breaker也可以不抛出异常而是返回默认值来改善应用的响应。

## 需要考虑的问题

开发者在实现Circuit-Breaker模式的时候，有如下的一些地方需要注意：

* **异常的处理**。应用如果通过Circuit-Breaker来调用操作的话，就必须能够处理操作失效所引起的异常。而处理这些异常的代码将会和应用是高度相关的。举例来说，应用可以选择暂时降低其服务，调用其他的操作来达到完成相同的任务，或者获取相同的数据，或者抛出异常给用户令其稍后重试。
* **异常的类型**。请求失败可能有有多个原因，有些错误可能会比其他的错误更严重。举例来说，请求失败可能是因为远端的服务creash掉了，需要几分钟时间来恢复，或者只是因为服务过载而造成的暂时性服务超时。Circuit-Breaker也能够判断错误的类型，来调整不同的策略。举例来说，针对超时配置的错误计数阈值可以配置的比服务失效的阈值更高。
* **日志**。Circuit-Breaker应该把所有的失败请求都记录日志（和可能成功的请求），这样可以让管理员监控到外部调用的健康状态。
* **恢复性**。开发者应该为其保护的远端调用进行合理的配置来匹配远端调用的恢复模式。举例来说，如果Circuit-Breaker配置的停留在打开状态很久的话，就算远端服务已经可用了，因为Circuit-Breaker的打开状态，会令服务的状态仍然处于不可用状态。类似的，如果配置Circuit-Breaker的恢复时间太快，也会让应用在打开状态和半开状态之间不断震荡。
* **测试失效操作**。在打开状态，相对于使用Timer来判断什么时间转换为半开状态，Circuit-Breaker也可以选择间隔性的ping远端的服务（资源）来决定其是否可用。ping操作也可以作为一种尝试远端调用的方式，当然，也可以用远端服务提供的其他接口来测试服务是否正常。这在Health-Endpoint监控模式中有所描述。
* **手动覆盖**。在某个系统中，如果其失效的时间是可见的，Circuit-Breaker也可以提供一些手动恢复的选项，来令管理员强制的关闭Circuit-Breaker。类似的，管理员也可以在远端服务暂时失效的情况下强制性配置Circuit-Breaker进入打开状态（重启超时Timer）
* **并发**。同一个Circuit-Breaker很可能被应用的实例大量并发访问。所以其实现应该是非阻塞的或者对于每个请求都增加更多的消费。
* **资源的不同**。需要注意的是，当为一种类型的资源配置一个Circuit-Breaker但是可能有多个独立的提供资源的服务时要尤其小心。举例来说，在一个包含多个Shard的数据仓库中，其中的一个Shard没问题而另一个可能短时间内访问有问题。如果错误的返回在上面的场景中混合在了一起，即使错误很类似，应用也会尝试访问，从而阻塞的。
* **加速断路**。有的时候，错误的应答信息足够判断当前的状态而让Circuit-Breaker立刻触发。举个例子：如果一个Shard的返回信息表示，不建议立刻重试，希望在几分钟后重试的时候，那么Circuit-Breaker就不需要计数器到达阈值在进入Open状态了。
> HTTP协议定义：503表示服务不可用，如果请求的服务当前在web服务器上面不可用，就可以返回503。这个应答信息就可以包含额外的信息，比如期望的延迟重试时间。

* **重演失败请求**。在**打开**状态，相对于让服务快速的失败，Circuit-Breaker也可以记录具体的请求，然后在稍后的时间，重新令这些失败的请求再来请求。
* **外部请求上的不恰当超时时间**。如果对于外部请求的超时时间配置的过长的话，Circuit-Breaker可能很难保护应用。如果超时时间过长，运行Circuit-Breaker的线程可能在认为服务失败之前，就被完全阻塞了。这种情况下，应用的实例就算是Circuit-Breaker触发了，进入了Open状态，也会有相当数量的线程处于阻塞状态的。

## 何时使用该模式

使用该模式：

* 当需要阻止应用不断尝试调用远端服务或者访问共享资源，并且这些请求很容易失败的时候使用Circuit-Breaker模式很合适。

什么场景不适合使用该模式：

* 当用来处理访问本地资源，比如内存中的数据结构的时候，不适合使用。在这种场景下，Circuit-Breaker只会给应用带来额外的负担。
* 将Circuit-Breaker作为处理应用中的业务逻辑中的异常处理的一部分也是不合适的。

## Circuit-Breaker使用举例

在web应用中，有些页面是需要从外部的服务来获取数据的。如果系统实现了最小额度的缓存，那么页面的大量访问可能就会引起大量的调用。如果web应用和外部服务之间配置了超时（比如60s）的话，如果外部服务没有响应，页面会认为服务失效并抛出异常。
然而，如果服务失败了，而系统仍然非常的频繁访问，用户可能会被迫等待60秒，然后看到无结果。最后，像是内存，连接数，以及线程等资源都不足了，就算用户不再访问外部资源，可能服务也会被拒绝的。
当然，增加web服务器和使用负载均衡等方式都能一定程度上防止资源的耗尽，但是，这样仍然无法解决用户长时间等待没有响应页面的问题。
通过使用Circuit-Breaker来包裹链接外部服务的逻辑可以有效削弱上面提到的问题。用户请求将会失败，但是请求会立刻失败，但是不会导致请求资源的阻塞。
`CircuitBreaker`类通过内部一个`ICircuitBreakerStateStore`对象来维护Circuit-Breaker的状态信息。参考如下代码：

```
interface ICircuitBreakerStateStore
{
    CircuitBreakerStateEnum State { get; }
    Exception LastException { get; }
    DateTime LastStateChangedDateUtc { get; }
    void Trip(Exception ex);
    void Reset();
    void HalfOpen();
    bool IsClosed { get; }
}
```
其中的`State`属性表示Circuit-Breaker当前的状态，其中包含前面所提到的三个状态`Open`,`HalfOpen`,`Closed`。`IsClose`属性在状态为`Closed`的时候就会返回`true`。`Trip(Exception ex)`方法会将Circuit-Breaker的状态，转换到`Open`的状态，并且记录引起状态变化的异常信息，以及发生异常的时间等信息。`LastException`属性以及`LastStateChangeDateUtc`属性就是用来获取状态转换的异常以及时间信息的。`Reset()`方法则会关闭Circuit-Breaker，`HalfOpen()`方法则是将Circuit-Breaker的状态置为`HalfOpen`。

```
public class CircuitBreaker
{
    private readonly ICircuitBreakerStateStore stateStore =
        CircuitBreakerStateStoreFactory.GetCircuitBreakerStateStore();
    private readonly object halfOpenSyncObject = new object ();
    ...

    public bool IsClosed { get { return stateStore.IsClosed; } }
    
    public bool IsOpen { get { return !IsClosed; } }
    
    public void ExecuteAction(Action action)
    {
        ...
        if (IsOpen)
        {
            // The circuit breaker is Open.
            ... (see code sample below for details)
        }
        // The circuit breaker is Closed, execute the action.
        try
        {
            action();
        }
        catch (Exception ex)
        {
            // If an exception still occurs here, simply
            // re-trip the breaker immediately.
            this.TrackException(ex);
            // Throw the exception so that the caller can tell
            // the type of exception that was thrown.
            throw;
        }
    }
    private void TrackException(Exception ex)
    {
        // For simplicity in this example, open the circuit breaker on the first exception.
        // In reality this would be more complex. A certain type of exception, such as one
        // that indicates a service is offline, might trip the circuit breaker immediately.
        // Alternatively it may count exceptions locally or across multiple instances and
        // use this value over time, or the exception/success ratio based on the exception
        // types, to open the circuit breaker.
        this.stateStore.Trip(ex);
    }
}
```

`CircuitBreaker`会创建一个实现`ICircuitBreakerStateStore`的实例来维护CircuitBreaker的状态。其中的`ExecuteAction(Action action)`方法会包含一个可能出错的方法。当这个方法运行的时候，会首先检查Circuit-Breaker的状态，如果是关闭状态，则会正常的执行远端服务的调用。如果这个操作失败掉了，则会通过`TrackException(Exception ex)`方法将`Circuit-Breaker`的状态置为打开状态。下面参考IsOpen中的代码：

```
if (IsOpen)
{
    // The circuit breaker is Open. Check if the Open timeout has expired.
    // If it has, set the state to HalfOpen. Another approach may be to simply
    // check for the HalfOpen state that had be set by some other operation.
    if (stateStore.LastStateChangedDateUtc + OpenToHalfOpenWaitTime < DateTime.UtcNow)
    {
        // The Open timeout has expired. Allow one operation to execute. Note that, in
        // this example, the circuit breaker is simply set to HalfOpen after being
        // in the Open state for some period of time. An alternative would be to set
        // this using some other approach such as a timer, test method, manually, and
        // so on, and simply check the state here to determine how to handle execution
        // of the action.
        // Limit the number of threads to be executed when the breaker is HalfOpen.
        // An alternative would be to use a more complex approach to determine which
        // threads or how many are allowed to execute, or to execute a simple test
        // method instead.
        bool lockTaken = false;
        try
        {
            Monitor.TryEnter(halfOpenSyncObject, ref lockTaken)
            if (lockTaken)
            {
                // Set the circuit breaker state to HalfOpen.
                stateStore.HalfOpen();
                // Attempt the operation.
                action();
                // If this action succeeds, reset the state and allow other operations.
                // In reality, instead of immediately returning to the Open state, a counter
                // here would record the number of successful operations and return the
                // circuit breaker to the Open state only after a specified number succeed.
                this.stateStore.Reset();
                return;
            }
        }
        catch (Exception ex)
        {
            // If there is still an exception, trip the breaker again immediately.
            this.stateStore.Trip(ex);
            // Throw the exception so that the caller knows which exception occurred.
            throw;
        } 
        finally
        {
            if (lockTaken)
            {
                Monitor.Exit(halfOpenSyncObject);
            }
        }
    }
}
```

上面的Circuit-Breaker的策略很简单，就是等待一定的时间，然后才进入`HalfOpen`状态，如果`action()`成功了，则重新恢复到`Close`状态。如果`action()`失败了则Circuit-Breaker重新进入到`Open`状态。

## 相关的其他模式
下面的模式跟Circuit-Breaker模式也是相关的：

* **Retry模式**：重试模式属于Circuit-Breaker模式的一个附属。主要处理的问题是当访问远程服务不可用的时候，令应用如何来处理可以预期的短时间的错误。
* **Health Endpoint Monitoring模式**：Circuit-Breaker可以通过发送请求到远端的服务提供的特殊的服务来监控对面服务的的健康状态。该服务需要返回一些信息来展示其健康的状态。

# Retry

Retry模式能够通过重复之前失败的操作来处理那些在调用远端服务或者网络资源的时候发生的一些可以预期的临时性的错误。Retry模式可以提高应用的稳定性。

## 问题

应用中，负责链接其他服务的组件必须要对环境中可能发生的临时性错误十分敏感。这些错误包括瞬间的网络连接丢失，服务的暂时不可用，或者是服务繁忙导致的超时等等。这些错误都属于不需要额外操作就能够自我修复的错误，只需要过了一定的时间延迟再重复之前的失败操作就能轻易修复。举例来说，如果有一个数据库服务处理大量的并发请求，该服务实现了一个简单的熔断策略，当到达阈值后，会直接拒绝多余的请求。那么，如果开发者的应用调用这个服务被拒绝了，那么只要等待一段时间再进行重试即可修复这个问题。

## 解决方案
在云环境中，临时性的错误是较为常见的，应用应该在设计的时候就应该能够处理这类错误，以减少这类错误对于业务的影响。

如果应用在通过请求访问远程服务的时候，检测到了错误，可以通过如下的策略来处理这些错误：

* 如果错误并非暂时性的，或者通过重复很难成功的话（举例来说，由于错误的用户名和密码造成的验证失败），应用就应该终止操作并抛出一个合适的异常信息。
* 如果发生了特殊的错误或者很少见的错误，可能是因为一些奇怪的状况，比如网络在传输的过程中丢包了。在这种情况下，应用可以立刻重试之前失败的请求。因为发生错误的状态不易重现，请求在这种情况下很可能成功。
* 如果错误是因为常见的连接性失败，或者因为远端服务繁忙引起的失败，远端服务可能会需要短暂的时间来恢复正常，或者需要一点时间来处理掉之前堆积的任务。那么应用应该等待一点时间再进行重试之前失败的操作。

对于大多数的常见的短暂性错误，第一次失败操作和第二次重试操作之间的时间间隔应该合理配置，来尽可能让应用的不同实例(云中的不同应用服务器)尽量保持发送请求的时间间隔。这样做可以防止服务过于繁忙而造成过载。如果应用的多个实例不断的高频率的发送重试的请求，远端服务的恢复时间可能会更久。

如果请求仍然失败，应用可以等待更久的时间然后再进行下一次的重试。如果有必要的话，重试的失败后可以增加延迟时间，直到这个时间到达一个阈值，然后再将服务置为失败。重试的间隔时间可以是逐渐递增的，也可以使用其他的时间策略，比如指数增长，或者是根据失败请求的一些特性来配置都是可以的。

下图展示了Retry模式的流程，如果远端服务的请求失败了，则继续重复请求，一旦失败的次数超过了预设的失败次数，应用就视该次请求为异常情况，然后在进行后续的处理。

![](Retry.png)

1. 应用调用宿主的服务，请求失败了，响应的是HTTP500，是服务器内部错误。
2. 应用等待了一段时间，然后继续重复之前的失败请求。请求仍然失败了，仍然返回了HTTP500。
3. 应用等待了更长的时间再次重试，这次请求成功了，服务返回HTTP200.

应用应该将所有尝试访问远程的服务都通过Retry模式的策略来包裹起来。而访问不同的远端服务，也应该使用不同的时间策略，有些宿主服务还会提供封装了重试的库。而这些实现库通过参数来配置策略，而开发者可以通过指定这些参数来配置重试次数，时间策略等信息。

应用中的检测错误和重试失败操作的代码应该记录所有失败操作的日志信息。这些信息对于后续的维护是很有用的。如果应用的日志经常显示远端服务不可用或者繁忙，通常可能是因为远端服务的资源耗尽了。开发者可以通过对远端服务的扩展来降低这些错误。比如，如果一个数据库服务经常过载，可以将数据库进行分离，将负载分散到多个服务器来缓解这个问题。

> Windows Azure提供了很多对于Retry模式的支持。[短暂错误处理模式和实践](http://msdn.microsoft.com/library/hh680934%28v=pandp.50%29.aspx)一文中描述了在Windows Azure服务中可以使能很多重试策略来处理短暂性错误。
微软的[Entity Framework version 6](http://msdn.microsoft.com/data/dn456835.aspx)一书中提供了很多关于重试数据库操作的工具。另外，很多Windows Azure的服务和存储API都实现了重试模式的。

## 需要考虑的问题

在实现Retry模式的时候也需要考虑如下一些问题：

* 重试策略最好能够很好的适配应用的业务需求和错误的特点。在有的时候，令一些非关键操作快速失败可能比多次重试要更好。举个例子，在一个交互式的web应用中，当应用对远端服务请求失败的时候，打印出失败的信息（比如说，“请稍后再试”之类的信息）要比令用户持续等待，没什么响应要好的多。而对于分批类型的应用，将重试的等待时间以指数增长可能更为合适。
* 如果一个重试的策略配置了很小的等待间隔，并且配置了很大数量的重试次数，那么这个策略可能令一个繁忙的服务的资源更容易耗尽。重试策略如果不断重试执行失败的操作，也会影响到应用的其他正常服务的响应。
* 如果对于远端的请求在大量重试之后仍然失败的话，令应用在一段时间不再继续请求远端服务，并且立刻抛出错误可能更为合适。当配置的时间过期后，应用再试探性的发送请求去请求远端资源来判断远端服务是否恢复正常。关于更多的信息可以参考[Circuit-Breaker模式](../Circuit-Breaker/circuit-breaker-pattern.md)来了解更多的信息。
* 应用访问的远端服务的操作需要是幂等的。例如，如果应用的请求已经到达远端服务了，并且已经正常处理了，但是因为临时性错误，并没有发送成功执行的响应回到应用端，那么应用中的重试逻辑可能会继续重复之前的操作将重复的操作再次发送，因为应用并没有收到服务调用成功的应答信息。这样的话，非幂等的远端服务就可能改变了应该持有的状态。
* 根据场景的不同，对远端服务的请求失败的原因也是多种多样的，产生的异常信息也各有不同。有些错误可以告诉开发者远端服务可以很快恢复，而有些异常信息则可能意味着错误会持续很久的时间。重试策略最好根据异常的不同来调整重试的时间间隔。
* 实现Retry模式也需要考虑重试操作对于全局事物一致性的影响。最好尽可能的配置好Retry策略，让事物操作能够尽可能的成功，以减少全部的回滚操作。
* 使用Retry模式一定要确保所有的重试代码针对不同的错误情况都有测试用例覆盖到。也需要检查Retry模式是否有严重影响了应用的性能或者可靠性。因为过多的加载远端服务和资源可能产生竞争或者瓶颈。
* 实现Retry模式的时候，整个上下文一定要理解失败的操作是如何处理的。举例来说，如果一个包含了重试策略的任务调用了另一个包含了重试策略的任务，外层处理会对大大增加处理的时间。这样的话，最好就将内层的任务配置为快速失败，并且将失败的异常信息交给调用它的外层任务。这样，外层的任务可以根据其自己的策略来决定如何处理错误了。
* 将每次重试的链接失败信息记录日志都是十分必要的，这些日志可以帮忙识别出应用，远端服务或者资源的一些潜在问题。
* 查看在请求远端服务所容易产生的错误，判断这些错误是不是会持续很久而无法恢复。如果是这类问题，最好将这类错误定义为一类异常。应用将这类异常记录下来，并且可以尝试调用替代的服务（如果存在的话）或者直接拒绝服务。想了解更多如何检查和处理长时间的错误，可以参考[Circuit-Breaker模式](../Circuit-Breaker/circuit-breaker-pattern.md)。

## 何时使用该模式

何时该使用Retry模式：

* 当应用可能在与远端服务交互或者请求远端资源的时候发生短暂性错误的时候就可以考虑实现Retry模式。这些错误是那种短暂性的，能够自我恢复的。这样，通过重复之前失败的操作就可以令应用正常。

何时不该使用Retry模式：

* 当错误是属于长时间持续的时候不该使用Retry模式。因为当错误无法自己恢复的时候，持续的请求外部服务只会影响应用本身的响应。应用的继续重试只会浪费时间和应用服务器的资源(线程，连接数等等)。
* 当错误并非短暂性错误的时候不该使用Retry模式。比如具体业务中引起的内部异常。
* 考虑到系统的弹性处理，如果应用频繁碰到远端服务繁忙的错误，那么就该考虑是否该扩展被请求的服务或者资源了。


## Retry模式使用举例

下面的例子展示了实现Retry模式的一个方案。下面的`OperationWithBasicRetryAsync`方法，通过`TransientOperationAsync`调用了一个外部的异步服务。

```
C#
private int retryCount = 3;
...
public async Task OperationWithBasicRetryAsync()
{
    int currentRetry = 0;
    for (; ;)
    {
        try
        {
            // Calling external service.
            await TransientOperationAsync();
            // Return or break.
            break;
        }
        catch (Exception ex)
        {
            Trace.TraceError("Operation Exception");
            currentRetry++;
            // Check if the exception thrown was a transient exception
            // based on the logic in the error detection strategy.
            // Determine whether to retry the operation, as well as how
            // long to wait, based on the retry strategy.
            if (currentRetry > this.retryCount || !IsTransient(ex))
            {
                // If this is not a transient error
                // or we should not retry re-throw the exception.
                throw;
            }
        }

        // Wait to retry the operation.
        // Consider calculating an exponential delay here and
        // using a strategy best suited for the operation and fault.
        Await.Task.Delay();
    }
}

// Async method that wraps a call to a remote service (details not shown).
private async Task TransientOperationAsync()
{
    ...
}
```

封装在**try-catch**代码块中的方法调用被封装到一个`for`循环中。`for`循环只有当`TransientOperationAsync`方法成功的返回或者抛出异常才能退出`for`循环。如果`TransientOperationAsync`方法失败了，那么`catch`代码块会检查产生错误的原因，如果错误属于临时性错误，那么应用会等待一会，然后再重新进行之前的操作。

`for`循环也会监控外部调用的调用的次数，如果代码失败次数超过了3次，那么应用就会认为错误还会继续持续一段时间。如果异常并非是暂时性错误，或者这个错误持续的时间可能比较久，`catch`代码块就会直接抛出异常。这个异常也一样可以退出`for`循环，并且这个异常最好由调用`OperationWithBasicRetryAsync`方法的地方来处理。

下面的`IsTransient`方法会检查异常的类型，来判断异常是否属于内部错误还是属于`WebException`或者是临时性错误。其中的`OperationTransientException`就代表调用的操作发生了临时性错误。参考如下代码:

```
private bool IsTransient(Exception ex)
{
    // Determine if the exception is transient.
    // In some cases this may be as simple as checking the exception type, in other
    // cases it may be necessary to inspect other properties of the exception.
    if (ex is OperationTransientException)
        return true;

    var webException = ex as WebException; if (webException != null)
    {
        // If the web exception contains one of the following status values
        // it may be transient.
        return new[] {
            WebExceptionStatus.ConnectionClosed,
            WebExceptionStatus.Timeout,
            WebExceptionStatus.RequestCanceled }. Contains(webException.Status);
    }

    // Additional exception checking logic goes here.
    return false;
}
```

## 相关的其它模式

在文章中有提到过最多的就是[Circuit-Breaker模式](../Circuit-Breaker/circuit-breaker-pattern.md)

* **[Circuit-Breaker模式](../Circuit-Breaker/circuit-breaker-pattern.md)**。Retry模式主要是用来处理临时性错误。如果错误的持续时间较长，使用Retry模式可能会使应用的资源很容易被耗尽，更适合考虑实现Circuit-Breaker模式。当然，两者可以考虑配合使用，能够更好的扩展服务的稳定性。
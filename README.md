# Do. Or do not. There's no Try.
One of the great things about being a programmer is that you never stop learning. Even after having programmed for almost 35 years, I still improve on the way I write code. About a year ago the way I write code changed once again when I started to apply monads and especially the `Try` class.

## Port from Scala
During a recent Java project, one of the guys on my team created a small library that ports the behaviour of the Scala `Try` classes (including `Success` and `Failure`). The 'Try' class basically allows you to invoke a function that is passed as a parameter, and wrap either the result in an instance of the `Success` class, or in the case and exception occurs, wraps this exception in an instance of the `Failure` class. These can than be used to invoke a next function on the result of the previous invocation. Thus, `Try` allows for chaining method without any exception being thrown, resulting in highly compact and robust code. 

Although at first this new monad didn't appeal to me, I soon really started to appreciate this style of programming, where we concatenate series of `Map()` and `FlatMap()` methods, using the power of the `Try` monad, and thus avoiding abundant try-catch blocks, many `if` statements and null checks. Hence, I decided to port this library to C#. 

## Try and succeed
When programming using the `Try` classes, you can actually avoid most try-catch blocks in code, because the monad `Try` will wrap possible exceptions, and just return whether the actions and functions you've called fail or succeed.

In the (simple) code example below an instance of the class `Employee` is created, the name of the employee is fetched in the `Map()` statement where `e` is really the instance of `Employee` that was created above, and we check whether it ends with the character s. This will return a `Try<bool>`, and we use the property `Value` to get the actual value from the `Try`. 

```c#
    var result = Try<Employee>.Invoke(() => repo.Create("Kees"))
        .Map(e => e.Name)
        .Map(s => s.EndsWith("s"))
        .Value;

    Assert.IsTrue(result);
```
Every time either an `Invoke()`, `Map()` or `Flatmap()` method is executed, the function that is passed as a lambda expression is invoked within a try-catch block. If there is no exception thrown, these methods return a new instance of the `Success` class, with the result from the invocation inside it. If an exception is thrown, it will be wrapped in a new instance of the `Failure` class. As both of these inherit from the `Try` class, we can concatenate another round of  `Map()` or `Flatmap()` methods again and agina.  

Getting the `Value` property in the end will only return a valid value IF all of the statements above have executed succesfully - that is, if we have an instance of the `Success` class in the end. If not, we have an instance of `Failure` on our hands and getting the value will throw an exception.

## Try and fail
In the next example, some more of the power of `Try` is exposed. Here one of the statements fails (deliberately). 

```c#
    var result = Try<Employee>.Invoke(() => repo.Create("Kees"))
        .Map(e => e.WillThrowException());

    Assert.IsTrue(result.IsFailure);
```
Now, the first `Invoke()` call return an employee, but the second statement `Map()` throws an exception. But instead of crashing your code, the result will be an instance of the `Failure` class, holding the exception that was thrown. If one of the statements in your code returns an instance of `Failure` all following statements are being ignored.

Although this example seems trivial, once you get used to programming with `Try` you will soon realize that it is actually quite powerful in building more robust code, that also becomes much easier to test too. 

## Recovering from failure
The next thing you might want to do is to recover from failure and continue. To this aim there are the `Recover()` methods.

```c#
    var result = Try<Employee>.Invoke(() => repo.Create("Kees"))
        .FlatMap(e => e.WillThrowException())
        .Recover(ex => repo.Create("Jaap"));

    Assert.IsTrue(result.IsSuccess);
    Assert.AreEqual(result.Value.Name, "Jaap");
```

In this code example the second statement throws and will return an instance of `Failure`. What the `Recover()` method in the next statement will do is help you get back on track. It will create a new employee (with the name Jaap) and allow you to continue. Quite often, recover statements appear at the end of a block of statements, for instance to recover from REST calls that do not return anything useful.

In case your code can throw several different exceptions and you want to recover in different ways, the `Try` class offers some additional features as shown in the example below.

```c#
    var result = Try<Employee>.Invoke(() => repo.Create("Kees"))
        .FlatMap(e => e.WillThrowTryException())
        .Recover<TryException>(e => repo.Create("Jaap"))
        .Recover(ex => repo.Create("Jan"));

    Assert.IsTrue(result.IsSuccess);
    Assert.AreEqual(result.Get().Name, "Jaap");
```

Here, the generic `Recover<T>` method only fires when this specific type of exception is thrown in one of the invoked methods.

## What if?
Next to avoiding try-catch blocks in your code, and making the code robust, the `Try` monad also allows you to avoid many if-statements (the next big thing to avoid after goto-blocks). It allows you to define filters on the results of previous calls to `Invoke()`, `Map()` or `Flatmap()` methods, as in the example below.


```c#
    var result = Try<Employee>.Invoke(() => repo.Create("Kees"))
        .Filter(e => e.IsNameKees().Value);

    Assert.IsTrue(result.IsSuccess);
```
## Starting with return
All in all, after having used the `Try` monad in many different situations, and in different programming languages, my preferred style of coding has totally changed. These days almost all of the methods I write immediately start with a return statements, followed by a concatenations of `Map()` and `Flatmap` calls - along with a number of other useful features on the `Try` classes. As a result the code I write is smaller and much more compact, much better in adhering to the Single Responsibility Principle. And my code much more robust - as it also catches any exception you might have overlooked when writing more traditional code.

In the meantime, I have conteminated many programmers with this style. At one of my clients the developers actually make it a sport to always start every method with e return statement. Don't hesitate to join in.

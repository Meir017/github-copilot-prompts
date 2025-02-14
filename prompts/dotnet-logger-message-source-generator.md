Replace usages of ILogger methods with a log method annotated with [LoggerMessage] 

- Make the class `partial` if it's not already.
- The generated methods should be at the end of the class, after all members, the order of the properties should be
    1. Level
    2. if relevant, EventId
    3. Message
- Set the `EventId` parameters only if the original log message had it. never set the value to `0`.
- if there's a private field of type `ILogger` make the generated method `private partial void`
otherwise make it `private static partial void` as pass the ILogger as the first parameter
- If the containing type has no `ILogger` field, then the generated methods should be static passing in the `ILogger` as the first parameter. 
- If a logging method is `static`, the `ILogger` instance is required as a parameter.
- If the log method is wrapped in an `if (logger.IsEnabled(...))` block, remove if statement since it's implicit in the generated method.
- when a method call that is not the `.ToString()` is passed to the log message, create a struct that receives the object and overrides the ToString to call the method that was previously passed to the log message.
the struct should be a nested `private readonly struct` in the class using the primary-constructor syntax, if the implementation of the `ToString` method is just calling `.ToString` on the object, the struct should not be generated. The generated struct should be right before the generated methods that use it,  
example:
```cs
Arg arg = ...;
logger.LogInfromation("the message {arg}", arg.CustomStringifier());
```
should be converted to:
```cs
private readonly struct ArgLogValue(Arg arg)
{
    public override string ToString() => arg.CustomStringifier();
}

[LoggerMessage(
    Level = LogLevel.Infromation,
    Message = "the message {arg}"
)]
private partial void LogInfoTheMessage();

LogInfoTheMessage(new(arg));
```

Refactor all of the logger methods in the class.

Examples:

Example 1:
```cs
TimeSpan currentRequestActiveTime = ...;
Message message = ...;
Message blockingRequest = ...;
ActivationData activation = ...;
_shared.Logger.LogWarning((int)ErrorCode.Dispatcher_ExtendedMessageProcessing, "Current request has been active for {CurrentRequestActiveTime} for grain {Grain}. Currently executing {BlockingRequest}. Trying to enqueue {Message}.", currentRequestActiveTime, activation.ToDetailedString(), blockingRequest, message);
```

should be converted to a method call

```cs
TimeSpan currentRequestActiveTime = ...;
Message message = ...;
Message blockingRequest = ...;
ActivationData activation = ...;
LogWarningDispatcher_ExtendedMessageProcessing(_shared.Logger, currentRequestActiveTime, new(this), _blockingRequest, message)
```

where the generated method would be
```cs
private readonly struct ActivationDataLogValue(ActivationData activation, bool includeExtraDetails = false)
{
    public override string ToString() => activation.ToDetailedString(includeExtraDetails);
}

[LoggerMessage(
    EventId = (int)ErrorCode.Dispatcher_ExtendedMessageProcessing,
    Level = LogLevel.Warning,
    Message = "Current request has been active for {CurrentRequestActiveTime} for grain {Grain}. Currently executing {BlockingRequest}. Trying to enqueue {Message}."
)]
private static partial void LogWarningDispatcher_ExtendedMessageProcessing(ILogger logger, TimeSpan currentRequestActiveTime, ActivationDataLogValue grain, Message blockingRequest, Message message)
convert all of the remaining logger methods
```

Example 2:
```cs
if (logger.IsEnabled(LogLevel.Debug))
{
    logger.LogDebug("DeactivateActivations: {Count} activations.", list.Count);
}
```

should be converted to a method

```cs
LogDebugDeactivateActivations(list.Count)
```

where the signature of the method would be
```cs
[LoggerMessage(
    Level = LogLevel.Debug,
    Message = "DeactivateActivations: {Count} activations."
)]
private partial void LogDebugDeactivateActivations(int count)
```
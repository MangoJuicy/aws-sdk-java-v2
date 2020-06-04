**Design:** New Feature, **Status:** [Proposed](../../README.md)

# Waiters

"Waiters" are an abstraction used to poll a resource until a desired state is reached or until it is determined that
the resource will never enter into the desired state. This feature is supported in the AWS Java SDK 1.x and this document proposes 
how waiters should be implemented in the Java SDK 2.x. 

## Introduction

A waiter makes it easier for customers to wait for a resource to transition into a desired state. It comes handy when customers are
interacting with operations that are eventually consistent on the service side.

For example, when you invoke `dynamodb#createTable`, the service immediately returns a response with a TableStatus of `CREATING`
and the table will not be available to perform write or read until the status has transitioned to `ACTIVE`. Waiters can be used to help 
you handle the task of waiting for the table to become available.

## Proposed APIs

The SDK 2.x will support both sync and async waiters for service clients that have waiter-eligible operations. It will also provide a generic `Waiter` class
which makes it possible for customers to customize polling function, define expected success, failure and retry conditions as well as configurations such as `maxAttempts`. 

### Usage Examples

#### Example 1: Using sync waiters

```Java
DynamoDbClient client = DynamoDbClient.create();

DescribeTableResponse response = client.waiter().waitUtilTableExists(b -> b.tableName("table"));
```

#### Example 2: Using async waiters

```Java
DynamoDbAsyncClient asyncClient = DynamoDbAsyncClient.create();

CompletableFuture<DescribeTableResponse> responseFuture = 
    asyncClient.waiter().waitUtilTableExists(b -> b.tableName("table"));
```

*FAQ Below: "Why not create waiter operations directly on the client?"*

#### Example 3: Using the generic waiter

```Java
Waiter<DescribeTableResponse> waiter =
   Waiter.<DescribeTableResponse>builder()
        .addAcceptor(WaiterAcceptor.successAcceptor(r -> r.table().tableStatus().equals(TableStatus.ACTIVE)))
        .addAcceptor(WaiterAcceptor.retryAcceptor(t -> t instanceof ResourceNotFoundException))
        .addAcceptor(WaiterAcceptor.errorAcceptor(t -> t instanceof InternalServerErrorException))
        .maxAttempts(20)
        .backoffStrategy(BackoffStrategy.defaultStrategy())
        .build();

// run synchronousely 
DescribeTableResponse response = waiter.run(() -> client.describeTable(describeTableRequest));

// run asychronousely
CompletableFuture<DescribeTableResponse> responseFuture =
      waiter.runAsync(() -> asyncClient.describeTable(describeTableRequest));
```

### `{Service}Waiter` and `{Service}AsyncWaiter`

Two classes will be created for each waiter-eligible service: `{Service}Waiter` and `{Service}AsyncWaiter` (e.g. `DynamoDbWaiter`, `DynamoDbAsyncWaiter`). 
This follows the naming strategy established by the current `{Service}Client` and `{Service}Utilities` classes.

#### Example

```Java
/**
 * Waiter utility class that waits for a resource to transition to the desired state.
 */
@SdkPublicApi
@Generated("software.amazon.awssdk:codegen")
public interface DynamoDbWaiter {

    /**
     * Poller method that waits for the table status to transition to <code>ACTIVE</code> by
     * invoking {@link DynamoDbClient#describeTable}. It returns when the resource enters into a desired state or
     * it is determined that the resource will never enter into the desired state.
     *
     * @param describeTableRequest Represents the input of a <code>DescribeTable</code> operation.
     * @return {@link DescribeTableResponse}
     */
    default DescribeTableResponse waitUtilTableExists(DescribeTableRequest describeTableRequest) {
        throw new UnsupportedOperationException();
    }

    default DescribeTableResponse waitUtilTableExists(Consumer<DescribeTableRequest.Builder> describeTableRequest) {
        return waitUtilTableExists(DescribeTableRequest.builder().applyMutation(describeTableRequest).build());
    }

    /**
     * Poller method that waits until the table does not exists by invoking {@link DynamoDbClient#describeTable}.
     * It returns when the resource enters into a desired state or it is determined that the resource will never enter into the desired state.
     *
     * @param describeTableRequest Represents the input of a <code>DescribeTable</code> operation.
     */
    default void waitUtilTableNotExists(DescribeTableRequest describeTableRequest) {
        throw new UnsupportedOperationException();
    }

    default void waitUtilTableNotExists(Consumer<DescribeTableRequest.Builder> describeTableRequest) {
        waitUtilTableNotExists(DescribeTableRequest.builder().applyMutation(describeTableRequest).build());
    }
}

/**
 * Waiter utility class that waits for a resource to transition to the desired state asynchronously.
 */
@SdkPublicApi
@Generated("software.amazon.awssdk:codegen")
public interface DynamoDbAsyncWaiter {

    /**
     * Poller method that waits for the table status to transition to <code>ACTIVE</code> by
     * invoking {@link DynamoDbAsyncClient#describeTable}. It returns when the resource enters into a desired state or
     * it is determined that the resource will never enter into the desired state.
     *
     * @param describeTableRequest Represents the input of a <code>DescribeTable</code> operation.
     * @return A CompletableFuture containing the result of the DescribeTable operation returned by the service. It completes
     * successfully when the resource enters into a desired state or it completes exceptionally when it is determined that the
     * resource will never enter into the desired state.
     */
    default CompletableFuture<DescribeTableResponse> waitUtilTableExists(DescribeTableRequest describeTableRequest) {
        throw new UnsupportedOperationException();
    }

    default CompletableFuture<DescribeTableResponse> waitUtilTableExists(Consumer<DescribeTableRequest.Builder> describeTableRequest) {
        return waitUtilTableExists(DescribeTableRequest.builder().applyMutation(describeTableRequest).build());
    }

    /**
     * Poller method that waits until the table does not exists by invoking {@link DynamoDbAsyncClient#describeTable}.
     * It returns when the resource enters into a desired state or it is determined that the resource will never enter into the desired state.
     *
     * @param describeTableRequest Represents the input of a <code>DescribeTable</code> operation.
     * @return A CompletableFuture containing the result of the DescribeTable operation returned by the service. It completes
     * successfully when the resource enters into a desired state or it completes exceptionally when it is determined that the
     * resource will never enter into the desired state.
     */
    default CompletableFuture<Void> waitUtilTableNotExists(DescribeTableRequest describeTableRequest) {
        throw new UnsupportedOperationException();
    }

    default CompletableFuture<Void> waitUtilTableNotExists(Consumer<DescribeTableRequest.Builder> describeTableRequest) {
        return waitUtilTableNotExists(DescribeTableRequest.builder().applyMutation(describeTableRequest).build());
    }
}
```

*FAQ Below: "Why returning the service response for waiter operations with a success state of a specific response" and "Why not returning a response for waiter operations with a success state of error?"*.

#### Instantiation

This class can be instantiated from an existing service client

```Java
// sync waiter
DynamoDbClient dynamo = DynamoDbClient.create();
DynamoDbWaiter dynamoWaiter = dynamo.waiter();

// async waiter
DynamoDbClient dynamoAsync = DynamoDbAsyncClient.create();
DynamoDbAsyncWaiter dynamoAsyncWaiter = dynamoAsync.waiter();
```

#### Methods

A method will be generated for each operation that needs waiter support. There are two categories depending on the expected success state.

- Operations with a desired condition where a specific *successful* response is returned
  - sync: `{Operation}Response waitUtil{DesiredState}({Operation}Request)`
    ```java
    DescribeTableResponse waitUtilTableExists(DescribeTableRequest describeTableRequest)
    ```
  - async: `CompletableFuture<{Operation}Response> waitUtil{DesiredState}({Operation}Request)`
    ```java
    CompletableFuture<DescribeTableResponse> waitUtilTableExists(DescribeTableRequest describeTableRequest)
    ```
- Operations with a desired condition where a specific *exception* is thrown
  - sync: `void waitUtil{DesiredState}({Operation}Request)`
    ```java
    void waitUtilTableNotExists(DescribeTableRequest describeTableRequest)
    ```
  - async: `CompletableFuture<Void> waitUtil{DesiredState}({Operation}Request)`
    ```java
    CompletableFuture<Void> waitUtilTableNotExists(DescribeTableRequest describeTableRequest)
    ```
    
### `Waiter<T>`

The generic `Waiter` class enables users to customize waiter configurations and provide their own `WaiterAcceptor`s which define the expected states and controls
the terminal state of the waiter.

#### Methods

```java
@SdkPublicApi
public final class Waiter<T> {
    /**
     * Runs the provided polling function. It completes when the resource enters into a desired state or
     * it is determined that the resource will never enter into the desired state.
     *
     * @param asyncPollingFunction the polling function to trigger
     * @return A CompletableFuture containing the result of the DescribeTable operation returned by the service. It completes
     * successfully when the resource enters into a desired state or it completes exceptionally when it is determined that the
     * resource will never enter into the desired state.
     */
    public CompletableFuture<T> runAsync(Supplier<CompletableFuture<T>> asyncPollingFunction) {
     ...
    }

    /**
     * Runs the provided polling function. It returns when the resource enters into a desired state or
     * it is determined that the resource will never enter into the desired state.
     *
     * @param pollingFunction Represents the input of a <code>DescribeTable</code> operation.
     * @return the response
     */
    public T run(Supplier<T> pollingFunction) {
        ...
    }
}
```

#### Inner-Class: `Waiter.Builder`

```java
    public interface Builder<T> {

        /**
         * Defines a list of {@link WaiterAcceptor}s to check if an expected state has met after executing an operation.
         *
         * @param waiterAcceptors the waiter acceptors
         * @return the chained builder
         */
        Builder<T> acceptors(List<WaiterAcceptor<T>> waiterAcceptors);

        /**
         * Add a {@link WaiterAcceptor}s
         *
         * @param waiterAcceptors the waiter acceptors
         * @return the chained builder
         */
        Builder<T> addAcceptor(WaiterAcceptor<T> waiterAcceptors);

        /**
         * Define the maximum number of attempts to try before transitioning the waiter to a failure state.
         */
        Builder<T> maxAttempts(int numRetries);

        /**
         * Define the {@link BackoffStrategy} that computes the delay before the next retry request.
         * @param backoffStrategy the backoff strategy
         * @return the chained builder
         */
        Builder<T> backoffStrategy(BackoffStrategy backoffStrategy);

        /**
         * Define the {@link ScheduledExecutorService} used to schedule async attempts
         *
         * @param scheduledExecutorService the schedule executor service
         * @return the chained builder
         */
        Builder<T> scheduledExecutorService(ScheduledExecutorService scheduledExecutorService);
    }
```

### `WaiterState`

`WaiterState` is an enum that defines possible states of a waiter to be transitioned to if a condition is met

```java
public enum WaiterState {
    /**
     * Indicates the waiter succeeded and must no longer continue waiting.
     */
    SUCCESS,

    /**
     * Indicates the waiter failed and must not continue waiting.
     */
    FAILURE,

    /**
     * Indicates that the waiter encountered an expected failure case and should retry if possible.
     */
    RETRY
}
```

### `WaiterAcceptor`

`WaiterAcceptor` is a class that inspects the response or error returned from the operation and determines whether an expected condition
is met and indicates the next state that the waiter should be transitioned to if there is a match.

```java
@SdkPublicApi
public interface WaiterAcceptor<T> {

    /**
     * @return the next {@link WaiterState} that the waiter should be transitioned to if this acceptor matches with the response or error
     */
    WaiterState waiterState();

    /**
     * Check to see if the response matches with the expected state defined by the acceptor
     *
     * @param response the response to inspect
     * @return whether it accepts the response
     */
    default boolean matches(T response) {
        return false;
    }

    /**
     * Check to see if the exception matches with the expected state defined by the acceptor
     *
     * @param throwable the exception to inspect
     * @return whether it accepts the throwable
     */
    default boolean matches(Throwable throwable) {
        return false;
    }

    /**
     * Creates a success waiter acceptor which determines if the response matches with the success state
     *
     * @param responsePredicate
     * @param <T> the response type
     * @return a {@link WaiterAcceptor}
     */
    static <T> WaiterAcceptor<T> successAcceptor(Predicate<T> responsePredicate) {
        return new WaiterAcceptor<T>() {
            @Override
            public WaiterState waiterState() {
                return WaiterState.SUCCESS;
            }

            @Override
            public boolean matches(T response) {
                return responsePredicate.test(response);
            }
        };
    }

    /**
     * Creates an error waiter acceptor which determines if the exception should transition the waiter to failure state
     *
     * @param errorPredicate
     * @param <T> the response type
     * @return a {@link WaiterAcceptor}
     */
    static <T> WaiterAcceptor<T> errorAcceptor(Predicate<Throwable> errorPredicate) {
        return new WaiterAcceptor<T>() {
            @Override
            public WaiterState waiterState() {
                return WaiterState.FAILURE;
            }

            @Override
            public boolean matches(Throwable t) {
                return errorPredicate.test(t);
            }
        };
    }

    /**
     * Creates a retry waiter acceptor which determines if the exception should transition the waiter to retry state
     *
     * @param errorPredicate
     * @param <T> the response type
     * @return a {@link WaiterAcceptor}
     */
    static <T> WaiterAcceptor<T> retryAcceptor(Predicate<Throwable> errorPredicate) {
        return new WaiterAcceptor<T>() {
            @Override
            public WaiterState waiterState() {
                return WaiterState.RETRY;
            }

            @Override
            public boolean matches(Throwable t) {
                return errorPredicate.test(t);
            }
        };
    }
}
```

## FAQ

### For which services will we generate waiters?

We will generate a `{Service}Waiter` class if the service has any operations that need waiter support.

### Why not create waiter operations directly on the client?

The options are: (1) create separate waiter utility classes or (2) create waiter operations on the client

The following compares Option 1 to Option 2, in the interest of illustrating why Option 1 was chosen.

**Option 1:** create separate waiter utility classes

```Java
dynamodb.waiter().waitUtilTableExists(describeTableRequest)
```

**Option 2:** create waiter operations on each service client

```Java
dynamodb.waitUntilTableExists(describeTableRequest)
```

**Option 1 Pros:**

1. consistent with existing s3 utilities and presigner method approach, eg: s3Client.utilities()
2. similar api to v1 waiter, and it might be easier for customers who are already using v1 waiter to migrate to v2.

**Option 2 Pros:**

1. slightly better discoverability

**Decision:** Option 1 will be used, because it is consistent with existing features and option2 might bloat the size
of the client, making it more difficult to use.

### Why returning the service response for waiter operations with a success state of a specific response?

The reason that the last response that has satisfied the waiter success state is returned is because it is a common pattern that customers creates a resource and then
retrieves the information of the resources. Without returning the response, customers will have to send an extra request when the waiter returns. 
See [feature request](https://github.com/aws/aws-sdk-java/issues/815)

### Why not returning a response for waiter operations with a success state of error?

For waiters that treats a specific exception as the success state, there is really no final service response to return. Alternatively, we could introduce 
a `WaiterResponse` that contains the exception, but it seems overkill considering that the waiter operations with a desired state of error
are mostly associated with deleting resources and the use case for the exception is not very clear.

## References

Github feature request links: 
- [Waiters](https://github.com/aws/aws-sdk-java-v2/issues/24)
- [Async requests that complete when the operation is complete](https://github.com/aws/aws-sdk-java-v2/issues/286) 


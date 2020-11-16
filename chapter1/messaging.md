# Introducing a middleware

Middlewares are basically every software that is used as a medium, a
communication channel between different and decoupled components in an
architecture, message queues, message brokers, buses are effectively
middlewares.

Many times their used in microservices architectures, think about asynchronous
communication between different services, where you just want to schedule some
kind of job without worrying of the immediate response, `RabbitMQ` or `SQS` is
often used in these scenarios.<br>
In our case, we're going to add a really simple message queue interface to
decouple the crawling logic of the web crawler, this, along with decoupling
responsibilities, come with the benefit of a simpler testing of the entire
applcation.

## Producer and consumer

The most famous and simple abstraction is the producer-consumer pattern, where
two actors are involved:

* The producer, generate traffic
* The consumer, consume the traffic

It's a simplification of the pub-sub pattern, but can be easily extended and
behave just like it, with multiple subscribers consuming the same source.

> *The bigger the interface, the weaker the abstraction*<br>
> ***Rob Pike***

Go is conceptually a minimalist-oriented language, or at least that's how I
mostly perceive it, so the best practice is to maintain the interfaces as
little as possible, and it really makes sense as every interface defines an
enforcement, a contract that must be fulfilled, the less you have to implement
to be compliant the better.

For our application it's not strictly required to follow the rule, but for the
sake of readability and extensibility for future additions we'll stick to the
rule adding three generic communication interfaces:

**messaging/queue.go**

```go
// Package messaging contains middleware for communication with decoupled
// services, could be RabbitMQ drivers as well as kafka or redis
package messaging

// Producer defines a producer behavior, exposes a single `Produce` method
// meant to enqueue an array of bytes
type Producer interface {
	Produce([]byte) error
}

// Consumer defines a consumer behavior, exposes a single `Consume` method
// meant to connect to a queue blocking while consuming incoming arrays of
// bytes forwarding them into a channel
type Consumer interface {
	Consume(chan<- []byte) error
}

// ProducerConsumer defines the behavior of a simple message queue, it's
// expected to provide a `Produce` function a `Consume` one
type ProducerConsumer interface {
	Producer
	Consumer
}

// ProducerConsumerCloser defines the behavior of a simple mssage queue
// that requires some kidn of external connection to be managed
type ProducerConsumerCloser interface {
	ProducerConsumer
	Close()
}
```

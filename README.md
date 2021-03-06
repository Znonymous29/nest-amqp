# Nest AMQP 1.0 Module

[![Build Status](https://api.travis-ci.org/team-supercharge/nest-amqp.svg?branch=master)](https://travis-ci.org/github/team-supercharge/nest-amqp)
[![Coverage Status](https://codecov.io/github/team-supercharge/nest-amqp/coverage.svg?branch=master)](https://codecov.io/github/team-supercharge/nest-amqp)
<a href="https://www.npmjs.com/@team-supercharge/nest-amqp" target="_blank"><img src="https://img.shields.io/npm/v/@team-supercharge/nest-amqp.svg" alt="NPM Version" /></a>
<a href="https://www.npmjs.com/@team-supercharge/nest-amqp" target="_blank"><img src="https://img.shields.io/npm/l/@team-supercharge/nest-amqp.svg" alt="Package License" /></a>

## Description

AMQP 1.0 module for [Nest](https://github.com/nestjs/nest). It is based on the 
[rhea-promise](https://www.npmjs.com/package/rhea-promise) package. This package is for
working with a message broker with extra data validation functionality. It uses the
[class-transformer](https://www.npmjs.com/package/class-transformer) and 
[class-validator](https://www.npmjs.com/package/class-validator) packages to transform
and validate the payload of the messages. With this we can easily verify the message's
payload structure and its data types with Data Transfer Objects. Besides, it gives 
control over the message how to handle it: accept, reject or release.

## Installation

```bash
$ npm install --save @team-supercharge/nest-amqp
```

To use the library, the peer dependencies must also be installed:
```bash
$ npm install --save class-transformer class-validator
```

## Usage

In the subsections you can see how to send and receive messages, how to handle message transfer and how to use DTO classes to message 
payload transformation and validation.

### Create connection

To create a connection, you have to import the `QueueModule.forRoot()` module into your application's root module. The `forRoot()`
static method has 2 parameters: the first and required is the connection URI string, and the second is the *optional* connection 
options object. The connection options object extends the [Rhea's connection options](https://www.npmjs.com/package/rhea#connectoptions) 
and accepts these new properties:
* **throwExceptionOnConnectionError**?: A boolean value. If it's `true` then AMQPModule will throw forward the exception which occurs 
  during the connection creation. Default value is `false`.
* **acceptValidationNullObjectException**?: A boolean value. If it's `true` then AMQPModule will accept the message when a
  `ValidationNullObjectException` error was thrown. (ValidationNullObjectException will be thrown when message body is null). Otherwise the
  message will be rejected on `ValidationNullObjectException` error. Default value is `false`.

> Note: the AMQPModule package does not support multiple connection!

```typescript
import { Module } from '@nestjs/common';
import { QueueModule } from '@team-supercharge/nest-amqp';

@Module({
  imports: [
    QueueModule.forRoot(
      'amqp://user:password@localhost:5672', 
      { 
        throwExceptionOnConnectionError: true
      }
    ),
    // ...
  ],
})
export class AppModule {}
```

### Send a message

You can send messages with the `QueueService`. First you have to inject the service instance in the constructor, then you can use it to
send a message. Here is an example:

```typescript
import { Injectable } from '@nestjs/common';
import { QueueService } from '@team-supercharge/nest-amqp';

@Injectable()
export class TestService {
  constructor(private readonly queueService: QueueService) {}

  public async sendMessage(): Promise<void> {
    const payload = { text: 'hello world', date: new Date() };
    await this.queueService.send<TestDto>('example', payload);
  }
}
```

In the example we send a message to the `example` queue with an object payload. The `QueueService` will stringify the object and send the 
message with the stringified object as payload. You can add a DTO class as generic type to the `send<T>()` method and TypeScript will 
check that the second argument is matches with the DTO class.

> Note: the `send<T>()` method won't validate and transform the payload, it only stringifies the payload to a string value and send objects
> as string payload.

The `send<T>()` method's third parameter is the options which is an optional object. You can set here the 
[rhea connection options](https://github.com/amqp/rhea/blob/2cbe55795b86a43375111f2f446089f55b948ecf/typings/connection.d.ts#L525)
as well. Other custom available options:
* **schedule.divideMinute**?: Send the message multiple times given by this number.
* **schedule.afterSeconds**?: Message sending delay in seconds.

### Listen to a queue

If you want to receive the messages which arrive at a queue then you have to listen on the specific queue. You can do this with the
`@Listen()` decorator, what will you add to a class method which has `@Injectable()` decorator. Here is an example:

```typescript
@Injectable()
export class ExampleListener {
  @Listen('example', { type: PayloadDto })
  public async listenForQueueMessages(data: PayloadDto, control: MessageControl): Promise<void> {
    console.log('new message arrived on the "example" queue:', data);
  }
}
```

The `@Listen()` decorator's first parameter is the name of the queue, which we are listening to. When a new message arrives, this method
will be invoked with 2 arguments in order: message payload and message control. The message payload is a Data Transfer Object (DTO) 
class's instance. The task of this DTO class is to describe the payload's data structure for validation and transformation. We do this with
decorators. The second argument is the message control object which gives control how to handle the message transfer. 

The `@Listen()` decorator has a second argument which is the options object. The options object can have these **optional** properties:
* **type**?: A class reference which is decorated with [class-transformer](https://www.npmjs.com/package/class-transformer) and
  [class-validator](https://www.npmjs.com/package/class-validator) decorators. The data validation and transformation will be done
  with this class.
* **noValidate**?: if it is `true` then the message payload won't be validated and transformed by the `type` property's value. The
  default value is `false`.
* **parallelMessageProcessing**?: The number of messages to be processed at one time. The default value is `1`.
* **transformerOptions**?: [transformation options](https://github.com/typestack/class-transformer/blob/develop/src/interfaces/class-transformer-options.interface.ts)
* **validatorOptions**?: [validation options](https://github.com/typestack/class-validator#passing-options)

### Message control

When a new message arrives at a queue, the assigned method with `@Listen()` decorator receives the transformed and validated message body
and the message control. The latter object is to control the message transfer. It is possible to accept, reject or release the transfer. 
Here are the examples:

```typescript
// accept the message
control.accept();

// reject the message
control.reject('processing failed');

// release the message
control.release();
```

Use `accept` when message has been handled normally. It will remove the message from the queue.
Use `reject` when message was unprocessable. It contained either malformed or semantically incorrect data. In other words
it can't be successfully processed in the future without modifications. It will remove the message from the queue.
Use `release` when a temporary problem happened during message handling, e.g. could not save record to DB, 3rd party service
errored, etc. The message is not malformed and theoretically can be processed at a later time without modifications. The
message will not be removed from the queue but will be processed by another consumer.

> If the message was not handled manually and the method with `@Listen()` decorator executed successfully then the message
> will be accepted. If the message was not handled manually and there was an `Error` or Exception then the message will be rejected 
> automatically.

### Payload validation and transformation

The module provides an opportunity for add validation and transformation process to the message payload. With validation, we can check
that the payload has the expected properties with valid values. With transformation, we can set what properties are important for us
and what properties should be skipped during the transformation. Here is an example DTO class:

```typescript
import { Exclude, Expose, Transform } from 'class-transformer';
import { IsIn, IsDateString, IsString } from 'class-validator';

@Exclude()
export class LogEntryDto {
  @Expose()
  @IsString()
  public readonly message: string;

  @Expose()
  @IsDateString()
  @Transform(value => new Date(value), { toClassOnly: true })
  public readonly date: Date;

  @Expose()
  @IsIn([1, 2, 3, 4, 5])
  public readonly level: number;

  constructor(props: Partial<LogEntryDto>) {
    Object.assign(this, props);
  }
}
```

The `@Expose()` decorator says that after the transformation, the properties with this decorator should stay in the `LogEntryDto` class
instance. The DTO class has a `@Exclude()` decorator which means that the properties that are not described in the class definition or
don't have `@Expose()` decorator will be removed during the transformation from the instance. With these 2 decorators we can specify which
properties want we from the payload when the message arrives at the consumer side. The `@Transform()` decorator is to manually cast or 
transform a value during the transformation process. In the example we can see that when the message arrives, the body will contain a 
JSON object which is converted to string. The stringified JSON object has only primitive values so the `date` property will be a string 
date. We want it as a Date object instance, so we manually transform it into a Date object with the `@Transform()` decorator. We have to 
notice that only when the payload will be transformed to the DTO object (i.e. when the message arrives at the consumer side) and not when
the object will be transformed to a string (i.e. before the sending on the producer side), because the DTO object will be represented as 
a string in the message payload. The other decorators are to validate the property values.

An object payload's lifetime during the whole sending-receiving process:
```
                                        send                               receive
Message's way:     sender (producer)  --------->  queue (message broker)  --------->  receiver (consumer)
                                      transform                           transform
Payload's format:   DTO instance or   --------->          string          --------->     DTO instance
                   JavaScript object
```

You can see that the sender sends a DTO object instance or a plain object which will be converted to string and the message broker 
receives this string as message body. When the consumer gets the message, the body is a string and the transformation process will 
transform it into a DTO instance.
Only one thing left: set this DTO class as message payload:

```typescript
@Listen('logsQueue', { type: LogEntryDto })
public async listenForLogsQueueMessages(data: LogEntryDto): Promise<void> {
  console.log('log entry:', data);
}
```

You can send any other primitive values as message payload but in this case you have to take care to disable the validation and 
transformation process. Here is the example:

```typescript
@Listen('stringQueue', { noValidate: true })
public async listenForStringQueueMessages(data: string): Promise<void> {
  console.log('message payload:', data);
}
```

### Module logs

The module uses [debug](https://www.npmjs.com/package/debug) package for logging. If you start the Nest application with the `DEBUG` 
environment variable, then you can set the level of logging:

```bash
# log everything
$ DEBUG=nest-amqp:* nest start

# log only the errors
$ DEBUG=nest-amqp:error:* nest start
```

### Shutdown

If you stop your application, then Nest will wait for the end of the currently processing messages and will exit after the processes 
finished.

## Example

First you have to import the Queue module into the app module. The `QueueModule.forRoot()` method's first parameter 
is the connection URI for the message broker server:

> Note: the `QueueModule.forRoot()` can be added only to the application's root module and only once because multiple
> message broker connections are not supported!

```typescript
import { QueueModule } from '@team-supercharge/nest-amqp';
// ...

@Module({
  imports: [
    QueueModule.forRoot('amqp://user:password@localhost:5672'),
    // ...
  ],
})
export class AppModule {} 
```

Then create a `user.module.ts` feature module what will give all the functionality which belongs to the users.

> Note: the `QueueModule.forFeature()` module must be imported to each feature module of the application. 

```typescript
import { Module } from '@nestjs/common';
import { QueueModule } from '@team-supercharge/nest-amqp';

@Module({
  imports: [QueueModule.forFeature()],
  controllers: [],
  providers: [],
})
export class UserModule {}
```

Import this `UserModule` in the `AppModule`.

After that create `user.dto.ts` file and add a data transfer object (DTO) class to it, which will be sent as
body in the queue message. The [class-transformer](https://www.npmjs.com/package/class-transformer) package 
is used to remove or not touch the object properties before send it to the queue and the
[class-validator](https://www.npmjs.com/package/class-validator) package is used to validate the received object
on the consumer side:

```typescript
import { Expose } from 'class-transformer';
import { IsInt, IsString } from 'class-validator';

@Expose()
export class AddUserDto {
  @IsString()
  public readonly name: string;

  @IsInt()
  public readonly age: number;

  constructor(userData: AddUserDto) {
    Object.assign(this, userData);
  }
}
```

After that create the `user.listener.ts` file and add  a new listener class to it which has a method 
with the `@Listen()` decorator to listen the specified queue's messages:

> Note: you can add the `@Listen()` decorator for any class which have `@Injectable()` decorator.

```typescript
import { Injectable } from '@nestjs/common';
import { Listen, MessageControl } from '@team-supercharge/nest-amqp';

import { AddUserDto } from './user.dto';

@Injectable()
export class UserListener {
  @Listen('addUser', { type: AddUserDto })
  public async listenForQueueNameMessages(data: AddUserDto, control: MessageControl): Promise<void> {
    console.log('new message arrived on the "addUser" queue:', data);
    control.accept();
  }
}
```

Then we create a `user.controller.ts` file and add a HTTP endpoint which will send the message 
to the queue with the payload what it gets as HTTP body:

```typescript
import { Body, Controller, Post } from '@nestjs/common';
import { QueueService } from '@team-supercharge/nest-amqp';

import { AddUserDto } from './user.dto';

@Controller('user')
export class UserController {
  constructor(private readonly queueService: QueueService) {}

  @Post()
  public async sendAddUserMessage(@Body() body: AddUserDto): Promise<string> {
    await this.queueService.send<AddUserDto>('addUser', body);

    return 'Add user message sent';
  }
}
```

We can see that the `send()` method is responsible to add a message to the given queue. 

The last thing is to add this controller to the corresponding module:

```typescript
import { UserController } from './user.controller';
import { UserListener } from './user.listener';
// ...

@Module({
  controllers: [UserController],
  providers: [UserListener],
})
export class UserModule {}
```

Finally, start the app with `npm run start` and it will listen on http://localhost:4444 URL. You can test the
functionality with sample HTTP requests which are in the `example/http-requests/add-user.http` file.

## License

@team-supercharge/nest-amqp is [MIT licensed](LICENSE).

# RabbitMQ-note
`ref: https://www.rabbitmq.com/tutorials/tutorial-one-php.html`

> producer ->  queue  -> consumer

- A producer is a user application that sends messages.
- A queue is a buffer that stores messages.
- A consumer is a user application that receives messages.

## How to use
不管是產生(producer)訊息還是處理(consumer)訊息，都需要建立rabbitmq連線、通道、宣吿要用哪個queue:
```php
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();
$channel->queue_declare('queue_name', false , false , false , false);
```

producer: 產生訊息丟進queue, [send.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/send.php)
```php
$msg = new AMQPMessage('Hello World!');
// 發布訊息
$channel->basic_publish($msg, '', 'queue_name'); 
// Here we use the default or nameless exchange: 
// messages are routed to the queue with the name specified by 'routing_key', if it exists. The 'routing_key' is the third argument to basic_publish
```

consumer: 處理queue中的訊息, [receive.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/receive.php)
```php
$callback = function ($msg) {
  echo ' [x] Received ', $msg->body, "\n";
};
// 處理訊息
$channel->basic_consume('queue_name', '', false, true, false, false, $callback);

while ($channel->is_consuming()) {
    $channel->wait();
}
```

最後關閉連線:
```php
$channel->close();
$connection->close();
```

### Message durability
確保訊息不會因為Rabbitmq server掛掉導致訊息丟失，還是有可能出意外但也夠用了，要更嚴謹的話去看 [publisher confirms](https://www.rabbitmq.com/confirms.html)
```php
// we need to declare it as durable. To do so we pass the third parameter to queue_declare as true:
$channel->queue_declare('task_queue', false, true, false, false);

// mark our messages as persistent - by setting the delivery_mode = 2 message property which AMQPMessage takes as part of the property array.
$msg = new AMQPMessage(
    $data,
    array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
);
```

### Fair dispatch
盡量平均分配訊息, 設定 prefetch_count=1 表示等consumer裡的訊息處理完並回傳ack後，才會dispatch新的訊息給這個consumer處理
```php
// we can use the basic_qos method with the prefetch_count = 1 setting. This tells RabbitMQ not to give more than one message to a worker at a time.
$channel->basic_qos(null, 1, null);
```


### Exchanges
在producer跟queue中間，有點像route的功能。exchange type決定了訊息要傳送到哪個queue
 - direct
 - topic
 - headers
 - fanout
 
```php
// 宣告一個 fanout exchange: it just broadcasts all the messages it receives to all the queues it knows.
$channel->exchange_declare('exchange_name', 'fanout', false, false, false);
$channel->basic_publish($msg, 'exchange_name');
```
### Bindings
把 exchange 跟 queue 建立關係
```php
// Now we need to tell the exchange to send messages to our queue. That relationship between exchange and a queue is called a binding.
$channel->queue_bind($queue_name, 'exchange_name');
```

## keywords

**channel**

``` which is where most of the API for getting things done resides. ```

**queue declare**

```Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.```


**Message acknowledgment**

```An ack(nowledgement) is sent back by the consumer to tell RabbitMQ that a particular message has been received, processed and that RabbitMQ is free to delete it.```

 




# RabbitMQ-note
> ref: https://www.rabbitmq.com/tutorials/tutorial-one-php.html

`producer ->  queue  -> consumer`

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
// messages are routed to the queue with the name specified by 'routing_key', if it exists. 
// The 'routing_key' is the third argument to basic_publish
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
// we can use the basic_qos method with the prefetch_count = 1 setting. 
// This tells RabbitMQ not to give more than one message to a worker at a time.
$channel->basic_qos(null, 1, null);
```


### Exchanges & Bindings
把 exchange 跟 queue 建立關係
```php
// Now we need to tell the exchange to send messages to our queue. 
// That relationship between exchange and a queue is called a binding.
$channel->queue_bind($queue_name, 'exchange_name');
```

* exchange type: fanout, direct, topic, headers
#### fanout
It just broadcasts all the messages it receives to all the queues it knows.
```php
// emit.php
$channel->exchange_declare('exchange_name', 'fanout', false, false, false);
$channel->basic_publish($msg, 'exchange_name');
// receive.php
$channel->queue_bind($queue_name, 'exchange_name');
```
#### direct
A message goes to the queues whose binding key exactly matches the routing key of the message.
```php
// emit.php
$channel->exchange_declare('direct_logs', 'direct', false, false, false);
$channel->basic_publish($msg, 'direct_logs', $routing_key);
// receive.php
$channel->queue_bind($queue_name, 'direct_logs', $binding_key);
```
#### topic
Messages sent to a topic exchange can't have an arbitrary routing_key - it must be a list of words, delimited by dots.
```
* (star) can substitute for exactly one word.
# (hash) can substitute for zero or more words.
```
> Topic exchange is powerful and can behave like other exchanges.

When a queue is bound with "#"(hash) binding key - it will receive all the messages, regardless of the routing key - like in fanout exchange.
When special characters "\*"(star) and "#"(hash) aren't used in bindings, the topic exchange will behave just like a direct one.
[see tutorial](https://www.rabbitmq.com/tutorials/tutorial-five-php.html)



### RPC (Remote procedure call)
```
producer ->  queue  -> consumer
↑(client)              ↓(server)
         <-  queue  <-  
```
需要另外的參數讓訊息返回client

#### Message properties
建立訊息的時候可以帶入的常用參數
```
correlation_id: Useful to correlate RPC responses with requests.
delivery_mode:  Marks a message as persistent (with a value of 2) or transient (1). You may remember this property from the second tutorial.
content_type:   Used to describe the mime-type of the encoding. For the often used JSON encoding it's a good practice to set this property to: application/json.
reply_to:       Commonly used to name a callback queue.

```
[rpc_client.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/rpc_client.php)
```php
//  In order to receive a response we need to send a 'callback' queue address with the request.
list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$msg = new AMQPMessage(
    (string) $n,
    array(
        'correlation_id' => $this->corr_id, // which is set to a unique value for every request
        'reply_to' => $this->callback_queue // which is set to the callback queue
    )
);

$channel->basic_publish($msg, '', 'rpc_queue');
```
[rpc_server.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/rpc_server.php)
```php
$callback = function ($req) {
    $n = intval($req->body);
    echo ' [.] fib(', $n, ")\n";

    $msg = new AMQPMessage(
        (string) fib($n),
        array('correlation_id' => $req->get('correlation_id'))
    );

    $req->delivery_info['channel']->basic_publish(
        $msg,
        '',
        $req->get('reply_to')
    );
    $req->delivery_info['channel']->basic_ack(
        $req->delivery_info['delivery_tag']
    );
};

// We use basic_consume to access the queue.
$channel->basic_consume('rpc_queue', '', false, false, false, false, $callback);

// Then we enter the while loop in which we wait for request messages, do the work and send the response back.
while ($channel->is_consuming()) {
    $channel->wait();
}
```



## keywords

**channel**

``` which is where most of the API for getting things done resides. ```

**queue declare**

```Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.```


**Message acknowledgment**

```An ack(nowledgement) is sent back by the consumer to tell RabbitMQ that a particular message has been received, processed and that RabbitMQ is free to delete it.```


 
 test for Personal access tokens




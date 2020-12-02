# RabbitMQ-note
producer -> queue -> consumer


## keywords

**channel**

``` which is where most of the API for getting things done resides. ```

**queue declare**

```Declaring a queue is idempotent - it will only be created if it doesn't exist already. The message content is a byte array, so you can encode whatever you like there.```


**Message acknowledgment**


**Exchanges**
 - direct
 - topic
 - headers
 - fanout
 
**Bindings**

**fanout**

https://medium.com/better-programming/implementing-rabbitmq-with-node-js-93e15a44a9cc

#First install the following library.
npm install amqplib --save

#Create the following file services/MQService.js

```javascript
import amqp from 'amqplib/callback_api';
const CONN_URL = 'amqp://gsgmnvnl:NITe9ThLkXQvKVLl7L6gEtMllb6obQmw@dinosaur.rmq.cloudamqp.com/gsgmnvnl';
```
We can use the connect() method provided by AMQP to create a connection. 

```javascript
let ch = null;
amqp.connect(CONN_URL, function (err, conn) {
   conn.createChannel(function (err, channel) {
      ch = channel;
   });
});

```

Then let’s create a method to send messages to the queue which implements the sendToQueue method.

```javascript
export const publishToQueue = async (queueName, data) => {
   ch.sendToQueue(queueName, new Buffer(data));
}

```
#The MQService file looks like:
```javascript

import amqp from 'amqplib/callback_api';
const CONN_URL = 'amqp://gsgmnvnl:NITe9ThLkXQvKVLl7L6gEtMllb6obQmw@dinosaur.rmq.cloudamqp.com/gsgmnvnl';
let ch = null;
amqp.connect(CONN_URL, function (err, conn) {
   conn.createChannel(function (err, channel) {
      ch = channel;
   });
});
export const publishToQueue = async (queueName, data) => {
   ch.sendToQueue(queueName, new Buffer(data));
}
process.on('exit', (code) => {
   ch.close();
   console.log(`Closing rabbitmq channel`);
});

```
#Let’s add another route in our routes/User.js file. First let us import the publishToQueue method we created in the service.

import {publishToQueue} from '../services/MQService';

Then add the following route method.

```javascript

router.post('/msg',async(req, res, next)=>{
  let { queueName, payload } = req.body;
  await publishToQueue(queueName, payload);
  res.statusCode = 200;
  res.data = {"message-sent":true};
  next();
})

```

Create a file like worker-1.js in the project.
Write the following code in the worker file.

```javascript
var amqp = require('amqplib/callback_api');
const CONN_URL = 'amqp://gsgmnvnl:NITe9ThLkXQvKVLl7L6gEtMllb6obQmw@dinosaur.rmq.cloudamqp.com/gsgmnvnl';
amqp.connect(CONN_URL, function (err, conn) {
  conn.createChannel(function (err, ch) {
    ch.consume('user-messages', function (msg) {
      console.log('.....');
      setTimeout(function(){
        console.log("Message:", msg.content.toString());
      },4000);
      },{ noAck: true }
    );
  });
});

```

#If we set the noAck field as true, then the queue will delete the message the moment it is read from the queue.

In a real life scenario however, the consumer might crash while doing some operation. In that scenario, we want the message to go back to the queue, so that the message can be consumed by another worker, or by the same worker when we spawn it again.
To achieve this, we have to do a couple of things.
The first thing to do, is to change the third parameter value in the consume method. noAck : false
As we have set the noAck as false, we have to explicitly call the channel.ack() method. Now our consume method looks like:

```javascript

ch.consume('user-messages', function (msg) {
    console.log('.....');
    setTimeout(function(){
      console.log("Message:", msg.content.toString());  
      ch.ack(msg);
    },8000);
  },
  { noAck: false }
);

```


#What if our RabbitMQ instance went down? How do we ensure that the payload is not lost?

We have to declare our queues to be durable, which we’ve done already in the RabbitMQ admin console.
Secondly, we have to add a third parameter to the sendToQueue method in our producer code.

{persistent: true}

So the function looks like:
```javascript
export const publishToQueue = async (queueName, data) => {
  ch.sendToQueue(queueName, new Buffer(data), {persistent: true});
}
```

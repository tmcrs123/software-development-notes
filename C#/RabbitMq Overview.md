
## What's RBMQ?

Rabbit is way of decoupling applications. Technically is an implementation of AMQP (Advanced Message Queueig Protocol). It allows you to share information (in the form of messages) between applications.

## Building blocks

- Producer - Who produces the message and sends it to the queue
- Consumer - Who listens to the queue and processes messages. It's also who acknowledges a message as been received
- Queue - This is the buffer where messages are stored temporarily until they are retrieved by a consumer. Queues are by default LIFO.
- Exchange - This is the entity that routes messages. You can have more than one queue and it's the exchange job to route them to the appropriate queue based on a routing key. There are several types of exchanges:
	- Direct: route messages with a specific key
	- Fanout: route messages to all bound queues (like SNS)
	- Topic: route messages based on pattern matching between the routing key and the binding
	- Headers: route messages based on header attributes
- Binding - is the relationship between an exchange and a queue.
- Routing key - the routing key used to determine which queue to use
- Message - the date being shared
- Acknowledgments - what consumers send back to rabbit to tell a message has been successfully processed
- Dead Letter Exchange - where message end up if they cannot be processed due to expiration or rejection

## Secondary concepts
### Expiration/Rejection

It means what it says on the box but it's good to know that you can define expiration at the message level (meaning a message has a specific TTL) or you can do it at queue level (meaning all messages in the queue have the same TTL)


## The most simple example possible

*note: assumes you have rabbit running on docker*

```C#
using Microsoft.AspNetCore.Mvc;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using System.Text;

namespace User_Service.Controllers
{
    public class RabbitController : BaseController
    {
        [HttpPost]
        public async Task CreateQueue()
        {
            var factory = new ConnectionFactory() { HostName = "localhost", Port = 5672 };

            using (var connection = await factory.CreateConnectionAsync())
            {
                using (var channel = await connection.CreateChannelAsync())
                {
                    await channel.QueueDeclareAsync("bananas", durable: false, exclusive: false, autoDelete: false, arguments: null);
                }
            }
        }

        [HttpPut]
        public async Task Publish()
        {
            var factory = new ConnectionFactory() { HostName = "localhost", Port = 5672 };

            using (var connection = await factory.CreateConnectionAsync())
            {
                using (var channel = await connection.CreateChannelAsync())
                {
                    string message = "I am banana";

                    var body = Encoding.UTF8.GetBytes(message);

                    await channel.BasicPublishAsync("", "bananas", body);
                    Console.WriteLine(" [x] Sent {0}", message);
                }
            }
        }


        [HttpPut]
        [Route("register")]
        public async Task Register()
        {
            this.RegisterConsumer();
        }

        [HttpGet]
        public async Task<IResult> ManualRead()
        {
            var factory = new ConnectionFactory() { HostName = "localhost", Port = 5672 };

            using (var connection = await factory.CreateConnectionAsync())
            {
                using (var channel = await connection.CreateChannelAsync())
                {
                    var getResult = await channel.BasicGetAsync("bananas", true);

                    var resString = Encoding.UTF8.GetString(getResult.Body.ToArray());

                    return Results.Ok(resString);
                }
            }
        }

        public async Task RegisterConsumer()
        {
            var factory = new ConnectionFactory() { HostName = "localhost", Port = 5672 };

            using (var connection = await factory.CreateConnectionAsync())
            {

                using (var channel = await connection.CreateChannelAsync())
                {
                    await channel.QueueDeclareAsync("bananas", durable: false, exclusive: false, autoDelete: false, arguments: null);


                    var consumer = new AsyncEventingBasicConsumer(channel);

                    consumer.ReceivedAsync += async (sender, ea) =>
                    {
                        var res = ea.Body.ToArray();
                        Console.WriteLine($"In consumer: {Encoding.UTF8.GetString(res)}");
                    };

                    await channel.BasicConsumeAsync("bananas", true, consumer);

                    Console.ReadLine();
                }
            }
        }
    }
}

```
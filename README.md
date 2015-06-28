# RabbitMQ on EC2

[RabbitMQ](http://www.rabbitmq.com/) is a fast and open-source
general-purpose message broker. The principal idea is pretty simple: it
accepts and forwards messages.

The following are quick instructions for setting up a 
RabbitMQ server on a *single* EC2 instance.
This means if the instance goes down, the RabbitMQ server will not 
respond. To prevent this, you can set up cluster nodes, which is not
covered here.

## Requirements
* A new EC2 instance
* The security group is open to the following incoming ports: 22, 5672,
  15672, 25672.
* (Optional) Set up and assign an Elastic IP Address
  [(EIP)](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
 to the EC2 instance.

## Instructions
These instructions need to be run only once per server.

### Install RabbitMQ
SSH into your EC2 instance and set up the RabbitMQ.
The following were borrowed from [here](https://www.rabbitmq.com/ec2.html).

```bash
echo "deb http://www.rabbitmq.com/debian/ testing main" | \
sudo tee /etc/apt/sources.list.d/rabbitmq.list

curl https://www.rabbitmq.com/rabbitmq-signing-key-public.asc -o \
/tmp/rabbitmq-signing-key-public.asc

sudo apt-key add /tmp/rabbitmq-signing-key-public.asc
rm /tmp/rabbitmq-signing-key-public.asc

sudo apt-get -qy update
sudo apt-get -qy install rabbitmq-server
```

### Configure server host mapping
Modify `/etc/hosts`:

```
123.12.1.123 ip-123-12-1-123
```

Where the IP address is the **private IP** of the EC2 instance.

You can find that out by typing: `hostname`.

### Create new user
Create a new user so remote hosts can access the server.
```
sudo rabbitmqctl add_user <user> <password>
sudo rabbitmqctl set_user_tags <user> administrator
sudo rabbitmqctl set_permissions -p / <user> ".*" ".*" ".*"

# Delete the default user.
sudo rabbitmqctl delete_user guest

sudo /etc/init.d/rabbitmq-server restart
```

Remote clients (e.g Celery) can now access access the RabbitMQ
server with the credentials `user:password`.

### Enable RabbitMQ management plugin
[The rabbitmq-management
plugin](https://www.rabbitmq.com/management.html)
provides management and monitoring services for the RabbitMQ server.

```
sudo rabbitmq-plugins enable rabbitmq_management

sudo /etc/init.d/rabbitmq-server restart
```

Now you can visit the RabbitMQ management page:
`http://<ec2-public-ip>:15672`

## Examples

### Celery
[Celery](http://www.celeryproject.org/) 
is an asynchronous task queue/job queue based multiple brokers including RabbitMQ.

`tasks.py`:
```py
from celery import Celery

RABBIT_URL = 'amqp://<user>:<password>@<ec2-public-ip>//'

celery = Celery(__name__, broker=RABBIT_URL, backend=RABBIT_URL)

@celery.task
def multiply(x, y):
    return x * y
```

Start the worker:

```celery -A tasks worker --loglevel=debug```

Call the tasks:
```
>>> import tasks

>>> res = tasks.multiply.delay(7, 191)
>>> res.get()
1337
```

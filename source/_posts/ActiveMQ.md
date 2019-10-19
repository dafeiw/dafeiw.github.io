---
title: ActiveMQ
date: 2019-07-27 00:01:58
tags:
- Java
- Web
categories: MOM
---

# Basic
```java
public class JMSQueueProducer {

    public static void main(String[] args) {
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://localhost:61616");
        Connection connection = null;
        try {
            connection = connectionFactory.createConnection();
            connection.start();
            Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
            Destination destination = session.createQueue("myQueue");
            MessageProducer producer = session.createProducer(destination);
            TextMessage message = session.createTextMessage("Hello World");
            producer.send(message);
            session.commit();
            session.close();
        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

You can also speicify null destination when creating producer.
```java
MessageProducer producer = session.createProducer(null);
...
producer.send(someDestination, message);
...
producer.send(anotherDestination, message);
```

It's better to create Connection, Session, MessageProducer, MessageConsumer upfront and then reuse them. 

<!--more-->

```java
public class JMSQueueReceiver {

    public static void main(String[] args) {
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://localhost:61616");
        Connection connection = null;
        try {
            connection = connectionFactory.createConnection();
            connection.start();
            Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
            Destination destination = session.createQueue("myQueue");
            MessageConsumer consumer = session.createConsumer(destination);
            TextMessage message = (TextMessage) consumer.receive();
            System.out.println(message.getText());
            session.commit();
            session.close();
        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

```java
public class JMSQueueListener {

    public static void main(String[] args) {
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://localhost:61616");
        Connection connection = null;
        try {
            connection = connectionFactory.createConnection();
            connection.start();
            Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
            Destination destination = session.createQueue("myQueue");
            MessageConsumer consumer = session.createConsumer(destination);

            MessageListener messageListener = new MessageListener() {
                public void onMessage(Message message) {
                    try {
                        System.out.println( ((TextMessage)message).getText());
                    } catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
            };

            while (true) {
                consumer.setMessageListener(messageListener);
                session.commit();
            }
//            session.close();
        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

* Message header
* Message Body
  * Text
  * Map
  * Bytes
  * Stream
  * Object

* Message properties


* Topics
  * Only subscribers who had an active subscription at the time the broker receives the message will get a copy of the message.

But activeMQ provides durable subscriber
```java
public class JMSPersistentTopicConsumer {

    public static void main(String[] args) {
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://localhost:61616");
        Connection connection = null;
        try {
            connection = connectionFactory.createConnection();
            connection.setClientID("Mic001");
            connection.start();
            // each producer or consumer can have individual session
            Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
            Topic destination = session.createTopic("myTopic");

            // JMS specification dictates that the identification of S is done by a combination of the clientID and the durable subscriber name. 
            MessageConsumer consumer = session.createDurableSubscriber(destination, "Mic001"); 
            TextMessage message = (TextMessage) consumer.receive();
            System.out.println(message.getText());
            session.commit();
            session.close();
        } catch (JMSException e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```


# Deliver Mode

Durability of messages is defined by the MessagerProducer. 
```java
public interface DeliveryMode {
    static final int NON_PERSISTENT = 1; // stored only in memory

    static final int PERSISTENT = 2;  // broker stores that message in a store on disk
}
```
```java
MessageProducer producer = ...;
producer.setDeliveryMode(DeliveryMode.PERSISTENT);
```

# Transaction and redelivery in JMS

In JMS, a transaction organizes a message or message group into an atomic processing unit; failure to deliver a message may result in redelivery of that message or message group.

* Acknowledgement options:
  * Auto mode: once-only message delivery guarantee
  * Duplicates okay mode: at-least-onece message delivery guarantee (acknowledged lazily so might be delivered more than once)
  * Client mode: 

* message processing phases
  * consumer receives message
  * consumer processes message
  * message is acknowledge

* session mechansim
  * transaction
    *
    ```java
            session.commit();
            session.rollback();
    ```
  * When operations are carried out on a transacted (or XA transacted) session, a transaction command is sent to the broker, with a unique transaction ID which is then followed by all the usual commands (send message, acknowledge message etc). When a commit() or rollback() is called on the Session, this command is sent to the broker for it to commit or rollback the transaction

* ActiveMQ supports sending messages to a broker in sync or async mode.

  * non-persistent delivery mode is sent asynchronously
  * persistent delivery mode and non-transactional is sent synchronously
  * transactional is sent asynchronously


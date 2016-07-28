JMS INPUT OPERATOR
=====================

### Introduction: About JMS Input Operator

The JMS input operator consumes data from a messaging system using the JMS client API. JMS not being a communication protocol, the operator needs an underlying JMS client API library to talk to a messaging system. Currently the operator has been tested with the Amazon SQS and Apache ActiveMQ System through their respective JMS client API libraries.

### Why is it needed ?

You will need the operator to read data from a messaging system (e.g. Apache ActiveMQ) via the JMS client API. The operator supports both the publish-subscribe (topics) and point-to-point (queues) modes. Although the operator is fault-tolerant, it currently does not support partitioning and dynamic scalability.

### AbstractJMSInputOperator

This abstract implementation serves as the base class for consuming generic messages from an external messaging system. Concrete subclasses implement conversion and/or the emit method to emit tuples for the concrete type. JMSStringInputOperator is one such subclass in the library used for String messages. JMSObjectInputOperator is another one used for multiple message types where the user has the ability to get String, byte array, Map or POJO messages on the respective output ports.

#### Configuration Parameters
Common configuration parameters are described here.
<table>
<col width="25%" />
<col width="75%" />
<tbody>
<tr class="odd">
<td align="left"><p>Parameter</p></td>
<td align="left"><p>Description</p></td>
</tr>
<tr class="even">
<td align="left"><p>windowDataManager</p></td>
<td align="left"><p>This is an instance of WindowDataManager that implements idempotency. Idempotency ensures that an operator will process the same set of messages in a window before and after a failure. For example, say the operator completed window 10 and failed before or during window 11. If the operator gets restored at window 10, it will replay the messages of window 10 which were saved from the previous run before the failure. Although important, idempotency comes at a price because an operator needs to persist some state at the end of each window. Default Value = org.apache.apex.malhar.lib.wal.FSWindowDataManager</p></td>
</tr>
<tr class="odd">
<td align="left"><p>connectionFactoryBuilder</p></td>
<td align="left"><p>The operator uses the builder pattern that requires the user to specify an instance of com.datatorrent.lib.io.jms.JMSBase.ConnectionFactoryBuilder. This builder creates the connection factory that encapsulates the underlying JMS client API library (e.g. ActiveMQ or Amazon SQS). By default the operator uses com.datatorrent.lib.io.jms.JMSBase.DefaultConnectionFactoryBuilder which is used for ActiveMQ. One of the examples below describes the Amazon SQS use-case. </td>
</tr>
</tbody>
</table>

#### Abstract Methods

The following abstract methods need to be implemented by concrete subclasses.

T convert(Message message): This method converts a JMS Message object to type T.

void emit(T payload): This method emits a tuple given the payload extracted from a JMS message.



### Concrete Classes

1.  JMSStringInputOperator :
This class extends AbstractJMSInputOperator to deliver String payloads in the tuple.

2.  JMSObjectInputOperator:
This class extends AbstractJMSInputOperator to deliver String, byte array, Map or POJO payloads in the tuple.

### Application Examples

#### ActiveMQ Example

The source code for the tutorial can be found here:

[https://github.com/DataTorrent/examples/tree/master/tutorials/jmsActiveMQ](https://github.com/DataTorrent/examples/tree/master/tutorials/jmsActiveMQ)

The default connectionFactoryBuilder supports ActiveMQ so there is no need to set this value. However the following ActiveMQ related values need to be set using the appropriate setter methods:

<table>
<col width="25%" />
<col width="75%" />
<tbody>
<tr class="odd">
<td align="left"><p>Value</p></td>
<td align="left"><p>Description</p></td>
</tr>
<tr class="even">
<td align="left"><p>connectionFactoryProperties</p></td>
<td align="left"><p>This is a Map of key and value strings and can be set directly from configuration as in the example above. The table below describes the most important properties.</p></td>
</tr>
<tr class="odd">
<td align="left"><p>topic</p></td>
<td align="left"><p>This boolean value is set to true for the publish-subscribe case and false for the PTP (point-to-point) case.</p></td>
</tr>
<tr class="even">
<td align="left"><p>subject</p></td>
<td align="left"><p>This is the queue name for PTP (point-to-point) use-case and topic name for the publish-subscribe use case.</p></td>
</tr>
<tr class="odd">
<td align="left"><p>durable</p></td>
<td align="left"><p>This boolean value is set to true for durable queues, false otherwise. Durable queues keep messages around persistently for any suitable consumer to consume them. Used only when the clientId (see below) is set.</p></td>
</tr>
<tr class="even">
<td align="left"><p>clientId</p></td>
<td align="left"><p>The client-ID for this ActiveMQ consumer in the durable (persistent queue) mode as described above.</p></td>
</tr>
<tr class="odd">
<td align="left"><p>transacted</p></td>
<td align="left"><p>This boolean value is set to true for transacted JMS sessions as described in 
<a href="https://docs.oracle.com/javaee/7/api/javax/jms/Session.html">Session</a>.</p></td>
</tr>
<tr class="even">
<td align="left"><p>ackMode</p></td>
<td align="left"><p>This string value sets the acknowledgement mode as described in 
<a href="https://docs.oracle.com/javaee/7/api/javax/jms/Session.html#field.summary">Session fields</a>.</p></td>
</tr>
</tbody>
</table>

The following table describes the string properties to be set in the map that is passed in the connectionFactoryProperties value described above.

<table>
<col width="25%" />
<col width="75%" />
<tbody>
<tr class="odd">
<td align="left"><p>Property Name</p></td>
<td align="left"><p>Description</p></td>
</tr>
<tr class="even">
<td align="left"><p>brokerURL</p></td>
<td align="left"><p>The <a href="http://activemq.apache.org/configuring-transports.html">connection URL</a> 
 used to connect to the ActiveMQ broker</p></td></tr>
<tr class="even">
<td align="left"><p>userName</p></td>
<td align="left"><p>The JMS userName used by connections created by this factory (optional)</p></td>
</tr>
<tr class="even">
<td align="left"><p>password</p></td>
<td align="left"><p>The JMS password used for connections created from this factory (optional)</p></td>
</tr>
</tbody>
</table>



#### SQS Example

The source code for the tutorial can be found here:

[https://github.com/DataTorrent/examples/tree/master/tutorials/jmsSqs](https://github.com/DataTorrent/examples/tree/master/tutorials/jmsSqs)

For SQS you will have to provide a custom connectionFactoryBuilder as shown in the example above and in [SQSConnectionFactory.java](https://github.com/awslabs/amazon-sqs-java-messaging-lib/blob/master/src/main/java/com/amazon/sqs/javamessaging/SQSConnectionFactory.java). The builder is typically used to supply AWS region and credential information that cannot be supplied via any JMS interfaces.

The following code snippet shows a typical Builder implementation that can be supplied to the operator. The AWS credentials are supplied via a [PropertiesFileCredentialsProvider](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/auth/PropertiesFileCredentialsProvider.html) object in which sqsCredsFilename is the fully qualified path to a properties file from which the AWS security credentials are to be loaded. For example `/etc/somewhere/credentials.properties`



```java
static class MyConnectionFactoryBuilder implements JMSBase.ConnectionFactoryBuilder {

String sqsCredsFilename;

MyConnectionFactoryBuilder()
{
}

@Override
public ConnectionFactory buildConnectionFactory() 
{
  // Create the connection factory using the properties file credential provider.
  // Connections this factory creates can talk to the queues in us-east-1 region. 
  SQSConnectionFactory connectionFactory =
    SQSConnectionFactory.builder()
      .withRegion(Region.getRegion(Regions.US_EAST_1))
      .withAWSCredentialsProvider(new PropertiesFileCredentialsProvider(sqsCredsFilename))
      .build();
    return connectionFactory;
  }
}
```

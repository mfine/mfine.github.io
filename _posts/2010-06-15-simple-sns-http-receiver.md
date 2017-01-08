---
layout: post
title: A Simple HTTP Receiver for Amazon's Simple Notification Service
---

{{ page.title }}
================

*15 June 2010*

Amazon's Simple Notification Service enables you to publish messages to subscribers in a scalable, cost-effective manner. Notifications can be delivered either to email or HTTP endpoints. This post helps get you started setting up embedded HTTP servers and processing notifications in Java using the [AWS SDK for Java](http://aws.amazon.com/sdkforjava/), the [Jetty HTTP Server](http://jetty.codehaus.org/jetty/), and the [Jackson JSON processor](http://jackson.codehaus.org/).

Publishing to Topics
--------------------

Let's look at the code for setting up a publisher. First, instantiate a SNS client with your AWS security credentials:

```java
// Create a client 
BasicAWSCredentials creds = new BasicAWSCredentials(ACCESS_KEY, SECRET_KEY)
AmazonSNSClient service = new AmazonSNSClient(creds);
```

Then create a topic named **"MyTopic"** with a CreateTopic request to the SNS client:

```java
// Create a topic
CreateTopicRequest createReq = new CreateTopicRequest()
   .withName("MyTopic");
CreateTopicResult createRes = service.createTopic(createReq);
```

Finally, publish a timestamped example message with a Publish request to the SNS client using the topic ARN returned from the CreateTopic result:

```java
// Publish to a topic
PublishRequest publishReq = new PublishRequest()
   .withTopicArn(createRes.getTopicArn())
   .withMessage("Example notification sent at " + new Date());
service.publish(publishReq);
```

That's it! You are now publishing messages to a topic!

Subscribing to Topics
---------------------

With a publisher up and running, let's now look at the code for setting up a receiver. First, subscribe a local HTTP endpoint with a SubscribeRequest to the SNS client using the topic ARN returned from the CreateTopic result:

```java
// Subscribe to topic
String address = InetAddress.getLocalHost().getHostAddress();
SubscribeRequest subscribeReq = new SubscribeRequest()
   .withTopicArn(createRes.getTopicArn())
   .withProtocol("http")
   .withEndpoint("http://" + address + ":" + port);
service.subscribe(subscribeReq);
```

Then instantiate and start a local HTTP server:

```java
// Create and start HTTP server
Server server = new Server(port);
server.setHandler(new AmazonSNSHandler());
server.start();
```

The local HTTP server's handler processes a HTTP request encoded in JSON by first scanning the request into a string:

```java
// Scan request into a string
Scanner scanner = new Scanner(request.getInputStream());
StringBuilder sb = new StringBuilder();
while (scanner.hasNextLine()) {
   sb.append(scanner.nextLine());
}
```

Then parsing the JSON request into a message map between fields and values:

```java
// Build a message map from the JSON encoded message
InputStream bytes = new ByteArrayInputStream(sb.toString().getBytes());
Map<String, String> messageMap = new ObjectMapper().readValue(bytes, Map.class);
```

Finally, enqueuing the message map to a local queue for the receiver and setting the HTTP response:

```java
// Enqueue message map for receive loop
messageQueue.add(messageMap);

// Set HTTP response
response.setContentType("text/html");
response.setStatus(HttpServletResponse.SC_OK);
((Request) request).setHandled(true);
```

With the local HTTP server started and enqueuing message maps to a local queue, let's now look at the code for the rest of the receiver. After receiving a message map on a local queue, look for a subscription confirmation token to confirm your subscription with a ConfirmSubscription request using the topic ARN returned from the CreateTopic result and the token:

```java
// Wait for a message from HTTP server
Map<String, String> messageMap = messageQueue.take();

// Look for a subscription confirmation Token
String token = messageMap.get("Token");
if (token != null) {

   // Confirm subscription
   ConfirmSubscriptionRequest confirmReq = new ConfirmSubscriptionRequest()
      .withTopicArn(createRes.getTopicArn())
      .withToken(token);
   service.confirmSubscription(confirmReq);
}
```

With your subscription confirmed, begin receiving the timestamped example messages published above:

```java
// Wait for a message from HTTP server
Map<String, String> messageMap = messageQueue.take();

// Check for a notification
String message = messageMap.get("Message");
if (message != null) {
   System.out.println("Received message: " + message);
}
```

That's it! You are subscribed to a topic and are now receiving messages from a topic!

You can now setup multiple publishers sending messages to multiple subscribers over HTTP endpoints using Java in a scalable, cost-effective manner. The complete code for both senders and receivers can be found at [AmazonSNSExample](http://github.com/mfine/AmazonSNSExample).

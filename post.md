## Motivation

You have a food ordering application. Your system is logging almost everything, your source code and configuration is version controlled. Somebody comes to you and says:

"We turned on a feature at 2PM five days ago, then we turned it off at 3PM. What was the actual configuration? How many users were impacted? What was the application version? How many of these users like to search for a phrase 'apple'? What do they like to buy? Can we recommend something for them?"

What do you need to do to answer these questions?<br>
If you are motivated enough, jump to [the idea](#the-idea) :)

**Visioning, developing, building, deploying, waiting, iterating**

In the world of IT, several roles impact an application's lifecycle. The product roles (product manager, chief, sales, project manager etc), the creator (developer, QA, UX etc), infrastructure roles and any mixture of these. 
These roles all require understanding what is going on in the application but they are all interested in a different or overlapping aspect.

From a product point of view we might be interested how the users are using our product, from sales point of view we will want to see what are they paying for, for developer point of view we would be probably more interested what happens in the system.

We also want to find a connection between application behaviour, user behaviour and infrastructure behaviour because a change in any of these impacts the others.

**Understanding our users**

To be more concrete let's assume our product has a UI which talks through an API to our server. If this is a web application or an app, and we are talking about understanding users, we probably end up including some analytics like Google Analytics or mouse click trackers. That works great, but how do we map this information to the state we have in our server or to a specific deployment version we have? We create some mechanism designed for that, a tool, or if we don't do that we usually just check it in our version control system.

**Understanding our system across iterations**

We have our current system. How do we check what happened 5 days ago? We probably go to our logs or the manager system sitting on top of it and filter out what we need. 
How do we know how many active users do we have currently and what are they doing? We probably check that in our database, session storage, or we send a push message to get some response. How do we check how many users tried out a feature which was turned on two months ago between 1PM and 2PM in a specific hotfix deployment? We probably go to our logs again, or to the aggregation manager designed for this, then we map it to our version controlled configuration and we also do a query in our database.

Lots of tools, tool managers, painful solutions sometimes and don't forget how many people you need to involve to answer these questions.

## The idea

The set of possible states of an application is a graph. 
When a set of events happens in our system there is an indicator, that creates a graph.
When a user travels across our application it creates a graph with its path.

Let's put all these graphs into one place! And here comes the key. Let's connect these graphs! 

### How?

To understand this let's define some (not all!) event groups which can happen across our application, across different abstraction levels.

**The user path events**<br>
A user is using our client app, and for the interaction with the server it calls the server's API (REST, SOAP, TCP, Websocket etc.). We can build a graph based on these events. Even when the API design is stateless we can assume the state of the client application as we created it (or we can have some kind of mapping mechanism).

**System run events**<br>
All kinds of events happening in our system belong here. Expected events because they are part of the application's functionality, unexpected events like errors, but also low level events like memory allocation. The level of abstraction depends on us. It feels like I am talking about the log level right? :)

**Application lifecycle events**<br>
Events like deploying, instantiating, destroying or even load balancing across the application.

So we are focusing on these event groups and the possible graph model they form. We will want to store these graphs and do queries on them. 

The powerful [Neo4j](http://neo4j.com/) database is perfect for this. Let's see an example!

### Building yet another food ordering application

Let's define the events first. We won't list all of them, but these should be enough. We will add some minimalistic list of properties which our event nodes would contain.

**Application lifecycle stage nodes**
<ui>
  <li style="padding-left:30px">START_NODE</li>
  <li style="padding-left:30px">DEPLOYING (timestamp, version, ticket)</li>
  <li style="padding-left:30px">DEPLOYMENT_ERROR (timestamp, message)</li>
  <li style="padding-left:30px">STARTING_ERROR (timestamp, message)</li>
  <li style="padding-left:30px">RUNNING (timestamp)</li>
  <li style="padding-left:30px">RUNTIME_ERROR (timestamp, message)</li>
  <li style="padding-left:30px">STOPPED (timestamp, message)</li>
</ui>

**The user event nodes**
<ui>
  <li style="padding-left:30px">USER_SESSION (timestamp, session_id)</li>
  <li style="padding-left:30px">USER_REGISTER (timestamp, username)</li>
  <li style="padding-left:30px">USER_LOGIN (timestamp, username)</li>
  <li style="padding-left:30px">USER_SEARCH (timestamp)</li>
  <li style="padding-left:30px">USER_PICK (timestamp)</li>
  <li style="padding-left:30px">USER_CHECKOUT (timestamp)</li>
  <li style="padding-left:30px">USER_PAY (timestamp)</li>
  <li style="padding-left:30px">USER_LOGOUT (timestamp)</li>
</ui>

**Some imaginary system runtime event nodes**
<ui>
  <li style="padding-left:30px">FEATURE_SWITCH (timestamp, ticket)</li>
  <li style="padding-left:30px">LOAD_ALERT (timestamp, message)</li>
  <li style="padding-left:30px">OUT_OF_STOCK (timestamp, item_id)</li>
  <li style="padding-left:30px">SCHEDULED_REPORT (timestamp, report_id)</li>
</ui>


Now let's jump into world of Neo4j and Cypher.

### Neo4j and Cypher

Neo4j has a query language called [Cypher](http://neo4j.com/developer/cypher-query-language/). Its usage is very intuitive and for our experiment we need just a running Neo4j instance and our Cypher queries.

In real life we would use a real [Neo4j](http://neo4j.com/download/) instance, but there is a super cool online [Neo4j console](http://console.neo4j.org/) which we can use to do our Cypher queries and share the database!

To get our feet wet, let's see how we create a new lifecycle stage node using Cypher.
```
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'DEPLOYING', timestamp: timestamp(), version:'v112'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);
```
We first find to which LifecycleStage the LifecyclePointer points, then we delete the pointer, we create a new LifecycleStage and we point the previous to it, and in the end we point to the latest stage. 

A user event does something similar, the difference is that every UserPointer contains the userID. If the user's visit is happening first time in this running instance, we have to connect the UserEvent to the RUNNING LifecycleStage. It looks like this:
```
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (ue:UserEvent {name:'USER_SESSION', timestamp: timestamp(), sessionID:'id1'})-[:USER_OF]->(stage);

MATCH (ue:UserEvent {sessionID:'id1'})
CREATE (up:UserPointer{sessionID:'id1'})-[:IS_CURRENT_UE]->(ue);
```

A FeatureSwitch uses the same pattern, but a new FeatureSwitch is connected to the RUNNING instance and to all the latest UserEvents.
```
CREATE (fs:FeatureSwitch{name:'FEATURE_SWITCH', featureName:'feature1', value:true})
CREATE (fp:FeaturePointer)-[:IS_CURRENT_FE]->(fs);

MATCH (fsl:FeaturePointer)-[:IS_CURRENT_FE]->(fs:FeatureSwitch)
MATCH (up:UserPointer)-[:IS_CURRENT_UE]->(ue:UserEvent)
CREATE (fs)-[:WILL_IMPACT]->(ue);

MATCH (fsl:FeaturePointer)-[:IS_CURRENT_FE]->(fs:FeatureSwitch)
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (fs)-[:RELATES_TO]->(stage);
```

These queries should be wrapped into transactions of course. Neo4j [supports them](http://neo4j.com/docs/stable/transactions.html) on node and relation level. 

### Now the story!

Let's assume the following scenario:

"We deploy version v111 of our application successfully, but cannot startup. Then we fix the build and do this again with version v112. Now it starts up and runs. A user connects, looks around, registers, logs in, explores, picks some items, checks out, pays. Another connects, starts exploring. We turn on a feature. One of the users searches. We turn off the feature. A new user connects."

Let's execute the Cypher queries reflecting this scenario and see what graph we get in our Neo4j instance!<br> If you are just interested in the outcome check out [this graph](http://console.neo4j.org/?id=gqddar) and jump to the [review](#let-s-review-what-we-got).
First the part where we deploy v111 of our app and get a STARTING_ERROR. 
```
//We point to the START_NODE in the start of the world
CREATE (current:LifecyclePointer)-[:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage {name:'START_NODE', timestamp: timestamp()});

//DEPLOYING
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'DEPLOYING', timestamp: timestamp(), version:'v111'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);

//DEPLOYED
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'DEPLOYED', timestamp: timestamp(), version:'v111'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);

//STARTING_ERROR
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'STARTING_ERROR', timestamp: timestamp(), version:'v111'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);
```
**Check it live: http://console.neo4j.org/r/6618ts**

Then we deploy and run a new version (v112) successfully, which means we grow a new graph from the START_NODE node.
```
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer;

MATCH (startline:LifecycleStage{name:'START_NODE'})
MATCH (current:LifecyclePointer)
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (startline);

MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'DEPLOYING', timestamp: timestamp(), version:'v112'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);

MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'DEPLOYED', timestamp: timestamp(), version:'v112'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);

MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'RUNNING', timestamp: timestamp(), version:'v112'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);
```
**Check it live: http://console.neo4j.org/?id=nkeg43**


A user(1) connects and interacts.
```
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (ue:UserEvent {name:'USER_SESSION', timestamp: timestamp(), sessionID:'id1'})-[:USER_OF]->(stage);
MATCH (ue:UserEvent {sessionID:'id1'})
CREATE (up:UserPointer{sessionID:'id1'})-[:IS_CURRENT_UE]->(ue);

MATCH (up:UserPointer{sessionID:'id1'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_SEARCH', timestamp: timestamp(), phrase:'searchphrase'})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);

MATCH (up:UserPointer{sessionID:'id1'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_PICK', timestamp: timestamp(), itemID:'item1', quantity:2})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);

MATCH (up:UserPointer{sessionID:'id1'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_PICK', timestamp: timestamp(), itemID:'item2', quantity:1})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);

MATCH (up:UserPointer{sessionID:'id1'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_PICK', timestamp: timestamp(), itemID:'item3', quantity:10})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);

MATCH (up:UserPointer{sessionID:'id1'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_CHECKOUT', timestamp: timestamp()})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);

MATCH (up:UserPointer{sessionID:'id1'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_PAY', timestamp: timestamp()})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);
```
**Check it live: http://console.neo4j.org/r/ylzlvy**


Another user(2) connects and starts searching.
```
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (ue:UserEvent {name:'USER_SESSION', timestamp: timestamp(), sessionID:'id2'})-[:USER_OF]->(stage);

MATCH (ue:UserEvent {sessionID:'id2'})
CREATE (up:UserPointer{sessionID:'id2'})-[:IS_CURRENT_UE]->(ue);

MATCH (up:UserPointer{sessionID:'id2'})-[last:IS_CURRENT_UE]->(ue:UserEvent)
DELETE last
CREATE (ue)-[:NEXT]->(ue2:UserEvent{name:'USER_SEARCH', timestamp: timestamp(), phrase:'searchphrase'})
CREATE (up)-[:IS_CURRENT_UE]->(ue2);
```
**Check it live: http://console.neo4j.org/r/4wtim4**


We turn on a feature.
```
CREATE (fs:FeatureSwitch{name:'FEATURE_SWITCH', featureName:'feature1', value:true})
CREATE (fp:FeaturePointer)-[:IS_CURRENT_FE]->(fs);

MATCH (fsl:FeaturePointer)-[:IS_CURRENT_FE]->(fs:FeatureSwitch)
MATCH (up:UserPointer)-[:IS_CURRENT_UE]->(ue:UserEvent)
CREATE (fs)-[:WILL_IMPACT]->(ue);

MATCH (fsl:FeaturePointer)-[:IS_CURRENT_FE]->(fs:FeatureSwitch)
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (fs)-[:RELATES_TO]->(stage);
```
**Check it live: http://console.neo4j.org/r/vjuish**


User(1) searches for something then we turn off a feature.
```
CREATE (fs:FeatureSwitch{name:'FEATURE_SWITCH', featureName:'feature1', value:true})
CREATE (fp:FeaturePointer)-[:IS_CURRENT_FE]->(fs);

MATCH (fsl:FeaturePointer)-[:IS_CURRENT_FE]->(fs:FeatureSwitch)
MATCH (up:UserPointer)-[:IS_CURRENT_UE]->(ue:UserEvent)
CREATE (fs)-[:WILL_IMPACT]->(ue);

MATCH (fsl:FeaturePointer)-[:IS_CURRENT_FE]->(fs:FeatureSwitch)
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (fs)-[:RELATES_TO]->(stage);
```
**Check it live: http://console.neo4j.org/r/26iiqj**

User(3) connects.
```
MATCH (fp:FeaturePointer)-[lastfe:IS_CURRENT_FE]->(fs)
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:LifecycleStage)
CREATE (ue:UserEvent {name:'USER_SESSION', timestamp: timestamp(), sessionID:'id3'})-[:USER_OF]->(stage)
CREATE (fs)-[:WILL_IMPACT]->(ue);
MATCH (ue:UserEvent {sessionID:'id3'})
CREATE (up:UserPointer{sessionID:'id3'})-[:IS_CURRENT_UE]->(ue);

```
**Check it live: http://console.neo4j.org/r/gqddar**


## Let's review what we got

This is just a rough example, but by looking to the resulting graph, we see clearly if a feature switching impacts a user, and when.

By going through each user's steps we can understand how they are using our application and we can also easily check what items they are interested in, so we can apply some recommendation algorithms.

By following the connection between the deployment and the user or feature switch, we can easily decide what was the version or what was the configuration at a particular time.

By checking the timestamps we can easily measure the latencies relevant to us.

**All this can be done using simple Cypher queries!**

## Next steps
I don't really see people logging their system and user events into graphs, but I think it is definitely worth doing further experiments in this field as it can result great patterns and frameworks.

**Thinking in aspects**

Persisting events is definitely something injected horizontally across our application. We should create our own aspects for particular levels, and event groups we need. Executing them should be as lightweight as possible.

**The levels of tracking**

The graph model should reflect the application's functionality, but we should not be afraid of persisting low level events if it helps us understanding our application better! 

**Using our existing log base to generate graphs**

If our logs are detailed enough, we can easily generate graphs for the past! So technically, it is possible to build an application graph without touching our current application. 

**Talk with our colleagues**

It is interesting to see what information others want to get from our "application graph" before we start modelling it.

**Please share your thoughts!**
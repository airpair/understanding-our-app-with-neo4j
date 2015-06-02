## Visioning, developing, building, deploying, waiting, iterating

In the world of IT, several roles impact an application's lifecycle. The product roles (product manager, chief, sales, project manager etc), the creator (developer, QA, UX etc), infrastructure roles and any mixture of these. 
These roles all require understanding what is going on in the application but they are all interested in a different or overlapping aspect.

In a product point of view we might be interested how the users are using our product, in sales point of view we will want to see what are they paying for, in developer point of view we would be probably more interested what happens in the system.

We also want to learn from the previous states, and know exactly what is the current state.

### Understanding our users

To be more concrete let's assume our product has a UI which talks through an API to our server. If this is a web application or an app, and we are talking about understanding users, we probably end up including some analytics like Google Analytics or mouse click trackers. That works great, but how do we map this information to the state we have in our server or to a specific deployment version we have? We create some mechanism designed for that. A tool.

### Understanding our system across iterations

We have our current system. How do we check what happened 5 days ago? We probably go to our logs or the manager system sitting on top of it and filter out what we need. 
How do we know how many active users do we have currently and what are they doing? We probably check that in our database, session storage, or we send a push message to get some response. How do we check how many users tried out a feature we turned on two months ago in a specific hotfix deployment? We go to our logs again, or to the aggregation manager designed for this. 

Lots of tools, tool managers, painful solutions sometimes and don't forget how many people you have to involve to answer these questions.

## The idea

The set of possible states of an application is a graph. 
When a set of events happens in our system there is an indicator, that creates a graph.
When a user travels across our application it creates a graph with its path.

Lets put all these graphs into one place! And here comes the key. Let's connect these graphs! 

### How?

To understand this let's define some (not all!) event groups which can happen across our application, across different abstraction levels.

#### The user path events
A user is using our client app, and for the interaction with the server it calls the server's API (REST, SOAP, TCP, Websocket etc.). We can build a graph based on these events. Even when the API design is stateless we can assume the state of the client application as we created it (or we can have some kind of mapping mechanism).

#### System run events
Here belong all kinds of events happening in our system. Good events which are expected to happen because they are part of the application's functionality, bad events like errors, but also low level events like memory allocation. The level of abstraction depends on us. It feels like I am talking about the log level right? :)

#### Application lifecycle events
Events like deploying, instantiating, destroying or even load balancing across the application.

So we are focusing on these event groups and the possible graph model they form. We will want to store these graphs and do queries on them. 

The powerful [Neo4j](http://neo4j.com/) database is perfect for this. Lets see an example!

### Building yet another food ordering application

Neo4j has a query language called [Cypher](http://neo4j.com/developer/cypher-query-language/). Its usage is very intuitive and for our experiment we need just a running Neo4j instance and our Cypher queries.

So I have my [Neo4j Community](http://neo4j.com/download/) instance running.

Let's define our Cypher queries.

##### First the application lifecycle stage nodes
<table style="display:block; max-width:600px">
    <tr>
        <td style="padding: 30px;"> Name </td>
        <td>Cypher query</td>
        <td>Description</tdv
    </tr>
    <tr>
        <td  style="padding: 30px;"> ACTUAL </td>
        <td>CREATE (stage:LifecycleStage {name:'ACTUAL'})</td>
        <td>Connected to the actual state always.</tdv
    </tr>
    <tr>
        <td  style="padding: 30px;"> STARTLINE </td>
        <td>CREATE (stage:LifecycleStage {name:'STARTLINE', timestamp: 'x'})</td>
        <td>The start NODE of our application</tdv
    </tr>
    <tr>
        <td  style="padding: 30px;"> DEPLOYING </td>
        <td>CREATE (stage:LifecycleStage {name:'DEPLOYING', timestamp: 'x', version:'x'})</td>
        <td>When we are in a deploying process.</td>
    </tr>
    <tr>
        <td  style="padding: 30px;"> DEPLOYMENT_ERROR </td>
        <td>CREATE (stage:LifecycleStage {name:'DEPLOYMENT_ERROR', timestamp: 'x', version:'x'})</td>
        <td>On error during deploying.</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> DEPLOYED </td>
        <td>CREATE (stage:LifecycleStage {name:'DEPLOYED', timestamp: 'x', version:'x'})</td>
        <td>On successfull deploy.</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> STARTING_ERROR </td>
        <td>CREATE (stage:LifecycleStage {name:'STARTING_ERROR', timestamp: 'x', version:'x'})</td>
        <td>On error in startup. </td>
    </tr>
    <tr>
        <td style="padding: 30px;"> RUNNING </td>
        <td>CREATE (stage:LifecycleStage {name:'RUNNING', timestamp: 'x', version:'x'})</td>
        <td>On successful startup.</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> STOPPED </td>
        <td>CREATE (stage:LifecycleStage {name:'STOPPED', timestamp: 'x', version:'x'})</td>
        <td>On stop.</td>
    </tr>
</table>

##### The user event nodes

<table style="display:block; max-width:600px">
    <tr>
        <td style="padding: 30px;"> Name </td>
        <td>Cypher query</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_SESSION </td>
        <td>CREATE (ue:UserEvent {name:'USER_SESSION', timestamp: 'x', sessionID:'id'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_REGISTER </td>
        <td>CREATE (ue:UserEvent {name:'USER_REGISTER', timestamp: 'x', username:'u'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_LOGIN </td>
        <td>CREATE (ue:UserEvent {name:'USER_LOGIN', timestamp: 'x', username:'u'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_SEARCH </td>
        <td>CREATE (ue:UserEvent {name:'USER_SEARCH', timestamp: 'x', username:'u'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_PICK </td>
        <td>CREATE (ue:UserEvent {name:'USER_PICK', timestamp: 'x', username:'u'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_CHECKOUT </td>
        <td>CREATE (ue:UserEvent {name:'USER_CHECKOUT', timestamp: 'x', username:'u'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_PAY </td>
        <td>CREATE (ue:UserEvent {name:'USER_PAY', timestamp: 'x', username:'u'})</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> USER_LOGOUT </td>
        <td>CREATE (ue:UserEvent {name:'USER_LOGOUT', timestamp: 'x', username:'u'})</td>
    </tr>
</table>

##### Some imaginary system runtime event nodes
<table style="display:block; max-width:600px">
    <tr>
        <td style="padding: 30px;"> Name </td>
        <td>Cypher query</td>
        <td>Description</tdv
    </tr>
    
    <tr>
        <td style="padding: 30px;"> FEATURE_SWITCH </td>
        <td>CREATE (re:RuntimeEvent {name:'FEATURE_SWITCH', timestamp: 'x', reason:'', ticketID:''})</td>
        <td>On switching feature.</tdv
    </tr>
    <tr>
        <td style="padding: 30px;"> LOAD_ALERT </td>
        <td>CREATE (re:RuntimeEvent {name:'LOAD_ALERT', timestamp: 'x', message:''})</td>
        <td>Some kind of load alert sent to the alerting component.</td>
    </tr>
    <tr>
        <td style="padding: 30px;"> OUT_OF_STOCK </td>
        <td>CREATE (re:RuntimeEvent {name:'OUT_OF_STOCK', timestamp: 'x', itemID:'x'})</td>
        <td>When an item is out of stock.</tdv
    </tr>
    <tr>
        <td style="padding: 30px;"> SCHEDULED_REPORT </td>
        <td>CREATE (re:RuntimeEvent {name:'SCHEDULED_REPORT', timestamp: 'x', reportType:'x'})</td>
        <td>A scheduled task happening sometimes.</tdv
    </tr>
    
</table>

As you see, almost all events have a timestamp. The lifecycle events have a version as well, users have username. These are the most basic properties.

Let's assume this scenario:

"We deploy version v111 of our application successfully, but cannot startup. Then we fix the build and do this again with version v112. Now it starts up and runs. A user connects, looks around, registers, logs in, explores, picks some items, checks out, pays. We turn on a feature. The user searches and we get an error. We turn off the feature. A new user connects."

Let's execute the Cypher queries reflecting this scenario and see what graph we get in our Neo4j instance!

First the part where we deploy v111 of our app and get a STARTING_ERROR. The graph looks like this:
![Alt STARTING_ERROR](http://i61.tinypic.com/3588ex2.jpg)

Then we deploy and run a new version (v112) successfully, which means we grow a new graph from the STARTLINE node.
![Alt RUNNING](http://i60.tinypic.com/2pydedh.jpg)

A user connects and interacts.
![Alt User interacts](http://i57.tinypic.com/oaupog.jpg)

We turn on a feature.
![Alt 1st feature](http://i57.tinypic.com/2ii7t5v.jpg)

User searches and we turn off a feature, a new user connects.
![Alt 1st feature](http://i61.tinypic.com/ng1z5l.jpg)

## What we have so far

This is just a rough idea, but by looking to the resulting graph, we see clearly if a feature impacts a user.

By going through each user's steps we can understand how they are using our application and we can also easily check what items they are interested in, so we can apply some recommendation algorithms.

By following the connection between the deployment and the user or feature switch, we can easily decide what was the version or what was the configuration at a particular time.

By checking the timestamps we can easily measure the latencies relevant to us.

## Additional notes

### Thinking in aspects

Persisting events is definitely something injected horizontally across our application. We should create our own aspects for particular levels, and event groups we need. Executing them should be as lightweight as possible.

### The levels of tracking

The graph model should reflect the application's functionality, but we should not be afraid of persisting low level events if it helps us understanding our application better! 

### Using our existing log base to generate graphs

If our logs are detailed enough, we can easily generate graphs for the past! So technically, it is possible to build an application graph without touching our current application. 

### Talk with our colleagues

It is interesting to see what information others want to get from our "application graph" before we start modelling it.

### Please share your thoughts!
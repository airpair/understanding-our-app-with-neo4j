## Visioning, developing, building, deploying, waiting, iterating

In the world of IT, several roles impact an application's lifecycle. The product roles (product manager, chief, sales, project manager etc), the creator (developer, QA, UX etc), infrastructure roles and any mixture of these. 
These roles all require understanding what is going on in the application but they are all interested in a different or overlapping aspect.

From a product point of view we might be interested how the users are using our product, from sales point of view we will want to see what are they paying for, for developer point of view we would be probably more interested what happens in the system.

We also want to find a connection between application behaviour, user behaviour and infrastructure behaviour because a change in any of these impacts the others.

### Understanding our users

To be more concrete let's assume our product has a UI which talks through an API to our server. If this is a web application or an app, and we are talking about understanding users, we probably end up including some analytics like Google Analytics or mouse click trackers. That works great, but how do we map this information to the state we have in our server or to a specific deployment version we have? We create some mechanism designed for that, a tool, or if we don't do that we usually just check it in our version control system.

### Understanding our system across iterations

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

#### The user path events
A user is using our client app, and for the interaction with the server it calls the server's API (REST, SOAP, TCP, Websocket etc.). We can build a graph based on these events. Even when the API design is stateless we can assume the state of the client application as we created it (or we can have some kind of mapping mechanism).

#### System run events
All kinds of events happening in our system belong here. Expected events because they are part of the application's functionality, unexpected events like errors, but also low level events like memory allocation. The level of abstraction depends on us. It feels like I am talking about the log level right? :)

#### Application lifecycle events
Events like deploying, instantiating, destroying or even load balancing across the application.

So we are focusing on these event groups and the possible graph model they form. We will want to store these graphs and do queries on them. 

The powerful [Neo4j](http://neo4j.com/) database is perfect for this. Let's see an example!

### Building yet another food ordering application

Let's define the events first. We won't list all of them, but these should be enough. We will add some minimalistic list of properties which our event nodes would contain.


#### The application lifecycle stage nodes
<table style="display:block; max-width:600px">
    <tr>
        <td style="padding: 12px 30px"> Name </td>
        <td style="padding: 12px 30px">Properties</td>
        <td style="padding: 12px 30px">Description</tdv
    </tr>
    <tr>
        <td style="padding: 12px 30px"> START_NODE </td>
        <td style="padding: 12px 30px"></td>
        <td style="padding: 12px 30px">The start NODE of our application</tdv
    </tr>
    <tr>
        <td style="padding: 12px 30px"> DEPLOYING </td>
        <td style="padding: 12px 30px">timestamp, version, ticket</td>
        <td style="padding: 12px 30px">When we are in a deploying process.</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> DEPLOYMENT_ERROR </td>
        <td style="padding: 12px 30px">timestamp, message</td>
        <td style="padding: 12px 30px">On error during deploying.</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> DEPLOYED </td>
        <td style="padding: 12px 30px">timestamp</td>
        <td style="padding: 12px 30px">On successfull deploy.</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> STARTING_ERROR </td>
        <td style="padding: 12px 30px">timestamp, message</td>
        <td style="padding: 12px 30px">On error in startup. </td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> RUNNING </td>
        <td style="padding: 12px 30px">timestamp</td>
        <td style="padding: 12px 30px">On successful startup.</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> RUNTIME_ERROR </td>
        <td style="padding: 12px 30px">timestamp, message</td>
        <td style="padding: 12px 30px">On successful startup.</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> STOPPED </td>
        <td style="padding: 12px 30px">timestamp</td>
        <td style="padding: 12px 30px">On stop.</td>
    </tr>
</table>

#### The user event nodes
<table style="display:block; max-width:600px">
    <tr>
        <td style="padding: 12px 30px"> Name </td>
        <td style="padding: 12px 30px">Properties</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_SESSION </td>
        <td style="padding: 12px 30px">timestamp, session_id</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_REGISTER </td>
        <td style="padding: 12px 30px">timestamp, username</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_LOGIN </td>
        <td style="padding: 12px 30px">timestamp, username</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_SEARCH </td>
        <td style="padding: 12px 30px">timestamp</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_PICK </td>
        <td style="padding: 12px 30px">timestamp</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_CHECKOUT </td>
        <td style="padding: 12px 30px">timestamp</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_PAY </td>
        <td style="padding: 12px 30px">timestamp</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> USER_LOGOUT </td>
        <td style="padding: 12px 30px">timestamp</td>
    </tr>
</table>

#### Some imaginary system runtime event nodes
<table style="display:block; max-width:600px">
    <tr>
        <td style="padding: 12px 30px"> Name </td>
        <td style="padding: 12px 30px">Properties</td>
        <td style="padding: 12px 30px">Description</tdv
    </tr>
    <tr>
        <td style="padding: 12px 30px"> FEATURE_SWITCH </td>
        <td style="padding: 12px 30px">timestamp, ticket</td>
        <td style="padding: 12px 30px">On switching feature. Ticket id added for further info.</tdv
    </tr>
    <tr>
        <td style="padding: 12px 30px"> LOAD_ALERT </td>
        <td style="padding: 12px 30px">timestamp, message</td>
        <td style="padding: 12px 30px">Some kind of load alert sent to the alerting component.</td>
    </tr>
    <tr>
        <td style="padding: 12px 30px"> OUT_OF_STOCK </td>
        <td style="padding: 12px 30px">timestamp, item_id</td>
        <td style="padding: 12px 30px">When an item is out of stock.</tdv
    </tr>
    <tr>
        <td style="padding: 12px 30px"> SCHEDULED_REPORT </td>
        <td style="padding: 12px 30px">timestamp, report_type, report_id</td>
        <td style="padding: 12px 30px">A scheduled task happening sometimes.</tdv
    </tr>
</table>

Now let's jump into world of Neo4j and Cypher.

### Neo4j and Cypher

Neo4j has a query language called [Cypher](http://neo4j.com/developer/cypher-query-language/). Its usage is very intuitive and for our experiment we need just a running Neo4j instance and our Cypher queries.

In real life we would use a real [Neo4j](http://neo4j.com/download/) instance, but there is a super cool online [Neo4j console](http://console.neo4j.org/) which we can use to do our Cypher queries and share the database!

To get our feet wet, let's see how we create a new lifecycle stage node using Cypher.
```
MATCH (current:LifecyclePointer)-[pointer:IS_CURRENT_LIFECYCLE]->(stage:Lifec
DELETE pointer
CREATE (stage)-[:NEXT]->(nextStage:LifecycleStage {name:'STAGENAME', timestamp: '123', version:'v111'})
CREATE (current)-[:IS_CURRENT_LIFECYCLE]-> (nextStage);
```
We will use a pointer which points always to the latest stage so we have to repoint and add a new node to our event graph. These queries would be wrapped into transactions of course as Neo4j [supports them](http://neo4j.com/docs/stable/transactions.html) on node and relation level. 

### Now the story!

Let's assume the following scenario:

"We deploy version v111 of our application successfully, but cannot startup. Then we fix the build and do this again with version v112. Now it starts up and runs. A user connects, looks around, registers, logs in, explores, picks some items, checks out, pays. Another connects, starts exploring. We turn on a feature. One of the users searches and we get an error. We turn off the feature. A new user connects."

Let's execute the Cypher queries reflecting this scenario and see what graph we get in our Neo4j instance!
First the part where we deploy v111 of our app and get a STARTING_ERROR. The graph looks like this:
##### **Check it live: http://console.neo4j.org/r/4i9sh9**
<center>![Alt STARTING_ERROR](https://openmerchantaccount.com/img/starting_error.png)</center>

Then we deploy and run a new version (v112) successfully, which means we grow a new graph from the START_NODE node.
![Alt RUNNING](http://i60.tinypic.com/2pydedh.jpg)

A user connects and interacts.
![Alt User interacts](http://i57.tinypic.com/oaupog.jpg)

We turn on a feature.
![Alt 1st feature](http://i57.tinypic.com/2ii7t5v.jpg)

User searches and we turn off a feature, a new user connects.
![Alt 1st feature](http://i61.tinypic.com/ng1z5l.jpg)

## Let's review what we got

This is just a rough idea, but by looking to the resulting graph, we see clearly if a feature switching impacts a user, and when.

By going through each user's steps we can understand how they are using our application and we can also easily check what items they are interested in, so we can apply some recommendation algorithms.

By following the connection between the deployment and the user or feature switch, we can easily decide what was the version or what was the configuration at a particular time.

By checking the timestamps we can easily measure the latencies relevant to us.

## Next steps
I don't really see people logging their system and user events into graphs, but I think it is definitely worth doing further experiments in this field, as it can result great patterns and frameworks.

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
# Part 5: Other Matters

We've now built up the sample application. This final section features a few topics that are worth considering if you plan to build a production system in this style. The Edument CQRS starter kit really is just that: a start. It's up to you to finish it with the things you need for a production ready system.

## Two Commands, One Aggregate

What happens if two commands targeting the same aggregate arrive at around the same time, and the (overlapping) command handler executions both yield events? Since aggregates are built up per command, there is no risk of the two executions interfering. However, only one will manage to get its events stored. The second will lose: it is not allowed to store its events because they were produced based on an inconsistent view of history. The rule underlying this is really simple: the number of events the aggregate had at the point it was loaded must match the number stored for the aggregate at the point we try to persist the new events. If more got stored in the meantime, we know there is a conflict.

This has some interesting consequences. One is that the system has a global progress bound: the only way it may fail to persist new events is because something else was successful. However, more interesting is what to do when that situation arises.

When this happens with the starter kit as provided, it just results in an exception being thrown. However, there is another option. You could catch such exceptions in MessageDispatcher and try to re-run the command handler. This means it will act on the latest events, and (provided nothing beats it again) can get its work done. Since command handlers only look at the aggregate's fields and the incoming command, they are pure. This means trying again is harmless.

Of course, there's nothing to enforce such purity on you. For example, you could potentially write a command handler that sends an email. This is a bad idea, since a conflict may follow and the command handler may have to run again. This could lead to a duplicate email, or worse the second attempt to handle the comamnd could fail due to the new events, meaning the email now tells that receiver about a reality that never came to be.

Do side-effects in an event handler instead.

## Transactionality of Event Handlers

Another concern we may have is what happens if events are persisted, but then we fail to update the read models. In this case, it's easy to see the risk of things getting desynchronized, which is clearly undesirable. There are a few options.

If all of the stores involved can participate in a transaction, then you could potentially surround the call to the command handler, applying of events and calls to the event subscribers in a TransactionScope. This is a relatively easy change to the MessageDispatcher, and it gives a pretty good promise of consistency. However, it's not quite a free lunch. Here's some things you may need to be aware of if you take this approach.

* You need to be using an event store that knows about transactions. If you use a event store that is really a couple of SQL Server database tables, for example, then you're covered.
* You need the read models to also be persisted in a way that participates in the transaction.
* The failure of one read model cascades out to a complete failure to apply the command. (Note that whether this is good or bad is system specific.)
* Not everything can be transactional (like sending email).

There are a couple of alternatives that are worth mentioning. One is to use a store-and-forward message queue to implement the publish/subscribe of events. The placing of the events into the queue and the persistence of the events to storage need to happen within a transaction. MSMQ and SQL Server can be used this way, for example (but beware the distributed transaction coordinator). The benefit of this is that as soon as the event is in the subscriber's queues, you're done. Read models can individually and transactionally receive their copy of the message. This means:

* Failure of a single read model affects only that read model
* There are multiple short-lived transactions rather than one big one
* Read models become **eventually consistent**. This is a big deal and you must plan how to deliver a good user experience in the light of it!
* The message queue provides a strategy to scale out over multiple processes, hardware, etc.
* BUT, you've now got much more of a distributed system!

Since using such a setup brings the system to a state of eventual consistency anyway, it may be simpler to switch to a pull-based approach. Instead of using a message queue, the read models simply poll for the latest events in the store. By giving each event stored an ascending number, it is possible for read models to know the last event they incorporated into the model and query for any new ones. This means that the update rate is chosen by the read model, which allows fine-grained contorl over SLA. It is also easy to scale out, as events are easily replicated. AtomPub is one way to do this. Using the Event Store product, you get an atom interface built in.

## Coping with Eventual Consistency

Of the approaches above, two of them lead to eventual consistency. There is no way to sweep that under the carpet: it will have consequences for the way you build the rest of the system. However, to repeat the advice already given many times in this tutorial, the approach suggested here is **aimed at your core domain**.

The typical CRUD system, with a grid of things to edit, has potential to be a nightmare if you have eventual consistency. Why? Because when the user adds or edits items, they expect to return to the grid and see it updated with what they just did. If the grid is being updated by doing a query each time, then there's no promise the update will have already taken place in the appropriate read model.

Task based UIs, because they're about taking the user through a workflow, tend to be less problematic in this regard, however. And doing task based UIs for the core domain is a worthwhile investment, because that's where efficient user experience will have the most benefit.
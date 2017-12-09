# Part 3: Read Models

## Read Models are about Queries

The domain logic that we built in the previous section accepted commands from the outside world. Sending a command is a void operation; either the command is accepted and life goes on, or the command is rejected and an exception is thrown. So how does the outside world ever ask questions about current state? This is where read models come in.

A read model is a **model specialized for reads, that is, queries**. It takes events produced by the domain and uses them to build and maintain a model that is suitable for answering the client's queries. It can draw on events from different aggregates and even different types of aggregate. What it does from there is unimportant. It may build and maintain...

* An in-memory model of the data
* A relational model of the data, stored in an RDBMS
* Documents, stored in a document database
* A graph model of the data, stored in a graph database
* A search index
* An XML file, CSV file, Excel sheet, etc.

You get to choose - and furthermore, you get to make the decision per read model (providing a concrete way to realize polyglot persistence). Moreover, multiple read models can be built from the same event. After all, events represent facts that have already been incorporated into the system. They are the normalized part of our system, and read models are a point of potential denormalization. If that makes your inner relational animal feel scared, it's worth remembering that you probably turn normalized data into denormalized data all the time - using the JOIN operator in SQL. We're just choosing to do the denormalization at a different point - potentially paying for it once per change rather than once per query.

A read model could even, instead of exposing methods to do queries, expose push collections, using Rx to turn streams of events from the domain into objects suited to maintaining some kind of reactive UI. Thus, this approach fits very well with reactive programming also.

In this tutorial, we'll build in-memory read models. However, the step up to making a persistent read model, stored in a database or file of some kind, is small. You can use all the familiar tools you wish to do it, too; there's nothing wrong with implementing your read model using Linq to SQL, if it is the most expedient way and will give you the performance you desire.

## What read models will we need

To get an idea of what read models are needed, we should start by looking at different scenarios where we need to present information to the users of the system. Wait staff need to see:

* The current list of open tabs
* The current tab for a particular table, with an indication of the status of each item
* The drinks that they need to serve
* Any food that is prepared and ready to be served to the table

Chefs need to see:

* The current list of food that has been ordered and needs to be prepared

Cashiers need to see:

* The tab for a table, and the total amount to be paid

The queries that involve tab viewing seem highly related, and can probably just be methods returning information based on a single model. The queries where a particular member of wait staff can see the food/drink they should serve as soon as possible also feel related; we may even want to return the two together. The query for showing chefs food that needs preparing feels like a separate concern, and so we may wish to keep that distinct.

Overall, it feels like there are three read models to build:

* A chef todo list read model
* A wait staff todo list read model
* A tab read model

We'll look in detail at building the chef todo list.

## The Chef Todo List

Maintaining the todo list for chefs involves reacting to two events:

* `FoodOrdered`, which adds food to prepare onto the todo list
* `FoodPrepared`, which removes prepared food from the todo list

We'll implement read models as classes, which implement the ISubscribeTo generic interface in order to subscribe to events:

```csharp
public class ChefTodoList :
    ISubscribeTo<FoodOrdered>,
    ISubscribeTo<FoodPrepared>
{
    // ...
}
```

While the naive approach is just to push all items to prepare into a list, typically food that is ordered together should be delivered together. Thus, the grouping should be retained - while realizing that a chef may violate it in certain circumstances. Therefore,we'll declare two DTOs to represent the todo list entries.

```csharp
public class TodoListItem
{
    public int MenuNumber;
    public string Description;
}

public class TodoListGroup
{
    public Guid Tab;
    public List<TodoListItem> Items;
}
```

The read model will maintain a List of TodoListGroup:

```csharp
private List<TodoListGroup> todoList = new List<TodoListGroup>();
```

Since changes to the todo list and requests for it may arrive concurrently, we must take care to be thread safe. Furthermore, we should never leak the internal todo list. In fact, we must not leak the internal list of items per group either, since those can also change over time. Thus, we should write the GetTodoList method something like this:

public List<TodoListGroup> GetTodoList()
{
    lock (todoList)
        return (from grp in todoList
                select new TodoListGroup
                {
                    Tab = grp.Tab,
                    Items = new List<TodoListItem>(grp.Items)
                }).ToList();
}
Note that we need to take care of the locking here as we're keeping a mutable model around in memory. If your builders always wrote into a database and your query methods already read from one, then you are most likely free of these issues (because you've left them for the database to deal with).

What about the methods that will build the todo list? New orders are a fairly simple bit of mapping:

```csharp
public void Handle(FoodOrdered e)
{
    var group = new TodoListGroup
    {
        Tab = e.Id,
        Items = new List<TodoListItem>(
            e.Items.Select(i => new TodoListItem
            {
                MenuNumber = i.MenuNumber,
                Description = i.Description
            }))
    };

    lock (todoList)
        todoList.Add(group);
}
```

The FoodPrepared handler has a little more to think about. First, it should remove all of the prepared items from the group. Then, if nothing remains in the group, the group itself should also be removed.

```csharp
public void Handle(FoodPrepared e)
{
    lock (todoList)
    {
        var group = todoList.First(g => g.Tab == e.Id);

        foreach (var num in e.MenuNumbers)
            group.Items.Remove(
                group.Items.First(i => i.MenuNumber == num));

        if (group.Items.Count == 0)
            todoList.Remove(group);
    }
}
```

## Two Become One

The next read model to implement is the wait staff todo list. There's nothing particularly new from an implementation perspective. It subscribes to all of the events, since the whole lifecycle matters for maintaining the todo list for wait staff. Of note, the descriptions are included in the event issued at the point of ordering, so even though we don't need to present food items in the list at that point, we still want to record the information associated with them, adding them to the list of things to serve at the point the FoodPrepared event arrives.

After implementing this read model, the key data structure looked something like this:

```csharp
public class ItemTodo
{
    public int MenuNumber;
    public string Description;
}

public class TableTodo
{
    public int TableNumber;
    public string Waiter;
    public List<ItemTodo> ToServe;
    public List<ItemTodo> InPreparation;
}

private Dictionary<Guid, TableTodo> todoByTab =
    new Dictionary<Guid,TableTodo>();
```

With the to-serve list per query looking as follows:

```csharp
public Dictionary<int, List<ItemTodo>> TodoListForWaiter(string waiter)
{
    lock (todoByTab)
        return (from tab in todoByTab
                where tab.Value.Waiter == waiter
                select new
                {
                    TableNumber = tab.Value.TableNumber,
                    ToServe = CopyItems(tab.Value)
                })
                .Where(t => t.ToServe.Count > 0)
                .ToDictionary(k => k.TableNumber, v => v.ToServe);
}
```

What's striking is that we essentially end up storing information about the tab overall. With a few extensions, this read model would also be able to answer queries related to tabs. Therefore, we can simplify our original plan, having just two read models. The resulting new read model is called OpenTabs, because its focus is on queries related to tabs that are currently open. See the sample application for the full details.

It's important to remain open to course corrections such as this during the development process. Things that a developer can see as they build a system differ from those a domain expert can see, which can lead to simplifications and new opportunities.

## Safety Through Interfaces

As handling of events is done through interface implementation, the Handle methods are public. However, we really don't want those writing query code to call them. Also, coupling directly to classes is bad object oriented design. Therefore, we'll extract a couple of interfaces from our read models:

```csharp
public interface IChefTodoListQueries
{
    List<ChefTodoList.TodoListGroup> GetTodoList();
}

public interface IOpenTabQueries
{
    List<int> ActiveTableNumbers();
    OpenTabs.TabInvoice InvoiceForTable(int table);
    OpenTabs.TabStatus TabForTable(int table);
    Dictionary<int, List<OpenTabs.TabItem>> TodoListForWaiter(string waiter);
}
```

These will be used by the web frontend, which we will look at next.
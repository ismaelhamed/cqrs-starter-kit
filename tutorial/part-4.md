# Part 4: Web Application Integration

## Planning

We'll build our front end application using ASP.NET MVC. The overall idea is to use queries to obtain data and send commands in order to effect changes in the system. Since the commands and queries were designed with user tasks in mind, the mapping should be easy or non-existent (that is, we use the query DTOs or command objects directly as the view model).

The web application needs to provide:

* Creation of new tabs when visitors arrive and take a table
* Adding of food and drink orders to a tab
* Viewing of a tab's current status: what is to serve, in preparation and already served
* A list of outstanding items to serve per member of wait staff
* A list of food to prepare, for chefs
* A way for a cashier to obtain an invoice and, upon payment, close the tab

## Menus and Wait Staff

Having seen how nicely we implemented tabs, using events and writing intentful tests, it may seem tempting to go ahead and create aggregates for the menu and the wait staff list. But wait...what will we gain from this? Tabs are at the heart of the cafe's operations. They follow an interesting workflow, which happens many times a day and needs to be got right. By contrast, there is very little to validate, and nearly no invariants to enforce, when it comes to a menu and a list of wait staff. It's just...data. And what do we do with it? CRUD.

For the sake of this example, we won't even build tools for manipulating it. Instead, we'll just have a file of static data:

```csharp
public class MenuItem
{
    public int MenuNumber;
    public string Description;
    public decimal Price;
    public bool IsDrink;
}

public static List<MenuItem> Menu = new List<MenuItem>
{
    new MenuItem
    {
        MenuNumber = 1, Description = "Coke", Price = 1.50M, IsDrink = true
    },
    new MenuItem
    {
        MenuNumber = 10, Description = "Mushroom & Bacon Pasta", Price = 6.00M
    },
    // ...
};

public static List<string> WaitStaff = new List<string>
{
    "Jack", "Lena", "Pedro", "Anastasia"
};
```

The takeaway message here is that not everything in your system will benefit from intentful testing, event sourcing, DDD and so forth. Apply it **in the core domain**. For the stuff that needs to be there but isn't the core driver of business value, **do the cheapest thing - but couple to it loosely**.

## Setup

We need instances of the message dispatcher to send commands. We also need instances of the read models, and need the message dispatcher to send events to them. We may wish to dependency inject all of these things into our controllers - but to keep this example simple, we'll just have a static class with a setup method called from our Global.asax.cs.

```csharp
public static class Domain
{
    public static MessageDispatcher Dispatcher;
    public static IOpenTabQueries OpenTabQueries;
    public static IChefTodoListQueries ChefTodoListQueries;

    public static void Setup()
    {
        Dispatcher = new MessageDispatcher(new InMemoryEventStore());

        Dispatcher.ScanInstance(new TabAggregate());

        OpenTabQueries = new OpenTabs();
        Dispatcher.ScanInstance(OpenTabQueries);

        ChefTodoListQueries = new ChefTodoList();
        Dispatcher.ScanInstance(ChefTodoListQueries);
    }
}
```

## Common Links

To make for easy navigation, we want all pages to display a set of links for each member of wait staff's todo lists, and another set linking to the status page for each open tab. We'll use ASP.NET MVC's ViewBag to store this data, accessing it from the page layout. Since we will want this for almost all of the pages, we'll factor out the logic to add it to an action filter.

```csharp
public class IncludeLayoutDataAttribute : ActionFilterAttribute
{
    public override void OnResultExecuting(ResultExecutingContext filterContext)
    {
        if (filterContext.Result is ViewResult)
        {
            var bag = (filterContext.Result as ViewResult).ViewBag;
            bag.WaitStaff = StaticData.WaitStaff;
            bag.ActiveTables = Domain.OpenTabQueries.ActiveTableNumbers();
        }
    }
}
```

Notice how this makes a call to a read model. The layout can then incorporate this information into the navigation.

```csharp
<li class="nav-header">Wait Staff Todo Lists</li>
@foreach (var w in ViewBag.WaitStaff as List<string>)
{
<li>@Html.ActionLink(w, "Todo", "WaitStaff", new { id = w }, null)</li>
}
<li class="nav-header">Open Tabs</li>
@foreach (var t in ViewBag.ActiveTables as List<int>)
{
<li>@Html.ActionLink(t.ToString(), "Status", "Tab", new { id = t }, null)</li>
}
```

## Opening A Tab

Opening a tab is done by sending an OpenTab command. The job of the web application is to collect information to populate that command. One obvious thing we could do is use the command as the model for the page - and in fact, that will work out very nicely here. Thus, here is the Razor view:

```csharp
@model Cafe.Tab.OpenTab

<h2>Open Tab</h2>

@using (Html.BeginForm())
{
  <fieldset>
    <label>Table Number</label>
    @Html.TextBoxFor(m => m.TableNumber)

    <label>
      Waiter/Waitress
    </label>
    @Html.DropDownListFor(m => m.Waiter, (ViewBag.WaitStaff as List<string>)
        .Select(w => new SelectListItem { Text = w, Value = w }))
    <br>

    <button type="submit" class="btn">Submit</button>
  </fieldset>
}
```

We can then use model binding in order to have the MVC framework instantiate and populate the command object for us, leaving us to fill in a Guid for the tab we wish to open (a fresh Guid because this command should happen on a new aggregate) and send the command using the dispatcher.

```csharp
public ActionResult Open()
{
    return View();
}

[HttpPost]
public ActionResult Open(OpenTab cmd)
{
    cmd.Id = Guid.NewGuid();
    Domain.Dispatcher.SendCommand(cmd);
    return RedirectToAction("Order", new { id = cmd.TableNumber });
}
```

This almost works - but not quite! It turns out that MVC model binding will not work with fields, only with properties. This is not a big problem; we just tweak our command a little:

```csharp
public class OpenTab
{
    public Guid Id;
    public int TableNumber { get; set; }
    public string Waiter { get; set; }
}
```

Note this makes explicit what fields we welcome model binding of and which should not be controllable from the web.

## Ordering

Placing an order also involves issuing a command. This time, it's not quite so easy to have the command as the model, but we can define a model that will be easy to map to the command.

```csharp
public class OrderModel
{
    public class OrderItem
    {
        public int MenuNumber { get; set; }
        public string Description { get; set; }
        public int NumberToOrder { get; set; }
    }

    public List<OrderItem> Items { get; set; }
}
```

This can be built up from the static menu data:

```csharp
public ActionResult Order(int id)
{
    return View(new OrderModel
    {
        Items = (from item in StaticData.Menu
                 select new OrderModel.OrderItem
                 {
                     MenuNumber = item.MenuNumber,
                     Description = item.Description,
                     NumberToOrder = 0
                 }).ToList(),
    });
}
```

And rendered using a relatively straightforward view:

```csharp
@model WebFrontend.Models.OrderModel

<h2>Place Order</h2>

@using (Html.BeginForm())
{
  <fieldset>
    <table class="table">
      <thead>
        <tr>
          <td>Menu #</td>
          <td>Description</td>
          <td>Number To Order</td>
        </tr>
      </thead>
      <tbody>
        @for (int i = 0; i < Model.Items.Count; i++)
        {
            <tr>
              <td>
                @Model.Items[i].MenuNumber
                @Html.HiddenFor(m => m.Items[i].MenuNumber)
              </td>
              <td>@Model.Items[i].Description</td>
              <td>@Html.TextBoxFor(m => m.Items[i].NumberToOrder)</td>
            </tr>
        }
      </tbody>
    </table>

    <button type="submit" class="btn">Place Order</button>
  </fieldset>
}
```

We can again use model binding to have the form data mapped into OrderModel. Then, we do a little bit of mapping to turn it into the PlaceOrder command and send it using the dispatcher. We also need to resolve the table number to its tab's ID.

```csharp
[HttpPost]
public ActionResult Order(int id, OrderModel order)
{
    var items = new List<Events.Cafe.OrderedItem>();
    var menuLookup = StaticData.Menu.ToDictionary(k => k.MenuNumber, v => v);
    foreach (var item in order.Items)
        for (int i = 0; i < item.NumberToOrder; i++)
            items.Add(new Events.Cafe.OrderedItem
            {
                MenuNumber = item.MenuNumber,
                Description = menuLookup[item.MenuNumber].Description,
                Price = menuLookup[item.MenuNumber].Price,
                IsDrink = menuLookup[item.MenuNumber].IsDrink
            });

    Domain.Dispatcher.SendCommand(new PlaceOrder
    {
        Id = Domain.OpenTabQueries.TabIdForTable(id),
        Items = items
    });

    return RedirectToAction("Status", new { id = id });
}
```

Note that TabIdForTable is a method added to the read model, having realized while building the web application that it would be useful. This is a normal thing to do; after all, the queries are there to serve the needs of clients.

## The Wait Staff Todo List

Queries are meant to be close to what we expect to present to the user. As a result, the DTO returned by a query will often be suitable as a model for the view. In that case, we can do something as simple as:

```csharp
[IncludeLayoutData]
public class WaitStaffController : Controller
{
    public ActionResult Todo(string id)
    {
        ViewBag.Waiter = id;
        return View(Domain.OpenTabQueries.TodoListForWaiter(id));
    }
}
```

The view simply takes the model and renders it out as HTML. Note that we take care to include a helpful link to the tab status page for each table we should serve to.

```csharp
@model Dictionary<int, List<CafeReadModels.OpenTabs.TabItem>>

<h2>Todo List For @ViewBag.Waiter</h2>

@foreach (var table in Model)
{
    <h3>Table @table.Key</h3>

    <table class="table">
    <thead>
        <tr>
        <th>Menu #</th>
        <th>Description</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var item in table.Value)
        {
            <tr>
            <td>@item.MenuNumber</td>
            <td>@item.Description</td>
            </tr>
        }
    </tbody>
    </table>

    @Html.ActionLink("Go To Tab", "Status", "Tab",
        new { id = table.Key }, null)
}
```

There's nothing particularly exciting here; we really are just taking data out of the read model and rendering it using the view.

## The Chef Todo List

The action method is no more involved:

```csharp
public class ChefController : Controller
{
    public ActionResult Index()
    {
        return View(Domain.ChefTodoListQueries.GetTodoList());
    }
}
```

The view, however, is a bit more interesting this time. We need to offer a way to mark items as prepared. This needs a little care, because it's possible a chef will manage to prepare only some of the items together, so we must offer a way to mark out those that have been prepared. We'll use check boxes, to keep things relatively simple.

```csharp
@model List<CafeReadModels.ChefTodoList.TodoListGroup>

<h2>Meals To Prepare</h2>

@foreach (var group in Model)
{
    var i = 0;
    using (Html.BeginForm("MarkPrepared", "Chef"))
    {
        @Html.Hidden("id", group.Tab)
        <table class="table">
        <thead>
            <tr>
            <th>Menu #</th>
            <th>Description</th>
            <th>Prepared</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var item in group.Items)
            {
                <tr>
                <td>@item.MenuNumber</td>
                <td>@item.Description</td>
                <td>@Html.CheckBox(string.Format("prepared_{0}_{1}",
                        i++, item.MenuNumber.ToString()))</td>
                </tr>
            }
        </tbody>
        </table>

        <button type="submit" class="btn">Mark Prepared</button>
        <hr />
    }
}
```

Often, views will end up playing this dual role: showing data from a read model and also providing a way to issue some command to do the next piece of work.

## The Rest

The other operations that make up the web application are variations on the things we've already seen in this tutorial, so we won't go over them in more detail here. See the sample application if you're curious.

## Task Based UI

Intentful testing leads us to commands that express domain-significant acts. This in turn tends to be reflected in the UI, **which is now focused on tasks rather than editing**. Since queries are developed according to the kinds of reports that we need to present, they can also focus on what the user really needs to see as they use the system.

This helps us to avoid the trap, common in systems that evolve from designing a relational database, of everything really being about editing. Of course, just creating commands isn't a magical promise of everything. If we'd failed to focus on the language of the domain language, we may have ended up with commands like CreateTab, UpdateTab and DeleteTab. In essence, we'd have gone to all this effort only to recreate a CRUD system - one we could have realized in a much more expedient manner!

All the pieces - intentful testing, a focus on the verbs, separating out the handling of commands and queries, and task based UI - make a coherent whole. They are best applied to **the core domain**, where it makes sense to invest in building domain understanding, solid tests, and loose coupling with the outside world.
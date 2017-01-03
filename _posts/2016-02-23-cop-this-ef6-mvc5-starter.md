---
layout: post
title: Cop this - a starter for EF6 and MVC5
categories:
- Programming
tags:
- entity framework
- mvc
- ddd
date: 2016-02-23 22:00:00 +1000
---
[Just take me to the code](https://github.com/xwipeoutx/CopThis)

As many of you know, I've recently started at [SSW](https://ssw.com.au), and of course that means I get to go File->New project much more frequently than when I was 100% product development.  

I have a few rules when it comes to project layout which should be par for the course

* All domain logic is in its own project
* All database logic is in its own project
* All UI logic is in its own project

Unfortunately, people doing the scaffolding for MVC and EF6 seem to have a different set of priorities

* All views should be modelled straight off an entity
* All entities should contain all the validation required for the UI

Of course, this works fine for CRUD cases, but the moment you need some business logic, you're in the deep end with no handrail.

<!--break-->
Now, this is a project where the business logic is not so crazy that it warrants going full DDD - but it still deserves to have the domain easily tested and separated from the view, for maintainability.  

I went for a design that gave View Models, Command Handlers, Repositories and Entities, but was limited to Transactional Consistency.  It also allowed fast read-models and flexible Unit of Work usage.  I thought I'd share the layout here.  It is in EF6 code-first on ASP.NET MVC5, and is available on [GitHub](https://github.com/xwipeoutx/CopThis)

# 1. Domain

The sample domain is an application for police to record speeding tickets. They enter in the license plate, the speed limit and the vehicles speed.  They may optionally enter the vehicle's make and model.  

If a vehicle in the system already has the license plate, then the ticket is linked to the corresponding vehicle.  Otherwise, a new vehicle is created and will eventually be synced with an external system.

Further, if the vehicle was _not_ in the system, and the make and model was provided, it is used as the initial values for the vehicle.  Otherwise, it is ignored.

At a later date, the officer may mark the speed ticket as paid.

## Entities

There are two entities here - a speed ticket and a vehicle.

These will exist as POCOs in the domain project.

{% highlight cs %}
public class Ticket
{
    public int Id { get; set; }
    public Vehicle Vehicle { get; set; }
    public int SpeedLimit { get; set; }
    public int ActualSpeed { get; set; }
    public bool IsPaid { get; set; }
}

public class Vehicle
{
    public int Id { get; set; }
    public string LicensePlate { get; set; }
    public string Make { get; set; }
    public string Model { get; set; }
}
{% endhighlight %}

## Commands

All commands are in the domain project, and are simply serialized method calls.

{% highlight cs %}
public class IssueTicketCommand
{
    public string LicensePlate { get; set; }
    public int SpeedLimit { get; set; }
    public int ActualSpeed { get; set; } 
    public string Make { get; set; }
    public string Model { get; set; }
}

public class PayTicketCommand
{
    public int TicketId { get; set; }
}
{% endhighlight %}

## Command Handlers

Note this is _not_ CQS. DB-generated ids are prevelant, and I thought it an ok tradeoff for the nicer URL that it offers over using guids. So commands that create entities will return the entity that was created.

This also lives in the domain project. 

{% highlight cs %}
public Ticket Handle(IssueTicketCommand command) 
{
    if (command.ActualSpeed <= command.SpeedLimit)
      throw new SpeedLimitNotExceededException();
      
    var vehicle = _vehicleRepository.Find(command.LicensePlate)
      ?? CreateVehicle(command.LicensePlate, command.Make, command.Model);
      
    var ticket = new Ticket
    {
        Vehicle = vehicle,
        SpeedLimit = command.SpeedLimit,
        ActualSpeed = command.ActualSpeed
    };
    _ticketRepository.Create(ticket);
    return ticket;
}

private Vehicle CreateVehicle(string licensePlate, string make, string model) 
{
    var vehicle = new Vehicle
    {
        LicensePlate = licensePlate,
        Make = make,
        Model = model
    };
    
    _vehicleRepository.Create(vehicle);
    
    return vehicle;
}

public void Handle(PayTicketCommand command)
{
    var ticket = _ticketRepository.Get(command.TicketId);
    
    if (ticket == null)
      throw new TicketDoesNotExistException();
    
    if (ticket.IsPaid)
      throw new TicketIsAlreadyPaidException();
        
    ticket.IsPaid = true;
    _ticketRepository.Save(ticket);
}
{% endhighlight %}

These methods are trivially testable, and do not make any assumptions about persistence - in fact, they're quite close to the usual DDD repository pattern, with a repository per aggregate root.

However, I've been loose with transactions here - there are none in the command handlers, so it's up to the application to set up the `TransactionScope` or `UnitOfWork` or whatever is cool. 

The sample application will use a per-request UnitOfWork that's committed in the controller.

 Of course, in a perfect DDD world, the vehicle would likely be eventually consistent, and not related via a primary key, but I feel that's complicating this simple project.

# 2. Data Access

I have purposely not extracted a generic EntityFrameworkRepository as base classes for these, 
for brevity.

{% highlight cs %}
public interface IUnitOfWork
{
    void Commit();
}

public class CopContext : DbContext, IUnitOfWork
{
    public CopContext() : base("CopContext") 
    {
        Configuration.LazyLoadingEnabled = false;
        Configuration.AutoDetectChangesEnabled = false;
    }
    
    public DbSet<Ticket> Tickets { get; set; }
    public DbSet<Vehicle> Vehicles { get; set; }
    
    public void Commit() {
        SaveChanges();
    }
}

public class EntityFrameworkTicketRepository : ITicketRepository
{
    private readonly CopContext _context;
    
    public EntityFrameworkTicketRepository(CopContext context) {
        _context = context;
    }
    
    public Ticket Get(int id) 
    {
        return _context.Tickets.AsNoTracking().FirstOrDefault(t => t.Id == id);
    }
    
    public void Create(Ticket ticket) 
    {
        _context.Tickets.Add(ticket);
    }
    
    public void Save(Ticket ticket) {
        _context.Tickets.Attach(ticket);
        _context.Entry(ticket).State = EntityState.Modified;
    }
}

public class EntityFrameworkVehicleRepository : IVehicleRepository
{
    // ... etc.
    public Vehicle Find(string licensePlate) 
    {
        return _context.Vehicles.AsNoTracking().FirstOrDefault(
            v => v.LicensePlate == licensePlate
        );
    }
    // ... etc.
}

{% endhighlight %}
  
The design here is that the repositories don't do persistence themselves, but rely on the contexts to be the same instance, and be commited externally via the unit of work implementation.

To be honest, the unit of work implementation here is pretty close to pure laziness, but it works well enough for our use case.

**Important**: Search methods in this pattern should _never_ return tracked entities Save methods need to ensure that they mark the model as modified after attaching, or it will never be saved.

# 3. UI

The views are trivial given the right models, so here's the controller

{% highlight cs %}
// View Model example
public class TicketRow
{
    public int Id { get; set; }
    public int SpeedExceededBy { get; set; }
    public string LicensePlate { get; set; }
    public string Make { get; set; }
    public string Model { get; set; }
    
    public static readonly Expression<Func<Ticket, TicketRow>> Projection = ticket => new TicketRow {
        Id = ticket.Id,
        SpeedExceededBy = ticket.ActualSpeed - ticket.SpeedLimit,
        LicensePlate = ticket.Vehicle.LicensePlate,
        Make = ticket.Vehicle.Make,
        Model = ticket.Vehicle.Model
    };
}

public class TicketsController
{
    public TicketsController(TicketCommandHandler ticketCommandHandler, IUnitOfWork unitOfWork, CopContext context) {
        _ticketCommandHandler = ticketCommandHandler;
        _unitOfWork = unitOfWork;
        _context = context;
    }
    
    public ActionResult Index()
    {
        IEnumerable<TicketRow> tickets = _context.Tickets.Include("Vehicle")
            .Where(t => !t.IsPaid)
            .Select(TicketRow.Projection);

        return View(tickets);
    }

    public ActionResult Create([Bind]IssueTicketCommand issueTicket)
    {
        var ticket = _ticketCommandHandler.Handle(issueTicket);
        _unitOfWork.Commit();

        return RedirectToAction("View", new {id = ticket.Id});
    }

    public ActionResult Pay([Bind]PayTicketCommand payTicket)
    {
        _ticketCommandHandler.Handle(payTicket);
        _unitOfWork.Commit();

        return Redirect("Index");
    }
}
{% endhighlight %}

We've made a critical decoupling here that will save your views from polluting the domain - view models are simply projections from the db sets - with `.AsNoTracking()`. Of course, this could be abstracted again.

This is pretty powerful - the projections will be done in the database query, so you'll only be getting what you need for the view model, not everything from every entitiy. Filtering is also trivially done from expressions - so again, in the query itself.

# 4. Setup injection

[Autofac](http://autofac.org) will be our container of choice - specifically the [Autofac MVC QuickStart](http://docs.autofac.org/en/latest/integration/mvc.html#quick-start).

Beyond this though, let's look at the additional setups - command handlers, repositories, and our data layers all need to be set up

{% highlight cs %}
public class CopModule : Module
{
    public override void Load(ContainerBuilder builder)
    {
        var domainAssembly = typeof(Ticket).Assembly;
        var dataAssembly = typeof(CopContext).Assembly;
        
        // Register command handlers
        builder.RegisterAssemblyTypes(domainAssembly)
            .Where(t => t.Name.EndsWith("CommandHandler"))
            .AsSelf().AsImplementedInterfaces();
            
        // Register repositories
        builder.RegisterAssemblyTypes(dataAssembly)
            .Where(t => t.Name.EndsWith("Repository"))
            .AsImplementedInterfaces();
            
        // Register database stuff
        builder.RegisterType<CopContext>()
            .InstancePerRequest()
            .AsSelf().As<IUnitOfWork>();        
    }
}
{% endhighlight %}

Everything except the context and unit of work are transient dependencies by default, and we've made our per request.

# 5. Win!

That's it! Run your project and be happy that you've isolated your domain, used entity framework and MVC, got dependency injection and everything is super squeaky clean.

Be sure to check out the [sample project on GitHub](https://github.com/xwipeoutx/CopThis)! 

Please if you have any suggestions for improvement - especially a more reliable (but still simple) transactional consistency, let me know on the twitters!

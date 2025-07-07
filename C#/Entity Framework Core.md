
## Repos

- https://github.com/gavilanch/Introduction-To-Entity-Framework-Core/blob/main/6.0/Module%209/End/EFCoreMovies/EFCoreMovies/Entities/Configurations/InvoiceConfig.cs
## Introduction

- EF Core is an evolution of EF core, has more features like being able to work with relational databases

## Code First

We start with the code and then create the database via code. Advantage is 1 - database is always in sync with the app and 2- if you need to deploy in multiple environments you don't need to worry about creating the database. 

## Database First

We start with an already created database and configure EF to use that database.

## When to use EF Core

1 - When you have a lot of CRUDs, EF does this out of the box and you can re-use a lot of code
2 - When you need to support multiple platforms (linux pe) or multiple database engines (sql, pgsql)

## When NOT to use EF Core

1 - When speed is key. EF Core is fast BUT it's will never be faster than a pure query
2 - Bulk operations - when you need to delete 100000000 rows
3 - (obvs) when EF core is not supported by the DB engine


## Configuring EF Core in .NET

First thing you need to do is to create an `ApplicationDbContext` . This needs to inherit from `DbContext`. This is where you tell EF that you want to use a specific database engine or connection string.

```C#
internal class ApplicationDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("Server=localhost,1433;Database=EFCoreConsoleApp;User Id=SA;Password=YourStrong!Passw0rd;TrustServerCertificate=True;");
    }
}
```

The next thing is how to tell EF what your entities are. To do that, you define your entity as a property of type `DbSet` inside the `ApplicationDbContext` class

```C#
internal class ApplicationDbContext : DbContext
{
    public DbSet<Person> People { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("Server=localhost,1433;Database=EFCoreConsoleApp;User Id=SA;Password=YourStrong!Passw0rd;TrustServerCertificate=True;");
    }
}
```

*note: `DbContext` comes from the `EfCore` package installed via Nuget.*
### For WebApps

If you are doing the setup for a web app, you need to register the EF core service with the dependency injection container and you most likely want the configuration string to come from your environment file. Therefore in this case you need to do more setup in `Program.cs`

```C#
builder.Services.AddDbContext<ApplicationDbContext>(options =>
{
    options.UseSqlServer("name=DefaultConnection");
});
```

Note that in the example above, `name=DefaultConnection` is an alternative way of referring to your `appSettings.json` file. This simply means "use the key `DefaultConnection`in the settings file as the connection string". Also, in this case I used SQL Server but if I wanted to use a different provider you would do `options.YourProvider`

note: `options.UseSqlServer` comes from the `EfCore.SqlServer` package installed via Nuget.

# Configuring primary keys

In EF core there are some conventions that rule the way a database is created, unless you specify otherwise.

1.  if an entity has an `Id` property, then it will be the primary key
2. if an entity is named `{something}Id`, then it will also be the primary key

# Configuring everything else

What if you want or need to break the convention? In that case you have 2 options: use data annotations or use fluent API.

Find a few examples below:

## Conventions

```C#
// in AppDbContext
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    base.ConfigureConventions(configurationBuilder);

    configurationBuilder.Properties<DateTime>().HaveColumnType("date");
    configurationBuilder.Properties<string>().HaveMaxLength(150);
}
```

### Changing primary key

```C#
//data annotations
public class Genre
{
    [Key]
    public int Identifier { get; set; }
    public string Name { get; set; }
}
```

```C#
//fluent-API, this is done in ApplicationDbContext
 protected override void OnModelCreating(ModelBuilder modelBuilder)
 {
     base.OnModelCreating(modelBuilder);

     modelBuilder.Entity<Genre>().HasKey(p => p.Identifier);
 }
```

### Define property requirements

```C#
public class Genre
{
    [Key]
    public int Identifier { get; set; }
    [StringLength(150)]
    [Required]
    public string Name { get; set; }
}
```

```C#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<Genre>().HasKey(p => p.Identifier);
    modelBuilder.Entity<Genre>().Property(p => p.Name).HasMaxLength(150).IsRequired();
}
```

### Change the Database schema (renaming things)

```C#
 modelBuilder.Entity<Genre>()
     .ToTable(name: "GenresTable", schema: "movies")
     .HasKey(p => p.Identifier);

 modelBuilder.Entity<Genre>()
     .Property(p => p.Name)
     .HasColumnName("GenreName")
     .HasMaxLength(150)
     .IsRequired();
```
	
```C#
[Table(name: "GenresTbl", Schema ="movies")]
public class Genre
{
    public int Identifier { get; set; }

	[Column("GenreName")]
    public string Name { get; set; }
}
```

### Changing a data type in the database (ie changing the type of a column)

```c#
 public class Actor
 {
     public int Id { get; set; }
     public string Name { get; set; }
     public string? Biography { get; set; }

     [Column(TypeName ="date")]
     public DateTime DateOfBirth { get; set; }
 }
```

```C#
 modelBuilder.Entity<Actor>()
     .Property(p => p.DateOfBirth)
     .HasColumnType("date");
```

*note about `DateTime` in C# vs how you store dates in SQL: a `DateTime` in C# holds a date and a time. Sometimes you may not want that - imagine for example storing date of birth. In that case you need to tell Entity Framework that the type of the column (if using SQL Server) needs to be of type `date` and not `datetime`

### Changing the precision of floating point numbers

```C#
[Precision(precision:9, scale: 2)]
public decimal Price { get; set; }
```

```C#
modelBuilder.Entity<Cinema>()
    .Property(p => p.Price)
    .HasPrecision(9,2)
    .IsRequired();
```

*note: 9,2 means "I will have 9 numbers in total with 2 being to the right of the comma"*

## Modelling relationships between models

### One-to-one relationships

```C#
public class Cinema
{
    public string Name { get; set; }
    public int Id { get; set; }

    public decimal Price { get; set; }

    public Point Location { get; set; }
    public CinemaOffer CinemaOffer { get; set; }
}
```

### One-To-Many relationships

Many to many relationships are trickier because you need to decide if you want control over the intermediate table that connects the 2 entities. Imagine you want to model a relationship between movies and genres, where a movie can belong to many genres AND a genre can have many movies. In SQL you need to create a table that links movies and genres. This will have a composite primary key with `MovieId` and `GenreId` (because the combination of movie and genre needs to be unique). The questions you need to ask is: is this table going to need to have more information other than the linking? If the answer is "no", you can model the many-to-many relationship in a "skip" style. What does that mean? It means you let EF deal with creating the intermediate linking table. You don't have to create a "linking" model manually.

Here's an example of a many-to-many linking between movies and genres in a skip style fashion.

```c#
public class Movie
{
    public int Id { get; set; }
    public string Title { get; set; }
    public bool InCinemas { get; set; }
    public DateTime ReleaseDate { get; set; }
    public string PosterUrl { get; set; }

    public HashSet<Genre> Genres { get; set; }
}
```

```C#
public class Genre
{
    public int Id { get; set; }

    public string Name { get; set; }

    public HashSet<Movie> Movies { get; set; }
}
```

There are scenarios where you may need some extra information in your linking table other than just the actual link. Imagine you have a linking table that links actors and movies but you also other information such as the name of the character the actor played in the movie or the order in which it appeared in. In that case, the linking table is not just the link, there's also other information there. Therefore you cannot let EF do the work for you, you need to create the linking model yourself  and specify what will get added to the linking table.

Here's an example of what the linking model looks like in this case:

```c#
public class MovieActor
{
    public int MovieId { get; set; }
    public int ActorId { get; set; }
    public string Character { get; set; }
    public int Order { get; set; }
    
    public Movie Movie { get; set; } //this property is just a navigation property to allow easier querying
    public Actor Actor { get; set; } //this property is just a navigation property to allow easier querying
}
```

Your `Movie` and `Actor` classes will also need to have a reference to this model:

```C#
public class Movie
{
    public int Id { get; set; }
    public string Title { get; set; }
    public bool InCinemas { get; set; }
    public DateTime ReleaseDate { get; set; }
    public string PosterUrl { get; set; }

    public HashSet<Genre> Genres { get; set; }
    public HashSet<CinemaHall> CinemaHalls { get; set; }
    public HashSet<MovieActor> MoviesActors { get; set; }
}
```

```C#
public class Actor
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string? Biography { get; set; }

    public DateTime DateOfBirth { get; set; }
    public HashSet<MovieActor> MoviesActors { get; set; }

}
```

The last thing you need to do now is tell EF how to create the primary key for the linking table. This is done in `AppDbContext`

```c#
 modelBuilder.Entity<MovieActor>()
     .HasKey(p => new { p.MovieId, p.ActorId });
```

## Seeding data

The simplest way to seed data is to do it manually. This is how you do it:

```C#
var drama = new Genre { Id = 5, Name = "Drama" };

modelBuilder.Entity<Genre>().HasData(action, animation, comedy, scienceFiction, drama);
```

If you are trying to seed a linking entity that has created automatically (meaning you did it in skip style), you need to let EF the name of the table.



```C#
 var entityGenreMovie = "GenreMovie";

 modelBuilder.Entity(entityGenreMovie).HasData(
     new Dictionary<string, object>
     {
         [genresId] = action.Id,
         [moviesId] = avengers.Id
     },
      new Dictionary<string, object>
      {
          [genresId] = scienceFiction.Id,
          [moviesId] = avengers.Id
      }
     );
```

## Querying data

Querying data always means using the DB context in a LINQ like way. Here are a few examples:

```C#
[HttpGet]
public async Task<IEnumerable<Genre>> Get()
{
    return await this.dbContext.Genres.ToListAsync();
}

  [HttpGet("first")]
  public async Task<ActionResult<Genre>> First()
  {
      var genre = await this.dbContext.Genres.FirstAsync(g => g.Name.Contains("m"));

      if(genre == null)
      {
          return NotFound();
      }

      return Ok(genre);
  }

  [HttpGet("filter")]
  public async Task<ActionResult<Genre>> Filter(string name)
  {
      var genres = await this.dbContext.Genres.Where(g => g.Name.Contains(name)).ToListAsync();

      return Ok(genres);
  }

[HttpGet("paged")]
public async Task<ActionResult<IEnumerable<ActorDTO>>> Paged(int page, int recordsToTake)
{
    var actors = await this.dbContext.Actors
        .OrderBy(g => g.Name)
        .Select(a => new ActorDTO { Id = a.Id, Name = a.Name, DateOfBirth = a.DateOfBirth})
        .Paginate(page, recordsToTake)
        .ToListAsync();

    return Ok(actors);
}
```

## Eager Loading vs Explicit Loading vs SelectLoading Vs LazyLoading vs DeferExecution

Eager loading means  "load a navigation property". The way you do that is by using the `Include` method of the db context class

```C#
var movie = await dbContext.Movies
    .Include(m => m.Genres)
    .FirstOrDefaultAsync(m => m.Id == id);
```

What's crucial here is to understand the circular dependency error that will come up with your models have circular dependencies. If your movies has a list of genres and your genres have a list of movies you have a circular dependency. This is NOT a database problem, your database is fine. This is a serialization problem. It just means EF doesn't know how to map the return from the database into JSON due to that circular dependency. Therefore you need to tell it to ignore circular dependencies. The strange thing here is that this is not something you tell EF, this is something you configure at application level. Your App has a JsonSerializer that EF uses.

```C#
builder.Services.AddControllers().AddJsonOptions(options =>
{
    options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
});
```

ExplicitLoading is when you do things one step at a time. First you get the movies, then you get the genres, etc...

```C#
[HttpGet("explicitloading/{id:int}")]
public async Task<ActionResult<MovieDTO>> ExplicitLoading(int id)
{
    var movie = await dbContext.Movies.FirstOrDefaultAsync(m => m.Id == id);

    if (movie is null) return NotFound();

    await dbContext.Entry(movie).Collection(p => p.Genres).LoadAsync();

    var count = dbContext.Entry(movie).Collection(p => p.Genres).Query().CountAsync();

    var movieDTO = mapper.Map<MovieDTO>(movie);
    return Ok(movieDTO);
}
```

In terms of performance this is worse than doing things in one go because it's more queries. Still, it can be useful if you want to do something like getting a count or conditional logic. 

SelectLoading is a more flexible way of doing queries. Essentially you ignore `include`, `thenInclude` and Automapper and just go rogue and build an anonymous object the way you want it to be. A downside I see if this approach is that your type safety goes a bit out the window. Notice that in the example below, you tell C# you are returning a `MovieDTO` but the anonymous object can have whatever you want in it.

```C#
 [HttpGet("selectloading/{id:int}")]
 public async Task<ActionResult<MovieDTO>> SelectLoading(int id)
 {
     var movieDTO = await dbContext.Movies.Select(m => new
     {
         Id = m.Id,
         Title = m.Title,
         Genres = m.Genres.Select(g => g.Name).ToList(),
         Fruit = "Bananas"
     }
     ).FirstOrDefaultAsync(m => m.Id == id);

     if (movieDTO is null) return NotFound();
     return Ok(movieDTO);
 }
```

LazyLoading is when you defer the loading of navigation properties only to the point where they are needed in the code. For example:

```C#
var movies = context.Movies.ToList(); // 1 query - load the movies

foreach (var movie in movies)
{
    Console.WriteLine(movie.Genre.Name); // 1 query per movie (lazy loading) - load the genres
}
```

This is efficient when you know that you only need related (navigational) data sometimes and don't want to manually load things like when doing eager loading (because you manually type the code to the loading). However this is really dangerous.

This is dangerous because of the n+1 problem. That simply means, as in the example above, that if you are looping through a list you will fire 1 query to get the data for a single object. This is really inefficient, you are far better off using a `JOIN`. It's n+1 because n refers to the first query to load the parent entity (movies) and +1 because it's one extra database roundtrip for each navigation property on the entity (genres).

Still, if you want to use it, you need to:
1 - Install the proxies package
2 - Mark the navigation properties virtual
3 - Configure EF core to use Lazy Loading

DefferedExecution is a bit of a weird one. Essentially you build the query but only fire it when you materialize it (like calling `toList`). Think of this like writing queries in SQL. You write stuff out but the query only executes once you press F5. The advantage is that you can do conditional logic like this without firing a query all of a sudden.

```C#
[HttpGet("filter/{id:int}")]
public async Task<ActionResult<IEnumerable<MovieDTO>>> Filtered([FromQuery] MovieFilterDTO movieFilterDTO)
{
    var moviesQuery = dbContext.Movies.AsQueryable();

    if (!string.IsNullOrEmpty(movieFilterDTO.Title))
    {
        moviesQuery = moviesQuery.Where(m => m.Title.Contains(movieFilterDTO.Title));
    }

    if (movieFilterDTO.InCinemas)
    {
        moviesQuery = moviesQuery.Where(m => m.InCinemas);
    }

    if (movieFilterDTO.UpcomingReleases)
    {
        var today = DateTime.Today;
        moviesQuery = moviesQuery.Where(m => m.ReleaseDate > today);
    }

    if (movieFilterDTO.GenreId != 0)
    {
        moviesQuery = moviesQuery
                        .Where(m => m.Genres.Select(g => g.Id).Contains(movieFilterDTO.GenreId));
    }

    var movies = await moviesQuery.Include(m => m.Genres).ToListAsync();

    return mapper.Map<List<MovieDTO>>(movies);
}
```
## Include Vs ThenInclude

`Include` is when you want to load a navigation property, `thenInclude` is when you want to load a property on a navigation property. *You cannot perform ordering in the `thenInclude` block*

## Ordering/Filtering and a note on Automapper

Stuff like ordering and filtering, if not using automapper, is always done in a LINQ like way:

```C#
  var movie = await dbContext.Movies
      .Include(m => m.Genres.OrderByDescending(g => g.Name)) //ordering
      .Include(m => m.CinemaHalls)
          .ThenInclude(ch => ch.Cinema)
      .Include(m => m.MoviesActors)
          .ThenInclude(ma => ma.Actor)
      .FirstOrDefaultAsync(m => m.Id == id);
```

However, if you are using automapper, then the filtering/ordering logic needs to be added to the mapping profile instead:

```C#
 public AutoMapperProfiles()
 {
     CreateMap<Movie, MovieDTO>()
         .ForMember(dto => dto.Genres, ent => ent.MapFrom(m => m.Genres.OrderByDescending(g => g.Name)))
 }
```

Your db context if using automapper just need to use the `ProjectTo` method to tell EF core to use automapper

```C#
 var movieDTO = await dbContext.Movies
     .ProjectTo<MovieDTO>(mapper.ConfigurationProvider)
     .FirstOrDefaultAsync(m => m.Id == id);
```
 
## Creating, Updating and Deleting data

First thing before doing any of this is to understand the concept of *Tracking*

### Tracking

In EF, when date/entities are loaded or saved in the database, EF starts a process called tracking. Tracking is the process that monitors changes to the entities so that at later point (when you call `context.saveChanges()`), EF knows what needs to be updated in the database and what needs to be skipped.

By default tracking is *enabled*. This default behaviour is useful when you need to do Create, Update or Delete operations.

```C#
var product = context.Products.FirstOrDefault(p => p.Id == 1);
// Changes will be tracked
product.Price = 20;
context.SaveChanges(); // Updates the Price in the DB
```

In contrast, you can use EF in *no-tracking* mode. The advantage of using this is it frees up memory because EF does not need to track all entities. This is ok to use when you are only doing Read operations because the entity will/shouldn't change anyway.

```C#
var product = context.Products.AsNoTracking().FirstOrDefault(p => p.Id == 1);
product.Price = 20;
context.SaveChanges(); // No effect — EF is not tracking this entity
```

#### Tracking States

An entity that is tracked by EF will always have a state, and the state will change according to the operations that are performed on the entity.

Available states are:

|**State**|**Description**|
|---|---|
|`Added`|Entity is new and will be inserted into the database on `SaveChanges()`|
|`Unchanged`|Entity hasn't changed since it was queried or attached|
|`Modified`|Entity has been changed and will be updated in the database|
|`Deleted`|Entity has been marked for deletion and will be removed from the database|
|`Detached`|Entity is not being tracked by the context at all|

A very important note on status: if you call `Add` on an entity, **it will only be added if it was previously untracked by EF**. For example, if you do:

```C#
var movie = mapper.Map<Movie>(movieCreationDTO);

movie.Genres.ForEach(g => dbContext.Entry(g).State = EntityState.Unchanged);
movie.CinemaHalls.ForEach(ch => dbContext.Entry(ch).State = EntityState.Unchanged);

if (movie.MoviesActors is not null)
{
    for (int i = 0; i < movie.MoviesActors.Count; i++)
    {
        movie.MoviesActors[i].Order = i + 1;
    }
}

dbContext.Add(movie); //Only movie is marked as "Added", Genres and CinemaHalls don't
await dbContext.SaveChangesAsync();
```

...only movie is marked as "Added", Genres and CinemaHalls don't. If an entity as a status, then it won't be affected by the `Add` method.

### Using backing fields (Flexible mapping)

Sometimes you may want to use getters/setters to do transformations before the entity gets saved in the database. If you follow the convention of having the name of your field starting with `_{PropertyName}` you don't need to do anything else, EF will be able to pick this up out of the box. If you break away from this convention, you need to explicitly tell EF that the field exists.

```C#
	builder.Property(p => p.Name).HasField("whateverTheNameOfMyFieldIs");
```

### Connected Model vs Disconnected Model

`ConnectedModel` is when you use the same instance of the `dbContext` object to do things. For example a PUT where you first get an entity by Id and then update it using the same context object. The key aspect here is that you retrieved the entity via the context so EF can track it

```C#
[HttpPut("{id:int}")]
public async Task<ActionResult> Put(ActorCreationDTO actorCreationDTO, int id)
{
    var actorDB = await dbContext.Actors.FirstOrDefaultAsync(p => p.Id == id);

    if (actorDB is null)
    {
        return NotFound();
    }

    actorDB = mapper.Map(actorCreationDTO, actorDB);
    await dbContext.SaveChangesAsync();
    return Ok();
}
```


`DisconnectedModel` is when you don't retrieve (or just create a new entity) in a way EF cannot possibly know about changes to that entity. In this scenario 2 things can happen:

1. you call `context.Update(entity)` as EF will assume that this is a full update, meaning the generated SQL will update all the columns in the database regardless if they have changed or not or;
2. you explicitly tell EF what columns you want to modify, so that it keeps the generated SQL only with the columns you will change

```C#

        [HttpPut("disconnected/{id:int}")]
        public async Task<ActionResult> PutDisconnected(ActorCreationDTO actorCreationDTO, int id)
        {
            var existsActor = await dbContext.Actors.AnyAsync(p => p.Id == id);

            if (!existsActor)
            {
                return NotFound();
            }

            var actor = mapper.Map<Actor>(actorCreationDTO);
            actor.Id = id;

            dbContext.Update(actor);
            await dbContext.SaveChangesAsync();
            return Ok();
        }
```

```C#
// Step 1: Get data from DB (maybe in a previous request or layer)
Product productFromClient = new Product { Id = 1, Price = 200 };

// Step 2: Attach and mark it as modified
using (var context = new AppDbContext())
{
    context.Attach(productFromClient);
    context.Entry(productFromClient).Property(p => p.Price).IsModified = true;
    context.SaveChanges(); // EF updates only the Price column
}

```

## Deleting / Soft - Deleting records

Deleting as the name implies means removing a record from the table.
Soft-Deleting means do not remove the record from the database but don't load it via EF. Think analytics or auditing. Note that is NOT something specific or EF magic. This is really just adding a "deleted" column to the database and update/return records accordingly. You can do this with any other framework.

## Query Filters

Soft deleting records as per above as one problem: you need to filter every single time you query the database. To fix this EF allows to configure filters that are applied to the entity every time you query the database

```C#
 public class GenreConfig : IEntityTypeConfiguration<Genre>
 {
     public void Configure(EntityTypeBuilder<Genre> builder)
     {
             builder.Property(p => p.Name)
             .HasMaxLength(150)
             .IsRequired();

         builder.HasQueryFilter(p => !p.IsDeleted); //apply this to every database query
     }
 }
```

On the other hand, you now have a problem because you never return the (soft) deleted items. To bypass that, EF allows you to ignore the query filters for certain queries.

```c#
var genre = await dbContext.Genres.IgnoreQueryFilters().FirstOrDefaultAsync(p => p.Id == id);
```

## Configuring Properties

Primary Keys / GUIDs: By convention, the property named "Id" will be a primary key. Know that if you make this a GUID, EF will add the record as a GUID in the database. These GUIDs are sequential for better performance. You can however configure EF not to do that and just create the GUIDs manual. **Note that every primary key is automatically configured as an index**

Ignoring properties / classes - sometimes you may not want to have a property or a class mapped as entity in the database. For example your user can have an "Age" field but you can derive that from the date of birth. Or, for example, you can have an Address class that you populate from a postcode search API. In scenarios like this you can tell EF not to map the entity to the database.

```C#
[NotMapped] //dont add this entity to the database
public class Address
{
    public int Id { get; set; }
    public string Street { get; set; }
    public string Province { get; set; }
    public string Country { get; set; }
}
```

```C#
//AppDbContext

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);
    modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    Module3Seeding.Seed(modelBuilder);
    modelBuilder.Ignore<Address>();
}
```

Creating Indexes - you can tell EF to create other indexes via configuration. For example:

```C#
[Index(nameof(Name), IsUnique = true)]
public class Genre
{
    public int Id { get; set; }
    public string Name { get; set; }

    public HashSet<Movie>? Movies { get; set; }
    public bool IsDeleted { get; set; }
}
```

```C#
public void Configure(EntityTypeBuilder<Genre> builder)
{
    builder.HasIndex(p => p.Name).IsUnique();
}
```

Indexes with filters - this one is a bit weird but essentially means "only include rows where `isDeleted` is false". So only check for duplicates where `isDeleted=false` . The use case for this is soft deletes: rows marked as deleted (`isDeleted=true`) are **excluded** from the uniqueness constraint.

```C#
 public void Configure(EntityTypeBuilder<Genre> builder)
 {
     builder.HasIndex(p => p.Name).IsUnique().HasFilter("isDeleted=false");
 }
```
 
Value conversions - Sometimes it's useful to tell EF that things like enums are to be stored as their string representation in the database. To do that we use value conversions.

```C#
void IEntityTypeConfiguration<CinemaHall>.Configure(EntityTypeBuilder<CinemaHall> builder)
{
    builder
        .Property(p => p.CinemaHallType)
        .HasDefaultValue(CinemaHallType.TwoDimensions)
        .HasConversion<string>();
}
```

Custom conversions - same logic as before but you can specify your own custom conversions.

Keyless Entities - as the name suggests this when you have entities that do not have a key. This is useful for cases when you need to arbitrary run a SQL query or your data does not have/need a key. Think for example retrieving the name of all customers or all cinemas. The advantage of this is that you still get all the benefits of EF in terms of mapping a (keyless) entity to a C# model. This is useful for running arbitrary queries or readonly models.

Views - this links a bit to keyless entities. One of the cool things you can do is to create a view in your table (create an empty migration an just dump the view SQL there), and use EF to map that view into a keyless model. This is great for read only queries

Shadow Properties - useful for data in your table that is not meant to be in your entity. Think fields like lastUpdatedAt or CreatedDate

How do you tell EF that it needs to add a column? Like this:

```C#
 builder.Property<DateTime>("CreatedDate").HasDefaultValueSql("GetDate()").HasColumnType("datetime2");
```

How do you use that column considering it's "hidden"? Like this:

```C#
[HttpGet]
public async Task<IEnumerable<Genre>> GetLogs()
{
    return await dbContext.Genres.AsNoTracking()
        .OrderByDescending(g => EF.Property<DateTime>(g, "CreatedDate"))
        .ToListAsync();
}
```

## Configuring relationships between entities

### Cascade on Delete

By default, navigation properties that are required, have a delete cascade behaviour. However there are other types of delete behaviour}

 - Cascade
 - Client Cascade - same as cascade but requires you to load the related entity first. Think of this as "delete from the client"
 - No Action - do nothing
 - Client No Action - do nothing
 - Restrict - This means "throw an exception if the parent entity has a related entity". The idea here is that you force your user to first delete the related entity first and then the parent entity
 - SetNull - set related entity to null
 - Client SetNull - same logic as before

### Non-required navigation properties

This one is a bit weird but imagine you have an optional navigation property. In terms of SQL, EF will still configure this as a PK->FK relationship, so if you try to delete the PK without updating the FK this will cause an error (FK violation). EF solves this is in a weird way: essentially you need to eager load the relationship so that EF knows that it needs to update the FK:

```C#
// by loading CinemaHalls, EF knows that it needs to update the FK to NULL
var cinema = await dbContext.Cinemas.Include(p => p.CinemaHalls).FirstOrDefaultAsync(p => p.Id == id);

if (cinema is null)
{
    return NotFound();
}

dbContext.Remove(cinema);
await dbContext.SaveChangesAsync();
return Ok();
```

### Foreign Keys

By default, the way EF knows that it needs to create a FK relationship is via naming conventions. For example, if you have a `Cinema` entity, and a `CinemaHall` , EF will assume that `CinemaId` is the FK between the two entities. 

```C#
 public class Cinema
 {
     public string Name { get; set; }
     public int Id { get; set; }
     public Address Address { get; set; }
     public Point Location { get; set; }
     public CinemaOffer CinemaOffer { get; set; }
     public HashSet<CinemaHall> CinemaHalls { get; set; }
 }

public class CinemaHall
{
    public int Id { get; set; }
    public decimal Cost { get; set; }
    public int CinemaId { get; set; } // This will be the foreign key
    public Cinema Cinema { get; set; }
    public CinemaHallType CinemaHallType { get; set; }
    public HashSet<Movie> Movies { get; set; }
    public Currency Currency { get; set; }
}
```

If you want to change this, you need to do it explicitly via data annotations:

```C#
public class CinemaHall
{
    public int Id { get; set; }
    public decimal Cost { get; set; }
    public int TheCinemaId { get; set; } // This will be the foreign key
    [ForeignKey(nameof(TheCinemaId))]
    public Cinema Cinema { get; set; }
    public CinemaHallType CinemaHallType { get; set; }
    public HashSet<Movie> Movies { get; set; }
    public Currency Currency { get; set; }
}
```
### Inverse Property attribute

This only applies when you have a "more than one FK" relationship. WTF is that? Consider the following:

```C#
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Message> SentMessages { get; set; }
    public List<Message> ReceivedMessages { get; set; }
}

public class Message
{
    public int Id { get; set; }
    public string Content { get; set; }
    public int SenderId { get; set; }
    public Person Sender { get; set; }
    public int ReceiverId { get; set; }
    public Person Receiver { get; set; }
}
```

In this case of a chat application, a person will have received and sent messages. But how can EF know what messages to load? You are specifying a one-to-many relationship between Person -> Message but remember EF only  looks at the type (`Person` and `List<Message>`), `SentMessages` or `ReceivedMessages` means nothing to EF.

So if you ask for "give me all the sent messages for a user" how can EF know if you mean sent or received messages just from `List<Message>`? You need to tell it.

```C#
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    [InverseProperty("Sender")]
    public List<Message> SentMessages { get; set; }
    
    [InverseProperty("Receiver")]
    public List<Message> ReceivedMessages { get; set; }
}
```
### Configuring relationships using the Fluent API

#### 1 to 1

```C#
// cinema
builder.hasOne(c => c.CinemaOffer).hasOne() //one cinema has one cinemaOffer.

//note that if a CinemaOffer had a navigation property on Cinema, then you'd need to include it in the hasOne()
```

#### 1 to many

```C#
builder.hasMany(c => c.CinemaHalls).withOne(ch => ch.Cinema)
```

#### Many to Many

If using an intermediate entity (no-skip style) you need to specify both relationships

```C#
builder.HasOne(p => p.Actor).WithMany(p => p.MoviesActors).HasForeignKey(p => p.ActorId);

builder.HasOne(p => p.Movie).WithMany(p => p.MoviesActors).HasForeignKey(p => p.MovieId);
```

If not using an intermediate entity you just need to specify a many-to-many

```C#
  builder.HasMany(p => p.Genres).WithMany(p => p.Movies);
```

### Table Splitting

The point of table splitting is breaking a complex table (and just ONE table) into multiple entities. Imagine you have Cinema and Cinema Details but the details of the cinema are all kept in the cinemas table in the database. In order to make your Cinema model smaller on the C# side of things you can tell EF that you want CinemaDetail to be its own model. 

To do that you need to add a navigation property AND you need to tell EF that CinemaDetail goes to the "cinema" table.

*note: there's a weird thing you need to do which is to make one of the properties on the related entity required*

```C#
public class CinemaConfig : IEntityTypeConfiguration<Cinema>
{
    public void Configure(EntityTypeBuilder<Cinema> builder)
    {
          builder.Property(p => p.Name)
          .IsRequired();

        builder.HasMany(p => p.CinemaHalls).WithOne(ch => ch.Cinema).HasForeignKey(ch => ch.CinemaId).OnDelete(DeleteBehavior.Restrict);

        builder.HasOne(c => c.CinemaDetail).WithOne(c => c.Cinema).HasForeignKey<CinemaDetail>(cd => cd.Id);
    }
}
```

```C#
 public class CinemaDetailConfig : IEntityTypeConfiguration<CinemaDetail>
 {
     public void Configure(EntityTypeBuilder<CinemaDetail> builder)
     {
         builder.ToTable("Cinemas");
     }
 }
```

### Owned Types

"We are centralising what an address is, and then we are using it several other entities"

The idea with owned types is to re-utilize models. Imagine you have an Address model that is used by the Person and Cinema models. You want every person to have an Address but you also want every cinema to have an Address. Also, you want the Address model to be exactly the same across your code base AND you don't want to have a table of addresses. OwnedTypes achieve just that. It's essentially a way of re-utilizing and doing model composition.

*note: the same weird thing about having to have a required property still applies*

```C#
 [Owned]
 public class Address
 {
     public string Street { get; set; }
     public string Province { get; set; }
     [Required]
     public string Country { get; set; }
 }
```

### Inheritance - Table per Hierarchy

Imagine you want to have a payments model and that your users can either pay by credit card or paypal. You want to store all this in one database. Besides if it's a card payment then store the last 4 digits of the card, if its a paypal payment then store the email. The table looks like this:


| Id  | Payment Type | Last 4 Digits | Email             | Amount | Date       |
| --- | ------------ | ------------- | ----------------- | ------ | ---------- |
| ABC | Paypal       | NULL          | someone@email.com | 20     | 21/06/2025 |
| DEF | Card         | 1234          | NULL              | 10     | 22/06/2025 |
How do you model this relationship in Entity Framework? Note that some properties are common to all payments such as amount and date.

In C# you would solve this using abstract classes.

```C#
 public abstract class Payment
    {
        public int Id { get; set; }
        public decimal Amount { get; set; }
        public DateTime PaymentDate { get; set; }
        public PaymentType PaymentType { get; set; }
    }

 public class PaypalPayment: Payment
    {
        public string EmailAddress { get; set; }
    }
 public class CardPayment: Payment
    {
        [StringLength(4)]
        public string Last4Digits { get; set; }
    }

```

But how do you translate that into EF? How do you tell EF that it needs to create just one table that can hold these 2 models?

You need to configure EF to use a discriminator:

```C#
 public class PaymentConfig : IEntityTypeConfiguration<Payment>
    {
        public void Configure(EntityTypeBuilder<Payment> builder)
        {
            builder.Property(p => p.Amount).HasPrecision(18, 2);

            builder.HasDiscriminator(p => p.PaymentType)
                .HasValue<PaypalPayment>(PaymentType.Paypal)
                .HasValue<CardPayment>(PaymentType.Card);
        }
    }
```

Essentially a discriminator is a way of telling EF what type of payment to save based on an enum:

```C#
 public enum PaymentType
    {
        Paypal = 1,
        Card = 2
    }
```


### Inheritance - Table per Type

Now imagine the opposite scenario where you have 2 classes that somehow derive from a parent class but are so distinct that you want to store them in separate tables. For example:

```c#
 public class Merchandising: Product
    {
        public bool Available { get; set; }
        public double Weight { get; set; }
        public double Volume { get; set; }
        public bool IsClothing { get; set; }
        public bool IsCollectionable { get; set; }
    }
    
public class RentableMovie: Product
    {
        public int MovieId { get; set; }
    }
```

In this case what you do is create a DbSet for the parent class - Product - but tell that Merchandising and RentableMovie have their own table:

```c#
 protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            
            modelBuilder.Entity<Merchandising>().ToTable("Merchandising");
            modelBuilder.Entity<RentableMovie>().ToTable("RentableMovies");
        }
```

Then, when you do need to load the data you tell the context what type of data you want.

```C#
[ApiController]
    [Route("api/products")]
    public class ProductsController: ControllerBase
    {
        private readonly ApplicationDbContext context;

        public ProductsController(ApplicationDbContext context)
        {
            this.context = context;
        }

        [HttpGet]
        public async Task<ActionResult<IEnumerable<Product>>> Get()
        {
            return await context.Products.ToListAsync();
        }

        [HttpGet("merch")]
        public async Task<ActionResult<IEnumerable<Merchandising>>> GetMerch()
        {
            return await context.Set<Merchandising>().ToListAsync();
        }

        [HttpGet("rentables")]
        public async Task<ActionResult<IEnumerable<RentableMovie>>> GetRentables()
        {
            return await context.Set<RentableMovie>().ToListAsync();
        }
    }
```

## Running migrations from within the application

```C#

// in Program.cs
using (var scope = app.Services.CreateScope())
{
    var applicationDbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    applicationDbContext.Database.Migrate();
}
```

Problems with this:
- Slower starts
- If migrations take too long to run database can timeout
- Hard to debug if things go wrong because application won't start.

## Compiled Models

Compiled models are a performance optimization in EF core that allows you to pre-compile your models so that startup time is improved. 

How do you build them? Like this:

`dotnet ef dbcontext optimize`

Where do you apply them? Here:

```C#
builder.Services.AddDbContext<ApplicationDbContext>(options => {
    options.UseSqlServer("name=DefaultConnection", sqlServer => sqlServer.UseNetTopologySuite());
    options.UseModel(ApplicationDbContextModel.Instance); //Compiled models here
    });
```

*note: compiled models don't work if you have query filters AND if your model changes you need to compile the models again using the command above.*

## DBContext

### OnConfiguring

`DbContext` has a method you can override to do configurations. You don't necesserailly need to do it in `Program.cs`.  Note that if you choose to do it here you still need to create an **empty** constructor because you will need to register the DBContext as a service in the DI container.

```C#
public ApplicationDbContext()
{
    
}

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer("name=DefaultConnection", sqlServer => sqlServer.UseNetTopologySuite());
}
```

### Changing the state of the entity directly

Essentially doing this:

```c#
context.Entry(genre).State = EntityState.Added
await context.saveChangesAsync()
```

Is the same as doing

```C#
context.Add(genre);
await context.saveChangesAsync()
```

### Updating only some columns

*note: this is the disconnected model of Entity Framework*

What if you just want to update one column rather than the whole model? You can do it like this:

```c#
context.Entry(actor).Property(p => p.Name).IsModified = true;
await context.SaveChangesAsync

```

### Override SaveChanges

Imagine you want to update a CreatedBy and UpdatedBy to every entity. Rather than doing it manually on EVERY model you can override `saveChanges` to do that:

```C#
 public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        {
            ProcessSaveChanges();
            return base.SaveChangesAsync(cancellationToken);
        }

        private void ProcessSaveChanges()
        {
            foreach (var item in ChangeTracker.Entries()
                .Where(e => e.State == EntityState.Added && e.Entity is AuditableEntity))
            {
                var entity = item.Entity as AuditableEntity;
                entity.CreatedBy = userService.GetUserId();
                entity.ModifiedBy = userService.GetUserId();
            }

            foreach (var item in ChangeTracker.Entries()
                .Where(e => e.State == EntityState.Modified && e.Entity is AuditableEntity))
            {
                var entity = item.Entity as AuditableEntity;
                entity.ModifiedBy = userService.GetUserId();
                item.Property(nameof(entity.CreatedBy)).IsModified = false;
            }
        }

```

### Injecting Services in the DBContext

This is the same thing as injecting any other service. However, services injected in the DBContext need to be **scoped** services. Why?

When you register the DbContext with the DI container, by default it will be created as a scoped service (same instance for request)

```c#
  builder.Services.AddDbContext<ApplicationDbContext>(options =>
  {
      options.UseSqlServer("name=DefaultConnection", sqlServer => sqlServer.UseNetTopologySuite());
  });
```

So, any services you want to inject here need to be scoped too. You cannot inject a singleton for example.

## State Events

Know that EF fires events when things change and you can handle those events if you so wish. Here's an example:

```c#
public ApplicationDbContext(DbContextOptions options, IUserService userService,
            IChangeTrackerEventHandler changeTrackerEventHandler) : base(options)
        {
            this.userService = userService;
            if (changeTrackerEventHandler is not null)
            {
                ChangeTracker.Tracked += changeTrackerEventHandler.TrackedHandler;
                ChangeTracker.StateChanged += changeTrackerEventHandler.StateChangeHandler;
                SavingChanges += changeTrackerEventHandler.SavingChangesHandler;
                SavedChanges += changeTrackerEventHandler.SavedChangesHandler;
                SaveChangesFailed += changeTrackerEventHandler.SaveChangesFailHandler;
            }
        }
```

```C#
public interface IChangeTrackerEventHandler
    {
        void SaveChangesFailHandler(object sender, SaveChangesFailedEventArgs args);
        void SavedChangesHandler(object sender, SavedChangesEventArgs args);
        void SavingChangesHandler(object sender, SavingChangesEventArgs args);
        void StateChangeHandler(object sender, EntityStateChangedEventArgs args);
        void TrackedHandler(object sender, EntityTrackedEventArgs args);
    }

    public class ChangeTrackerEventHandler: IChangeTrackerEventHandler
    {
        private readonly ILogger<ChangeTrackerEventHandler> logger;

        public ChangeTrackerEventHandler(ILogger<ChangeTrackerEventHandler> logger)
        {
            this.logger = logger;
        }

        public void TrackedHandler(object sender, EntityTrackedEventArgs args)
        {
            var message = $"Entity: {args.Entry.Entity}, state: {args.Entry.State}";
            logger.LogInformation(message);
        }

        public void StateChangeHandler(object sender, EntityStateChangedEventArgs args)
        {
            var message = $"Entity: {args.Entry.Entity}, previous state: {args.OldState} - new state: {args.NewState}";
            logger.LogInformation(message);
        }

        public void SavingChangesHandler(object sender, SavingChangesEventArgs args)
        {
            var entities = ((ApplicationDbContext)sender).ChangeTracker.Entries();

            foreach (var entity in entities)
            {
                var message = $"Entity: {entity.Entity} it's going to be {entity.State}";
                logger.LogInformation(message);
            }
        }

        public void SavedChangesHandler(object sender, SavedChangesEventArgs args)
        {
            var message = $"We processed {args.EntitiesSavedCount} entities.";
            logger.LogInformation(message);
        }

        public void SaveChangesFailHandler(object sender, SaveChangesFailedEventArgs args)
        {
            logger.LogError(args.Exception, "Error in SaveChanges");
        }
    }
```

## Arbitrary Queries

You can use EF to run arbitrary queries against the DB. Like this:

```c#

            var cinemaDB = await context.Cinemas.FromSqlInterpolated($"SELECT * FROM Cinemas WHERE Id = {id}")
                   .Include(c => c.CinemaHalls)
                   .Include(c => c.CinemaOffer)
                   .Include(c => c.CinemaDetail)
                .FirstOrDefaultAsync();
```

Note that you can also mix arbitrary queries with EF Core statements like Include. Another thing to note is you need to bring all columns if getting an entity. For example, if you are getting a genre doing `select name from genres` will not work. EF expects you to bring ALL columns so you need to do `select * from genres`

Also, you can use arbitrary queries to add data:

``` c#
    await context.Database.ExecuteSqlInterpolatedAsync($@"
                                                    INSERT INTO Genres(Name)
                                                    VALUES({genre.Name})");

```

Note that if you do it like this, EF will automatically run the query, you don't need `SaveChanges()`

One other thing you can do with arbitrary queries is to map them to a model. This is similar to mapping a sql view to a model.

```c#
 modelBuilder.Entity<MovieWithCounts>().ToSqlQuery(@"SeLeCt Id, Title,
(Select count(*) FROM GenreMovie where MoviesId = movies.Id) as AmountGenres,
(Select count(distinct moviesId) from CinemaHallMovie
	INNER JOIN CinemaHalls
	ON CinemaHalls.Id = CinemaHallMovie.CinemaHallsId
	where MoviesId = movies.Id) as AmountCinemas,
(Select count(*) from MoviesActors where MovieId = movies.Id) as AmountActors
FROM Movies");
```

## Stored Procedures

You can use stored procedures with EF. The way to do is to create an empty migration and add your stored procedure there. However stored procedures are "Non Composable Queries". What does that mean?

It means you cannot use LINQ queries with them. You cannot do things like `Where().Filter().OrderBy()`. So stay away from them

## Transactions

By default EF uses transactions under the hood. You don't need to do anything to use them. What this means is everything that happens before `saveChanges` in invoked is wrapped in a transaction and it's atomic (all or nothing). For example:

```c#
 var cinema = mapper.Map<Cinema>(cinemaCreationDTO);
 
 dbContext.Add(cinema);
 dbContext.Update(actor);
 dbContext.Remove(genre);
 
 await dbContext.SaveChangesAsync(); // all the 3 operations above will be wrapped in a transaction

 return Ok();
```

However there is still the concept of "Transactions" in EF. I think this is confusing so I'll call it "EF Core transaction"

An "EF Core transaction" is when you need to make sure a "all or nothing" behaviour between 2 `SaveChanges()` invocations. For example the classic case of updating products and invoices:

```c#
  [HttpPost]
        public async Task<ActionResult> Post()
        {
            using var transaction = await context.Database.BeginTransactionAsync();

            try
            {
                var invoice = new Invoice()
                {
                    CreationDate = DateTime.Now
                };
                
                context.Add(invoice);
                await context.SaveChangesAsync();

                //throw new ApplicationException("this is an error");

                var invoiceDetail = new List<InvoiceDetail>()
            {
                new InvoiceDetail()
                {
                    InvoiceId = invoice.Id,
                    Product = "Product A",
                    Price = 123
                },
                 new InvoiceDetail()
                {
                     InvoiceId = invoice.Id,
                    Product = "Product B",
                    Price = 456
                }
            };

                context.AddRange(invoiceDetail);
                await context.SaveChangesAsync();
                await transaction.CommitAsync();
                return Ok("all good");
            }
            catch(Exception ex)
            {
                return BadRequest("There was an error");
            }
        }
```

## Advanced Concepts

### User Defined Functions

Imagine you have a database of users and users have invoices (like Amazon for example). Now imagine you want to know what is the sum of all invoices for a user. If you are just using C# you would need to do something like:

```c#
context.Invoices.Where(i => i.id === userId).Reduce(i => i.value)
```

This means you have to do things in C# that you could do in SQL. An alternative for this is to create a user defined function.

A user defined function is nothing more than a function you define in SQL and then invoke via EF.

To define that function you do the same as before: create an empty migration and add the function code there.

Then you need to register those functions with EF. To do that you can register functions directly with `modelBuilder` or you can create a static class as below (cleaner):

```c#
public static class Scalars
    {
        public static void RegisterFunctions(ModelBuilder modelBuilder)
        {
            modelBuilder.HasDbFunction(() => InvoiceDetailAverage(0));
        }

        public static decimal InvoiceDetailAverage(int invoiceId)
        {
            return 0;
        }
       
    }
```

*note: what you return from the function doesn't matter. In the example above we return 0 but this will in truth return whatever SQL returns.* 

The next step is to register the functions with EF:

```C#
 protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            (...)
            Scalars.RegisterFunctions(modelBuilder);
        }
```

In order to use the function, you go via the dbContext has usual.

```C#
 [HttpGet("Scalars")]
        public async Task<ActionResult> GetScalars()
        {
            var invoices = await context.Invoices.Select(f => new
            {
                Id = f.Id,
                Total = context.InvoiceDetailSum(f.Id),
                Average = Scalars.InvoiceDetailAverage(f.Id)
            }).OrderByDescending(f => context.InvoiceDetailSum(f.Id))
            .ToListAsync();

            return Ok(invoices);
        }
```

### Table value functions

Same logic as above, but instead of returning a single value, you now return a result set. Config is slightly different, usage is the same.

```c#
// config
  public IQueryable<MovieWithCounts> MoviesWithCounts(int movieId)
        {
            return FromExpression(() => MoviesWithCounts(movieId));
        }
```

### Computed columns

As the name suggests this is for you to add columns that are the derived result of other columns in the DB via EF. Think for example calculating totals if you have a quantity and a price. You can decide whether this is meant to be saved or computed on the fly. You need to be careful if this is a slow computation because it will slow things down.

```C#
  public class InvoiceDetailConfig : IEntityTypeConfiguration<InvoiceDetail>
    {
        public void Configure(EntityTypeBuilder<InvoiceDetail> builder)
        {
            builder.Property(p => p.Total).HasComputedColumnSql("Quantity * Price");
        }
    }
```

### Creating a sequence

Sometimes you cannot rely solely in EF to create a sequence because you can have "jumps" (imagine for example you delete an item you created and item with id 5 but then you delete it, add a new one, and EF will give you id 7 but you logically wanted id 6 ). In order to fix that you can create sequences manually:

```c#
  modelBuilder.HasSequence<int>("InvoiceNumber", "invoice");
```

The last (weird) thing you need to do is to tell EF how to build the sequence:

```C#
builder.Property(p => p.InvoiceNumber)
                .HasDefaultValueSql("NEXT VALUE FOR invoice.InvoiceNumber");
```

### Concurrency handling

#### At Column level

This is the classic scenario of 2 users updating a column at the same time and you want to throw an error to the user who saves last. This is fairly easy to configure in EF. You have 2 options:

```C#
	// using data annotations
 public class Genre: AuditableEntity
    {
        public int Id { get; set; }
        
        [ConcurrencyCheck]
        public bool IsDeleted { get; set; }
    }
```

```c#
// using fluent api

builder.Property(p => p.name).IsRequired().IsConcurrencyToken()
```

#### At row level

This is optimistic locking. EF makes this easy to check, you just need to add a version row and EF will automatically check the row id on save for you and throw if it's invalid.

```c#
// data annotations
 public class Genre: AuditableEntity
    {
        [Timestamp]
        public byte[] Version { get; set; }
    }
```

```c#
//fluent api

builder.property(p => p.Version).IsRowVersion()
```

## Temporal Tables

Temporal tables are not a feature of EF, they are a feature of SQL. Essentially temporal tables are tables that under the hood have a "history" table. When you update the main table, the "history" table will also update to contain the old data. This means you can query past data or even do rollbacks. In EF you set it up like this:

```c#
  builder.ToTable(name: "Genres", options =>
            {
                options.IsTemporal(t => {
	                t.HasPeriodStart("Start"), //this is OPTIONAL, this is how yoi configure the name of the col
	                t.HasPeriodEnd("End"), //this is OPTIONAL, this is how yoi configure the name of the col
	                t.UseHistoryTable("bananas") //this is OPTIONAL, this is how yoi configure the name of the table
                });
            });

```

note: there's a requirement in SQL Server that temporal tables need to have at least 2 fields of type `datetime2` . Also note that the history will only have records when the main table is either updated or deleted. 

How do you query the history table from EF? Like this, there are quite a few methods you can invoke on the db context:

```c#

        [HttpGet("temporalAll/{id:int}")]
        public async Task<ActionResult> GetTemporalAll(int id)
        {
            var genres = await context.Genres.TemporalAll()
                .Select(p =>
                new {
                    Id = p.Id,
                    Name = p.Name,
                    PeriodStart = EF.Property<DateTime>(p, "PeriodStart"),
                    PeriodEnd = EF.Property<DateTime>(p, "PeriodEnd")
                })
                .Where(p => p.Id == id).ToListAsync();

            return Ok(genres);
        }

        [HttpGet("temporalAsOf/{id:int}")]
        public async Task<ActionResult> GetTemporalAsOf(int id, DateTime date)
        {
            var genre = await context.Genres.TemporalAsOf(date)
                .Select(p =>
                new {
                    Id = p.Id,
                    Name = p.Name,
                    PeriodStart = EF.Property<DateTime>(p, "PeriodStart"),
                    PeriodEnd = EF.Property<DateTime>(p, "PeriodEnd")
                })
                .Where(p => p.Id == id).FirstOrDefaultAsync();

            return Ok(genre);
        }

        [HttpGet("TemporalFromTo/{id:int}")]
        public async Task<ActionResult> GetTemporalAll(int id, DateTime from, DateTime to)
        {
            var genres = await context.Genres.TemporalFromTo(from, to)
                .Select(p =>
                new {
                    Id = p.Id,
                    Name = p.Name,
                    PeriodStart = EF.Property<DateTime>(p, "PeriodStart"),
                    PeriodEnd = EF.Property<DateTime>(p, "PeriodEnd")
                })
                .Where(p => p.Id == id).ToListAsync();

            return Ok(genres);
        }

        [HttpGet("TemporalContainedIn/{id:int}")]
        public async Task<ActionResult> GetTemporalContainedIn(int id, DateTime from, DateTime to)
        {
            var genres = await context.Genres.TemporalContainedIn(from, to)
                .Select(p =>
                new {
                    Id = p.Id,
                    Name = p.Name,
                    PeriodStart = EF.Property<DateTime>(p, "PeriodStart"),
                    PeriodEnd = EF.Property<DateTime>(p, "PeriodEnd")
                })
                .Where(p => p.Id == id).ToListAsync();

            return Ok(genres);
        }

        [HttpGet("TemporalBetween/{id:int}")]
        public async Task<ActionResult> GetTemporalBetween(int id, DateTime from, DateTime to)
        {
            var genres = await context.Genres.TemporalBetween(from, to)
                .Select(p =>
                new {
                    Id = p.Id,
                    Name = p.Name,
                    PeriodStart = EF.Property<DateTime>(p, "PeriodStart"),
                    PeriodEnd = EF.Property<DateTime>(p, "PeriodEnd")
                })
                .Where(p => p.Id == id).ToListAsync();

            return Ok(genres);
        }
```


### Restoring a deleted record

This is a bit of a weird one. There's no built in EF way to simply restore a deleted record. The record lives in the history table and you need to restore it manually. You need to be careful with the IDs though. You cannot simply add a new record because, if using auto generated ids, SQL will add the historical record again but with a different ID. This means you will end up with a broken link between the historic record you restored and the history for that record in the history table. Therefore you need to make sure the record gets inserted with the ID it had before and that is achieve manually. Like this:

```c#
 [HttpPost("restore/{id:int}")]
        public async Task<ActionResult> Restore(int id, DateTime date)
        {
            var genre = await context.Genres.TemporalAsOf(date)
                .IgnoreQueryFilters().FirstOrDefaultAsync(p => p.Id == id);

            if (genre is null)
            {
                return NotFound();
            }

            try
            {
                await context.Database.ExecuteSqlInterpolatedAsync(@$"
                SET IDENTITY_INSERT Genres ON;

                INSERT INTO GENRES (Id, Name)
                VALUES({genre.Id}, {genre.Name})

                SET IDENTITY_INSERT Genres OFF;");
            }
            finally
            {
                await context.Database.ExecuteSqlInterpolatedAsync(@$"SET IDENTITY_INSERT Genres OFF;");
            }

            return Ok();
        }

```


---
# Concepts

Entity - A class that represents a table in our database.

DbFirst - You already have an existing database with data and you want EF to read the data from that database and turn that into C# models

CodeFirst - database does not exist yet. You write your code and EF will create the database from your code.

Navigation property - A navigation property is a way of doing configuration by convention. Imagine you have the following class:

```C#
public class Cinema
{
    public string Name { get; set; }
    public int Id { get; set; }

    public decimal Price { get; set; }

    public Point Location { get; set; }
    public CinemaOffer CinemaOffer { get; set; }
}
```

A `Cinema` has a `CinemaOffer` and this is what's called a navigation property. By virtue of doing things like this, EF understands that there's a relationship between `Cinema` and `CinemaOffer` without you specifying anything. In SQL this will create a Foreign Key

---
## EF Core commands

### Create a migration

`dotnet ef migrations add <your-migration-name>`

## Remove the latest migration

`dotnet ef migrations remove`

If you need to remove a migration that has already been added to the database.  This will drop the database table.

`dotnet ef migrations remove --force`

## Partially update a migration

Once a migration has been applied there's no such thing as updating only part of a migration. If you want to do only a small change rather than reverting the whole migration you need to add a new migration with the partial change you want.

### Apply a migration

`dotnet ef database update`

## List migrations

`dotnet ef migrations list`

## Drop Database

`dotnet ef database drop`

## Bundle migrations

Bundle migrations means "turn all your migrations in a windows exe file that you can run". Useful if you need to apply/share migrations.

`dotnet ef migrations bundle --Configuration Bundle`

## Script migrations

Generate a SQL script for ALL the migrations you have done. Use the `idempotent` flag to make sure your migrations do not clash with what you have in the database (for example what if the generated script adds a table that already exists? `idempotent` fixes that by wrapping things in an if statement)

`dotnet ef migrations script --idempotent`

note: if you have customized code like for example table views, use bundles. Migrations don't work in this scenario
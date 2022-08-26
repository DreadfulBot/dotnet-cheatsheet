# Testing

## Short story

Let's imagine that you have modern application project made in times of .NET core 6 release. Needless to say, it's structure is a little bit different from older .net versions (where you had 2 separate files - `Startup.cs` and `Program.cs`)

Let's deep dive through old and new structure differences, that would be useful for further explanations

## .NET Core 5 Times Project structure

### Startup.cs

Here we used to register all services, configs and classes.

```c#
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }
    //...
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env) {
        if(env.IsDevelopment()) {
            app.UseDeveloperExceptionPage();
            app.UseSwagger();
            // ...
        }
        // ...
    }
}
```

### Program.cs

In fact, this is a starting point of application. Here, for example, I added few lines of code for creating scope for Entity Context and running initial database migrations.

```c#
public class Program
{
    public static void Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();

        using (var scope = host.Services.CreateScope())
        {
            var services = scope.ServiceProvider;
            try
            {
                var context = services.GetRequiredService<DataContext>();
                // context.Database.Migrate();
                DataSeed.SeedDataAsync(context, services).Wait();
            }
            catch (Exception ex)
            {
                var logger = services.GetRequiredService<ILogger<Program>>();
                logger.LogError(ex, "An error occurred during migration");
            }
        }

        host.Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder => { webBuilder.UseStartup<Startup>(); });
}
```

## .NET Core 6 Project structure

Both old files `Program.cs` and `Startup.cs` are now merged together into `Program.cs` file, and there are not any classes inside anymore. Only pure functions for adding new modifications.

### Typical code of `Program.cs` .NET Core 6+ project

```c#
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddControllers();

// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
//...
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseAuthorization();
```

## .NET Core 5 vs 6 structure -  Advantages and disadvantages (+ solving problems)

### Advantages

- 1 simple file instead of 2
- less class wrappings - more simple and clean code
- new webApplicationBuilder was presented

### Disadvantages

- We don't have any classes in program to make relations. For example, for adding such libs as `Mediator` or `FluentValidation` you need to get assembly type by name, example:

```c#
builder.Services.AddMediatR(typeof(Program).Assembly);
```

### Solving root assembly name in .NET Core 6

One of possible solutions is adding this line to the end of file `Program.csz`:

```c#
// Make the implicit Program class public so test projects can access it
public partial class Program { }
```

Now, you can use structures like `typeof(Program)` or `WebApplicationFactory<Program>` (while doing testing stuff)

## Configuring .NET Core 5 Test server

Here we have nu-get package called `Microsoft.TestPlatform.TestHost` with `Microsoft.AspNetCore.TestHost`.

Here is TestServerFixture from one of my projects, that relates to Program from real application:

```c#
public class TestServerFixture : IDisposable
{
    private readonly TestServer _testServer;
    public HttpClient Client { get; }

    public TestServerFixture()
    {
        var builder = new WebHostBuilder()
            // .UseContentRoot(GetContentRootPath())
            .UseEnvironment("Development")
            .UseConfiguration(FakeConfiguration.GetInstance())
            .UseStartup<Startup>();  // Uses Start up class from your API Host project to configure the test server

        _testServer = new TestServer(builder);
        Client = _testServer.CreateClient();
    }
    // ...
}
```

Typical use-case:

```c#
[Fact]
public async void ExceptionIfPasswordNotValid()
{
    using var testServer = new TestServerFixture();

    // Arrange
    const string password = "123";

    var command = new RegisterCommand()
    {
        Email = Faker.Internet.Email(),
        Password = password,
        PasswordConfirmation = password,
        RuleAgreement = true
    };

    // Act
    var (response, _) = await PostAsync<ValidationException>("api/auth/Register", command);
    var responseData = await response.Content.ReadAsStringAsync();
    // ...

```

## Configuring .NET Core 6 Test server

Unit test for controllers and utils is good method, but I also prefer to have real running instance of application for my tests, with ability to send real `POST` and `GET` request.

So, lot's of people in internet starting to build another separate configuration for running such testing virtual node, but in fact we have all necessary configurations in our `Program.cs` file. Let's use it! But we need to make some changes in it, for example - to use not real physical database, but memory one.

First thing first, now we can relate to class `Program.cs` and use it in combination with `WebApplicationFactory` class:

My testing fixture class is looking like this:

```c#
using Microsoft.AspNetCore.Mvc.Testing;

namespace TestHelpers
{
    public class TestServerFixture : IDisposable
    {
        protected readonly WebApplicationFactory<Program> WebApplicationFactory;
        protected HttpClient Client { get; }

        public TestServerFixture()
        {
            WebApplicationFactory = new TestingWebAppFactory();
            Client = WebApplicationFactory.CreateDefaultClient();
        }

        public void Dispose()
        {
            Client.Dispose();
            WebApplicationFactory.Dispose();
        }
    }
}

```

`Program` has been user right from main application project (example above);

Also, I changed it a little bit with that class wrapper. Here I add virtual database + run initial seeding migrations:

```c#
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace TestHelpers;

public class TestingWebAppFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        base.ConfigureWebHost(builder);
        builder.ConfigureServices(services =>
        {
            var descriptor =
                services.SingleOrDefault(d => d.ServiceType == typeof(DbContextOptions<DatabaseContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            services.AddEntityFrameworkInMemoryDatabase();
            services.AddDbContext<DatabaseContext>(o =>
            {
                o.UseInMemoryDatabase("InMemoryAynnTest");
            });

            var sp = services.BuildServiceProvider();

            using var scope = sp.CreateScope();
            using var appContext = scope.ServiceProvider.GetRequiredService<DatabaseContext>();
            appContext.Database.EnsureCreated();
        });
    }
}
```

Now I can easily use this `TestingFixture` in my xuint tests. Real get and post requests go here:

```c#
using TestHelpers;
using Xunit;

namespace Tests;

public class TestTestController: TestServerFixture
{
    [Fact]
    public async Task Ping_OnSuccess_ReturnsTrue()
    {
        var response = await Client.GetAsync("/api/Test/Ping");
        var stringResult = await response.Content.ReadAsStringAsync();
        Assert.Equal("Pong", stringResult);
    }
}
```

## Testing GraphQL HotChocolate request through Unit tests

Let's take the strongest case - running GraphQL mutation through application that is very close to real one.

I will be using the same `TestServerFixture` class here:

```c
[Fact]
public async Task TestRegister_OnSuccess_ReturnsUser()
{
    // arrange
    var query = @"
        mutation register {
            register(
                payload: {
                    email: ""test@t10.com""
                    password: ""1A?a456""
                    passwordConfirmation: ""1A?a456""
                    ruleAgreement: true
            }
            ) {
                id
            }
        }
    ";

    // act
    var request = QueryRequestBuilder.New()
        .SetQuery(query)
        .Create();

    var result = await WebApplicationFactory.Services.ExecuteRequestAsync(request);
    var json = await result.ToJsonAsync();
    
    // assert
    Assert.Null(result.Errors);
    Assert.Contains("data", json);
    Assert.Contains("id", json);
    
    Assert.Matches(
        @"(\{){0,1}[0-9a-fA-F]{8}\-[0-9a-fA-F]{4}\-[0-9a-fA-F]{4}\-[0-9a-fA-F]{4}\-[0-9a-fA-F]{12}(\}){0,1}",
        json);
}
```

This is fully functional example, that works with our test server fixture, and, in that way - with injected virtual EntityFramework database.
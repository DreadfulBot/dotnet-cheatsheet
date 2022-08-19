# dotnet cheat sheet

## Adding secrets to project

```shell
dotnet user-secrets init

dotnet user-secrets set "Redis:ConnectionString" "12345"
```

## Accessing configuration (for secrets - similar)

```c#
IConfiguration configuration;

// with get
Configuration.GetSection("Redis")

// indexed key style
Configuration["Redis:ConnectionString"]
```

## Mapping configuration to POCO

```c#
var moviesConfig = Configuration.GetSection("Redis").Get<RedisOptions>();
```

## Accessing configuration object through Dependency Injection

```c#
class RedisOptions {
    public string ConnectionString {get; set;}
    //...
}

// binding configuration section to runtime object
services.Configure<RedisOptions>("Redis");

// Accessing anywhere in constructor through DependencyInjection
class Controller {
    public Controller(IOptions<JwtSettingsDto> jwtOptions)
    {
        //...
    }
}
```

## Accessing connection string

```c#
Configuration.GetConnectionString("DefaultConnectionString");
```

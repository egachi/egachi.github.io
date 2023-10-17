---
layout: post
title: Building a custom API Gateway with YARP
---

Whenever you are designing an architecture  with microservices, you might encounter in how to implement an `API Gateway`, since you need a way to communicate and consume multiple services, generally through APIs. A possible solution is to have a single entry point for all your clients and implement an `API Gateway`, which will handle all the requests and route those to appropiate microservices. 

There are different ways to implement an API Gateway or pay for built-in services in cloud hostings.

In this post I will pick the easiest way that I found to create one for a microservice architecture using ``.NET`` and `YARP`. Here is a general overview of a microservice architecture.

![architecture](/media/2023/10/api-gateway-architecture.png)

## YARP 

`YARP` (which stands for "Yet Another Reverse Proxy") is an open-source project that Microsoft built for improving routing through internal services using a built-in reverse proxy server. This become very popular and was implemented for several Azure products as [App Service](https://azure.github.io/AppService/2022/08/16/A-Heavy-Lift.html).

- To get started, you need to create an ASP.NET core empty project. I chose .NET 7.0 for this post. 
{% highlight console %}
dotnet new web -n ApiGateway -f net7.0
{% endhighlight %}
- Install [YARP nuget package](https://www.nuget.org/packages/Yarp.ReverseProxy/) or add the project reference `.csproj`.
{% highlight microsoftshell %}
Install-Package Yarp.ReverseProxy
{% endhighlight %}

{% highlight xml %}
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Yarp.ReverseProxy" Version="2.0.1" />
  </ItemGroup>

</Project>
{% endhighlight %}
- Add the YARP middleware to your `Program.cs`.
{% highlight csharp %}
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
var app = builder.Build();
app.MapReverseProxy();
app.Run();
{% endhighlight %}
- To add the YARP configuration you will use `appsettings.json` file. YARP uses routes and clusters, regularilly inside `ReverseProxy` object defined in builder configuration section. You can find more information about different configuration settings [here](https://github.com/microsoft/reverse-proxy/blob/main/docs/docfx/articles/config-files.md). 

- In this example, I am using `Products` and `Employee` microservices. So I will have routes like `employee-route` and `product-route` and clusters as `product-cluster` and `employee-cluster` pointing to destinations. Open your `appsettings.json` and apply the following configuration. 

{% highlight json %}
{
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "Match": {
          "Path": "/p/{**catch-all}",
          "Hosts": ["*"]
        },
        "Transforms": [
          {
            "PathPattern": "{**catch-all}"
          }
        ]
      },
      "employee-route": {
        "ClusterId": "employee-cluster",
        "Match": {
          "Path": "/e/{**catch-all}",
          "Hosts": ["*"]
        },
        "Transforms": [
          {
            "PathPattern": "{**catch-all}"
          }
        ]
      }
    },
    "Clusters": {
      "product-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://localhost:3500/v1.0/invoke/product-api/method/"
          }
        }
      },
      "employee-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "http://localhost:3500/v1.0/invoke/employee-api/method/"
          }
        }
      }
    }
  }
}
{% endhighlight %}
- In scenarios that you need to allow CORS to specific origins you can add a cors policy described in this [Microsoft Doc](https://learn.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-7.0). Here is a configuration example:

{% highlight csharp %}
var builder = WebApplication.CreateBuilder(args);
var allowOrigins = "_allowOrigins";
builder.Services.AddCors(options =>
{
    options.AddPolicy(allowOrigins, policy =>
    {
        policy
          .WithOrigins("http://localhost:3000", "http://127.0.0.1")
          .SetIsOriginAllowedToAllowWildcardSubdomains()
          .AllowAnyHeader()
          .WithMethods("GET", "POST", "PUT", "DELETE")
          .AllowCredentials();
    });
});
var app = builder.Build();
app.UseCors(allowOrigins);
{% endhighlight %}
- Then add this CORs policy inside `Routes` as followed:
{% highlight json %}
"Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "CorsPolicy": "_allowOrigins",
        "Match": { "..." },
        "Transforms": [ "..."]
      },
}
{% endhighlight %}
- Finally if you want to get more information about YARP logging for future debugging or production information, you can add the YARP log level (`Information`,`Warning` or `Error`) inside `Logging` object as followed:

{% highlight json %}
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Yarp": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
{% endhighlight %}
- Run the application with `dotnet run` and use Postman or curl to test the endpoints defined in your paths e.g. `http://localhost:5212/p/v1/`.
> Check all possible configurations and transformations in the [YARP documentation](https://github.com/microsoft/reverse-proxy/blob/main/docs/docfx/articles/config-files.md). 
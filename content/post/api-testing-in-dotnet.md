---
title: "API Testing In .NET with Expected.Request"
date: 2017-10-05T19:33:35-06:00
draft: true
tags: ["dotnet", "dotnet core", "api testing", "testing", "integration testing"]
---

Alright let's get started with the question that I'm sure you are all asking, what is Expected.Request? Simply put it's a Fluent API I've created to test REST APIs, and even better it's open source! Woot! Expected.Request abstracts away the HttpClient class in favor of clear and cohesive chaining. If you would like to see the full documentation for this package check that out [here](https://wright-development.github.io/Expected.Request/), otherwise let's jump into some examples.

**Concidentally this post ties pretty well into my previous post about integration testing with docker in .NET, if you are interested check it out [here](/post/using-docker-for-net-core/)**

## Project Setup

In order to get started we will need an API to test against, and a test project. If you would like to avoid the setup feel free to download the starting project from [here](https://github.com/wright-development/expected-request-sample-project/archive/start.zip). Otherwise feel free to follow along with your own code.

Once you have downloaded the code, unzip and open the project in your favorite editor.

**If you are following along with your own code you will need to install Microsoft.AspNetCore.TestHost@2.0.0 (Optional, if you don't use this package then you will need to provide your own HttpClient, or allow Expected.Request to created it for you.), Newtonsoft.JSON@10.0.3 and Expected.Request@1.0.10**

## Writing Tests

Alright now that the project is all setup, let's create a few tests and see what Expected.Request buys us. We'll start with a simple get request on the ValuesController.

Without expected request the code will look something like this...

``` csharp
private readonly TestServer _server;
private readonly HttpClient _client;
        
public ValuesControllerTests()
{
    _server = new TestServer(new WebHostBuilder()
        .UseStartup<Startup>());
    _client = _server.CreateClient();
}

[Fact]
public async Task http_client_test()
{
    var response = await _client.GetAsync("/api/values");

    var status = response.StatusCode;
    var content = await response.Content.ReadAsStringAsync();
    var contentObject = JsonConvert.DeserializeObject<IEnumerable<string>>(content);

    Assert.Equal(status, HttpStatusCode.OK);
    Assert.Equal(contentObject.ElementAt(0), "value1");
    Assert.Equal(contentObject.ElementAt(0), "values");
}
```

While this code isn't bad, it's easy to see that after a few tests there will be alot of code duplication. In addition the code code stand to be a little 
clearer, thus enters Expected.Request.

``` csharp
private readonly TestServer _server;
private readonly HttpClient _client;
        
public ValuesControllerTests()
{
    _server = new TestServer(new WebHostBuilder()
        .UseStartup<Startup>());
    _client = _server.CreateClient();
}

[Fact]
public async Task expected_request_test()
{
    await new Request(_client)
     .Get("/api/values")
     .Next(x => x.ExpectOk())
     .Next(x => x.Expect<IEnumerable<string>>((reponse) => {
         Assert.Equal(reponse.ElementAt(0), "value1");
         Assert.Equal(reponse.ElementAt(1), "value2");
     }))
     .Done();
}
```

With the syntax of expected request, we can now read our API test a little better.

- Start a new Request
- Perform a Get to the url '/api/values'
- Next I expect the status to be ok
- Next I want to check the content matches my expectations
- I'm Done with the Request

Expected Request shines even more when you want to perform multiple requests, for example after a POST to an API I want to perform a GET to retrieve the object back that I posted.

``` csharp
[Fact]
public async Task expected_request_multiple_requests_test()
{
    int id = 0;

    await new Request(_client)
     .Post("/api/values", "value")
     .Next(x => x.ExpectOk())
     .Next(x => x.Map<int>(model => id = model))
     .Next(x => x.Get($"/api/values/{id}", _client))
     .Next(x => x.ExpectContent("value"))
     .Done();
}
```


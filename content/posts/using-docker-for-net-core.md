---
title: "Using Docker for .NET Core Integration Testing"
date: 2017-09-16T13:39:28-06:00
draft: false
tags: ["docker", "docker compose", "dotnet", "dotnet core", "web service", "testing", "integration test", "MySql"]
---

Recently at work, we have been discussing how to perform integration tests on .NET Core services. From previous experience, integration testing can be quite a messy process especially when performing reads and writes to a database. 
<br>
<br>
Have you ever had an issue with maintaining consistently correct data? Sharing a database with multiple developers? Or even setting up your own data without interfering with your teammates? If any of these problems sound familiar then docker can be the solution for you.
<br>
<br>
Docker provides a great way for you to stand up services, databases, and other resources locally through containers. In addition, with docker compose, we can set up multiple containers and define the interactions between them. For example, we can start our service and have it communicate with a container running a MySQL database.

<br>
## Setting up the project
This post assumes that you have some experience with using Docker, .NET Core, and writing tests with XUnit. If not, check out the links below for additional help. 

* [Docker Getting Started](https://docs.docker.com/get-started/#conclusion)
* [.NET Core Getting Started](https://docs.microsoft.com/en-us/dotnet/core/get-started)
* [XUnit writing your first test](https://xunit.github.io/docs/getting-started-desktop.html#write-first-tests)
<br>
<br>
**Tools Needed: docker, dotnet cli**
<br>
**[Source Code](https://github.com/wright-development/dotnet-docker-integration-testing)**
<br>

Before working with docker, we need to set up the project structure, install the packages that we need, create some code to test and create an integration test to run. So let's get started!
<br>
<br>
Feel free to set up the project using the dotnet cli, Visual Studio, or any other means. Be sure to set the project up in the following format.

<pre>
TodoService/
├── src/
│   └── TodoService/
│       └── TodoService.csproj
├── test/
│  └── TodoService.Integration.Tests/
│       └── TodoService.Integration.Tests.csproj
└── TodoService.sln
</pre>

The TodoService is a WebAPI project **(Target Framework netcoreapp1.1)**, and the TodoService.Integration.Tests project is a XUnit test project **(Target Framework netcoreapp1.1)**.
<br>
**Check out the [source code](https://github.com/wright-development/dotnet-docker-integration-testing) for more clarification.**
<br>
<br>
Now that the service and test projects have been set up, we will need to install the following packages.

- TodoService
    - Dapper@1.50.2
    - Microsoft.AspNetCore@1.1.2
    - Microsoft.AspNetCore.Mvc@1.1.3
    - MySql.Data.Core@7.0.4-ir-19
    - Newtonsoft.JSON@10.0.3
- TodoService.Integration.Tests 
    - **This project will need to reference the TodoService project**
    - Dapper@1.50.2
    - MySql.Data.Core@7.0.4-ir-19
    - Newtonsoft.JSON@10.0.3
    - Shouldly@2.8.3 (Or a different assertion framework.)

<br>
Next up, we need to implement the TodoController.
<br>
<br>

``` csharp
// *********************************************
// src/TodoService/Controllers/TodoController.cs
// *********************************************

namespace TodoService.Controllers
{
    //Our simple todo model that we will use to map objects in dapper
    public class TodoModel
    {
        [JsonProperty("text")]
        public string Text { get; set; }

        [JsonProperty("id")]        
        public string Id { get; set; }

        [JsonProperty("checked")]        
        public bool Checked {get;set;}
    }

    [Route("api/[controller]")]
    public class TodoController
    {

        private string _connectionString;

        public TodoController()
        {
            //This environment variable will be assigned in the docker-compose file to connect to the database
            _connectionString = Environment.GetEnvironmentVariable("CONNECTION_STRING");
        }

        [HttpPost]
        public TodoModel Post([FromBody]TodoModel model)
        {
            using (var connection = new MySqlConnection(_connectionString))
            {
                var id = connection.Query<string>("INSERT INTO todo (text, checked) values (@Text, @Checked); SELECT LAST_INSERT_ID();", model).Single();
                return Get(id);
            }
        }

        [HttpGet("{id}")]
        public TodoModel Get(string id)
        {
            using(var connection = new MySqlConnection(_connectionString))
            {
                return connection.Query<TodoModel>("SELECT id,checked,text FROM todo WHERE id=@Id", new {Id=id}).FirstOrDefault();
            }
        }
    }
}
```
Now let's add an integration test.
<br>
<br>
``` csharp

//***************************************
// test/TodoService.IntegrationTests/TodoEndpointTests.cs
//***************************************
namespace TodoService.IntegrationTests
{
    public class TodoEndpointTests : IDisposable
    {
        private string _endpoint = "/api/todo";
        private string _url;
        private string _connectionString;

        public TodoEndpointTests()
        {
            // This environment variable will be supplied by the docker compose, we will use this to make our api calls against our service.
            _url = Environment.GetEnvironmentVariable("API_URL") + _endpoint;
            _connectionString = Environment.GetEnvironmentVariable("CONNECTION_STRING");
        }
        
        [Fact]
        public async void should_assign_id_on_post()
        {
            var client = new HttpClient();
            var expectedModel = new TodoModel{ Checked = false, Text = "Test Text" };
            var result = await client.PostAsync(_url, new StringContent(JsonConvert.SerializeObject(expectedModel),Encoding.UTF8, "application/json"));

            var jsonString= await result.Content.ReadAsStringAsync();
            Console.WriteLine("Results: " + jsonString);
            var actualModel = JsonConvert.DeserializeObject<TodoModel>(jsonString);

            actualModel.Id.ShouldNotBeEmpty();
        }

        public void Dispose()
        {
            using(var connection = new MySqlConnection(_connectionString))
            {
                //Clear the data after each test
                connection.Execute("truncate todo");
            }
        }
    }
}
```

Finally, we need table to store the todos that will be created by the application. This SQL script will be located in the **Scripts** folder at the root of the solution.

``` sql
# /Scripts/InitialSchema.sql
USE todosdb;

CREATE TABLE IF NOT EXISTS todo (id SERIAL, text VARCHAR(100), checked BOOLEAN)
```

Alright, now that the project has been set up let's write up some docker files. **All of the docker files will be located at the root of the solution (/TodoService).**
<br>
<br>
## Docker Setup
 
Oh boy! The moment that we have all been waiting for, it's Docker time!
<br>
<br>
There are a few docker files needed to run our integration tests.

1. Dockerfile - This will be used to build the service, publish the service's content, and copy the published content into a container.
2. Dockerfile.integration - This will be used to build and restore the integration test project, getting it ready to run the tests.
3. docker-compose-integration.xml - This will be used to stand up our MySql database, Todo service, and run our integration tests against the service.

### Dockerfile
<br>
For the **Dockerfile** we are going to use [multi stage builds](https://docs.docker.com/engine/userguide/eng-image/multistage-build/) to both build and publish our service into an image.
<br>
<br>
``` dockerfile
# /Dockerfile
FROM microsoft/dotnet:1.1.2-sdk as builder
COPY . /code
WORKDIR /code/src/TodoService
RUN dotnet restore && dotnet publish -c Release -o publish

FROM microsoft/dotnet:1.1.2-runtime
COPY --from=builder /code/src/TodoService/publish /app
WORKDIR /app
ENV ASPNETCORE_URLS="http://*:5000"
EXPOSE 5000
# This is used to wait until resources have started, in the docker compose
RUN curl https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh > wait_for_it.sh
ENTRYPOINT [ "dotnet", "/app/TodoService.dll" ]
```

Next up is the **Dockerfile.integration** file. This is fairly simple. We copy the solution's code over and run a restore in the integration test directory.
<br>
<br>
``` dockerfile
# /Dockerfile.integration
FROM microsoft/dotnet:1.1.2-sdk as builder
COPY . /app
WORKDIR /app/test/TodoService.IntegrationTests
RUN curl https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh > /app/wait_for_it.sh \
    && dotnet restore
```

Last but not least, we need our **docker-compose-integration.yml** file.
<br>
<br>

``` yaml
# /docker-compose-integration.yml
version: '3'

services:
  integration:
    build: 
      context: .
      dockerfile: Dockerfile.integration
    environment:
      - API_URL=http://web:5000
      - CONNECTION_STRING=Server=db;Database=todosdb;Uid=root;Pwd=password;SslMode=None;      
    entrypoint: bash /app/wait_for_it.sh web:5000 -t 0 -- dotnet test
    depends_on:
      - web
      - db
  web:
    build: .
    ports: 
      - 5000:5000
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - CONNECTION_STRING=Server=db;Database=todosdb;Uid=root;Pwd=password;SslMode=None;
    entrypoint: bash /app/wait_for_it.sh db:3306 -t 0 -- dotnet /app/TodoService.dll
    depends_on:
      - db
  db:
    image: mysql
    ports:
      - 3306:3306
    # Start the container with a todosdb, and password as the root users password
    environment: 
      - MYSQL_DATABASE=todosdb
      - MYSQL_ROOT_PASSWORD=password
    # Volume the scripts folder over that we setup earlier.
    volumes: 
      - ./Scripts:/docker-entrypoint-initdb.d

```

Great! All the files that we need have been set up. Are you ready to run some tests? Find your nearest terminal that supports Docker, cd to the folder that contains the **docker-compose-integration.yml** file, and run the following command.
<br>
<br>
``` bash 
docker-compose -f docker-compose-integration.yml up
```
**Personally, I like to run the following command:**
``` bash 
docker-compose -f docker-compose-integration.yml up --build --abort-on-container-exit
``` 
**This will stop the docker compose after the integration tests have completed. In addition, it will rebuild your images before starting the containers.**
<br>
<br>

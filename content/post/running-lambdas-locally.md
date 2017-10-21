---
title: "Running Lambda's Locally"
date: 2017-09-16T13:39:11-06:00
draft: true
---

**Tools Needed: [awsCli](https://aws.amazon.com/cli/), [docker](https://docs.docker.com/engine/installation/), and [dotnet core](https://www.microsoft.com/net/download/core)**
<br>
<br>
Very recently a co-worker of mine brought up an open source project for testing the creation of AWS resources locally called [localstack](https://github.com/localstack/localstack). Naturally interested I pull up the github page and figured out how to run it to see what all the hype was about.
<br>
<br>
After a while of messing around with the project for a bit, I found it was absolutely amazing! Granted it doesn't have the complete functionality of AWS, what has been completed is remarkable. 
<br>
<br>
## Launching LocalStack

Therefore without futher adieu, let's run a Lambda function locally. I am a huge fan of docker so to make things easier for us I will be standing up the localstack project using a docker compose file.
<br>
<br>
``` yml
# docker-compose.yml
version: '3'
services:
  aws:
    image: localstack/localstack
    ports:
    - "8080:8080" # web
    - "4574:4574" # lambda
```

**Check out the localstack [documentation](https://github.com/localstack/localstack) for the other ports used for the services available.**
<br>
<br>
Now within a terminal where the **docker-compose.yml** is located run **docker-compose up**, and with little patience we the localstack service will start up.
<br>
<br>
Navigate to the website at localhost:8080, and you will see the localstack dashboard for your deployed resources. Huzzah! Now that we have everything started up for us to start launching services locally.
<br>
<br>
## Publishing our Dotnet Code
<br>
In the interest of saving time we will add the lambda templates to the dotnet cli using the following command.
<br>
``` bash
dotnet new -i Amazon.Lambda.Templates::*
```
<br>
Then create an empty lambda function.
<br>
``` bash
dotnet new lambda.EmptyFunction -n SampleApp
```
<br>
Once the app has been created cd into the directory containing the csproj and publish.
``` bash
cd SampleApp/src/SampleApp
dotnet publish -c Release -o output
```
Once that has been published, zip up the contents of the **output** directory into a zip called SampleApp.zip.
<br>
## Publishing our Lambda
<br>
Now for the moment all have been waiting for, we will publish our dotnet code as a lambda to our localstack. In the directory where the zip file, **SampleApp.zip**, was created run the following aws cli command.

``` bash
```
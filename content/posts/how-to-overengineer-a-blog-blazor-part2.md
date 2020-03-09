---
author: "Danny van der Biezen"
date: 2020-03-08
linktitle: How to overengineer a blog using Blazor WebAssembly - The code
title: How to overengineer a blog using Blazor WebAssembly - The code
---

[In my previous post](https://dannybies.github.io/posts/how-to-overengineer-a-blog-blazor/) I talked about how I decided on the architecture I chose to create a blog application. In this part I will be talking about how I created the blog  using Blazor, SignalR and MongoDB.

**TL;DR, you can look at the source code on my [github](https://github.com/dannyBies/BiesBlogBlazor).**

To start off we are going to add the latest template for Blazor WebAssembly using the following command:
```
dotnet new -i Microsoft.AspNetCore.Blazor.Templates::3.2.0-preview1.20073.1
```
Also make sure to [install the .NET Core 3.1 SDK](https://dotnet.microsoft.com/download/dotnet-core/3.1).

After the installation is complete we can create a new solution using the newly available "Blazor WebAssembly App" template in Visual Studio. Under advanced make sure to select "ASP.NET Core Hosted" as we will be using a server for our Change Stream and SignalR.

![create-blazor](/images/create-blazor.png)

After creating the solution you can confirm that everything works by using Visual Studio to run the application and the default web app will be opened:

![run-blazor](/images/run-blazor.png)

## Setting up the database layer
After confirming that everything works we can begin setting up our database. We will be using MongoDB Change Streams, so we can either run MongoDB locally or use [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) to save time from setting everything up ourselves. As we will be using Change Streams, a requirement is that our database needs to be set up as either a replica set or a sharded cluster because we need an [Oplog](https://docs.mongodb.com/manual/core/replica-set-oplog/) for Change Streams to work. MongoDB Atlas will manage all of this for us and it has a free tier which is all we will need for this project. You can use Atlas or run it locally, just make sure that you have a valid connectionstring to connect to the database.

In the shared project we can now add a new class to represent our BlogPost document. To keep things simple this is the only collection we will be creating. Make sure to add the MongoDB.Bson NuGet package in order to use the attributes.
```C#
public class BlogPost
{
    [BsonId(IdGenerator = typeof(StringObjectIdGenerator))]
    [BsonRepresentation(BsonType.ObjectId)]
    public string Id { get; set; }
    [Required]
    public string CreatedBy { get; set; }
    [Required]
    public DateTime CreatedOn { get; set; }
    [Required]
    public string Title { get; set; }
    [Required]
    public string Description { get; set; }
    [Required]
    public string Content { get; set; }
}
```
Now let's add an appsettings.json file in our server project in which we will add our database settings:
```JSON
{
  "Mongo": {
    "ConnectionString": "Your-connection-string",
    "Database": "Blog"
  }
}
```
We will make use of ASP.NET Core's Dependency Injection to inject these settings into our classes. To configure this we add the following constructor to our Startup.cs.
```C#
private readonly IConfiguration Configuration;

public Startup(IConfiguration configuration)
{
    Configuration = configuration;
}
```
Then we can add the appsettings as a singleton to be able to inject it into our classes.
```C#
services.AddSingleton(Configuration.GetSection("Mongo").Get<MongoOptions>());
```
to the ConfigureServices method in Startup.cs. 
Finally we add a simple class to contain the settings:
```C#
public class MongoOptions
{
    public string ConnectionString { get; set; }
    public string Database { get; set; }
}
```
## Setting up a change stream
Now that we have set up our database let's look at using a change stream to stream data being inserted in the database back to our Server. We will need to add the MongoDB.Driver NuGet package to connect to our database.
```C#
public class MongoDbChangeStreamFeed<T>
{
    private readonly IMongoCollection<T> _collection;
    private readonly TimeSpan _retryIfNoDocumentsFoundDelay;

    public MongoDbChangeStreamFeed(IMongoCollection<T> collection, TimeSpan retryIfNoDocumentsFoundDelay)
    {
        _collection = collection;
        _retryIfNoDocumentsFoundDelay = retryIfNoDocumentsFoundDelay;
    }

    public async IAsyncEnumerable<T> FetchFeed(PipelineDefinition<ChangeStreamDocument<T>, ChangeStreamDocument<T>> filterPipeline = null, [EnumeratorCancellation]CancellationToken cancellationToken = default)
    {
        using var changeStreamFeed = await _collection.WatchAsync(filterPipeline, cancellationToken: cancellationToken);
        while (!cancellationToken.IsCancellationRequested)
        {
            while (await changeStreamFeed.MoveNextAsync(cancellationToken))
            {
                using var changeStreamFeedCurrentEnumerator = changeStreamFeed.Current.GetEnumerator();
                while (changeStreamFeedCurrentEnumerator.MoveNext())
                {
                    yield return changeStreamFeedCurrentEnumerator.Current.FullDocument;
                }
            }
            await Task.Delay(_retryIfNoDocumentsFoundDelay, cancellationToken);
        }
    }
}
```
> Note the use of C# 8's IAsyncEnumerable here to make sure that we process data as it comes through instead of processing it in batches. If you want to know more about IAsyncEnumerable I recommend reading [this article](https://dotnetcoretutorials.com/2019/01/09/iasyncenumerable-in-c-8/) because it explains what it does quite nicely.

With this in place we can continuously listen for changes in our BlogPost collection. To do this we will be spinning up a hosted service, so we can listen and process BlogPosts in the background. 

```C#
public class BlogPostFeedHostedService : BackgroundService
{
    private readonly MongoOptions _mongoOptions;

    public BlogPostFeedHostedService(MongoOptions mongoOptions)
    {
        _mongoOptions = mongoOptions;
    }

    protected override async Task ExecuteAsync(CancellationToken cancellationToken)
    {
        var client = new MongoClient(_mongoOptions.ConnectionString);
        var database = client.GetDatabase(_mongoOptions.Database);
        var collection = database.GetCollection<BlogPost>(nameof(BlogPost));
        
        var insertOperationsOnlyFilter = new EmptyPipelineDefinition<ChangeStreamDocument<BlogPost>>().Match(x => x.OperationType == ChangeStreamOperationType.Insert);
        var blogPostFeed = new MongoDbChangeStreamFeed<BlogPost>(collection, TimeSpan.FromSeconds(5));

        try
        {
            await foreach (var blogPost in blogPostFeed.FetchFeed(insertOperationsOnlyFilter, cancellationToken))
            {
                //Process data
            }
        }
        catch (OperationCanceledException) { }
    }
}
```
Because we previously used 
```C#
services.AddSingleton(Configuration.GetSection("Mongo").Get<MongoOptions>()); 
``` 
we can make use of the built-in
 Dependency Injection that ASP.NET Core provides to get our MongoOptions so we can set up our connection to the database.

The final step is to register the background service in Startup.cs
```C#
services.AddHostedService<BlogPostFeedHostedService>();
```

Let's see if this everything is setup correctly. To do this we'll add 
```C#
Debug.WriteLine("A new blog post was just created!"); 
```
to our await foreach code block in BlogPostFeedHostedService.cs.

This should now log the above message every time you insert a document into the BlogPostCollection. To quickly test if it works without using the database we can change our FetchFeed method to yield return default:
```C#
public async IAsyncEnumerable<T> FetchFeed(PipelineDefinition<ChangeStreamDocument<T>, ChangeStreamDocument<T>> filterPipeline = null, [EnumeratorCancellation]CancellationToken cancellationToken = default)
{
    using var changeStreamFeed = await _collection.WatchAsync(filterPipeline, cancellationToken: cancellationToken);
    while (!cancellationToken.IsCancellationRequested)
    {
        while (await changeStreamFeed.MoveNextAsync(cancellationToken))
        {
            yield return default;
        }
        await Task.Delay(_retryIfNoDocumentsFoundDelay, cancellationToken);
    }
}
```
With this setup you can run the program and you should get the same message every second.
![create-blazor](/images/fetch-feed-output.png)

## Setting up SignalR

Now that we've got our background service running we can work on adding SignalR to this service in order to notify our Blazor client when a new BlogPost gets added. To get this working we set up a simple [strongly typed SignalR hub](https://docs.microsoft.com/en-us/aspnet/core/signalr/hubs?view=aspnetcore-3.1)
```C#
public class BlogHub : Hub<IBlogFeed>
{
}
public interface IBlogFeed
{
    Task BlogPostCreated(BlogPost blogPost);
}
```
We have to configure our server to use SignalR, we do that in our startup.cs. We add the following line to ConfigureServices()
```C#
services.AddSignalR();
```
And finally add the following line to the endpoint mapping in Configure()
```C#
endpoints.MapHub<BlogHub>("/blogHub");
```

We can now use this hub to send messages to all clients that are connected to /blogHub. We can add a few lines of code to our background service to send a message to every connected client every time a BlogPost has been created. We make use of ASP.NET Core's Dependency Injection one more time to inject the BlogHub into our constructor.
```C#
private readonly MongoOptions _mongoOptions;
private readonly IHubContext<BlogHub, IBlogFeed> _blogHub;
public BlogPostFeedHostedService(MongoOptions mongoOptions, IHubContext<BlogHub, IBlogFeed> blogHub)
{
    _mongoOptions = mongoOptions;
    _blogHub = blogHub;
}
```
Then we can simply call the BlogPostCreated method we defined on the IBlogFeed interface to send the BlogPostCreated message to all clients.
```C#
await foreach (var blogPost in blogPostFeed.FetchFeed(insertOperationsOnlyFilter, cancellationToken))
{
    await _blogHub.Clients.All.BlogPostCreated(blogPost);
}
```

## Setting up Blazor WebAssembly

The only thing left for us to do is to set up SignalR on the client-side and show the data.

In our Index.razor file we can add the following code

```C#
@page "/"
@using Microsoft.AspNetCore.SignalR.Client
@inject NavigationManager NavigationManager

@foreach (var blogPost in _blogPosts)
{
    <article>
        <h1><a href="blog/@blogPost.Id">@blogPost.Title</a></h1>
        @blogPost.Description
    </article>
}
@code {
    private HubConnection _hubConnection;
    private List<BiesBlogBlazor.Shared.Entities.BlogPost> _blogPosts = new List<BiesBlogBlazor.Shared.Entities.BlogPost>();

    protected override async Task OnInitializedAsync()
    {
        _hubConnection = new HubConnectionBuilder()
            .WithUrl(NavigationManager.ToAbsoluteUri("/blogHub"))
            .Build();

        _hubConnection.On("BlogPostCreated", (BiesBlogBlazor.Shared.Entities.BlogPost blogPost) =>
        {
            _blogPosts.Add(blogPost);
            StateHasChanged();
        });

        await _hubConnection.StartAsync();
    }
}
```
When we navigate to our index.razor page we will open a websocket to our Server using SignalR and subscribe to all BlogPostCreated messages. When we receive these we update our list of blogPosts and the corresponding HTML.

And that's it! We've got everything ready now :D. You can try it out by directly inserting the document below into your database.
```JSON
{
    "CreatedBy": "Danny",
    "CreatedOn": {
        "$date": {
            "$numberLong": "1582284312358"
        }
    },
    "Title": "My first blog",
    "Description": "My description",
    "Content": "My content"
}
```
With some small changes to the styling of the page (which you can in the [sourcecode](https://github.com/dannyBies/BiesBlogBlazor))) this will result in the following view:
![create-blazor](/images/blazor-result.png)

## Conclusion
It turned out to be quite easy to get everything working even though Blazor WebAssembly is on a preview version. The generated template by Visual Studio and documentation around Blazor and SignalR was all I really needed to get this up and running within hours. You can check out the source code on my [github](https://github.com/dannyBies/BiesBlogBlazor)).

I will be writing more of these posts in the future. If you enjoyed reading this or have any feedback please let me know on [Twitter](https://twitter.com/DannyBies)!
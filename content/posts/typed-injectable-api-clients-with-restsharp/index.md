+++
date = 2024-01-11T12:35:17+01:00
title = "Typed injectable API clients with RestSharp"
description = "When working with external APIs, it is useful to create a typed API client that can be injected in your code through the Dependency Injection mechanism. RestSharp can help with this."
authors = ["Stefano Previato"]
tags = ["C#", "RestSharp"]
categories = ["C#", "Programming"]
+++

## Preface

In my day-to-day work, I often find myself needing to access external data via an external API.

If you're using C# and recent versions of .NET (both ASP.NET Core and Worker flavors) and need to do something similar, you may find useful to create a client for the external API and inject it whenever you need it via Dependency Injection.

Writing everything from scratch can lead to a lot of boilerplate. Luckily, we can use the help of a great library called [RestSharp](https://restsharp.dev/) ([GitHub](https://github.com/restsharp/RestSharp)).

## Required packages

Make sure to install the NuGet package for RestSharp in your project. This article was written with RestSharp v108.0.1, but you may check for updates and migration paths for newer versions on the official docs.

## Creating the contract for the API

Let's start by declaring the shape of the API that we need to call:

```csharp
public interface IMusicClient
{
  Task<IEnumerable<SongData>> GetSongsAsync(Guid artistId, CancellationToken cancellationToken);
}
```

and create an implementation of it:

```csharp
public class MusicClient : IMusicClient
{
  private readonly RestClient _restClient;

  public MusicClient(RestClient restClient)
  {
    _restClient = restClient;

    // provide the base URL for all calls
    // this should be coming from an IConfiguration or an IOption setting possibly
    _restClient.Options.BaseUrl = new Uri("https://your_external_api_uri");

    // optionally inject default parameters in all calls
    // could be useful for setting the User-Agent, or maybe passing an authorization token or an API key
    _restClient.DefaultParameters.AddParameter(new HeaderParameter("Custom-Header", "CustomValueXYZ"));
  }

  Task<IEnumerable<SongData>> GetSongsAsync(Guid artistId, CancellationToken cancellationToken)
  {
    // compose the URI of the resource (will be appended to the base URL)
    // you can pass parameters without string interpolation, they will be added a few lines after this line
    const string uri = "MusicLibrary/Artists/{artistId}/Songs";

    // create the request object
    RestRequest restRequest = new(uri);

    // replace the URI parameters provided above
    restRequest.AddUrlSegment("artistId", artistId);
    // there are also other utility methods such as AddQueryParameter();

    // fire the request
    RestResponse<IEnumerable<SongData>> response = await _restClient.ExecuteAsync<IEnumerable<SongData>>(restRequest, cancellationToken);

    // check if the response was successful, if not then throw an exception
    response.ThrowIfNotSuccessful();

    // the content of the response inside .Data is typed!
    return response.Data;
  }
}
```

As you can see, the implementation is really straightforward, and we get a lot of benefits from using RestSharp:

- URI parameters handling
- automatic deserialization of response data
- header customization for all requests
- error checking and exception throwing in case of failure
- automatic cancellation handling via `CancellationToken`
- ...and a lot more that you can find in the docs!

## Adding the services to the DI container

In your `Program.cs` (or wherever you have your host initialization code) add the required code for RestSharp and your custom implementations:

```csharp
public static void Main(string[] args)
{
  IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
      // add RestSharp's client
      services.AddTransient<RestClient>();

      // add the custom-made API client library
      services.AddSingleton<IMusicClient, MusicClient>();

      // your other services...
    })
    .Build()
    .Run();
}
```

## Using the API client

At this point, you are ready to use the API client in your classes by injecting it via costructor injection for example:

```csharp
public class Worker : BackgroundService
{
  private readonly IMusicClient _musicClient;

  public Worker(IMusicClient musicClient)
  {
    _musicClient = musicClient;
  }

  protected override async Task ExecuteAsync(CancellationToken stoppingToken)
  {
    // call your API client methods
    _musicClient.GetSongsAsync(Guid.NewGuid(), stoppingToken);

    // rest of your code...
  }
}
```

That's it!

## Testing

Creating a contract for your API client is a good way to abstract away the implementation details and enable you to test consumer code way better.

If you followed up until here, the `Worker` class is highly testable (at least from a unit-test perspective) because you can provide a fake implementation for the `IMusicClient` interface or even a mock and limit your tests to the logic of the worker itself, rather than focusing on testing the API calls too.

If you need to test the client itself without firing the actual requests over the network, you should check out a very good library called `RichardSzalay.MockHttp`.

Here's how a typical test would look like (using `NUnit`, `FluentAssertions` and `Moq`):

```csharp
[TestFixture]
public class MusicClientTests
{
  [SetUp]
  public void SetUp()
  {
    // setup a mock http message handler, so we don't actually connect to the network for the tests
    _mockHttpMessageHandler = new MockHttpMessageHandler();

    // setup the RestSharp client with the mock http message handler
    _restClient = new RestClient(_mockHttpMessageHandler);
  }

  private MockHttpMessageHandler _mockHttpMessageHandler = null!;
  private RestClient _restClient = null!;

  [Test]
  public async Task GetSongsAsync_ShouldMakeTheCorrectRequest_WhenCalled()
  {
    // arrange
    Guid artistId = Guid.NewGuid();

    IEnumerable<SongData> songData = CreateFakeSongData(); // omitted for brevity

    _mockHttpMessageHandler
      .Expect(HttpMethod.Get, $"/MusicLibrary/Artists/{artistId}/Songs")
      .Respond("application/json", JsonSerializer.Serialize(songData));

    MusicClient sut = new(_restClient);

    // act
    List<SongData> result = (await sut.GetSongsAsync(artistId, CancellationToken.None)).ToList();

    // assert
    _mockHttpMessageHandler.VerifyNoOutstandingExpectation();

    result.Should().NotBeNull();
    result.Should().HaveCount(songData.Count());
  }
}
```

## Conclusion

Whenever possible, you should try to take advantage of static and strong typing in C#, to allow you to reuse code and apply refactorings in a secure manner. On top of that, clean code is easier code usually, and RestSharp makes it really easy to parse the code visually while reading it.

## TUI Musement tech challenge solution description
My name is Samy AADDAJ ELOUDHRIRI.

This document provides some insight into some of the implementation choices that have been made in solving this interesting challenge.

### A quick overview of the project
Here you can see the main building blocks of the application:
![alt project summary graph](../README-images/ProjectSummaryGraph.png)

From bottom to top, we have:
- `MusementApiClient` which sends a request for the list of cities from TUI Musement's API
- `WeatherApiClient` which sends a request for the weather forecast from weatherapi for each of the cities
- `WeatherForecastService` which orchestrate the different requests to get the list of cities and its weather forecasts and print it all out to the console
- `Program` which is the main entry point for the application and makes the call to process all cities

The following paragraphs will discuss some key interesting choices in each of these blocks

## Program.cs
This is the main entry point for the application.

Being relatively small, an image will provide a good introduction:
![alt Program.cs](../README-images/Program.png)

### Configuration reading [lines 7 to 11 above]
There is two ways of configuring the application to provide, amongst other thing, the weatherapi key:
- by filling the available appsetting.json file
- by setting environment variables

The appsettings.json file is rather classic but the environment variable is quite interesting.

It takes precedence over the json file and is useful when running docker as it allows for a reconfiguration of the application without having to rebuild the image.
It might not be particularly pertinent for such a small application but it is non-the-less intersting.

### Dependency injection [line 15 above]
All the depencies are registered throught the `RegisterAll(...)` extension method. This allows for a clean separation.

This method mainly retrieves the weather api key and configures typed HttpClient for both API clients using IHttpClientFactory with specific handler lifetime and retry policy.

The handler lifetime for each client is configurable using the appsettings.json file (or by using environment variables as described previously).

Here is the full appsettings.json file:
![alt appsettings.json](../README-images/appsettings.json.png)

The retry policies are as follow:
#### MusementApiClient
![alt MusementApiClient retry policy.cs](../README-images/MusementApiClientRetryPolicy.png)
The retry policy used for `MusementApiClient` is a simple exponential backoff with an initial waiting period of two seconds an a maximum retry count of ten times.

This seems to be pertinent as the targeted API is part of TUI Musement's own infrastructure on the one hand.

On the other hand it also seems to be pertinent because the application would more than likely be run a handful of times during a day with a single call each time (provided that the result are not paginated).

#### WeatherApiClient
![alt WeatherApiClient retry policy](../README-images/WeatherApiClientRetryPolicy.png)
The retry policy used for `WeatherApiClient` is a more complex exponential backoff that generates sleep durations in a jittered manner.

It is a pretty robust policy more suited to weatherapi which is an external API.

## WeatherForecastService.cs
This is the orchestrator between both API clients.
![alt WeatherForecastService.cs](../README-images/WeatherForecastService.png)

The elephant in the room here is the use of console output for what could/should be logged.

Not that it would be particularly complicated, we just need and instance of ILogger that we would get from dependency injection.

I chose to keep it simple for this demonstration and not open the door to a multitude of questions, such as:
- Which message to ouput, which one to log ?
- Should we still log to the console ?
- Should we exclusively log to a file ?
- If so, which library should we use (as .NET does not provide one) ?

So I decided to keep it simple ...

## MusementApiClient.cs
Nothing really interesting going on here, sorry.
![alt MusementApiClient.cs](../README-images/MusementApiClient.png)

## WeatherApiClient.cs
Here we have two interesting things.
![alt WeatherApiClient.cs](../README-images/WeatherApiClient.png)

### Retrieval of the weatherapi key [constructor]
The api key is injected throught an instance of `IOptions<WeatherApiOptions>` which represents the corresponding section in the appsettings.json file:
![alt WeatherApiOptions](../README-images/WeatherApiOptions.png)

### Treatment of the error code returned by weatherapi
The error code are treated as described in the weaterapi documentation available here: https://www.weatherapi.com/docs/#intro-error-codes

I have categorised them in three categories:
- What is equivalent to a not found ressource returned by the API as a bad request with a 1006 error code for which we return a null reference
- What is in essence a configuration error in the endpoint or API key returned with different HTTP status and error code for which we return a specific `ApiConfigurationError`
- What is declared a an internal application error, which could be transient, returned by the API as bad request with a 9999 error code for which we return a specific `ApiInternalError`

The choice to return specific objects instead of exceptions is to accelerate the flow of the caller without making any assumptions on what he should be doing.

For example all of the `ApiConfigurationError` are more than likely to be permanent errors, meaning that we will get the same result for all the calls made to the weatherapi until we rectify something (the API ley, an endpoint).

So we could say that the consumer (`WeatherForecastService`) will have to terminate because there is no point in sending more requests and we could therefore afford, without much performance penalty, to throw an exception.

But I think this choice should be made by the consumer and not this client.

## Final note
Thank you very much for this interesting challenge that puts the mind in some of the problematics directly linked to TUI Musement business.


## Mock HttpClient/HttpClientFactory

Essentially mock the inner workings of the `HttpClient` via `HttpClientHandler` to return what you want. `HttpClientHandler` only has a `Send` and `SendAsync` method. All the stuff the `HttpClient` does - `Get, GetAsync, Post, etc` - it's all using `Send` under the hood.

```C#
// arrange
var mockLogger = new Mock<ILogger<GeocodingService>>();

var mockSettings = new Mock<IOptions<AppSettings>>();

mockSettings.Setup(s => s.Value)
   .Returns(new AppSettings()
   {
       GeocodingApiKey = "111",
   });

var mockHttpMessageHandler = new Mock<HttpMessageHandler>();

var url = "https://api.geoapify.com/v1/geocode/search?limit=1&text=neverland&apiKey=111";

mockHttpMessageHandler.Protected()
    .Setup<Task<HttpResponseMessage>>("SendAsync", ItExpr.Is<HttpRequestMessage>(req =>
        req.Method == HttpMethod.Get && req.RequestUri.ToString() == url),
    ItExpr.IsAny<CancellationToken>())
    .ReturnsAsync(
        new HttpResponseMessage()
        {
            Content = new StringContent(jsonResponse)
        }
    );

var mockHttpClient = new HttpClient(mockHttpMessageHandler.Object);

var mockHttpFactory = new Mock<IHttpClientFactory>();

mockHttpFactory.Setup(f => f.CreateClient(It.IsAny<string>()))
    .Returns(mockHttpClient);

return new GeocodingService(mockHttpFactory.Object, mockSettings.Object, mockLogger.Object);
```
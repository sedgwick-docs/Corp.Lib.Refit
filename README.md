# Corp.Lib.Refit

[![NuGet Version](https://img.shields.io/badge/NuGet-10.1.5-blue)](https://www.nuget.org/)
[![.NET](https://img.shields.io/badge/.NET-10.0-purple)](https://dotnet.microsoft.com/)

A streamlined library for configuring and integrating [Refit](https://github.com/reactiveui/refit) HTTP clients in ASP.NET Core applications with built-in support for certificate-based authentication, correlation ID tracking, and flexible JSON serialization options.

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Architecture Overview](#architecture-overview)
- [Detailed Usage Guide](#detailed-usage-guide)
  - [Step 1: Define Your API Interface](#step-1-define-your-api-interface)
  - [Step 2: Create a Service Interface](#step-2-create-a-service-interface)
  - [Step 3: Implement the Service](#step-3-implement-the-service)
  - [Step 4: Register the Refit Client](#step-4-register-the-refit-client)
  - [Step 5: Use the Service](#step-5-use-the-service)
- [Configuration Options](#configuration-options)
- [Extension Methods Reference](#extension-methods-reference)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Dependencies](#dependencies)
- [License](#license)

## Features

- **Simplified Refit Integration**: Extension methods for `WebApplicationBuilder` and `IServiceCollection` to streamline Refit client configuration
- **Certificate-Based Authentication**: Built-in support for PKCS#12 certificates with TLS 1.2 security, as well as certificate store lookup by thumbprint
- **Correlation ID Propagation**: Automatic correlation ID tracking across service calls via `CorrelationIdMessageHandler`
- **Flexible JSON Serialization**: Support for both `System.Text.Json` (default, recommended) and `Newtonsoft.Json`
- **HTTP Version Control**: Configure HTTP version policies for optimal performance
- **HttpClientFactory Integration**: Leverages `IHttpClientFactory` for proper connection pooling and lifecycle management

## Installation

Install the package via NuGet Package Manager:

```bash
dotnet add package Corp.Lib.Refit
```

Or via the Package Manager Console:

```powershell
Install-Package Corp.Lib.Refit
```

## Prerequisites

- **.NET 10.0** or later
- **ASP.NET Core** application (Web API, Razor Pages, or MVC)
- **Corp.Lib.Logging** package (automatically included as a dependency)
- Valid **PKCS#12 certificate** (`.pfx` file) for client authentication, **or** a certificate installed in the **local machine certificate store** (referenced by thumbprint)

## Quick Start

Here's a minimal example to get you started:

```csharp
// Program.cs
using Corp.Lib.Refit.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Register your Refit client
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com/v1",
    certificatePath: "/certs/client.pfx",
    password: "your-certificate-password"
);

var app = builder.Build();
app.Run();
```

Alternatively, if you have a certificate in the local machine store, reference it by thumbprint:

```csharp
// Program.cs
using Corp.Lib.Refit.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Register your Refit client using a certificate thumbprint
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com/v1",
    certificateThumbprint: "AB12CD34EF56..."
);

var app = builder.Build();
app.Run();
```

## Architecture Overview

Corp.Lib.Refit uses a three-layer architecture pattern for clean separation of concerns:

```
???????????????????????????????????????????????????????????????
?                    Your Application                         ?
???????????????????????????????????????????????????????????????
?  TServiceInterface (e.g., IWeatherService)                  ?
?  - Business logic contract                                  ?
?  - Injected into controllers/pages                          ?
???????????????????????????????????????????????????????????????
?  TServiceImplementation (e.g., WeatherService)              ?
?  - Implements TServiceInterface                             ?
?  - Contains business logic, error handling, mapping         ?
?  - Injects TRefitServiceInterface                           ?
???????????????????????????????????????????????????????????????
?  TRefitServiceInterface (e.g., IWeatherApiClient)           ?
?  - Refit interface with HTTP attributes                     ?
?  - Auto-implemented by Refit                                ?
?  - Handles HTTP communication                               ?
???????????????????????????????????????????????????????????????
?  Corp.Lib.Refit Infrastructure                              ?
?  - HttpClientFactory integration                            ?
?  - Certificate authentication                               ?
?  - Correlation ID propagation                               ?
?  - JSON serialization                                       ?
???????????????????????????????????????????????????????????????
```

## Detailed Usage Guide

### Step 1: Define Your API Interface

Create a Refit interface that defines the HTTP API contract. This interface will be automatically implemented by Refit.

```csharp
using Refit;

namespace MyApp.ApiClients;

/// <summary>
/// Refit interface for the Weather API.
/// This interface is auto-implemented by Refit.
/// </summary>
public interface IWeatherApiClient
{
    [Get("/forecasts")]
    Task<IEnumerable<WeatherForecast>> GetForecastsAsync(CancellationToken cancellationToken = default);

    [Get("/forecasts/{id}")]
    Task<WeatherForecast> GetForecastByIdAsync(int id, CancellationToken cancellationToken = default);

    [Get("/forecasts/city/{city}")]
    Task<WeatherForecast> GetForecastByCityAsync(string city, CancellationToken cancellationToken = default);

    [Post("/forecasts")]
    Task<WeatherForecast> CreateForecastAsync([Body] CreateForecastRequest request, CancellationToken cancellationToken = default);

    [Put("/forecasts/{id}")]
    Task<WeatherForecast> UpdateForecastAsync(int id, [Body] UpdateForecastRequest request, CancellationToken cancellationToken = default);

    [Delete("/forecasts/{id}")]
    Task DeleteForecastAsync(int id, CancellationToken cancellationToken = default);

    [Get("/forecasts/search")]
    Task<IEnumerable<WeatherForecast>> SearchForecastsAsync([Query] ForecastSearchParameters parameters, CancellationToken cancellationToken = default);
}
```

### Step 2: Create a Service Interface

Define a business-level service interface that your application will consume. This abstracts away the HTTP details.

```csharp
namespace MyApp.Services;

/// <summary>
/// Business service interface for weather operations.
/// This is what your controllers/pages will inject.
/// </summary>
public interface IWeatherService
{
    Task<IEnumerable<WeatherForecastDto>> GetAllForecastsAsync(CancellationToken cancellationToken = default);
    Task<WeatherForecastDto?> GetForecastByIdAsync(int id, CancellationToken cancellationToken = default);
    Task<WeatherForecastDto?> GetForecastByCityAsync(string city, CancellationToken cancellationToken = default);
    Task<WeatherForecastDto> CreateForecastAsync(CreateForecastDto dto, CancellationToken cancellationToken = default);
    Task<WeatherForecastDto> UpdateForecastAsync(int id, UpdateForecastDto dto, CancellationToken cancellationToken = default);
    Task<bool> DeleteForecastAsync(int id, CancellationToken cancellationToken = default);
}
```

### Step 3: Implement the Service

Create a service implementation that wraps the Refit client with business logic, error handling, and data mapping.

```csharp
using Microsoft.Extensions.Logging;
using Refit;

namespace MyApp.Services;

/// <summary>
/// Implementation of IWeatherService that uses the Refit API client.
/// </summary>
public class WeatherService : IWeatherService
{
    private readonly IWeatherApiClient _apiClient;
    private readonly ILogger<WeatherService> _logger;

    public WeatherService(IWeatherApiClient apiClient, ILogger<WeatherService> logger)
    {
        _apiClient = apiClient ?? throw new ArgumentNullException(nameof(apiClient));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    }

    public async Task<IEnumerable<WeatherForecastDto>> GetAllForecastsAsync(CancellationToken cancellationToken = default)
    {
        try
        {
            _logger.LogInformation("Fetching all weather forecasts");
            var forecasts = await _apiClient.GetForecastsAsync(cancellationToken);
            return forecasts.Select(MapToDto);
        }
        catch (ApiException ex)
        {
            _logger.LogError(ex, "API error while fetching forecasts. Status: {StatusCode}", ex.StatusCode);
            throw new WeatherServiceException("Failed to retrieve weather forecasts", ex);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Network error while fetching forecasts");
            throw new WeatherServiceException("Network error occurred while fetching forecasts", ex);
        }
    }

    public async Task<WeatherForecastDto?> GetForecastByIdAsync(int id, CancellationToken cancellationToken = default)
    {
        try
        {
            _logger.LogInformation("Fetching weather forecast with ID: {ForecastId}", id);
            var forecast = await _apiClient.GetForecastByIdAsync(id, cancellationToken);
            return MapToDto(forecast);
        }
        catch (ApiException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            _logger.LogWarning("Forecast with ID {ForecastId} not found", id);
            return null;
        }
        catch (ApiException ex)
        {
            _logger.LogError(ex, "API error while fetching forecast {ForecastId}. Status: {StatusCode}", id, ex.StatusCode);
            throw new WeatherServiceException($"Failed to retrieve forecast with ID {id}", ex);
        }
    }

    public async Task<WeatherForecastDto?> GetForecastByCityAsync(string city, CancellationToken cancellationToken = default)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(city);

        try
        {
            _logger.LogInformation("Fetching weather forecast for city: {City}", city);
            var forecast = await _apiClient.GetForecastByCityAsync(city, cancellationToken);
            return MapToDto(forecast);
        }
        catch (ApiException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            _logger.LogWarning("Forecast for city {City} not found", city);
            return null;
        }
        catch (ApiException ex)
        {
            _logger.LogError(ex, "API error while fetching forecast for city {City}. Status: {StatusCode}", city, ex.StatusCode);
            throw new WeatherServiceException($"Failed to retrieve forecast for city {city}", ex);
        }
    }

    public async Task<WeatherForecastDto> CreateForecastAsync(CreateForecastDto dto, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(dto);

        try
        {
            _logger.LogInformation("Creating new weather forecast for city: {City}", dto.City);
            var request = MapToCreateRequest(dto);
            var forecast = await _apiClient.CreateForecastAsync(request, cancellationToken);
            _logger.LogInformation("Successfully created forecast with ID: {ForecastId}", forecast.Id);
            return MapToDto(forecast);
        }
        catch (ApiException ex)
        {
            _logger.LogError(ex, "API error while creating forecast. Status: {StatusCode}", ex.StatusCode);
            throw new WeatherServiceException("Failed to create weather forecast", ex);
        }
    }

    public async Task<WeatherForecastDto> UpdateForecastAsync(int id, UpdateForecastDto dto, CancellationToken cancellationToken = default)
    {
        ArgumentNullException.ThrowIfNull(dto);

        try
        {
            _logger.LogInformation("Updating weather forecast with ID: {ForecastId}", id);
            var request = MapToUpdateRequest(dto);
            var forecast = await _apiClient.UpdateForecastAsync(id, request, cancellationToken);
            _logger.LogInformation("Successfully updated forecast with ID: {ForecastId}", id);
            return MapToDto(forecast);
        }
        catch (ApiException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            _logger.LogWarning("Cannot update forecast - ID {ForecastId} not found", id);
            throw new WeatherServiceException($"Forecast with ID {id} not found", ex);
        }
        catch (ApiException ex)
        {
            _logger.LogError(ex, "API error while updating forecast {ForecastId}. Status: {StatusCode}", id, ex.StatusCode);
            throw new WeatherServiceException($"Failed to update forecast with ID {id}", ex);
        }
    }

    public async Task<bool> DeleteForecastAsync(int id, CancellationToken cancellationToken = default)
    {
        try
        {
            _logger.LogInformation("Deleting weather forecast with ID: {ForecastId}", id);
            await _apiClient.DeleteForecastAsync(id, cancellationToken);
            _logger.LogInformation("Successfully deleted forecast with ID: {ForecastId}", id);
            return true;
        }
        catch (ApiException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            _logger.LogWarning("Cannot delete forecast - ID {ForecastId} not found", id);
            return false;
        }
        catch (ApiException ex)
        {
            _logger.LogError(ex, "API error while deleting forecast {ForecastId}. Status: {StatusCode}", id, ex.StatusCode);
            throw new WeatherServiceException($"Failed to delete forecast with ID {id}", ex);
        }
    }

    // Mapping methods
    private static WeatherForecastDto MapToDto(WeatherForecast forecast) => new()
    {
        Id = forecast.Id,
        City = forecast.City,
        Date = forecast.Date,
        TemperatureC = forecast.TemperatureC,
        TemperatureF = forecast.TemperatureF,
        Summary = forecast.Summary
    };

    private static CreateForecastRequest MapToCreateRequest(CreateForecastDto dto) => new()
    {
        City = dto.City,
        Date = dto.Date,
        TemperatureC = dto.TemperatureC,
        Summary = dto.Summary
    };

    private static UpdateForecastRequest MapToUpdateRequest(UpdateForecastDto dto) => new()
    {
        TemperatureC = dto.TemperatureC,
        Summary = dto.Summary
    };
}
```

### Step 4: Register the Refit Client

Register the Refit client in your `Program.cs` using one of the provided extension methods.

#### Using WebApplicationBuilder Extension

```csharp
// Program.cs
using Corp.Lib.Refit.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Option 1: Basic registration with base address including version
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com/v1",
    certificatePath: builder.Configuration["Api:CertificatePath"]!,
    password: builder.Configuration["Api:CertificatePassword"]!
);

// Option 2: Registration with explicit version control
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com",
    version: new Version(2, 0),
    certificatePath: builder.Configuration["Api:CertificatePath"]!,
    password: builder.Configuration["Api:CertificatePassword"]!
);

// Add other services
builder.Services.AddControllers();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### Using IServiceCollection Extension

```csharp
// Program.cs
using Corp.Lib.Refit.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Using IServiceCollection extension
builder.Services.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com/v1",
    certificatePath: builder.Configuration["Api:CertificatePath"]!,
    password: builder.Configuration["Api:CertificatePassword"]!
);

var app = builder.Build();
app.Run();
```

#### Using Certificate Store Thumbprint

If you prefer to use a certificate installed in the local machine certificate store instead of a `.pfx` file:

```csharp
// Program.cs
using Corp.Lib.Refit.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Option 1: Basic registration with certificate thumbprint
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com/v1",
    certificateThumbprint: builder.Configuration["Api:CertificateThumbprint"]!
);

// Option 2: Registration with explicit version control and certificate thumbprint
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.weather.example.com",
    version: new Version(2, 0),
    certificateThumbprint: builder.Configuration["Api:CertificateThumbprint"]!
);

var app = builder.Build();
app.Run();
```

> **Note**: Thumbprint-based overloads always use `System.Text.Json` for serialization. The certificate must be installed in the `LocalMachine\My` store.

#### Configuration via appsettings.json

```json
{
  "Api": {
    "WeatherService": {
      "BaseAddress": "https://api.weather.example.com/v1",
      "CertificatePath": "/certs/client.pfx",
      "CertificatePassword": "your-secure-password"
    }
  }
}
```

```csharp
// Program.cs
var weatherConfig = builder.Configuration.GetSection("Api:WeatherService");

builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: weatherConfig["BaseAddress"]!,
    certificatePath: weatherConfig["CertificatePath"]!,
    password: weatherConfig["CertificatePassword"]!
);
```

### Step 5: Use the Service

Inject and use the service in your controllers, Razor Pages, or other components.

#### In a Controller

```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyApp.Controllers;

[ApiController]
[Route("api/[controller]")]
public class WeatherController : ControllerBase
{
    private readonly IWeatherService _weatherService;
    private readonly ILogger<WeatherController> _logger;

    public WeatherController(IWeatherService weatherService, ILogger<WeatherController> logger)
    {
        _weatherService = weatherService;
        _logger = logger;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<WeatherForecastDto>>> GetAll(CancellationToken cancellationToken)
    {
        var forecasts = await _weatherService.GetAllForecastsAsync(cancellationToken);
        return Ok(forecasts);
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<WeatherForecastDto>> GetById(int id, CancellationToken cancellationToken)
    {
        var forecast = await _weatherService.GetForecastByIdAsync(id, cancellationToken);
        
        if (forecast is null)
            return NotFound();
            
        return Ok(forecast);
    }

    [HttpGet("city/{city}")]
    public async Task<ActionResult<WeatherForecastDto>> GetByCity(string city, CancellationToken cancellationToken)
    {
        var forecast = await _weatherService.GetForecastByCityAsync(city, cancellationToken);
        
        if (forecast is null)
            return NotFound();
            
        return Ok(forecast);
    }

    [HttpPost]
    public async Task<ActionResult<WeatherForecastDto>> Create([FromBody] CreateForecastDto dto, CancellationToken cancellationToken)
    {
        var forecast = await _weatherService.CreateForecastAsync(dto, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = forecast.Id }, forecast);
    }

    [HttpPut("{id:int}")]
    public async Task<ActionResult<WeatherForecastDto>> Update(int id, [FromBody] UpdateForecastDto dto, CancellationToken cancellationToken)
    {
        var forecast = await _weatherService.UpdateForecastAsync(id, dto, cancellationToken);
        return Ok(forecast);
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id, CancellationToken cancellationToken)
    {
        var deleted = await _weatherService.DeleteForecastAsync(id, cancellationToken);
        
        if (!deleted)
            return NotFound();
            
        return NoContent();
    }
}
```

#### In a Razor Page

```csharp
// Pages/Weather/Index.cshtml.cs
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace MyApp.Pages.Weather;

public class IndexModel : PageModel
{
    private readonly IWeatherService _weatherService;
    private readonly ILogger<IndexModel> _logger;

    public IndexModel(IWeatherService weatherService, ILogger<IndexModel> logger)
    {
        _weatherService = weatherService;
        _logger = logger;
    }

    public IEnumerable<WeatherForecastDto> Forecasts { get; private set; } = [];

    public async Task OnGetAsync(CancellationToken cancellationToken)
    {
        try
        {
            Forecasts = await _weatherService.GetAllForecastsAsync(cancellationToken);
        }
        catch (WeatherServiceException ex)
        {
            _logger.LogError(ex, "Failed to load weather forecasts");
            ModelState.AddModelError(string.Empty, "Unable to load weather data. Please try again later.");
        }
    }
}
```

```html
<!-- Pages/Weather/Index.cshtml -->
@page
@model MyApp.Pages.Weather.IndexModel

<h1>Weather Forecasts</h1>

@if (!ViewData.ModelState.IsValid)
{
    <div class="alert alert-danger">
        @Html.ValidationSummary()
    </div>
}

<table class="table">
    <thead>
        <tr>
            <th>City</th>
            <th>Date</th>
            <th>Temperature (°C)</th>
            <th>Temperature (°F)</th>
            <th>Summary</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var forecast in Model.Forecasts)
        {
            <tr>
                <td>@forecast.City</td>
                <td>@forecast.Date.ToShortDateString()</td>
                <td>@forecast.TemperatureC</td>
                <td>@forecast.TemperatureF</td>
                <td>@forecast.Summary</td>
            </tr>
        }
    </tbody>
</table>
```

## Configuration Options

### HTTP Client Name

The `httpClientName` parameter provides a unique identifier for the HTTP client in the `IHttpClientFactory`. This is useful when you need to configure multiple Refit clients:

```csharp
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",  // Unique name for this client
    // ...
);

builder.AddRefitClient<IUserService, UserService, IUserApiClient>(
    httpClientName: "UserApi",  // Different name for another client
    // ...
);
```

### Service Base Address

The base URL for the API, optionally including the API version:

```csharp
// Without version in URL
serviceBaseAddress: "https://api.example.com"

// With version in URL (recommended)
serviceBaseAddress: "https://api.example.com/v1"
```

### HTTP Version

Control the HTTP protocol version used for requests:

```csharp
// Use HTTP/2.0
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.example.com/v1",
    version: new Version(2, 0),  // HTTP/2.0
    certificatePath: "/certs/client.pfx",
    password: "password"
);
```

### JSON Serialization

By default, the library uses `System.Text.Json` which is faster and more secure. If you need `Newtonsoft.Json` for compatibility:

```csharp
// Using Newtonsoft.Json (deprecated - use only when necessary)
#pragma warning disable CS0618
builder.AddRefitClient<IWeatherService, WeatherService, IWeatherApiClient>(
    httpClientName: "WeatherApi",
    serviceBaseAddress: "https://api.example.com/v1",
    certificatePath: "/certs/client.pfx",
    password: "password",
    useNewtonsoft: true  // Enable Newtonsoft.Json
);
#pragma warning restore CS0618
```

> ?? **Warning**: The `useNewtonsoft` parameter is marked as obsolete. `System.Text.Json` is strongly preferred as it's faster and more secure.

## Extension Methods Reference

### WebApplicationBuilder Extensions

| Method | Description |
|--------|-------------|
| `AddRefitClient<...>(builder, httpClientName, serviceBaseAddress, certificatePath, password)` | Basic registration with certificate file authentication |
| `AddRefitClient<...>(builder, httpClientName, serviceBaseAddress, version, certificatePath, password)` | Registration with certificate file and explicit HTTP version |
| `AddRefitClient<...>(builder, httpClientName, serviceBaseAddress, certificateThumbprint)` | Registration with certificate store thumbprint |
| `AddRefitClient<...>(builder, httpClientName, serviceBaseAddress, version, certificateThumbprint)` | Registration with certificate store thumbprint and explicit HTTP version |
| `AddRefitClient<...>(builder, httpClientName, serviceBaseAddress, certificatePath, password, useNewtonsoft)` | Registration with Newtonsoft.Json option (deprecated) |
| `AddRefitClient<...>(builder, httpClientName, serviceBaseAddress, version, certificatePath, password, useNewtonsoft)` | Full configuration with Newtonsoft.Json option (deprecated) |

### IServiceCollection Extensions

All the same methods are available on `IServiceCollection` for scenarios where you don't have access to `WebApplicationBuilder`:

```csharp
// Certificate file
services.AddRefitClient<TServiceInterface, TServiceImplementation, TRefitServiceInterface>(
    httpClientName,
    serviceBaseAddress,
    certificatePath,
    password
);

// Certificate store thumbprint
services.AddRefitClient<TServiceInterface, TServiceImplementation, TRefitServiceInterface>(
    httpClientName,
    serviceBaseAddress,
    certificateThumbprint
);
```

## Best Practices

### 1. Use the Three-Layer Pattern

Always separate your Refit interface (HTTP contract), service interface (business contract), and service implementation:

```
IWeatherApiClient    ? Refit HTTP interface (auto-implemented)
IWeatherService      ? Business service interface
WeatherService       ? Business service implementation
```

### 2. Handle API Exceptions Appropriately

Wrap Refit's `ApiException` in domain-specific exceptions:

```csharp
try
{
    return await _apiClient.GetDataAsync();
}
catch (ApiException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    return null;  // Or throw a domain-specific exception
}
catch (ApiException ex)
{
    _logger.LogError(ex, "API error: {StatusCode}", ex.StatusCode);
    throw new MyServiceException("Failed to get data", ex);
}
```

### 3. Store Certificates Securely

Never store certificate passwords in source code. Use:
- Azure Key Vault
- Environment variables
- User secrets (for development)
- Configuration providers

```csharp
// Development: User secrets
var certPath = builder.Configuration["Api:CertificatePath"];
var certPassword = builder.Configuration["Api:CertificatePassword"];

// Production: Azure Key Vault or environment variables
```

### 4. Use Cancellation Tokens

Always pass `CancellationToken` through your service methods:

```csharp
public interface IWeatherApiClient
{
    [Get("/forecasts")]
    Task<IEnumerable<WeatherForecast>> GetForecastsAsync(CancellationToken cancellationToken = default);
}
```

### 5. Configure Timeouts

Consider adding timeout policies for resilience:

```csharp
builder.Services.AddRefitClient<IWeatherApiClient>(/* ... */)
    .ConfigureHttpClient(c => c.Timeout = TimeSpan.FromSeconds(30));
```

### 6. Use Correlation IDs

The library automatically adds correlation ID tracking via `CorrelationIdMessageHandler`. Ensure your logging infrastructure from `Corp.Lib.Logging` is properly configured to take advantage of this feature for distributed tracing.

## Troubleshooting

### Certificate Errors

**Problem**: `CryptographicException` when loading certificate

**Solutions**:
- Verify the certificate path is correct and accessible
- Ensure the password is correct
- Check that the certificate is a valid PKCS#12 (.pfx) file
- On Linux, ensure proper file permissions

### Certificate Not Found in Store

**Problem**: `InvalidOperationException` — certificate with thumbprint not found

**Solutions**:
- Verify the thumbprint value is correct (no extra spaces or invisible characters)
- Ensure the certificate is installed in the `LocalMachine\My` (Personal) store
- Run the application with sufficient permissions to access the machine certificate store
- Use the Certificate Manager (`certlm.msc`) to confirm the certificate is present

### Connection Refused

**Problem**: `HttpRequestException` with connection refused

**Solutions**:
- Verify the `serviceBaseAddress` is correct
- Check network connectivity to the target server
- Ensure firewall rules allow outbound connections
- Verify the target service is running

### SSL/TLS Errors

**Problem**: SSL handshake failures

**Solutions**:
- Ensure the server supports TLS 1.2
- Verify the client certificate is trusted by the server
- Check certificate expiration dates

### Serialization Errors

**Problem**: JSON deserialization failures

**Solutions**:
- Ensure your DTOs match the API response structure
- Check for nullable property mismatches
- Consider using `[JsonPropertyName]` attributes for property mapping
- If using legacy APIs, consider the Newtonsoft.Json option

### Service Not Registered

**Problem**: `InvalidOperationException` - service not registered

**Solutions**:
- Ensure `AddRefitClient` is called before building the app
- Verify the generic type parameters are correct
- Check that all three types (interface, implementation, Refit interface) are specified

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| [Refit.HttpClientFactory](https://github.com/reactiveui/refit) | 10.0.1 | HttpClientFactory integration for Refit |
| [Refit.Newtonsoft.Json](https://github.com/reactiveui/refit) | 10.0.1 | Newtonsoft.Json serialization support |
| [Corp.Lib.Logging](https://dev.azure.com/IT-Specialty/Projects/_git/Corp.Solution.Logging) | 10.1.4 | Correlation ID message handler and logging |

## License

This package is proprietary software. All rights reserved.

---

## Support

For issues, feature requests, or questions, please contact the development team or open an issue in the Azure DevOps repository.

**Repository**: [Corp.Solution.Refit](https://dev.azure.com/IT-Specialty/Projects/_git/Corp.Solution.Refit)

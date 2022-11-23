
# Create an Open ID Connect Code Flow client with Razor Pages

Test user : alice@alice.ch
Test password: Alice!01

Test user : bob@bob.ch
Test password: Bobby!01

## Exercise Goal

The aim of this exerise is to learn how to implement an OpenID Connect Code flow authentication in an ASP.NET Core Razor Page application.

## Step 1 

Add the required nuget packages to the RazorPageOidcClient project.

```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.OpenIdConnect" Version="6.0.10" />
```

## Step 2 

Configure the Startup authentication. The configuration must exactly match the configuration in the STS. 

```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    Configuration.GetSection("OpenIDConnectSettings").Bind(options);
    options.Authority = Configuration["OpenIDConnectSettings:Authority"];
    options.ClientId = Configuration["OpenIDConnectSettings:ClientId"];
    options.ClientSecret = Configuration["OpenIDConnectSettings:ClientSecret"];

    options.SignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.ResponseType = OpenIdConnectResponseType.Code;

    options.SaveTokens = true;
    options.GetClaimsFromUserInfoEndpoint = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        NameClaimType = "name"
    };
});

services.AddAuthorization();
```

## Step 3

Add the following settings to the Razor page app.settings.

```json
  "OpenIDConnectSettings": {
    "Authority": "https://localhost:44318",
    "ClientId": "oidc-pkce-confidential",
    "ClientSecret": "oidc-pkce-confidential_secret"
  },
```

## Step 4

The UseRouting must be implemented before the UseAuthentication and the UseAuthorization

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
	JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();

	if (env.IsDevelopment())
	{
		app.UseDeveloperExceptionPage();
	}
	else
	{
		app.UseExceptionHandler("/Error");
		app.UseHsts();
	}

	app.UseHttpsRedirection();
	app.UseStaticFiles();

	app.UseRouting();

	app.UseAuthentication();
	app.UseAuthorization();

	app.UseEndpoints(endpoints =>
	{
		endpoints.MapRazorPages();
	});
}
```

## Step 5

Add the Authorize attribute to the protected razor pages, for example the Index.cshtml.cs file

```csharp
[Authorize]
```

A better way would be to force the whole application to be authorized and opt out for unsecure pages

```csharp
services.AddRazorPages().AddMvcOptions(options =>
{
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    options.Filters.Add(new AuthorizeFilter(policy));
});
```

## Step 6

Add a new Logout and SignedOut Razor page to the client project

Implement the sign out page

```csharp
[Authorize]
public class LogoutModel : PageModel
{
    public IActionResult OnGetAsync()
    {
        return SignOut(new AuthenticationProperties
        {
            RedirectUri = "/SignedOut"
        },
        CookieAuthenticationDefaults.AuthenticationScheme,
        OpenIdConnectDefaults.AuthenticationScheme);
    }
}
```

## Step 7

Add the user email to the shared _Layout page, display logout button only when logged in

```
@if (Context.User.Identity!.IsAuthenticated)
{
	<li class="nav-item">
		<a class="nav-link text-dark" asp-area="" asp-page="/Logout">Logout</a>
	</li>

	<span class="nav-link text-dark">Hi @Context.User.Identity.Name</span>
}
else
{
	<li class="nav-item">
		<a class="nav-link text-dark" asp-area="" asp-page="/Index">Login</a>
	</li>
}
```

### Step 8

Run the applications in IIS Express. Start both projects in Visual Studio and log in with the credentials given at the start of this file.

### Step 9 (Optional)

Have a look at the log txt files located in the solution folder. You will see some familiar communication between IDP and client.

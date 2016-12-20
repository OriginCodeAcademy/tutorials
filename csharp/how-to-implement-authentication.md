# Authentication in CSharp

In this tutorial, you will learn how to create an ASP.NET REST API that facilitates user authentication (Registration and login).

## Sources
[Token Based Authentication - Taiseer Joudeh](http://bitoftech.net/2014/06/01/token-based-authentication-asp-net-web-api-2-owin-asp-net-identity/)<br />
[ASP.NET Identity with Entity Framework](http://odetocode.com/blogs/scott/archive/2014/01/03/asp-net-identity-with-the-entity-framework.aspx)

## Table of Contents

- [ ] Project Setup
	- [ ] Convert project to OWIN project
	- [ ] Configure Web API
	- [ ] Install ASP Identity
	- [ ] Create a User model
	- [ ] Create a DataContext
	- [ ] Run Migrations
- [ ] Implement Registration
	- [ ] Create a RegistrationModel
	- [ ] Create a UsersController
	- [ ] Test UsersController
- [ ] Implement Authentication (Login)
- [ ] Implement Authorization (Protecting controllers)

## Project Setup

### Convert project to OWIN project

- [ ] Create a new Web API Project.
- [ ] Install the following three NuGet packages:

--------

`Microsoft.AspNet.WebApi.Owin`<br />
> This package allows you to host ASP.NET Web API within an OWIN server and provides access to additional OWIN features.

-------------

`Microsoft.Owin.Host.SystemWeb`<br />
> OWIN server that enables OWIN-based applications to run on IIS using the ASP.NET request pipeline.

---------

`Microsoft.Owin.Cors`
> This package contains the components to enable Cross-Origin Resource Sharing (CORS) in OWIN middleware.

---------

- [ ] Delete your `Global.asax` file.
- [ ] Add a new file called `Startup.cs` into your project.

```cs
using Microsoft.Owin;
using Owin;
using System.Web.Http;

[assembly: OwinStartup(typeof(Acme.Api.Startup))]
namespace Acme.Api
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            var config = new HttpConfiguration();
            WebApiConfig.Register(config);
            app.UseCors(Microsoft.Owin.Cors.CorsOptions.AllowAll);
            app.UseWebApi(config);
        }
    }
}
```

### Configure Web API

In `App_Start/WebApiConfig.cs`

- [ ] Turn off XML Serialization

```csharp
config.Formatters.Remove(config.Formatters.XmlFormatter);
```

- [ ] Turn on `CamelCasePropertyNamesContractResolver`

```csharp
config.Formatters.JsonFormatter.SerializerSettings.ContractResolver = 
	new CamelCasePropertyNamesContractResolver();
```


### Install ASP Identity System

- [ ] Install the following two NuGet packages:

---

`Microsoft.AspNet.Identity.Owin`<br />
> Owin implementation for ASP.NET Identity.

---

`Microsoft.AspNet.Identity.EntityFramework`
> ASP.NET Identity providers that use Entity Framework.

---

### Create a User model

- [ ] In your models folder, add the following class

```csharp
using Microsoft.AspNet.Identity.EntityFramework;

namespace Acme.Api.Models
{
    public class User : IdentityUser
    {
		// custom fields + relationships go here
    }
}
```

### Create a DataContext
- [ ] Create a DataContext class that inherits from the `IdentityDbContext` class. You can read more about this class on [Scott Allens Blog](http://odetocode.com/blogs/scott/archive/2014/01/03/asp-net-identity-with-the-entity-framework.aspx)

```cs
using Acme.Api.Models;
using Microsoft.AspNet.Identity.EntityFramework;
using System.Data.Entity;

namespace Acme.Api.Infrastructure
{
    public class AcmeDataContext : IdentityDbContext<User>
    {
        public AcmeDataContext() : base("Acme.Api")
        {

        }

        protected override void OnModelCreating(DbModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
        }
    }
}
``` 

### Run Migrations
- [ ] Open `Package Manager Console` and run the following commands

`Enable-Migrations`<br />
`Add-Migration InitialMigration`<br />
`Update-Database`

## Implement Registration

The first thing our backend needs to facilitate is user registration.

### Create a RegistrationModel

- [ ] Create a `RegistrationModel.cs` file that contains the fields that will eventually be provided by the registration form on the front end.

```cs
using System.ComponentModel.DataAnnotations;

namespace Acme.Api.Models
{
    public class RegistrationModel
    {
        [Required]
        public string EmailAddress { get; set; }

        [Required, MinLength(8), DataType(DataType.Password)]
        public string Password { get; set; }

        [Required]
        [Compare("Password", ErrorMessage = "Passwords do not match.")]
        public string ConfirmPassword { get; set; }
    }
}
```

### Create a UsersController

- [ ] Create an empty `UsersController.cs` file with the following contents

```cs
using Acme.Api.Infrastructure;
using Acme.Api.Models;
using Microsoft.AspNet.Identity;
using Microsoft.AspNet.Identity.EntityFramework;
using System.Web.Http;

namespace Acme.Api.Controllers
{
    public class UsersController : ApiController
    {
        private UserManager<User> _userManager;

        public UsersController()
        {
            var db = new AcmeDataContext(db);
            var store = new UserStore<User>();

            _userManager = new UserManager<User>(store);
        }

        // POST: api/users/register
        [AllowAnonymous]
        [Route("api/users/register")]
        public IHttpActionResult Register(RegistrationModel registration)
        {
            if(!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }

            var user = new User
            {
                UserName = registration.EmailAddress
            };

            var result = _userManager.Create(user, registration.Password);

            if(result.Succeeded)
            {
                return Ok();
            }
            else
            {
                return BadRequest("Invalid user registration");
            }
        }

        protected override void Dispose(bool disposing)
        {
            _userManager.Dispose();
        }
    }
}
```

### Test UsersController

Open Postman and send an HTTP Post request to the API endpoint. You should expect a `200` response.

```json
{
	"emailAddress": "john@smith.com",
	"password": "password123",
	"confirmPassword": "password123"
}
```

## Implement Authentication (Login)

- [ ] Install the OAuth Nuget package

---

`Microsoft.Owin.Security.OAuth`
> Middleware that enables an application to support any standard OAuth 2.0 authentication workflow.

---

- [ ] Add a new method named `ConfigureOAuth` to `Startup.cs`, and call it within the `Configuration` method.

```cs
public void ConfigureOAuth(IAppBuilder app)
{
    var authorizationOptions = new OAuthAuthorizationServerOptions
    {
        AllowInsecureHttp = true,
        TokenEndpointPath = new PathString("/api/users/login"),
        AccessTokenExpireTimeSpan = TimeSpan.FromDays(1),
        Provider = new LocalAuthorizationProvider()
    };

    var authenticationOptions = new OAuthBearerAuthenticationOptions();

    app.UseOAuthAuthorizationServer(authorizationOptions);
    app.UseOAuthBearerAuthentication(authenticationOptions);
}
```

- [ ] Create a new folder called `Providers` and add a new `LocalAuthorizationProvider.cs` file

```cs
using Acme.Api.Infrastructure;
using Acme.Api.Models;
using Microsoft.AspNet.Identity;
using Microsoft.AspNet.Identity.EntityFramework;
using Microsoft.Owin.Security.OAuth;
using System.Security.Claims;
using System.Threading.Tasks;

namespace Acme.Api.Providers
{
    public class LocalAuthorizationProvider : OAuthAuthorizationServerProvider
    {
        public override async Task GrantResourceOwnerCredentials(OAuthGrantResourceOwnerCredentialsContext context)
        {
            context.OwinContext.Response.Headers.Add("Access-Control-Allow-Origin", new[] { "*" });

            var db = new AcmeDataContext();
            var store = new UserStore<User>(db);

            using (var manager = new UserManager<User>(store))
            {
                var user = manager.Find(context.UserName, context.Password);

                if (user == null)
                {
                    context.SetError("invalid_grant", "Incorrect username or password");
                    return;
                }
            }

            var identity = new ClaimsIdentity(context.Options.AuthenticationType);
            identity.AddClaim(new Claim(ClaimTypes.Name, context.UserName));
            identity.AddClaim(new Claim(ClaimTypes.Role, "user"));

            context.Validated(identity);
        }

        public override async Task ValidateClientAuthentication(OAuthValidateClientAuthenticationContext context)
        {
            context.Validated();
        }
    }
}
```

### Test Login

Now, let's test the login functionality.

- [ ] Open up postman, issue an HTTP post request to `/api/users/login`

<img src="http://i.imgur.com/TayhDLN.png" />

You should see the following

<img src="http://i.imgur.com/BfvRaN4.png" />

## Implement Authorization (Protecting Controllers)

Now for the easy bit. Create your API as normal, then once you've done that - simply add the `[Authorize]` attribute above the methods you'd like to protect. Take this `OrdersController.cs` for example.

```csharp
using System.Collections.Generic;
using System.Web.Http;

namespace ListIt.Controllers
{
    public class OrdersController : ApiController
    {
        // GET: api/Orders
        [Authorize]
        public IEnumerable<string> Get()
        {
            // the usual
        }

        // GET: api/Orders/5
        [Authorize]
        public string Get(int id)
        {
            // the usual
        }

        // POST: api/Orders
        [Authorize]
        public void Post([FromBody]string value)
        {
            // the usual
        }

        // PUT: api/Orders/5
        [Authorize]
        public void Put(int id, [FromBody]string value)
        {
            // the usual
        }

        // DELETE: api/Orders/5
        [Authorize]
        public void Delete(int id)
        {
            // the usual
        }
    }
}
```

Once you've done this for your controllers, you'll notice that when you try to call endpoints marked with `[Authorize]` in Postman that you'll get a `401 Unauthorized` response back from the server.

In the next tutorial, we'll talk about how to "unlock" these now protected endpoints.

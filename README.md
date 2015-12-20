Guardian
========

An authentication framework for use with Elixir applications.

Guardian is based on similar ideas to Warden but is re-imagined
for modern systems where Elixir manages the authentication requirements.

Guardian remains a functional system. It integrates with Plug, but can be used
outside of it. If you're implementing a TCP/UDP protocol directly, or want to
utilize your authentication via channels, Guardian is your friend.

The core currency of authentication in Guardian is JWT. You can use the JWT to
authenticate web endpoints, channels, and TCP sockets and it can contain any
authenticated assertions that the issuer wants to include.

## Installation

Add Guardian to your application

mix.deps

```elixir
defp deps do
  [
    # ...
    {:guardian, "~> 0.8.0"}
    # ...
  ]
end
```

config.exs

```elixir
config :guardian, Guardian,
  allowed_algos: ["HS512"], # optional
  verify_module: Guardian.JWT,  # optional
  issuer: "MyApp",
  ttl: { 30, :days },
  verify_issuer: true, # optional
  secret_key: <guardian secret key>,
  serializer: MyApp.GuardianSerializer
```

The items in the configuration allow you to tailor how the JWT generation behaves.

* `secret_key` - The secret used to sign the key
* `allowed_algos` - The list of algorithms (must be compatible with JOSE). The first is used as the encoding key. Default is: ["HS512"]
* `verify_module` - Provides a mechanism to setup your own validations for items
  in the token. Default is `Guardian.JWT`
* `issuer` - The entry to put into the token as the issuer. This can be used in congunction with `verify_issuer`
* `ttl` - The default ttl of a token
* `verify_issuer` - If set to true, the issuer will be verified to be the same issuer as specified in the `issuer` field
* `secret_key` - The key to sign the tokens
* `serializer` The serializer that serializes the 'sub' (Subject) field into and out of the token.

## Serializer

The serializer knows how to encode and decode your resource into and out of the
token. A simple serializer:

```elixir
defmodule MyApp.GuardianSerializer do
  @behaviour Guardian.Serializer

  alias MyApp.Repo
  alias MyApp.User

  def for_token(user = %User{}), do: { :ok, "User:#{user.id}" }
  def for_token(_), do: { :error, "Unknown resource type" }

  def from_token("User:" <> id), do: { :ok, Repo.get(User, id) }
  def from_token(_), do: { :error, "Unknown resource type" }
end
```

## Plug API

Guardian ships with some plugs to help integrate into your application.

### Guardian.Plug.VerifySession

Looks for a token in the session. Useful for browser sessions.
If one is not found, this does nothing.

### Guardian.Plug.VerifyHeader

Looks for a token in the Authorization header. Useful for apis.
If one is not found, this does nothing.

### Guardian.Plug.EnsureAuthenticated

Looks for a previously verified token. If one is found, continues, otherwise it
will call the `:unauthenticated` function of your handler.

When you ensure a session, you must declare an error handler. This can be done
as part of a pipeline or inside a pheonix controller.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller

  plug Guardian.Plug.EnsureAuthenticated, handler: MyApp.MyAuthErrorHandler
end
```

The failure function must receive the connection, and the connection params.

### Guardian.Plug.EnsurePermissions

Looks for a previously verified token. If one is found, confirms that all listed
permissions are present in the token. If not, the `:unauthorized` function is called on your handler.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller

  plug Guardian.Plug.EnsurePermissions, handler: MyApp.MyAuthErrorHandler, default: [:read, :write]
end
```

### Pipelines

These plugs can be used to construct pipelines in Phoenix.

```elixir
pipeline :browser_session do
  plug Guardian.Plug.VerifySession
  plug Guardian.Plug.LoadResource
end

pipeline :api do
  plug :accepts, ["json"]
  plug Guardian.Plug.VerifyHeader
  plug Guardian.Plug.LoadResource
end

scope "/", MyApp do
  pipe_through [:browser, :browser_session] # Use the default browser stack
  # ...
end

scope "/api", MyApp.Api do
  pipe_through [:api] # Use the default browser stack
end
```

From here, you can either EnsureAuthenticated in your pipeline, or on a per-controller basis.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller

  plug Guardian.Plug.EnsureAuthenticated, handler: MyApp.MyAuthErrorHandler
end
```

## Sign in and Sign out

It's up to you how you generate the claims to encode into the token Guardian uses.
As an example, here's the important parts of a SessionController

```elixir
defmodule MyApp.SessionController do
  use MyApp.Web, :controller

  alias MyApp.User
  alias MyApp.UserQuery

  plug :scrub_params, "user" when action in [:create]

  def create(conn, params = %{}) do
    conn
    |> put_flash(:info, "Logged in.")
    |> Guardian.Plug.sign_in(verified_user) # verify your logged in resource
    |> redirect(to: user_path(conn, :index))
  end

  def delete(conn, _params) do
    Guardian.Plug.sign_out(conn)
    |> put_flash(:info, "Logged out successfully.")
    |> redirect(to: "/")
  end
end
```

### Guardian.Plug.sign\_in

You can sign in with a resource (that the serializer knows about)

```elixir
Guardian.Plug.sign_in(conn, user) # Sign in with the default storage
```

```elixir
Guardian.Plug.sign_in(conn, user, :token, claims)  # give some claims to use for the token jwt

Guardian.Plug.sign_in(conn, user, :token, key: :secret)  # create a token in the :secret location
```

To attach permissions to the token, use the `:perms` key and pass it a map.
Note. To add permissions, you should configure them in your guardian config.

```elixir
Guardian.Plug.sign_in(conn, user, :token, perms: %{ default: [:read, :write], admin: [:all] })

Guardian.Plug.sign_in(conn, user, :token, key: :secret, perms: %{ default: [:read, :write], admin: [:all]})  # create a token in the :secret location
```

### Guardian.Plug.sign\_out

```elixir
Guardian.Plug.sign_out(conn) # Sign out everything (clear session)
```

```elixir
Guardian.Plug.sign_out(conn, :secret) # Clear the token and associated user from the 'secret' location
```

### Current resource, token and claims

Access to the current resource, token and claims is useful. Note, you'll need to
have run the VerifySession/Header for token and claim access, and LoadResource to access the resource.

```elixir
Guardian.Plug.claims(conn) # Access the claims in the default location
Guardian.Plug.claims(conn, :secret) # Access the claims in the secret location
```

```elixir
Guardian.Plug.current_token(conn) # access the token in the default location
Guardian.Plug.current_token(conn, :secret) # access the token in the secret location
```

For the resource

```elixir
Guardian.Plug.current_resource(conn) # Access the loaded resource in the default location
Guardian.Plug.current_resource(conn, :secret) # Access the loaded resource in the secret location
```

### Without Plug

There are many instances where Plug might not be in use. Channels, and raw
sockets for e.g. If you need to do things your own way.

```elixir
{ :ok, jwt, encoded_claims } = Guardian.encode_and_sign(resource, <token_type>, claims_map)
```

This will give you a new JWT to use with the claims ready to go.
The token type is encoded into the JWT as the 'aud' field and is intended to be
used as the _type_ of token.

```elixir
{ :ok, jwt, full_claims } = Guardian.encode_and_sign(resource, :token)
```

Add some permissions

```elixir
{ :ok, jwt, full_claims } = Guardian.encode_and_sign(resource, :token, perms: %{ default: [:read, :write], admin: Guardian.Permissions.max})
```

Currently suggested token types are:

* `"token"` - Use for API or CORS access. These are basic tokens.

You can also customize the claims you're asserting.

```elixir
claims = Guardian.Claims.app_claims
         |> Dict.put(:some_claim, some_value)
         |> Guardian.Claims.ttl({3, :days})

{ :ok, jwt, full_claims } = Guardian.encode_and_sign(resource, :token, claims)
```

To verify the token:

```elixir
case Guardian.decode_and_verify(jwt) do
  { :ok, claims } -> do_things_with_claims(claims)
  { :error, reason } -> do_things_with_an_error(reason)
end
```

Accessing the resource from a set of claims:

```elixir
case Guardian.serializer.from_token(claims.sub) do
  { :ok, resource } -> do_things_with_resource(resource)
  { :error, reason } -> do_things_without_a_resource(reason)
end
```

### Permissions

Guardian includes support for including permissions. Declare your permissions in
your configuration. All known permissions must be included.

```elixir
config :guardian, Guardian,
       permissions: %{
         default: [:read, :write],
         admin: [:dashboard, :reconcile]
       }
```

JWTs need to be kept reasonably small so that they can fit into an authorization
header. For this reason, permissions are encoded as bits (an integer) in the
token. You can have up to 64 permissions per set, and as many sets as you like.
In the example above, we have the `:default` set, and the `:admin` set.

The bit value of the permissions within a set is determined by it's position in
the config.

```elixir
# Fetch permissions from the claims map

Guardian.Permissions.from_claims(claims, :default)
Guardian.Permissions.from_claims(claims, :admin)

# Check the permissions for all present

Guardian.Permissions.from_claims(claims, :default) |> Guardian.Permissions.all?([:read, :write], :default)
Guardian.Permissions.from_claims(claims, :admin) |> Guardian.Permissions.all?([:reconcile], :admin)

# Check for any permissions
Guardian.Permissions.from_claims(claims, :default) |> Guardian.Permissions.any?([:read, :write], :default)
Guardian.Permissions.from_claims(claims, :admin) |> Guardian.Permissions.any?([:reconcile, :dashboard], :admin)
```

You can use a plug to ensure permissions are present. See Guardian.Plug.EnsurePermissions

#### Setting permissions

When you generate (or sign in) a token, you can inject permissions into it.

```elixir
Guardian.encode_and_sign(resource, :token, perms: %{ admin: [:dashboard], default: Guardian.Permissions.max}})
```

By setting a permission using Guardian.Permission.max you're setting all the bits, so even if new permissions are added, they will be set.

You can similarly pass a `:perms` key to the sign\_in method to have the
permissions encoded into the token.

### Hooks

Often you'll need to take action on some event within the lifecycle of
authentication. Recording logins etc. Guardian provides hooks to allow you to do
this. Use the Guardian.Hooks module to setup. Default implementations are
available for all callbacks.

```elixir
defmodule MyApp.GuardianHooks do
  use Guardian.Hooks

  def after_sign_in(conn, location) do
    user = Guardian.Plug.current_resource(conn, location)
    IO.puts("SIGNED INTO LOCATION WITH: #{user.email}")
    conn
  end
end
```

By default, JWTs are not tracked. This means that after 'logout' the token can
still be used if it is stored outside the system. This is because Guardian does
not track tokens and only interprates them live. When using Guardian in this
way, be sure you consider the expiry time as this is one of the few options you
have to make your tokens invalid.

If you want more control over this you should implement a hook that tracks the
tokens in some storage. When calling `Guardian.revoke!` (called automatically
with sign\_out).

To keep track of all tokens and ensure they're revoked on sign out you can use
[GuardianDb](https://github.com/hassox/guardian_db). This is a simple
Guardian.Hooks module that implements database integration.

    config :guardian, Guardian,
           hooks: GuardianDb

    config :guardian_db, GuardianDb, repo: MyRepo

Configure Guardian to know which module to use.

```elixir
config :guardian, Guardian,
       hooks: MyApp.GuardianHooks,
       #…
```

### Refreshing Tokens

You can use Guardian to refresh tokens. This keeps most of the information in
the token intact, but changes the `iat`, `exp`, `jti` and `nbf` fields.

```elixir
case Guardian.refresh!(existing_jwt, exisiting_claims, %{ttl: {15, :days}}) do
  {:ok, new_jwt, new_claims} -> do_things(new_jwt)
  {:error, reason} -> handle_error(reason)
end
```

Once the new token is created, the old one is revoked before returning the new
token.

### Phoenix Controllers

Guardian provides some helpers for you to use with your controllers.

Provides a simple helper to provide easier access to the current user and their claims.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller
  use Guardian.Phoenix.Controller

  def index(conn, params, user, claims) do
    # do stuff in here
  end
end
```

You can specify the key location of the user if you're using multiple locations to store users.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller
  use Guardian.Phoenix.Controller, key: :secret

  def index(conn, params, user, claims) do
  # do stuff with the secret user
  end
end
```

### Phoenix Channels

Guardian uses JWTs to make the integration of authentication management as
seamless as possible. Channel integration is part of that.

```elixir
defmodule MyApp.UsersChannel do
  use Phoenix.Channel
  use Guardian.Channel

  def join(_room, %{ claims: claims, resource: resource }, socket) do
    { :ok, %{ message: "Joined" }, socket }
  end

  def join(room, _, socket) do
    { :error,  :authentication_required }
  end

  def handle_in("ping", _payload, socket) do
    user = Guardian.Channel.current_resource(socket)
    broadcast socket, "pong", %{ message: "pong", from: user.email }
    { :noreply, socket }
  end
end
```

Guardian picks up on joins that have been made and automatically verifies the
token and makes available the claims and resource making the request.

```javascript
let socket = new Socket("/ws");
socket.connect();
let guardianToken = jQuery('meta[name="guardian_token"]').attr('content');
let chan = socket.chan("pings", { guardian_token: guardianToken });
```

How to get the tokens onto the page?

```eex
<%= if Guardian.Plug.current_token(@conn) do %>
  <meta name='guardian_token' content="<%= Guardian.Plug.current_token(@conn) %>">
<% end %>
```

# Acknowledgements

Many thanks to Sonny Scroggin (@scrogson) for the name Guardian and great
feedback to get up and running.

### TODO

- [x] Flexible serialization
- [x] Integration with Plug
- [x] Basic integrations like raw TCP
- [x] Sevice2Service credentials. That is, pass the authentication results through many downstream requests.
- [ ] Create a "csrf" token type that ensures that CSRF protection is included
- [x] Integration with Phoenix channels
- [x] Integrated permission sets
- [x] Hooks into the authentication cycle
- [x] Revoke tokens
- [x] Refresh tokens

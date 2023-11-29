# Auth Me Backend, Phase 3: Jbuilder Views, CSRF Protection, And Error Handling

Although your app is functional, it's not quite complete: the data being
returned is too extensive, and you still haven't accounted for CSRF attacks.

## Jbuilder views

As mentioned before, it's not ideal that in several controller actions you are
rendering all of a user's data. Now is a good time to start building Jbuilder
views for more customized JSON responses.

Within __app__, define a __views__ directory with an __api__ subdirectory. All
your Jbuilder views will live in this __app/views/api/__ folder.

Next, create a subdirectory __users__ within __app/views/api/__ and create a
view template called __show.json.jbuilder__. This template assumes that an
`@user` instance variable has been defined and structures the data that should
be rendered for that `@user` in a JSON response:

```rb
# app/views/api/users/show.json.jbuilder

json.user do
  json.extract! @user, :id, :email, :username, :created_at, :updated_at
end
```

Next, replace every `render json:` of user data in your controller actions with
a `render` of the user `show` view. (Don't forget to set an `@user` instance
variable in your `session#show` action!)

```rb
# app/controllers/api/sessions_controller.rb

def show
  if current_user
    @user = current_user
    render 'api/users/show'
  # ...
end

def create
  # ...
  if @user
    # ...
    render 'api/users/show'
  # ...
end

# app/controllers/api/users_controller.rb

def create
  # ...
  if @user.save
    # ...
    render :show
  # ...
end
```

Go ahead and test these new Jbuilder views by making `fetch` requests in your
Chrome console to hit the three actions you changed: `users#create`,
`session#show`, and `session#create`. (You may also want to make a logout
request between testing the login and sign up routes.) Make sure the user data
returned includes only the `id`, `email`, `username`, `createdAt`, and
`updatedAt` attributes.

## CSRF protection

Remember Cross Site Request Forgery (CSRF) attacks? These occur when a user with
an active session visits a malicious website, which sends a non-GET request to
your app without the user's knowledge (triggered, perhaps, by the user clicking
a harmless looking link that secretly submits a form). Since the user's
`session` cookie is automatically included in their request, the server will
authenticate the request. The server doesn't know that the request is
malicious.

You can protect against CSRF attacks by requiring that each non-GET request
include an authenticity token in a header, signifying that it is legitimate.
This token must be obtained from the server.

> Rails ensures that these tokens are browser-session-specific: your
> authenticity token will only authenticate requests coming from your browser
> session. A malicious actor therefore cannot fake authentication by attaching
> an authenticity token generated on their browser to a form submitted from your
> browser--the token would not match your browser session.

Typically, Rails provides authenticity tokens in its HTML responses
automatically via `<meta>` tags in __application.html.erb__. Since your server
will operate solely as an API--i.e., no HTML responses--you'll instead need to
put an authenticity token inside a response header manually.

Head to your `ApplicationController`. First, you want to ensure Rails rejects
any non-GET requests that lack a valid authenticity token. To do so, add the
following two lines to the top of your controller:

```rb
# app/controllers/application_controller.rb

include ActionController::RequestForgeryProtection
  
protect_from_forgery with: :exception
```

> **Note:** You have to add the `RequestForgeryProtection` module because you
> used the `--api` flag when generating your initial Rails application. If your
> `ApplicationController` inherited from `ActionController::Base` instead of
> `ActionController::API`, then this module would already be included with the
> protection turned on by default.

Then, you want to make sure you generate a valid authenticity token and store it
in a header for each response. To do so, add a new `before_action` filter,
`attach_authenticity_token`:

```rb
# app/controllers/application_controller.rb

# add this line:
before_action :attach_authenticity_token

# add this method to the `private` section of your controller:
def attach_authenticity_token
  headers['X-CSRF-Token'] = masked_authenticity_token(session)
end
```

> Note: The `form_authenticity_token` helper that you've used in your views just
> calls `masked_authenticity_token(session)`. In fact, you could just set
> `headers['X-CSRF-Token'] = form_authenticity_token`, but since this method
> isn't specific to forms, go ahead and call `masked_authenticity_token`.

Time to test that you are protecting against CSRF attacks! Make sure your server
is running and open up your Chrome console.

First, make a GET request __which won't be blocked by CSRF protection__ in order
to obtain an authenticity token from the response headers:

```js
> response = await fetch('/api/session')
  token = response.headers.get('X-CSRF-Token')
```

Next, try making two POST requests to your `api/test` route (now you see why
it's a POST route!). One should include the authenticity token in the request
headers (at the key of `X-CSRF-Token`, where Rails expects it), and one should
omit the token. Only the one without the token should be rejected for having an
invalid authenticity token.

```js
> withToken = { method: 'POST', headers: { 'X-CSRF-Token': token } }
  withoutToken = { method: 'POST' }

> await fetch('/api/test', withToken).then(res => res.json())
// should get a 200 response

> await fetch('/api/test', withoutToken)
// should get a 422 response and a "Can't verify CSRF token authenticity." error
// message in the server log
```

### CSRF protection demonstration

Let's do one more set of tests to demonstrate how this CSRF protection works.

Using the same token that you set before, try logging in, fetching the current
user, logging out, and fetching the current user again:

```js
> await fetch('/api/test?login', withToken).then(res => res.json())
// => {user: {...}}

> await fetch('/api/test', withToken).then(res => res.json())
// => {user: {...}}

> await fetch('/api/test?logout', withToken).then(res => res.json())
// => ['No current user']

> await fetch('/api/test', withToken).then(res => res.json())
// => ['No current user']
```

Note that all of these requests succeed using the same token, even though
the user is logging in and out. Logging in a different user would also still
work. The CSRF token remains the same for the **browser** session, not the
**user's** session in the app.

Next, copy the value of `token` in the DevTools console. Open a new tab in the
same browser window and go to [`localhost:5000`] again. In the DevTools console,
create a new `token` with the same value as the previous tab's `token`, define
`withToken`, and fetch the current user:

```js
> token = '<pasted csrf token>'
> withToken = { method: 'POST', headers: { 'X-CSRF-Token': token } }
  await fetch('/api/test', withToken).then(res => res.json())
```

This sequence should still work: new tabs--and even new windows--are still part
of the same browser session.

Now open a new browser window in Incognito mode and try the same sequence:

```js
> token = "<pasted csrf token>"
> withToken = { method: 'POST', headers: { 'X-CSRF-Token': token } }
  await fetch('/api/test', withToken).then(res => res.json())
```

This time the fetch should fail with status 422 because of an invalid
authenticity token. You have just confirmed that your app will not allow a
malicious user to use a valid token created in **their** browser to authenticate
an action from **your** browser.

## Wrapping up

Awesome work! You just finished setting up the entire backend for this project!
**(Don't forget to commit your code.)** In the next part, you will implement the
React frontend.

[`localhost:5000`]: http://localhost:5000

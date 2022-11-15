# Authenticating Users

## Learning Goals

- Define the term "authentication".
- Understand how websites use login to authenticate users.
- Follow REST conventions for handling session data.

***

## Key Vocab

- **Identity and Access Management (IAM)**: a subfield of software engineering that
  focuses on users, their attributes, their login information, and the resources
  that they are allowed to access.
- **Authentication**: proving one's identity to an application in order to
  access protected information; logging in.
- **Authorization**: allowing or disallowing access to resources based on a
  user's attributes.
- **Session**: the time between a user logging in and logging out of a web
  application.
- **Cookie**: data from a web application that is stored by the browser. The
  application can retrieve this data during subsequent sessions.

***

## Introduction

We've covered how cookies can be used to store data in a user's browser.

One of the most common uses of cookies is for login. In this lesson, we'll cover
how to use the Flask session to log users in.

***

## Authentication

Nearly every website in the world uses what we like to call the "wristband"
pattern. A lot of nightclubs use this pattern as well.

You arrive at the club. The bouncer checks your ID. They put a wristband on your
wrist (or stamp your hand). They let you into the club.

If you leave and come back, the bouncer doesn't look at your ID, they just look
for your wristband. If you buy a drink, the bartender doesn't need to see your
ID, since your wristband proves you're old enough to buy alcohol.

You arrive at [gmail.com](http://mail.google.com). You submit your username and
password. Google's servers check to see if your credentials are correct. If they
are, Google's servers issue a cookie to your browser. If you visit another page
on gmail — or anywhere on google.com for that matter — your browser will show
the cookie to the server. The server verifies this cookie, and lets you load
your inbox.

The term we use to describe this process is **authentication**. When we talk
about authentication in our applications, we are describing how our application
can **confirm that our users are who they say they are**.

***

## How This Looks in Flask

Let's look at what the simplest possible login system might look like in a Flask
API/React application.

The flow will look like this:

- The user navigates to a login form on the React frontend.
- The user enters their username. There is no password (for now).
- The user submits the form, POSTing to `/login` on the Flask backend.
- In the login view we set a cookie on the user's browser by writing their user
  ID into the session hash.
- Thereafter, the user is logged in. `session['user_id']` will hold their user ID.

Let's write a view to handle our login route. This class will handle `POST`
requests to `/login`:

```py
class Login(Resource):
    
    def get(self):
        ...

    def post(self):
        ...

api.add_resource(Login, '/login')

```

> **NOTE: Check out the REST module for a reminder on setting up CRUD APIs
> with Flask RESTful! We left out the imports and whatnot here.**

Typically, your `post()` method would look up a user in the database, verify
their login credentials, and then store the authenticated user's id in the
session:

```py
class Login(Resource):

    def post(self):
        user = User.query.filter(
            User.username == request.get_json()['username']
        ).first()
        
        session['user_id'] = user.id
        return jsonify(user.to_dict())

```

There's no way for the server to log you out right now. To log yourself out,
you'll have to delete the cookie from your browser.

Here's what the login component might look like on the frontend:

```jsx
function Login({ onLogin }) {
  const [username, setUsername] = useState("");

  function handleSubmit(e) {
    e.preventDefault();
    fetch("/login", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ username }),
    })
      .then((r) => r.json())
      .then((user) => onLogin(user));
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

When the user submits the form, they'll be logged in! Our `onLogin` callback function
would handle saving the logged in user's details in state.

***

## Staying Logged In

Using the wristband analogy, in the example above, we've shown our ID at the
door (`username`) and gotten our wristband (`session[:user_id]`) from the
backend. So our backend has a means of identifying us with each request using
the session hash.

Our frontend also knows who we are, because our user data was saved in state after
logging in.

What happens now if we leave the club and try to come back in, by refreshing the
page on the frontend? Well, our **frontend** doesn't know who we are any more,
since we lose our frontend state after refreshing the page. Our **backend** does
know who we are though — so we need a way of getting the user data from the
backend into state when the page first loads.

Here's how we might accomplish that. First, we need a route to retrieve the user's
data from the database using the session hash:

```rb
get "/me", to: "users#show"
```

And a controller action:

```rb
class UsersController < ApplicationController
  def show
    user = User.find_by(id: session[:user_id])
    if user
      render json: user
    else
      render json: { error: "Not authorized" }, status: :unauthorized
    end
  end
end
```

Then, we can try to log the user in from the frontend as soon as the application
loads:

```jsx
function App() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch("/me").then((response) => {
      if (response.ok) {
        response.json().then((user) => setUser(user));
      }
    });
  }, []);

  if (user) {
    return <h2>Welcome, {user.username}!</h2>;
  } else {
    return <Login onLogin={setUser} />;
  }
}
```

This is the equivalent of letting someone use their wristband to come back into
the club.

***

## Logging Out

The log out flow is even simpler. We can add a new route for logging out:

```rb
delete "/logout", to: "sessions#destroy"
```

Then add a `SessionsController#destroy` method, which will clear the username
out of the session:

```rb
def destroy
  session.delete :user_id
  head :no_content
end
```

Here's how that might look in the frontend:

```jsx
function Navbar({ onLogout }) {
  function handleLogout() {
    fetch("/logout", {
      method: "DELETE",
    }).then(() => onLogout());
  }

  return (
    <header>
      <button onClick={handleLogout}>Logout</button>
    </header>
  );
}
```

The `onLogout` callback function would handle removing the information about the
user from state.

***

## Security Concerns

Cookies are stored as plain text in a user's browser. Therefore, the user can
see what's in them, and they can set them to anything they want.

If you open the developer console in your browser, you can see the cookies set
by the current site. In Chrome's console, you can find this under
`Application > Cookies`. You can delete any cookie you like. For example, if you
delete your `user_session` cookie on `github.com` and refresh the page, you will
find that you've been logged out.

You can also edit cookies, for example with [this extension][edit_this_cookie].

This presents a problem for us. If users can edit their `pageviews_remaining`
cookie, then they can easily give themselves an unlimited amount of page views.

***

## Flask Never Fails!

Fortunately, Flask has a solution to this. Instead of sending our cookies in
plain text, we can use Flask to **encrypt** and **sign** a special cookie known
as a session using the `session` module. The `session` module is imported
from `flask`, and it behaves like a dictionary:

```py
  session['pageviews_remaining'] = 5
```

You can store any simple Python object in the session.

By default, Flask manages all session data in a single cookie. It _serializes_
all the key/value pairs you set with `session`, converting them from a Python
object into a big string. Whenever you set a key with the `session` module,
Python updates the value of its session cookie to this big string.

When you set cookies this way, Flask **signs** them to prevent users from
tampering with them. Your Flask server has a key, configured in
your app:

```py
app.secret_key = 'BAD_SECRET_KEY'
```

The best strategy for generating a good secret key is using the `os` module
from the command line:

```shell
$ python -c 'import os; print(os.urandom(16))'
# => b'_5#y2L"F4Q8z\n\xec]/'

# keep this somewhere safe!
```

It's guaranteed that given the same message and key, Flask will produce the
same output. Also, without the key, it is practically impossible to know what
Flask would return for a given message. That is, signatures can't be forged.

Flask creates a signature for every session cookie it sets, and appends the
signature to the cookie.

When it receives a cookie, Flask verifies that the signature matches the
content.

This prevents cookie tampering. If a user tries to edit their cookie and change
the `pageviews_remaining`, the signature won't match, and Flask will silently
ignore the cookie and set a new one.

Cryptography is a deep rabbit hole. At this point, you don't need to understand
the specifics of how cryptography works, just that Flask and other frameworks
use it to ensure that session data which is set on the server can't be edited by
users.

***

## Conclusion

Cookies are foundational for the modern web.

Most sites use cookies, to let their users log in, keep track of their shopping
carts, or record other ephemeral session data. Almost nobody thinks these are
bad uses of cookies: nobody really believes that you should have to type in your
username and password on every page, or that your shopping cart should clear if
you reload the page.

But cookies let you store data in a user's browser, so by nature, they can be
used for more controversial endeavors.

For example, Google AdWords sets a cookie and uses that cookie to track what ads
you've seen and which ones you've clicked on. The tracking information helps
AdWords decide what ads to show you.

This is why, if you click on an ad, you may find that the ad follows you around
the internet. It turns out that this behavior is as effective as it is annoying:
people are far more likely to buy things from ads that they've clicked on once.

This use of cookies worries people and the EU
[has created legislation around the use of cookies][eu_law].

Cookies, like any technology, are a tool. In the rest of this module, we're going
to be using them to let users log in. Whether you later want to use them in such
a way that the EU passes another law is up to you.

***

## Check For Understanding

Before you move on, make sure you can answer the following questions:

<details>
  <summary>
    <em>1. What do we mean when we say that HTTP is stateless?</em>
  </summary>

  <p>Every HTTP request the server receives is independent. New requests do not
     remember the requests that came before.</p>
</details>
<br/>

<details>
  <summary>
    <em>2. What attribute do we need to set to ensure that data is encrypted in
        Flask sessions?</em>
  </summary>

  <p><code>app.secret_key</code></p>
</details>
<br/>

***

## Resources

- [Introduction to Identity and Access Management (IAM) - auth0](https://auth0.com/docs/get-started/identity-fundamentals/identity-and-access-management)
- [HTTP RFC Section 9 — Methods][http methods rfc]
- [RFC 6265 — HTTP State Management Mechanism (the cookie spec)][rfc_cookies]
- [Session data in Python Flask - pythonbasics.org][flask session]
- [General Data Protection Regulation][eu_law]
- [Legal Cookie Requirements][cookie policies]

[http methods rfc]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html "HTTP RFC 9 — Method Definitions"
[rfc_cookies]: http://tools.ietf.org/html/rfc6265 "HTTP State Management Mechanism"
[edit_this_cookie]: https://chrome.google.com/webstore/detail/editthiscookie/fngmhnnpilhplaeedifhccceomclgfbg?hl=en
[flask session]: https://pythonbasics.org/flask-sessions/
[eu_law]: https://en.wikipedia.org/wiki/General_Data_Protection_Regulation
[cookie policies]: https://termly.io/resources/templates/cookie-policy-template/#are-you-legally-required-to-have-a-cookie-policy

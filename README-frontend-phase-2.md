# Auth Me Frontend, Phase 2: Signup Form Page

The signup form page is the second page that you will add to your frontend
application. The flow will be very similar to what you did for the login form
page. Can you remember all the steps to implement it? If so, **try doing this on
your own before looking below for help!**

## Signup action

You will use the `POST /api/users` backend route to sign up a user.

In the session store file, add a `signup` thunk action that will hit the signup
backend route with `username`, `email`, and `password` inputs. After the
response from the AJAX call comes back, parse the JSON body of the response and
dispatch the action for setting the session user to the user in the response's
body.

Export the `signup` thunk action.

### Test the `signup` action

Test the `signup` thunk action.

Navigate to [http://localhost:5173]. In the browser's DevTools console, try
dispatching the `signup` thunk action with a new `username`, a new `email`, and
a `password`.

The `previous state` in the console should look like this:

```js
{
  session: {
    user: null // or previously logged-in user
  }
}
```

The `next state` in the console should look something like this:

```js
{
  session: {
    user: {
      createdAt: "<Some date time format>",
      email: "<new email>",
      id: "<new id>",
      updatedAt: "<Some date time format>",
      username: "<new password>"
    }
  }
}
```

If there is an error or if the previous or next state does not look like this,
check the logic in your signup action.

**Commit your code for the signup actions!**

### Example `signup` action

Again, there is no absolute "right" way of doing this. As long as your `signup`
action is displaying the expected initial state and the expected states after
each dispatched action, then your setup is fine.

Here's an example of the `signup` thunk action:

```js
// frontend/src/store/session.js

// ...
export const signup = (user) => async (dispatch) => {
  const { username, email, password } = user;
  const response = await csrfFetch("/api/users", {
    method: "POST",
    body: JSON.stringify({
      username,
      email,
      password
    })
  });
  const data = await response.json();
  dispatch(setUser(data.user));
  return response;
};
```

Here's an example of a `signup` thunk action test in the browser's DevTools
console:

```js
> store.dispatch(sessionActions.signup({
    username: 'NewUser',
    email: 'new@user.io',
    password: 'password'
  }))
```

## `SignupForm` component

After finishing the Redux action for the signup feature, the React components
are next.

Create a file in the __components__ directory named `SignupForm.jsx`.

Render a form with controlled inputs for the new user's username, email, and
password, and confirm password fields.

On submit of the form, validate that the confirm password is the same as the
password field, then dispatch the `signup` thunk action with the form input
values. Make sure to handle and display any errors from the `signup` thunk
action. If the confirm password is not the same as the password, display an
error message.

Export the `SignupForm` component as the default at the bottom of the file, then
add it in the router in __App.jsx__ with a `/signup` route.

If a user tries to access the `SignupForm` while there is a current session user
in the Redux store, navigate the user to the `/` path.

Test your component by navigating to the `/signup` page. (Log out via the
console if necessary.) Try submitting a new user's information on the form. Once
you sign up, you should be redirected to the `/` route. Check your code for the
`SignupForm` and the `App` component if this is not the flow that you are
experiencing.

Also try signing up with invalid fields to test your handling and displaying of
error messages.

**After testing, commit your `SignupForm` code!**

### Example `SignupForm` component

Again, there is no absolute "right" way of doing this. As long as your React
application is behaving as expected, then you don't need to make your code look
exactly like the example code.

Here's an example `SignupForm` component:

```js
// frontend/src/components/session/SignupForm.jsx

import { useState } from "react";
import { useDispatch, useSelector } from "react-redux";
import { Navigate } from "react-router-dom";
import * as sessionActions from "../../store/session";

function SignupForm() {
  const dispatch = useDispatch();
  const sessionUser = useSelector(state => state.session.user);
  const [email, setEmail] = useState("");
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [confirmPassword, setConfirmPassword] = useState("");
  const [errors, setErrors] = useState([]);

  if (sessionUser) return <Navigate to="/" replace={true}/>;

  const handleSubmit = (e) => {
    e.preventDefault();
    if (password === confirmPassword) {
      setErrors([]);
      return dispatch(sessionActions.signup({ email, username, password }))
        .catch(async (res) => {
        let data;
        try {
          // .clone() essentially allows you to read the response body twice
          data = await res.clone().json();
        } catch {
          data = await res.text(); // Will hit this case if the server is down
        }
        if (data?.errors) setErrors(data.errors);
        else if (data) setErrors([data]);
        else setErrors([res.statusText]);
      });
    }
    return setErrors(['Confirm Password field must be the same as the Password field']);
  };

  return (
    <>
      <h1>Sign Up</h1>
      <form onSubmit={handleSubmit}>
        <ul>
          {errors.map(error => <li key={error}>{error}</li>)}
        </ul>
        <label>
          Email
          <input
            type="text"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </label>
        <label>
          Username
          <input
            type="text"
            value={username}
            onChange={(e) => setUsername(e.target.value)}
            required
          />
        </label>
        <label>
          Password
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </label>
        <label>
          Confirm Password
          <input
            type="password"
            value={confirmPassword}
            onChange={(e) => setConfirmPassword(e.target.value)}
            required
          />
        </label>
        <button type="submit">Sign Up</button>
      </form>
    </>
  );
}

export default SignupForm;
```

### `SignupForm` CSS

Add a __SignupForm.css__ file in your __session__ folder. Import this CSS file
in __frontend/src/components/session/SignupForm.jsx__.

```js
// frontend/src/components/session/SignupForm.jsx

// ...
import './SignupForm.css';
// ...
```

Define all your CSS styling rules for the `SignupForm` component in the
__SignupForm.css__ file. Practice doing some CSS now to make your signup page
look better. Make sure to **commit your code afterwards!**

[http://localhost:5173]: http://localhost:5173

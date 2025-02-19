# Account creation and login

The first step towards supporting authentication in your web application is providing a way for users to uniquely identify themselves. This usually requires three service endpoints. One to initially `register`, a second to `login`, on future visits, and a third to `logout`. Once a user is authenticated we can control access to other endpoints. For example, web services often have a `getMe` endpoint that gives information about the currently authenticated user. We will implement these endpoints to demonstrate that authentication is actually working correctly.

## Endpoint design

The following uses HTTP to define what each of our authentication endpoints look like.

### Registration endpoint

This takes an email and password and returns a cookie containing the authentication token and user ID. If the email already exists it returns a 409 (conflict) status code.

```http
POST /api/auth HTTP/2
Content-Type: application/json

{
  "email":"marta@id.com",
  "password":"toomanysecrets"
}
```

```http
HTTP/2 200 OK
Content-Type: application/json
Set-Cookie: auth=tokenHere

{
  "email":"marta@id.com"
}
```

### Login authentication endpoint

This takes an email and password and returns a cookie containing the authentication token and user ID. If the email does not exist or the password is bad it returns a 401 (unauthorized) status code.

```http
POST /api/auth HTTP/2
Content-Type: application/json

{
  "email":"marta@id.com",
  "password":"toomanysecrets"
}
```

```http
HTTP/2 200 OK
Content-Type: application/json
Set-Cookie: auth=tokenHere

{
  "email":"marta@id.com"
}

```

### Logout authentication endpoint

This takes an email and password and returns a cookie containing the authentication token and user ID. If the email does not exist or the password is bad it returns a 401 (unauthorized) status code.

```http
DELETE /api/auth HTTP/2
Cookie: auth=tokenHere
```

```http
HTTP/2 200 OK
Content-Type: application/json

{
}
```

### GetMe endpoint

This uses the authentication token stored in the cookie to look up and return information about the authenticated user. If the token or user do not exist it returns a `401 (unauthorized)` status code.

```http
GET /api/user HTTP/2
Cookie: auth=tokenHere
```

```http
HTTP/2 200 OK
Content-Type: application/json

{
  "email":"marta@id.com"
}

```

## Web service

With our service endpoints defined, we can now build our web service using Express that simply has stubbed out functionality for each of our authentication endpoints.

**service.js**

```js
const express = require('express');
const app = express();

app.post('/api/auth', async (req, res) => {
  res.send({ email: 'marta@id.com' });
});

app.put('/api/auth', async (req, res) => {
  res.send({ email: 'marta@id.com' });
});

app.delete('/api/auth', async (req, res) => {
  res.send({});
});

app.get('/api/user', async (req, res) => {
  res.send({ email: 'marta@id.com' });
});

const port = 3000;
app.listen(port, function () {
  console.log(`Listening on port ${port}`);
});
```

Using the above code, use the following steps to create and start the authentication service.

1. Create a directory called `authTest` that we will work in.
1. Save the above content to a file named `service.js`. This is our starting web service.
1. Run `npm install express cookie-parser uuid bcryptjs` to install all of the packages we are going to use.
1. Run `node --watch service.js` or press `F5` in VS Code to start up the web service.
1. You can now open a console window and use `curl` to try out one of the endpoints.

   ```sh
   curl -X POST localhost:3000/api/auth -d '{"email":"test@id.com", "password":"a"}'
   ```

   ```sh
   {"email":"marta@id.com"}
   ```

## Handling requests

With our basic service created, we can now fill in the registration endpoint. The first step is to read the credentials from the body of the HTTP request. Since the body is designed to contain JSON we need to tell Express that it should parse HTTP requests, with a content type of `application/json`, automatically into a JavaScript object. We do this by using the `express.json` middleware. We can then read the email and password directly out of the `req.body` object. We can test that we can read the request processing is working by just turning around and sending them back in the response.

```js
app.use(express.json());

app.post('/api/auth', (req, res) => {
  res.send({
    email: req.body.email,
    password: req.body.password,
  });
});
```

```sh
curl -X POST localhost:3000/api/auth -H "Content-Type: application/json" -d '{"email":"test@id.com", "password":"a"}'
```

```sh
{"email":"test@id.com","password":"a"}
```

Now that we have proven that we can parse the request bodies correctly, we want to create a function that will actually create the user and store it in memory. We also want to properly management passwords so that we never remember the user's original password.

### Storing users and securing passwords

In order to properly decompose our code, we want to create a functions that creates a user and stores it in a memory data structure, as well as a function that will return a user based upon an email address or authorization token. In later instruction, we will replace our in memory storage by storing our users in a database.

```js
const users = [];

async function createUser(email, password) {
  // creation code
}

function getUser(field, value) {
  // retrieval code
}
```

Along with storing the user we, need to securely store our passwords. Failing to do so is a major security concern. If, and it has happened to many major companies, a hacker is able to access the database, they will have the passwords for all of your users. This may not seem like a big deal if your application is not very valuable, but users often reuse passwords. That means you will also provide the hacker with the means to attack the user on many other websites.

So instead of storing the password as clear text, we want to cryptographically hash the password so that we never store the actual password. When we want to validate a password during login, we can hash the login password and compare it to our stored hash of the password.

To hash our passwords we will use the `js` package. This creates a very secure one-way hash of the password. If you are curious about how [bcrypt](https://en.wikipedia.org/wiki/Bcrypt) works, it is definitely worth the time. Here is our resulting code.

### Working create and get user functions

```js
const bcrypt = require('bcryptjs');

const users = [];

async function createUser(email, password) {
  const passwordHash = await bcrypt.hash(password, 10);

  const user = {
    email: email,
    password: passwordHash,
  };

  users.push(user);

  return user;
}

function getUser(field, value) {
  return users.find((user) => user[field] === value);
}
```

### Working registration endpoint

Now we can completely implement the registration endpoint by first checking to see if we already have a user with that email address. If we do, then we immediately return a 409 (conflict) status code. Otherwise we create a new user and only return the user's email.

```js
app.post('/api/auth', async (req, res) => {
  if (await getUser('email', req.body.email)) {
    res.status(409).send({ msg: 'Existing user' });
  } else {
    const user = await createUser(req.body.email, req.body.password);
    res.send({ email: user.email });
  }
});
```

### Generating authentication tokens

We need to produce a random token that represents that the user has been authenticated and can make requests to protected endpoints. To generate a reasonable authentication token we use the `uuid` package. [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) stands for Universally Unique Identifier, and it does a really good job creating a hard to guess, random, unique ID.

```js
const uuid = require('uuid');

token: uuid.v4();
```

## Using cookies for authorization tokens

We now need to pass our generated authentication token to the browser when the login endpoint is called, and get it back on subsequent requests. To do this we use HTTP cookies. The `cookie-parser` package provides middleware for cookies and so we will leverage that.

We import the `cookieParser` object and then tell our app to use it. When a user is successfully created, or logs in, we set the cookie header. Since we are storing an authentication token in the cookie, we want to make it as secure as possible, and so we use the `httpOnly`, `secure`, and `sameSite` options.

- `httpOnly` tells the browser to not allow JavaScript running on the browser to read the cookie.
- `secure` requires HTTPS to be used when sending the cookie back to the server.
- `sameSite` will only return the cookie to the domain that generated it.

```js
const cookieParser = require('cookie-parser');
const uuid = require('uuid');

app.use(cookieParser());

app.post('/api/auth', async (req, res) => {
  if (await getUser('email', req.body.email)) {
    res.status(409).send({ msg: 'Existing user' });
  } else {
    const user = await createUser(req.body.email, req.body.password);
    setAuthCookie(res, user);

    res.send({ email: user.email });
  }
});

function setAuthCookie(res, user) {
  user.token = uuid.v4();

  res.cookie('token', user.token, {
    secure: true,
    httpOnly: true,
    sameSite: 'strict',
  });
}
```

## Login endpoint

The login authorization endpoint needs to get the hashed password from the database, compare it to the provided password using `bcrypt.compare`, and if successful set the authentication token in the cookie. If the password does not match, or there is no user with the given email, the endpoint returns status 401 (unauthorized).

```js
app.put('/api/auth', async (req, res) => {
  const user = await getUser('email', req.body.email);
  if (user && (await bcrypt.compare(req.body.password, user.password))) {
    setAuthCookie(res, user);

    res.send({ email: user.email });
  } else {
    res.status(401).send({ msg: 'Unauthorized' });
  }
});
```

## Logout endpoint

The logout authorization endpoint needs to remove the authorization token from the user get the hashed password from the database, compare it to the provided password using `bcrypt.compare`, and if successful set the authentication token in the cookie. If the password does not match, or there is no user with the given email, the endpoint returns status 401 (unauthorized).

```js
app.delete('/api/auth', async (req, res) => {
  const token = req.cookies['token'];
  const user = await getUser('token', token);
  if (user) {
    clearAuthCookie(res, user);
  }

  res.send({});
});

function clearAuthCookie(res, user) {
  delete user.token;
  res.clearCookie('token');
}
```

## GetMe endpoint

With everything in place to create credentials and login using the credentials, we can now implement the `getMe` endpoint to demonstrate that it all actually works. To implement this we get the user object from the database by querying on the authentication token. If there is not an authentication token, or there is no user with that token, we return status 401 (unauthorized).

```js
app.get('/api/user/me', async (req, res) => {
  const token = req.cookies['token'];
  const user = await getUser('token', token);
  if (user) {
    res.send({ email: user.email });
  } else {
    res.status(401).send({ msg: 'Unauthorized' });
  }
});
```

## The final login backend code

Here is the full example code.

```js
const express = require('express');
const app = express();
const cookieParser = require('cookie-parser');
const uuid = require('uuid');
const bcrypt = require('bcryptjs');

app.use(express.json());
app.use(cookieParser());

app.post('/api/auth', async (req, res) => {
  if (await getUser('email', req.body.email)) {
    res.status(409).send({ msg: 'Existing user' });
  } else {
    const user = await createUser(req.body.email, req.body.password);
    setAuthCookie(res, user);

    res.send({ email: user.email });
  }
});

app.put('/api/auth', async (req, res) => {
  const user = await getUser('email', req.body.email);
  if (user && (await bcrypt.compare(req.body.password, user.password))) {
    setAuthCookie(res, user);

    res.send({ email: user.email });
  } else {
    res.status(401).send({ msg: 'Unauthorized' });
  }
});

app.delete('/api/auth', async (req, res) => {
  const token = req.cookies['token'];
  const user = await getUser('token', token);
  if (user) {
    clearAuthCookie(res, user);
  }

  res.send({});
});

app.get('/api/user/me', async (req, res) => {
  const token = req.cookies['token'];
  const user = await getUser('token', token);
  if (user) {
    res.send({ email: user.email });
  } else {
    res.status(401).send({ msg: 'Unauthorized' });
  }
});

const users = [];

async function createUser(email, password) {
  const passwordHash = await bcrypt.hash(password, 10);

  const user = {
    email: email,
    password: passwordHash,
  };

  users.push(user);

  return user;
}

async function getUser(field, value) {
  return users.find((user) => user[field] === value);
}

function setAuthCookie(res, user) {
  user.token = uuid.v4();

  res.cookie('token', user.token, {
    secure: true,
    httpOnly: true,
    sameSite: 'strict',
  });
}

function clearAuthCookie(res, user) {
  delete user.token;
  res.clearCookie('token');
}

const port = 3000;
app.listen(port, function () {
  console.log(`Listening on port ${port}`);
});
```

## Experiment

With everything implemented, we can use `curl` to try it out. First start up the web service from VS Code by pressing `F5` and selecting `node.js` as the debugger if you have not already done that. You can set breakpoints on all of the different endpoints to see what they do and inspect the different variables. Then open a console window and run the following `curl` commands. You should see results similar to what is shown below. Note that the `-c` and `-b` parameters tell curl to store and use cookies with the given file.

```sh
curl -X POST localhost:3000/api/auth -H 'Content-Type:application/json' -d '{"email":"지안@id.com", "password":"toomanysecrets"}'
```

```sh
{"email":"지안@id.com"}
```

```sh
curl -c cookie.txt -X PUT localhost:3000/api/auth -H 'Content-Type:application/json' -d '{"email":"지안@id.com", "password":"toomanysecrets"}'
```

```sh
{"email":"지안@id.com"}
```

```sh
curl -b cookie.txt localhost:3000/api/user/me
```

```sh
{"email":"지안@id.com"}
```

## Login frontend code

With the backend service in place, we can create a simple React application that demonstrates the use of the authentication endpoints. First we need to follow the basic React setup that we discussed in the simple [Hello World React](../../webFrameworks/react/introduction/introduction.md#react-hello-world) app that we created in previous instruction. This includes:

1. Creating an NPM project, installing Vite, and installing React.
   ```sh
   npm init -y
   npm install vite@latest -D
   npm install react react-dom react-router-dom
   ```
1. Configuring Vite to proxy API requests through to the backend when debugging.
1. Creating a basic `index.html` file that loads your React application.
1. Creating your React application in `index.jsx`.

### The authentication components

In the index.jsx file we will set up some simple routing between a **login** component and a user **profile** component.

```jsx
function App() {
  return (
    <BrowserRouter>
      <main>
        <Routes>
          <Route path='/' element={<Login />} exact />
          <Route path='/profile' element={<Profile />} />
        </Routes>
      </main>
    </BrowserRouter>
  );
}
```

The login component will handle the authentication calls to the backend.

```jsx
function Login() {
  const navigate = useNavigate();
  const [email, setEmail] = React.useState('');
  const [password, setPassword] = React.useState('');

  function handleLogin() {
    createAuth('PUT');
  }

  function handleRegister() {
    createAuth('POST');
  }

  async function createAuth(method) {
    const res = await fetch('api/auth', {
      method: method,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });
    await res.json();
    if (res.ok) {
      navigate('/profile');
    } else {
      alert('Authentication failed');
    }
  }

  return (
    <div>
      <h1>Login</h1>
      <div>
        <label>Email:</label>
        <input type='text' onChange={(e) => setEmail(e.target.value)} required />
      </div>
      <div>
        <label>Password:</label>
        <input type='password' onChange={(e) => setPassword(e.target.value)} required />
      </div>
      <button type='submit' disabled={!(email && password)} onClick={handleLogin}>
        Login
      </button>
      <button type='button' disabled={!(email && password)} onClick={handleRegister}>
        Register
      </button>
    </div>
  );
}
```

And the profile component will display the current user and provide the ability to logout.

```jsx
function Profile() {
  const navigate = useNavigate();
  const [userInfo, setUserInfo] = React.useState('');

  React.useEffect(() => {
    (async () => {
      const res = await fetch('api/user/me');
      const data = await res.json();
      setUserInfo(data);
    })();
  }, []);

  function handleLogout() {
    fetch('api/auth', {
      method: 'DELETE',
    });
    navigate('/');
  }

  return (
    <div>
      <h1>Profile</h1>
      <div>Logged in as: {userInfo.email}</div>
      <button type='button' onClick={handleLogout}>
        Logout
      </button>
    </div>
  );
}
```

### The finished product

With the backend running on port 3000, and the frontend running with the Vite hot reloading HTTP server we will be able to debug the login application in both the browser, for the frontend, and with VS Code, for the backend.

![Login demo](loginDemo.gif)

## Experiment

You can find this complete example [here](exampleCode). To run it yourself, take the following steps:

1. Clone the repository.
1. Run `npm install` from a console window in the example root directory.
1. Run `npm install` from a console window in the example service subdirectory.
1. Open up the code in VS Code and review what it is doing.
1. Run and debug the example by pressing `F5` for the file `service/index.js`. You may need to select node.js as the debugger the first time you run.
1. Run `npm run dev` from a console window in the example root directory.
1. Open multiple browser windows and point them to http://localhost:5137.
1. Register, login, logout, and access the protected endpoint to get your user information.
1. Use the browser's debugger to view the authentication communication.

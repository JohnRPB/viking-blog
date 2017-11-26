---
layout: post
title:  "Web Sockets: concepts and implementation with Express"
date:   2017-11-25 13:57
categories: jekyll update
---

Most of the time, website features are not synchronized between users; different users accessing the site at different times will see different things. So if you type in your login information, for example, your buddy doesn't suddenly see it, ghost-like, appearing on his screen. For many simple use cases, however, sychronization is necessary---such as for forums or visitor counter buttons. All users on the same page should see (some of) the same things, and those things should be updating in "real time". How does this happen? 

## The problem with HTTP 

If you know a little something about HTTP, you probably recognize that for a user to see something on a page, their client (e.g. browser) has to make a request to that page's server; this is called "polling the server". If there are dozens or hundreds of clients all polling one server, such as in a chatroom---and simulatneously making POST requests to it---it is easy to see how keeping the user's experiences synchronized can be complicated. 

Because HTTP is a "client-driven" protocol, there are really only two ways of doing this: short polling and long polling. As their names imply, the former relies on constantly sending GET requests to the server, so that the client is always updated, while the second opens a connection between the client and the server and makes a request, but the *server* doesn't respond until something changes or the connection time is up (typically 20 to 30 seconds, as TCP connections are short-lived). This drastically reduces the number of requests and responses that the client and server have to make, respectively. 

There is still a better method, however, and that is to enable two-way communication between client and server. This is what Websockets does. The client and server can push information to each other over an open connection. Below, we will see how this works in node.js.

## Implementing a websocket with sockets.io



Sockets.io is a module that is available from the node package manager, so you have to require it. Below, we have a typical setup using Express. You might find this code in a high-level file such as 'bin/www' or 'app.js'. 

```javascript
// app.js

const express = require('express');
const app = express();
const server = require('http').createServer(app);
const io = require('socket.io')(server);

app.use('/socket.io',
  express.static(__dirname + 'node_modules/socket.io-client/dist/')
)
````

What this code does, essentially, is connect components. The `http` module is the core; by running the `.createServer()` method on it, we create a basic `server` object that can listen to endpoints. It reacts to every request with a single callback, supplied upon creation---which in this case, happens to be our Express app. 

Every request will be handled by Express-mediated functionality, some of which we code, some of which comes bundled with Express. The sockets module recieves the server object so that any code we attach to the "io" object has access to our server (and can, for instance, emit events to it). 

Finally, if the client makes a request to the `/socket.io` route (a "virtual" route, defined in this code, with no parallel in the file system), it is re-routed using express middleware to the file in the file system that contains the front-end implementation of sockets.io (the file is simply attached to the virtual route, which can be conferred by adding `/socket.io/socket.io.js` to the homepage URL and seeing what comes up). In our HTML file header, we load some scripts, one of which requests this route, 

```html
<html>
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <script src="/socket.io/socket.io.js"></script>
    ...
  </head>
</html>
````
That's it for the basic setup. Now we need to actually implement a web sockets connection. This is done server-side by attaching a "connection" listener to the 'io' object we created above. The "connection" event is initiated by the client-side code, which recieves the home address of our site and wraps it to create a client object. This code then initiates a connection event on 'io', and transmits with it the client object we just mentioned, 

````javascript
// script.js

let socket = io.connect('http://localhost:3000');
````

Server-side, the connection listener picks up this event, and recieves the client object in its callback, allowing us to set listeners on it, or emit events to it. A connection is born. 

```javascript
// app.js

io.on('connection', client => {
  // do something
})
````
Now, we can enable client-server communication. By attaching listeners and emitters to the 'socket' object in the front-end code, we can define which events we want to client to listen for, and which events we want it to emit to the server. Vice versa for the 'client' object in the back-end code,

````javascript
// script.js

let socket = io.connect('http://localhost:3000');

socket.on("event_from_server", () => {
  // doing something
});
socket.emit("event_from_client");

// app.js

io.on('connection', client => {
  client.on("event_from_client", () => {
    // doing something else
  }
  client.emit("event_from_server");
})
````
This is almost everything, but the perceptive reader will have noticed a discrepancy (this is just something writers like to say, don't worry about it). We began this post discussing "real time" updates for all clients connecting to a server, yet while our code certainly enables two-way communication between **one** client and one server, it certainly doesn't enable it for all clients at once. 

Suppose we were building a forum, and our client initated a "submit_post" event; the server would react to this event, update its stored posts, and then... what? No other clients connected to the server would be notified of the change. We need a way to send events to **all** clients, and we can do that, server-side, using 'io' directly:

````javascript
// app.js

io.on('connection', client => {
  client.on("post_from_client", () => {
    // update posts
    io.emit("post_update");
  }
})
````

That's it. We've established a web sockets connection, equipped it with events to trigger server-side and client-side code, and enabled our server to trigger client-side code on all clients. We should be able to do anything now. 

Anything. 

Let your amibitions run wild, young padawan.

### Extra goodies

Since `io.on("connection", ...)` is just a listener, which doesn't *create* a connection, you can put it anywhere in your code, as many times as you need---provided the `io` object has already been stored in memory. You should only *initiate* a new connection, however, once; that is, `io.connect(...)` should only be called a single time, in your client-side code.


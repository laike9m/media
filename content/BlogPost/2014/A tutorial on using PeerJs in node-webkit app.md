In this article, I'm going to talk about the combination usage of node-webkit and PeerJs, but first let's take a look at things we're going to talk about. [Node-webkit][1] is an app runtime that allows developers to use Web technologies (i.e. HTML5, CSS and JavaScript) to develop native apps. [PeerJS][2] wraps the browser's WebRTC implementation to provide a complete, configurable, and easy-to-use peer-to-peer connection API. If you haven't heard about them, I suggest you go to their websites and take a quick look at what they do, cause they both are really insteresting projects.

Why do I write this article?

There has been some [discussions][3] on running peerjs client in a node.js application, but apparently it's not easy to do this because PeerJs relies on WebRTC which is built into Webkit. Then, as node-webkit gets more and more popular, people start to think, why not use PeerJs in node-webkit so that we can build p2p apps? [Here][4] is an attempt, as you see it really works, basically it's the same as running PeerJs in browser.

Yet, simply using node-webkit as a browser and running PeerJs in it waste the most powerful feature node-webkit provides: the node.js runtime. Browser is cool, HTML5 give us the ability to cope with files stored in local computer, but those features are minor compared to what nodejs can do. If a node-webkit app don't make use of nodejs, why bother using node-webkit instead of writing a pure web app?

The node-webkit project I'm working on needs nodejs(db strorage, watching files, etc...) as well as PeerJs. First I tried something like this:

```html
<!DOCTYPE html>
<html>
<head lang="en">
  <script src="peer.js"></script>
</head>
<body>
<script>
  var peer = new Peer('username', {key: 'my-key'});
  var conn = peer.connect('hehe');
  var fs = require('fs');

  // sender side code
  conn.on('open', function(){
    console.log("connect to peer");
    var data = fs.readFileSync('file-you-want-to-transfer');
    conn.send(data);
    console.log('data sent');
  });

  // receiver side code
  peer.on('connection', function(conn) {
    conn.on('data', function(data){
      fs.writeFileSync('received_file', data);
      console.log("received complete: ", Date());
    });
  });
</script>
</body>
</html>
```   

It doesn't work because there's no node.js runtime in html, So you can't read from/write to local using `fs.write/readFileSync`. You probabaly know that node-webkit's node runtime can't really interact with DOM environmen——it lets you require a nodejs's script and call its function from DOM, but you CAN'T GET THE RETURN VALUE, the functon you invoked is run in node runtime and knows nothing about DOM, that's why the above code couldn't work.

Then my friend suggested me to run an express server on localhost and use socket.io to make node and DOM interact with each other. I've written a demo and put it on github to show how this works:

**[https://github.com/laike9m/peerjs-with-nodewebkit-tutorial](https://github.com/laike9m/peerjs-with-nodewebkit-tutorial)**

This demo could be run either on a single machine or two machines. What it does is transfering `.gitignore` file to the other end. Here's its GUI:

![](http://laike9m.com/media/content/BlogPost/images/peer-nw-1.jpg)

To run this app, `npm install` first, then launch it following [node-webkit's documentation][6]. Assume you only have one computer, click the `receive` button, then click `send` button, you'll see a new file called `received_gitignore` appear in app's directory. Be sure to click `receive` before `send`, whether running on single machine or two machines.

Finally, let's get down to business to explain how this demo works.

First is `package.json`:

```javascript
{
    "main": "main.html",
}
```

So `main.html` is the first HTML page node-webkit should display.

```html
<html>
<script>
  var main = require('./main.js');
  window.location.href = "http://127.0.0.1:12345/";
</script>
</html>
```

Here's the interestring part: our `main.html` doesn't contain anything to display, it's only purpose is calling `require('./main')` which will launch an express server listening on `127.0.0.0:12345`, then connects to it.

Let's see how it's done in `main.js`:

```javascript
// main.js part 1
var app = require('express')();
var server = require('http').Server(app);
var io = require('socket.io')(server);

server.listen(12345);
app.use(require('express').static(__dirname + '/static'));
app.get('/', function (req, res) {
  res.sendfile(__dirname + '/index.html');
});
```

Nothing special, just a regular express http server with socket.io.  
As we can see, visiting `http://127.0.0.1:12345/` gets `index.html` displayed. Here's `index.html`:

```html
<!DOCTYPE html>
<html>
<head lang="en">
  <meta charset="UTF-8">
  <meta name="author" content="laike9m">
  <title>demo</title>
  <script type="text/javascript" src="peer.js"></script>
  <script type="text/javascript" src="/socket.io/socket.io.js"></script>
</head>
<body>
  <script>
    window.socket = io.connect('http://localhost/', { port: 12345 });
    function clickSend() {
      var peer = new Peer('sender', {key: '45rvl4l8vjn3766r'});
      var conn = peer.connect('receiver');
      conn.on('open', function () {
        console.log("sender dataconn open");
        window.socket.on('sendToPeer', function(data) {
          console.log("sent data: ", Date());
          conn.send(data);
          peer.disconnect();
        });
        window.socket.emit('send');
      });
    }
    function clickRecv(){
      var peer = new Peer('receiver', {key: '45rvl4l8vjn3766r'});
      peer.on('connection', function(conn) {
        conn.on("open", function(){
          console.log("receiver dataconn open");
          conn.on('data', function(data){
            console.log("received data: ", Date());
            window.socket.emit('receive', data);
          });
        });
      });
    }
  </script>
  peerjs with nodewebkit demo
  <button onclick="clickSend()">send</button>
  <button onclick="clickRecv()">receive</button>
</body>
</html>
``` 

It contains two button: `send` and `receive`. When clicked, `clickSend` and `clickRecv` gets called. To understand what these two function do, I'll show the other part of code in `main.js`.

```javascript
// main.js part 2
io.on('connection', function(socket){
  socket.on('send', function(data){
    socket.emit('sendToPeer', fs.readFileSync('.gitignore'));
  });
  socket.on('receive', function(data){
    fs.writeFileSync('received_gitignore', data);
  });
});
```

So you clicked the `receive` button, PeerJs create a `Peer` with id `receiver` and a valid key I registered, then it waits for connection from sender. Then you clicked the `send` button, another `Peer` is created with id `sender`. `sender` tries to connect to `receiver` and if data connection is successfully built, `window.socket` will emit a `send` event, which is handled in `main.js`. What the handler does is simply reading file content and send it back to DOM environment. Note that socket.io supports binary data transfer from 1.0, so you can't use a lower version socket.io.

Coming back to code in `clickSend`, we've already created an event handler on `sendToPeer` before emitting the `send` event, now it gets fired. `conn.send(data)` sends data to Peer `receiver`. In function `clickRecv`, when `conn` receives data, it uses `window.socket` to emit a `receive` event and sending the data it received from sender to Node runtime. Finally `fs.writeFileSync('received_gitignore', data)` write data to disk, all done.

You might wonder if it actually works for large file transfer. It does. My project is working great, it can transfer large files with decent speed, the prototype is this demo. Of course you should write many many more lines to make this prototype a usable app, for instance, data needs to be sliced when transfering large files, and you should handle all kinds of PeerJs errors.

That's all for this tutorial.


[1]:https://github.com/rogerwang/node-webkit/
[2]:http://peerjs.com/
[3]:https://github.com/peers/peerjs/issues/103
[4]:https://github.com/rogerwang/node-webkit/issues/1526
[5]:https://github.com/rogerwang/node-webkit/wiki/Differences-of-JavaScript-contexts
[6]:https://github.com/rogerwang/node-webkit/wiki/How-to-run-apps
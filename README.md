# post-robot [:]-\\-<

Cross domain post-messaging on the client side, using a simple listener/client pattern.

Send a message to another window, and:

- [Get a response](#simple-listener-and-sender-with-error-handling) from the window you messaged
- [Pass functions](#functions) to another window, across different domains
- [Handle any errors](#simple-listener-and-sender-with-error-handling) that prevented your message from getting through
- Don't worry about serializing your messages; [just send javascript objects](#simple-listener-and-sender-with-error-handling)
- Use [promises](#listener-with-promise-response), [callbacks](#listener-with-callback-response) or [async/await](#async--await) to wait for responses from windows you message
- Set up a [secure message channel](#secure-message-channel) between two windows on a certain domain
- Send messages between a [parent and a popup window](#parent-to-popup-messaging) in IE

## Simple listener and sender with error handling

```javascript
postRobot.on('getUser', function(event) {
    return {
        id:   event.data.id,
        name: 'Zippy the Pinhead'
    };
});
```

```javascript
postRobot.send(someWindow, 'getUser', { id: 1337 }).then(function(event) {
    console.log(event.source, event.origin, 'Got user:', event.data.name);

}).catch(function(err) {
    console.error(err);
});
```

## Listener with promise response

```javascript
postRobot.on('getUser', function(event) {

    return getUser(event.data.id).then(function(user) {
        return {
            name: user.name
        };
    });
});
```

## Listener with callback response

```javascript
postRobot.on('getUser', { id: 1337 }, function(event, callback) {

    setTimeout(function() {
        callback(null, {
            id:   event.data.id,
            name: 'Captain Pugwash'
        });
    }, 500);
});
```

## One-off listener

```javascript
postRobot.once('init', function(event) {

    return {
        name: 'Noggin the Nog'
    };
});
```

## Listen for messages from a specific window

```javascript
postRobot.on('init', { window: window.parent }, function(event) {

    return {
        name: 'Guybrush Threepwood'
    };
});
```

## Listen for messages from a specific domain

```javascript
postRobot.on('init', { domain: 'http://zombo.com' }, function(event) {

    return {
        name: 'Manny Calavera'
    };
});
```

## Set a timeout for a response

```javascript
postRobot.send(someWindow, 'getUser', { id: 1337 }, { timeout: 5000 }).then(function(event) {
    console.log(event.source, event.origin, 'Got user:', event.data.name);

}).catch(function(err) {
    console.error(err);
});
```

## Send a message to a specific domain

```javascript
postRobot.send(someWindow, 'getUser', { id: 1337 }, { domain: 'http://zombo.com' }).then(function(event) {
    console.log(event.source, event.origin, 'Got user:', event.data.name);
});
```

## Send a message to the direct parent

```javascript
postRobot.sendToParent('getUser').then(function(event) {
    console.log(event.data);
});
```

## Async / Await

```javascript
postRobot.on('getUser', async ({ source, origin, data }) => {

    let user = await getUser(data.id);

    return {
        id:   data.id,
        name: user.name
    };
});
```

```javascript
try {
    let { source, origin, data } = await postRobot.send(someWindow, `getUser`, { id: 1337 });
    console.log(source, origin, 'Got user:', data.name);

} catch (err) {
    console.error(err);
}
```

## Secure Message Channel

For security reasons, it is recommended that you always explicitly specify the window and domain you want to listen
to and send messages to. This creates a secure message channel that only works between two windows on the specified domain:

```javascript
postRobot.on('getUser', { window: childWindow, domain: 'http://zombo.com' }, function(event) {
    return {
        id:   event.data.id,
        name: 'Frodo'
    };
});
```

```javascript
postRobot.send(someWindow, 'getUser', { id: 1337 }, { domain: 'http://zombo.com' }).then(function(event) {
    console.log(event.source, event.origin, 'Got user:', event.data.name);

}).catch(function(err) {
    console.error(err);
});
```

You can even set up a listener and sender instance in advance:

```javascript
var listener = postRobot.listener({ window: childWindow, domain: 'http://zombo.com' });

listener.on('getUser', function(event) {
    return {
        id:   event.data.id,
        name: 'Frodo'
    };
});
```

```javascript
var client = postRobot.client({ window: someWindow, domain: 'http://zombo.com' });

client.send('getUser', { id: 1337 }).then(function(event) {
    console.log(event.source, event.origin, 'Got user:', event.data.name);

}).catch(function(err) {
    console.error(err);
});
```

## Functions

Post robot lets you send across functions in your data payload, fairly seamlessly.

For example:

```javascript
postRobot.on('getUser', function(event) {
    return {
        id:     event.data.id,
        name:   'Nogbad the Bad',

        logout: function() {
            currentUser.logout();
        }
    };
});
```

```javascript
postRobot.send(myWindow, { id: 1337 }, 'getUser').then(function(event) {
    var user = event.data;

    user.logout().then(function() {
        console.log('User was logged out');
    });
});
```

The function `user.logout()` will be called on the **original** window. Post Robot transparently messages back to the
original window, calls the function that was passed, then messages back with the result of the function.

Because this uses post-messaging behind the scenes and is therefore always async, `user.logout()` will **always** return a promise, and must be `.then`'d or `await`ed.


## Parent to popup messaging

Unfortunately, IE blocks direct post messaging between a parent window and a popup, on different domains.

In order to use post-robot in IE9+ with popup windows, you will need to set up an invisible 'bridge' iframe on your parent page:

```
   [ Parent page ]

+---------------------+          [ Popup ]
|        xx.com       |
|                     |      +--------------+
|  +---------------+  |      |    yy.com    |
|  |    [iframe]   |  |      |              |
|  |               |  |      |              |
|  | yy.com/bridge |  |      |              |
|  |               |  |      |              |
|  |               |  |      |              |
|  |               |  |      |              |
|  |               |  |      +--------------+
|  +---------------+  |
|                     |
+---------------------+
```

a. Create a bridge path on the domain of your popup, for example `http://yy.com/bridge.html`, and include post-robot:

```html
<!-- http://yy.com/bridge.html -->

<script src="http://yy.com/js/post-robot.js"></script>
```

b. In the parent page, `xx.com`, which opens the popup, include the following javascript:

```html
<!-- http://xx.com -->

<script>
    postRobot.openBridge('http://yy.com/bridge.html');
</script>
```

c. Now `xx.com` and `yy.com` can communicate freely using post-robot, in IE.
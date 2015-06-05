---
permalink: /docs/patterns/index.html
layout: docs
---

Shared patterns for dealing with common Hubot scenarios.

## Renaming the Hubot instance

When you rename Hubot, he will no longer respond to his former name. In order to train your users on the new name, you may choose to add a deprecation notice when they try to say the old name. The pattern logic is:

* listen to all messages that start with the old name
* reply to the user letting them know about the new name

Setting this up is very easy:

1. Create a [bundled script](/docs/scripting/) in the `scripts/` directory of your Hubot instance called `rename-hubot.coffee`
2. Add the following code, modified for your needs:

```coffeescript
# Description:
#   Tell people hubot's new name if they use the old one
#
# Commands:
#   None
#
module.exports = (robot) ->
  robot.hear /^hubot:? (.+)/i, (res) ->
    response = "Sorry, I'm a diva and only respond to #{robot.name}"
    response += " or #{robot.alias}" if robot.alias
    res.reply response
    return

```

In the above pattern, modify both the hubot listener and the response message to suit your needs.

Also, it's important to note that the listener should be based on what hubot actually hears, instead of what is typed into the chat program before the Hubot Adapter has processed it. For example, the [HipChat Adapter](https://github.com/hipchat/hubot-hipchat) converts `@hubot` into `hubot:` before passing it to Hubot.

## Deprecating or Renaming Listeners

If you remove a script or change the commands for a script, it can be useful to let your users know about the change. One way is to just tell them in chat or let them discover the change by attempting to use a command that no longer exists. Another way is to have Hubot let people know when they've used a command that no longer works.

This pattern is similar to the Renaming the Hubot Instance pattern above:

* listen to all messages that match the old command
* reply to the user letting them know that it's been deprecated

Here is the setup:

1. Create a [bundled script](scripting.md) in the `scripts/` directory of your Hubot instance called `deprecations.coffee`
2. Copy any old command listeners and add them to that file. For example, if you were to rename the help command for some silly reason:

```coffeescript
# Description:
#   Tell users when they have used commands that are deprecated or renamed
#
# Commands:
#   None
#
module.exports = (robot) ->
  robot.respond /help\s*(.*)?$/i, (res) ->
    res.reply "That means nothing to me anymore. Perhaps you meant `docs` instead?"
    return

```

## Forwarding all HTTP requests through a proxy

In many corporate environments, a web proxy is required to access the Internet and/or protected resources. For one-off control, use can specify an [Agent](https://nodejs.org/api/http.html) to use with `robot.http`. However, this would require modifying every script your robot uses to point at the proxy. Instead, you can specify the agent at the global level and have all HTTP requests use the agent by default.

Due to the way node.js handles HTTP and HTTPS requests, you need to specify a different Agent for each protocol. ScopedHTTPClient will then automatically choose the right ProxyAgent for each request.

```coffeescript
proxy = require 'proxy-agent'
module.export = (robot) ->
  robot.globalHttpOptions.httpAgent  = proxy('http://my-proxy-server.internal', false)
  robot.globalHttpOptions.httpsAgent = proxy('http://my-proxy-server.internal', true)
```

## Middleware Examples

Middleware is often used for authentication and authorization.  There are three
kinds of middleware: Post-receive, Pre-listener, and Pre-reply.

 - Post-Receive middleware happens for every message, whether it's matched or not.
 - Pre-Listener middleware is called for every listener that matches a message,
  before it has been executed
 - Pre-reply middleware is called for every message hubot attempts to send to
  a chat room, regardless of reason or source.

Hooks are asynchronous by default, needing to call either `middleware.finish()`
to allow further processing, or `middleware.done()` to end processing for a
given message. All hooks can be created with a `Sync` variant.

**All of these are just examples. None of them are secure. Don't use them.**

Here's a simple example to blacklist users at the listener level.
```coffeescript
module.exports = (robot) ->
  robot.preListen (middleware) ->
    if middleware.listener.options.id == "banned.command"
      if middleware.response.message.user.id == "whitelistedUser"
        # User is allowed access to this command
        middleware.next()
      else
        # Restricted command, but user isn't in whitelist
        middleware.response.reply "I'm sorry, @#{context.response.message.user.name}, but you don't have access to do that."
        middleware.done()
    else
      # This is not a restricted command; allow everyone
      middleware.next()
```

Here's a synchronous version of the same example. As soon as this function returns,
the listener will run (or the next middleware, if any). If `middleware.done()` is
called, further processing will stop.

```coffeescript
module.exports = (robot) ->
  robot.preListenSync (middleware) ->
    if middleware.listener.options.id == "banned.command" &&
       !middleware.response.message.user.id == "whitelistedUser"
         middleware.response.reply "I'm sorry, @#{context.response.message.user.name}, but you don't have access to do that."
         middleware.done()
```

Here's a similar synchronous example, blacklisting particular users from *all*
commands:

```coffeescript
module.exports = (robot) ->
  robot.preReceiveSync (middleware) ->
    if middleware.response.message.user.id == "blacklistedUser"
       context.response.reply "I'm sorry, @#{context.response.message.user.name}, but you don't have access to do that."
       middleware.done()
```

Here's an example that will replace sensitive information from being sent
under any circumstances:

```coffeescript
module.exports = (robot) ->
  robot.preReply (middleware) ->
    if middleware.reply.text.match(/passwords lol/)
      middleware.reply.text = "Sorry meatbag, no passwords allowed"
    middleware.next()
```

You could also block the message entirely:

```coffeescript
module.exports = (robot) ->
  robot.preReply (middleware) ->
    if middleware.reply.text.match(/passwords lol/)
      middleware.done()
    else
      middleware.next()
```

Here's a more complicated example, using an external auth system to
authenticate users for particular listeners:

```coffeescript
module.exports = (robot) ->
  robot.preListen (middleware) ->
    robot.auth(middleware.response.user, middleware.listener.options) -> (auth)
      if auth.allowed()
        middleware.next()
      else
         middleware.response.reply "I'm sorry, @#{context.response.message.user.name}, but you don't have access to do that."
         middleware.done()
```

Lastly, here's a version of the two-person rule, requiring a second person
to validate commands tagged as requiring it.

```coffeescript
crypto = require('crypto')
module.exports = (robot) ->
  randomIdentifier = (len = 12) ->
    crypto.randomBytes(Math.ceil(len * 3 / 4)).toString('base64').slice(0, len)

  robot.commandsAwaitingVerification = {}

  robot.preListenSync (middleware) ->
    if middleware.listener.options.two_person_protected?
      if middleware.response.message.verified?
        middleware.response.send "Who am I to argue with *two* fleshy constructs?"
      else
        middleware.message.code = randomIdentifier()
        robot.commandsAwaitingVerification[middleware.message.code] = middleware.message
        middleware.response.reply "A bold strategy. One of your meatbag friends needs to verify it with /verify #{msg.code}"
        middleware.done()

  robot.preReceive (middleware) ->
    if match = middleware.message.text.match(/verify (\S+)/)
      code = match[1]
      if old_message = robot.commandsAwaitingVerification[code]
        if middleware.message.user.id == old_message.middleware.user.id
          middleware.response.reply "You can't verify your own commands, silly meatbag."
          middleware.done()
        else
          # stop further processing on the verification and send the old message through the pipeline again.
          middleware.done()
          old_message.verified = true
          robot.receive(old_message)
```

# Sending Telegram Messages with Bash Scripts!

## Intro

A few months ago I realized a handful of my automated tasks were missing something - I didn't always know if they were running correctly! There are tons of ways to tackle this problem, but with smaller tasks I was looking for a quick and easy way to get notified of success/failure. 

Telegram ended up having a very easy-to-use API that was simple to integrate into any script. I'll go over the setup as well as a few super quick ways to leverage this tool.

There are three steps to getting this set up:

1. Creating a Telegram bot
2. Getting chat ID or group ID
3. Calling the API from a bash script

## Creating a Telegram bot

Your Telegram bot is what will be doing the actual sending of the message. When you leverage the Telegram API, you'll get a message in Telegram from your Telegram bot.

Creating bots is actually done through the Telegram app by messaging the one bot to rule them all - `@BotFather`. This is an automated tool that helps you manage and create Telegram bots. 

You can start off by sending a quick message to `@BotFather`:

```
/start
```

`@BotFather` will respond with a helpful guide containing some commands you can use. For this guide, we will be using:

```
/newbot
```

You will now be asked to choose a new for your bot. This name does not have to be unique and can pretty much be whatever you want, I'm going to name this one:

```
Test Bot
```

You will now be asked to create a username. This name *does* have to be unique and can't be shared with any existing bots. You will also have to make sure it ends with the word "bot". I'll be naming mine:

```
SuperDuperAwesomeTestBot
```

And you're all done! `@BotFather` will respond letting you know that your bot was created as well as giving you the `bot token` you will use to interact with the API. Go ahead and store this somewhere safe. If you do happen to forget it, you can always check the chat history with `@BotFather`.

## Getting the chat ID

Now that we have our new bot up and running, we need to find *where* its messages are going to go. We can leverage the Telegram API for this in just a few easy steps.

First, send a quick message to your new bot. It doesn't have to be anything fancy:

```
hey
```

Now we are going to call the `getUpdates` endpoint of the Telegram API. This endpoint returns recent updates to the bot, including messages it received. Most importantly, it shows the ID of *who* sent the message. This is the ID you will need, in fact its *your* ID!

You can make the call like so, replacing `${bot_token}` with the token `@BotFather` gave you:

```
$ curl -s https://api.telegram.org/bot${bot_token}/getUpdates | jq '.result[0].message.chat.id'
```

You should get a number in the response, this is your chat ID!

If that command is giving you any trouble, you can try calling like this:


```
$ curl -s https://api.telegram.org/bot${bot_token}/getUpdates | jq
```

You'll see a response like so:

```
{
  "ok": true,
  "result": [
    {
      "update_id": XXX,
      "message": {
        "message_id": 3,
        "from": {
          "id": XXXXX, <---------- Your Chat ID
          "is_bot": false,
          "first_name": "You",
          "username": "your_username",
          "language_code": "en"
        },
        "chat": {
          "id": XXXXX, <---------- Your Chat ID
          "first_name": "You",
          "username": "your_username",
          "type": "private"
        },
        "date": 1709126913,
        "text": "Hey"
      }
    }
  ]
}

```

Grab the `id` from the response and you're ready to test out your bot!

## Sending a message

With the initial set up out of the way, you have everything you need to use the API. Lets bring it all together:

```
curl -s --data-urlencode "text=$message" --data "chat_id=$chat_id" "https://api.telegram.org/bot$bot_token/sendMessage"
```

Replace `$message` with the body of your message. Thanks to the `--data-urlencode` option, you can even include `\n` line breaks for better formatting! 

Now if you check Telegram, you should see a message from your bot containing your message text!

## Sending messages to a group

Optionally, you can have your bot send messages to a group instead of just through an individual DM. This can be useful if you have several bots and want to group them together, or if you want an easy way to send a single message to multiple people. The set up here is pretty easy, we just need a group. If you don't have one already:

1. In Telegram, tap the pencil Icon on the bottom right and select "New Group".
2. In the search bar, search for your new bot
3. Hit the next arrow, give your group a name, and hit the check box

If you already have a group set up, just add your bot as a member to that group.

Done! Now we just need to find the ID of this group. We can use the `getUpdates` endpoint again for this. Run the following:

```
curl -s https://api.telegram.org/bot${bot_token}/getUpdates | jq
```

Now you should see a much longer response than before, we're looking for the result that has `my_chat_member`:

```
....
},
{
    "update_id": XXXXXXX,
    "my_chat_member": {
        "chat": {
            "id": -XXXXXXX, <------------ This is the ID you want
            "title": "Test Group",
            "type": "group",
            "all_members_are_administrators": true
        },
        "from": {
            "id": XXXXXXX,
            "is_bot": false,
            "first_name": "You",
            "username": "your_username",
            "language_code": "en"
        },
        "date": 1709129601,
        "old_chat_member": {
            "user": {
            "id": XXXXXXX,
            "is_bot": true,
            "first_name": "Test Bot",
            "username": "SuperDuperAwesomeTestBot"
            },
            "status": "left"
        },
        "new_chat_member": {
            "user": {
            "id": XXXXXXX,
            "is_bot": true,
            "first_name": "Test Bot",
            "username": "SuperDuperAwesomeTestBot"
            },
            "status": "member"
        }
    }
},
}
....
```

Grab the `id` under `my_chat_member` > `chat` > `id`. This may or may not be a negative number, make sure to copy the `-` as well. 

And you're all set! You can make the same call as before, just pass in the `id` you just grabbed as the `chat_id`:

```
curl -s --data-urlencode "text=$message" --data "chat_id=$group_id" "https://api.telegram.org/bot$bot_token/sendMessage"
```

## A few example uses

There are a ton of ways to integrate this into existing workflows, here are a few of my favorites: 

### Reusable script

It might seem obvious, but I like to put the call to send a Telegram message in a reusable script that can be called from any other script. 

Check out my `send_telegram_notification.sh` script [here](https://github.com/rhysbratti/scripts/blob/main/ipaudit/send_telegram_notification.sh)

For this script I like to place my `bot_token` and `chat_id` in a file with limited permissions (`chmod 400` should do it). This restricts the script to only running with `sudo` permissions.

### Login notification

On a Linux machine, any scripts placed in `/etc/profile.d/` will be run when a user logs in. We can capitalize on this by adding a script to send a Telegram message any time somebody logs into our machine. I use this for my remote servers to ensure nobody is logging in but me. 

Check out the `login_notification.sh` script [here](https://github.com/rhysbratti/scripts/blob/7fd95d516a76c4aef65dc2250efa43a39b664923/ipaudit/login_notification.sh)
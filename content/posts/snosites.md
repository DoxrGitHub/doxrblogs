+++
title = 'Simple Commenter to Email Vulnerability - SNOSites/WordPress:sno-comment-verification'
date = 2025-03-31T19:55:34Z
draft = false
+++

# Email Leak Vuln Through sno-comment-verification WordPress Plugin

This is a small blog to log this finding. The issue is not very critical at all and really at the end of the day it's just changing a URL. This blog is more about the other stuff; if you're only interested in being able to find the email of a commenter for SNOSites, the tl;dr contains what you're looking for. The rest of the blog is basically just a bunch of the other stuff I did with the SNO site since this was my first time pentesting a WordPress website.

## TL;DR

If you have a SNOSites WordPress blog website (they're usually student newsletters and stuff) with comments enabled and email verification for comments, you can find the email of people who commented. No authentication necessary, most of these websites are made so only blog editors have accounts anyways.

To do this, create a comment on any blog, and you should get an email that gives you a verification URL as such:

`https://example.com/wp-json/sno-comment-verification/v1/comment/?id=[commentID]&verification=[verificationhash]`

You will see this:

`Email address verification successful for [your email]. Your comment still needs to be approved by a moderator before it will be publicly available.` or `Email address verification successful for [your email]. Your comment has been approved.` (some websites, like my school's newsletter, also need someone to manually approve your comment after email verification).

You can change the numeric comment ID decrementally in that URL (no need to mess with the verification hash), and if that comment ID aligns to a different comment *that is also up* (you may hit some comments that didn't have their email verified), it will return this instead:

`Email address verification successful for [comment owner's email]. Your comment has been approved.`

This means that all you need to find a commenter's email is their ID. You can find the ID of a comment pretty easily: press the "share" button for a comment:

![image of share button](https://i.imgur.com/QMLIK9p.png)

You'll get a URL like this:

`https://example.com/99999/slug/longer-title-slug/#comment-[comment ID]`

You can copy that comment ID and use it with the `sno-comment-verification/v1/comment` URL instead, and you get their email.

Unpatched as of 3/31/25 as I was unable to report this.

## Why Did I Start

I realized that I kept giving up extremely easily when it came to pentesting, after the low-hanging fruit was already thought up of. That's why I decided that I needed to actually spend more time without giving up, and decided to go with SNOSites WordPress websites (which is affiliated with my school). I've never used WordPress myself, and I had been ignorant of WordPress based vulnerabilities, since I didn't really care about WordPress.

However, SNOSites became my target , and I started going through the website. My ultimate goal was to be able to modify a blog post while having no affiliation with any of the staff team. Ultimately, I never reached this goal but maybe I will some other day.

## Fuzzing

I'm sure this is a staple of pentesting WordPress websites, and I used `fuff` with [this wordlist](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/CMS/wordpress.fuzz.txt) to find and store a bunch of open paths. Doing more research, I looked into `wp-json` and its namespaces for a whole lotta time. I found a couple of important things, and I think that the most important thing was probably finding all of the plugins that my student newsletter site used.

Obviously I went through a bunch of the open pages, and while I found some pages that probably didn't need to be up, not many were noticably important.

## Plugins List Through wc/v3 AKA WooCommerce

Apparently for some reason WooCommerce leaks every plugin? I saw a route in the wc/v3 namespace after vising /wp-json, `/wc/v3/ams-get-active-plugins`, and you can guess what I saw based on the route:

```
[
  {
    "Requires Yoast SEO": "",
    "Name": "AppMySite",
    "PluginURI": "https://www.appmysite.com",
    "Version": "3.13.1",
    "Description": "This plugin enables WordPress & WooCommerce users to sync their websites with native iOS and Android apps, created on \u003Ca href=\"https://www.appmysite.com/\"\u003E\u003Cstrong\u003Ewww.appmysite.com\u003C/strong\u003E\u003C/a\u003E",
    "Author": "AppMySite",
    "AuthorURI": "https://appmysite.com",
    "TextDomain": "appmysite",
    "DomainPath": "",
    "Network": false,
    "RequiresWP": "",
    "RequiresPHP": "",
    "UpdateURI": "",
    "RequiresPlugins": "",
    "Title": "AppMySite",
    "AuthorName": "AppMySite"
  },
  {
    "Requires Yoast SEO": "",
    "Name": "Classic Editor",
    "PluginURI": "https://wordpress.org/plugins/classic-editor/",
    "Version": "1.6.7",
    "Description": "Enables the WordPress classic editor and the old-style Edit Post screen with TinyMCE, Meta Boxes, etc. Supports the older plugins that extend this screen.",
    "Author": "WordPress Contributors",
    "AuthorURI": "https://github.com/WordPress/classic-editor/",
    "TextDomain": "classic-editor",
    "DomainPath": "/languages",
    "Network": false,
    "RequiresWP": "4.9",
    "RequiresPHP": "5.2.4",
    "UpdateURI": "",
    "RequiresPlugins": "",
    "Title": "Classic Editor",
    "AuthorName": "WordPress Contributors"
  },
  {
    "Requires Yoast SEO": "",
    "Name": "Delete Pending Comments",
    "PluginURI": "https://bulkwp.com",
    "Version": "1.0.0",
    "Description": "A quick way to delete all pending and spam comments. Useful for victims of spammer attacks.",
    "Author": "sudar",
    "AuthorURI": "https://sudarmuthu.com/",
    "TextDomain": "delete-pending-comments",
    "DomainPath": "translations/",
    "Network": false,
    "RequiresWP": "",
    "RequiresPHP": "",
    "UpdateURI": "",
    "RequiresPlugins": "",
    "Title": "Delete Pending Comments",
    "AuthorName": "sudar"
  }
  ...
```

I'm not sure if it should be that easy to get all installed plugins with a regular WordPress site, but this did drive me to look for vulnerable/old WordPress plugin versions, which I did actually find. However, I didn't really know how to craft a PoC with the limited details I had about their respective vulnerabilities, and I didn't really care about trying to abuse those older plugins

## Pentesting the Login Page

Long story short I tried a bunch of stuff and got nowhere. I was able to reverse a staff member to their email only because my school has a very specific format that allows you to get a student's email through their first and last name, as well as 4 digits that a managed Gmail client leaks for you. The staff members' full names were on the site, so I chose someone that I knew to pentest on. Anyways, I wasn't able to take over his account, but I did see that running a forget password thing and supplying an array instread of a string for the email crashed the website, in the sense that it didn't gracefully give you an error but rather just said that SNO was contacted regarding the 500 error.

## Comments

I tried to XSS the comments, but the server did their XSS filtering server-side (probably thanks to the plugin Really Simple Security that was installed, unless SNO's comment plugin actually also had filtering), and I gave up on XSS really quickly. I didn't think it would really help with my ultimate goal, so I stopped. However, I did try to make comments and when I saw the verification URL, I tried to change the comment ID which I didn't expect to work because of the verification hash, so that was kind of suprising (read the TL;DR now if you haven't).

## Something Funny

I found that you can replay the request to like a post and recommend a comment over and over again, leading to crazy numbers if you do it for a while. Same thing with polls, if you have that.

Here's the video PoC/demonstration:

<video width="800" controls>
  <source src="../../snosites.webm" type="video/webm">
  Your browser does not support this video.
</video>

## Conclusion

This blog doesn't cover all the nonsense I did but I didn't want all that time to go to waste so I wrote this blog about the only thing that might be a privacy concern to some people. This blog post was more about my experience pentesting a WordPress website. If you think about it, all it is, is just the email of some random person on the internet, so while it might be possible that hackers would go through SNOSites with comments and scraped the emails of everyone who comments for personalized scam bait, its unlikely (even though said vulnerability is extremely simple). Realistically, nothing of importance arises from this, but it should still be patched. The patch would be easy too... just don't return the email of the commenter when you visit the verification page?

- doxr
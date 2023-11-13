+++
title = "No Thank You Adobe"
date = "2023-07-19T16:41:37-05:00"
tags = ["adobe", "macOS"]
+++

Getting rid of Adobe's "Creative" processes with [santa](https://github.com/google/santa).

<!--more-->

> It is named Santa because it keeps track of binaries that are naughty or nice. [^1]

Every time I quit an adobe app, like photoshop, I ended up with a bunch of background processes:

{{< figure src="/blog/no-thank-you-adobe/process-list.webp" caption="Process List" class="center-fig">}}

(Don't forget we also have "Creative Cloud Helper", "Core Sync", "CCXProcess_", and "CCLibrary"...)

Simply removing the `com.adobe` plists from `/Library/LaunchAgents/` and `~/Library/LaunAgents` can prevent them from starting up at login, but not after quitting an adobe app.

And soon you realize, you can kill everything except the `AdobeIPCBroker`, every single time I tried to terminate the process, it will just come back. (sigh 😮‍💨)

After digging around the internet, found [this post](https://apple.stackexchange.com/a/265918) that gives the necessary steps to get rid of `AdobeIPCBroker`:

1. Kill Core Sync.
2. Kill CCLibrary.
3. Kill CCXProcess.
4. Kill AdobeIPCBroker.


**Introducing santa**

{{< figure src="https://raw.githubusercontent.com/google/santa/main/Source/gui/Resources/Images.xcassets/AppIcon.appiconset/santa-hat-icon-256.png"  caption="santa icon" class="center-fig">}}

> Santa is a binary authorization system for macOS. It consists of a system extension that monitors for executions, a daemon that makes execution decisions based on the contents of a local database, a GUI agent that notifies the user in case of a block decision and a command-line utility for managing the system and synchronizing the database with a server. [^1] 

Official git repo: [google/santa](https://github.com/google/santa)    
Latest release: [tag:latest](https://github.com/google/santa/releases/latest)

TL;DR: santa will match the identifier of binary and block it if it matches one of the preset rules:

```bash
sudo santactl rule --silent-block --path /Library/Application\ Support/Adobe/Adobe\ Desktop\ Common/IPCBox/AdobeIPCBroker.app

sudo santactl rule --silent-block --path /Applications/Utilities/Adobe\ Creative\ Cloud/ACC/Creative\ Cloud\ Helper.app/

sudo santactl rule --silent-block --path /Applications/Utilities/Adobe\ Creative\ Cloud/ACC/Creative\ Cloud.app/
```

additionally:

```bash
sudo santactl rule --silent-block --path /Library/Application\ Support/Adobe/Adobe\ Desktop\ Common/ADS/Adobe\ Desktop\ Service.app/

sudo santactl rule --silent-block --path /Library/Application\ Support/Adobe/Creative\ Cloud\ Libraries/CCLibrary.app/

sudo santactl rule --silent-block --path /Applications/Utilities/Adobe\ Creative\ Cloud\ Experience/CCXProcess/CCXProcess_.app/

sudo santactl rule --silent-block --path /Applications/Utilities/Adobe\ Sync/CoreSync/Core\ Sync.app/

sudo santactl rule --silent-block --path /Applications/Utilities/Adobe\ Creative\ Cloud/ACC/Creative\ Cloud\ Helper.app/
```

feel free to kill the process from **Activity Monitor**. You don't have to see it again in some times. You will need to update the rules after updating those apps since it uses sha256sum as identifiers.

incase you are wondering, photoshop 2022 and audition 2022 works just fine:

{{< figure src="/blog/no-thank-you-adobe/ps-2022.webp"  caption="picture from: https://www.deviantart.com/winkyna/art/CZ-Krtek-CZ-Mole-145481566" class="center-fig">}}


[^1]: https://github.com/google/santa#readme
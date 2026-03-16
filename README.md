# Phoniebox: RPi-Jukebox-RFID — Modern Web UI Fork

> 🔀 **Fork of [MiczFlor/RPi-Jukebox-RFID](https://github.com/MiczFlor/RPi-Jukebox-RFID)**
> Based on v2.x · Replacing the PHP/jQuery web interface with a modern SPA · All hardware untouched

[![Matrix chat](https://matrix.to/img/matrix-badge.svg)](https://matrix.to/#/#phoniebox_community:matrix.org)

## What's Different in This Fork

This fork keeps the **entire v2 backend** (RFID daemon, GPIO buttons, MPD, `playout_controls.sh`) and replaces **only the web interface** — the 77 PHP/jQuery/Bootstrap 3 files in `htdocs/` — with a modern stack:

| Component | Old (v2) | New (this fork) |
|-----------|----------|-----------------|
| **Frontend** | jQuery 1.12 + Bootstrap 3 + PHP templates | Svelte 5 + SvelteKit + Tailwind CSS |
| **Backend API** | PHP with `exec()` calls | FastAPI (Python) wrapping shell scripts safely |
| **Web server** | Lighttpd | Nginx (WebSocket proxy support) |
| **Real-time** | 5-second polling | WebSocket with MPD idle bridge |
| **Config** | 50+ flat files in `settings/` | SQLite (with flat-file writeback for shell scripts) |
| **Auth** | None | Three-tier PIN model (open / parent / expert) |
| **Security** | Command injection in PHP, `shell=True` | Subprocess argument lists, input validation |

### Why not v3?

We evaluated [Phoniebox v3 (future3)](https://github.com/MiczFlor/RPi-Jukebox-RFID/tree/future3/main) and decided against migrating. v2's backend works reliably — only the web UI needs replacement. See [our analysis](.planning/ANALYSIS.md) for the full evaluation.

### What stays the same

Everything that works stays untouched:

* RFID card reader (RC522/USB) and `daemon_rfid_reader.py`
* GPIO buttons and rotary encoder via `gpio_control.py`
* MPU6050 gyro sensor via [phonie-gyro](https://github.com/t13gazh/phonie-gyro)
* MPD audio playback and `playout_controls.sh` (1,153 lines, single source of truth)
* Card-to-folder assignment, resume position, playlists
* All existing hardware (speakers, DACs, arcade buttons)

## Status

🚧 **In development** — no releases yet.

| PR | Scope | Slices | Status |
|----|-------|--------|--------|
| PR 1 | API Foundation & Real-Time Layer | S01 + S02 | Not started |
| PR 2 | Player UI & Library Browser | S03 + S04 | Not started |
| PR 3 | Card/Content Management & Settings | S05 – S08 | Not started |
| PR 4 | Access Control, Gyro & PWA | S09 – S11 | Not started |

Development happens on the [`feature/webui-modernization`](https://github.com/t13gazh/RPi-Jukebox-RFID/tree/feature/webui-modernization) branch. PRs will be offered to the upstream repo as they complete.

## Additional Hardware

This fork adds support for:

* **MPU6050 gyro sensor** via [phonie-gyro](https://github.com/t13gazh/phonie-gyro) — tilt gestures to control playback (skip, volume, pause). Configurable via the new web UI (sensitivity profiles, calibration).

---

## Original Phoniebox Documentation

*Everything below is from the [original Phoniebox project](https://github.com/MiczFlor/RPi-Jukebox-RFID) and applies to both the original and this fork.*

---

## Introduction

Phoniebox is a contactless jukebox for the Raspberry Pi, playing audio files, playlists, podcasts, web streams and spotify triggered by RFID cards. All plug and play via USB, no soldering iron needed. It also features GPIO buttons control support.

## Everyone builds an original

This is one of the most inspiring and gratifying experiences: Phoniebox is about Open Source AND Open Design.

See innovation, upcycling and creativity in the [Phoniebox Gallery](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/GALLERY)

![Caption](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/gallery/Steph-20171215_h90-01.jpg)
![Caption](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/gallery/Elsa-20171210_h90-01.jpg)
![Caption](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/gallery/Geliras-20171228-Jukebox-01-h90.jpg)
![Caption](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/gallery/UlliH-20171210_h90-01.jpg)
![Caption](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/gallery/KingKahn-20180101-Jukebox-01-h90.jpg)
![Caption](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/gallery/hailogugo-20171222-h90-01.jpg)

## Install Phoniebox

### 🎶 Version 2

* ⭐ **[Releases](https://github.com/MiczFlor/RPi-Jukebox-RFID/releases?q=v2&expanded=true)**
* 🎵 **[Install Jukebox Version 2](#quick-install-version-2)**
* 🐛 [Report a bug](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues/new?assignees=&labels=bug%2Clegacy_v2%2Cneeds+triage&projects=&template=bug_v2.yaml&title=%F0%9F%90%9B+%7C+)
* ☑️ [Features](#features-version-2)
* 📖 [Documentation](#documentation)
* 🦄 Code: [Release Branch](https://github.com/MiczFlor/RPi-Jukebox-RFID/tree/master), [Development Branch](https://github.com/MiczFlor/RPi-Jukebox-RFID/tree/develop)

#### Features Version 2

* The **one-line install script** handles both **Classic** and **+Spotify** when [setting up your Phoniebox](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/INSTALL#one-line-install-command)
  * integrated selection of RFID readers
  * features *non-interactive* installs based on a config file
* [**RFID** control](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#cardcontrol) for playout and controlling your Phoniebox (e.g. set [maximum volume with RFID](https://github.com/MiczFlor/RPi-Jukebox-RFID/pull/633) cards).
* Support for various RFID readers, e.g.
  * USB
  * [RC522](components/rfid-reader/RC522/README.md)
  * [PN532](components/rfid-reader/PN532/README.md)
  * PC/SC
  * also [multiple readers](https://github.com/MiczFlor/RPi-Jukebox-RFID/pull/1012#issue-434052529) simultaneously
* [**GPIO** control](components/gpio_control/README.md) for [buttons](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/Using-GPIO-hardware-buttons) and much more to control your Phoniebox via GPIO.
* Control via smooth [**Web App**](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#webapp) running on ajax from your phone, tablet or PC. You can play, upload, move files, assign new RFID cards, control playout, settings, etc.
* Support for files with embedded chapters metadata (like m4a)
* Customizable poweroff command
* **[WiFi management](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#wifi-settings)**
  * RFID cards to toggle WiFi (or switch it on/off)
  * Read out the WiFi IP address (if you are connecting to a new network and don't know where to point your browser)
  * **Hotspot** Phoniebox: [ad-hoc hotspot](https://github.com/MiczFlor/RPi-Jukebox-RFID/pull/967) if no known network found (IP: 10.0.0.5 SSID: phoniebox Password: PlayItLoud)
* **Touchscreen** LCD display Player (file: `index-lcd.php` in web app)
* Integrate your [Phoniebox in your Smart Home with MQTT](components/smart-home-automation/MQTT-protocol/README.md).
* Player control
  * [Playout **Resume**](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#manage-playout-behaviour) switch for audio books, allowing you to jump straight back to where you were (unless you fell asleep...).
  * Playout **Shuffle** switch to mix up your playlists.
* Audio sources
  * Local audio files
  * Download from **YouTube** directly to your Phoniebox.
  * Support for **[Spotify](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/Spotify-FAQ)** integration.
  * [**Podcast feature** for Phoniebox](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#podcasts)

#### Quick install Version 2

* Check [Installation instructions](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/INSTALL) with the [One line install script](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/INSTALL#one-line-install-command) for Raspberry Pi OS.
* Two editions:
  * *Phoniebox Classic* supports local audio, web radio, podcasts, YouTube (download and convert), GPIO and/or RFID
  * **MUST READ** for users of *[Phoniebox +Spotify Edition](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/Spotify-FAQ)*
* Once everything has been installed and configured, [read the manual](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL) to change settings, register RFID cards, add audio.
* Adding push buttons to control volume, skipping tracks, pause, play: read the [GPIO Control Readme](components/gpio_control/README.md).
* There's also a [Complete Install Guide](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/INSTALL-COMPLETE-GUIDE)

## Contact and Support

* Please ask **general questions** in [Discussions](https://github.com/MiczFlor/RPi-Jukebox-RFID/discussions)
* **Bugs and enhancements** should still be in [Issues](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues).
* Want to discuss with others? Check out our [Matrix community](https://matrix.to/#/#phoniebox_community:matrix.org)

## Documentation

Documentation can be found in the [GitHub wiki for Phoniebox](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki). Please try to add content in the wiki regarding special hardware, software tweaks and the like.

### Manual

In the [Manual](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL) you will learn:

* [How to connect to the Phoniebox from any computer to add and edit audio files.](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#connect)
* [How to register new RFID cards, assign them a *human readable* shortcut and add audio files for each card.](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#registercardswebapp)
* [How to add webradio stations and other streams to the playout files](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#webstreams) - [and even mix web based and local files.](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#mixwebstreams)
* [Adding Podcasts the your Phoniebox](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#podcasts)
* [How to control the Phoniebox through the web app.](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#webapp)
* [How to assign cards specific tasks such as changing the volume level or shutting down the Phoniebox.](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/MANUAL#cardcontrol)

### Troubleshooting

There is a growing section of [troubleshooting](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/Troubleshooting-FAQ) including:

* I want to improve the onboard audio quality
* I am moving, how do I get the Phoniebox into my new WiFi network?
* The RFID Reader doesn't seem to work.
* Changing the volume does not work, but the playout works.
* Script `daemon_rfid_reader.py` only works via SSH not by RFID cards.
* Script daemon is closing down unexpectedly.
* Everything seems to work, but I hear nothing when swiping a card.
* I would like to use two cards / IDs to do the same thing.

## Hardware aka Shopping List

Here is a list of equipment needed. You can find a lot second hand online (save money and the planet). The links below lead to Amazon, not at all because I want to support them, but because their PartnerNet program helps to support the Phoniebox maintenance (a little bit...).

> [!NOTE]
> Depending on individual projects, the hardware requirements vary.

### Raspberry Pi

* [Raspberry Pi 4 Modell B](https://amzn.to/2Yuei04)
* [Raspberry Pi 3 Model B](https://amzn.to/3fqp8ef)
* [Raspberry Pi Zero WH](https://amzn.to/3fkfKc5)

You might be surprised how easy and affordable you can get an RPi second hand. Think about the planet before you buy a new one.

### RFID Reader and cards / fobs

* RFID Card Reader (USB): [Neuftech USB RFID Reader ID](https://amzn.to/2RrqScm) using 125 kHz - make sure to buy compatible cards, RFID stickers or key fobs working with the same frequency as the reader.
  * RFID cards: [125 KHz EM4100](https://amzn.to/37pjy9q) make sure the frequency matches the RFID card reader!  
  * RFID fobs / key rings: [EM4100 RFID-Transponder-Schlüsselring, 125 KHz](https://amzn.to/3hsuvLO) make sure the frequency matches the RFID card reader!
* RFID Kit RC522: [RC522 Reader, Chip, Card for Raspberry Pi 13.56MHz](https://amzn.to/2C7YZCZ)
  * RFID sticker / tags: [MIFARE RFID NFC Tags](https://amzn.to/30GfLDg) untested by me personally, but reported to work with work with RC522 and PN532 card readers.  

### Speakers / amps

* [USB Stereo Speaker Set (6 Watt, 3,5mm jack, USB-powered) black](http://amzn.to/2kXrard) | This USB powered speaker set sounds good for its size, is good value for money and keeps this RPi project clean and without the need of a soldering iron :)
* [USB A Male to Female Extenstion Cable with Switch On/Off](http://amzn.to/2hHrvkG) | I placed this USB extension between the USB power adapter and the Phoniebox. This will allow you to switch the Phoniebox on and off easily.
* [USB 2.0 Hub 4-port bus powered USB Adapter](http://amzn.to/2kXeErv) | Depending on your setup, you will need none, one or two of these. If you are using the external USB powered speakers, you need one to make sure the speakers get enough power. If you want to use the additional USB soundcard and have an older RPi, you might need a second one to make sure you can connect enough devices with the RPi.

### Arcade Buttons

* [USB Interface for Arcade buttons](https://amzn.to/3nRAtIS) if you insist on not soldering hardware.
* Arcade Buttons / Sensors (one of these might suit you)
  * [Arcade Buttons / Schalter in various colours](https://amzn.to/2QMxe9r) if you want buttons for the GPIO control.
  * [Arcade Buttons wit LED and custom icons](https://amzn.to/2MWQ6hq) as used by [@splitti](https://splittscheid.de/selfmade-phoniebox/#3C).
  * [Set: Arcade Buttons / Tasten / Schalter](https://amzn.to/2T81JTZ) GPIO Extension Board Starter Kit including cables and breadboard.
  * [Touch Sensor / Kapazitive Touch Tasten](https://amzn.to/2Vc4ntx) these are not buttons to press but to touch as GPIO controls.

### Special hardware

These are links to additional items, which will add an individual flavour to your Phoniebox setup. Consult the issue threads to see if your idea has been realised already.

* [Ground Loop Isolator / Entstörfilter Audio](https://amzn.to/2Kseo0L) this seems to [get rid off crackles in the audio out (a typical RPi problem)](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues/341)
* [Mechanical audio switch](https://amzn.to/35oOSCS) if you want to connect differen audio devices, this is the easiest way (in connection with the *Ground Loop Isolator* you will get good results)
* [Rotary Encoder / Drehregler / Dial](https://amzn.to/2J34guF) for volume control. Read here for more information on how to [integrate the rotary dial](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues/267)
* [HiFiBerry DAC+ Soundcard](https://amzn.to/2J36cU9) Read here for more information on how to [HifiBerry Soundcard integration](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/HiFiBerry-Soundcard-Details)
* [HDMI zu HDMI + Optisches SPDIF mit 3,5-mm-Stereo-HDMI Audio-Extractor | HDMI zu SPDIF Konverter](https://amzn.to/2N8KP8C) If you plan to use video, this might be the better solution than a USB soundcard or the hifiberry. If takes up some space, but will work with the HDMI audio out and split the signal to deliver audio through 3.5mm jack.

### Components

Special hardware is now organised in the folder [`components`](components/). If you have new hardware attached to your Phoniebox, please add to this library! It currently contains soundcards, displays, GPIO controls, RFID reader, smarthome integration.

## Support Phoniebox
<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD045 -->

<a href="https://www.buymeacoffee.com/MiczFlor" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/guidelines/download-assets-sm-2.svg"></a>

If you like your Phoniebox, consider to [buy me a coffee](https://www.buymeacoffee.com/MiczFlor)
or donate via [PayPal](https://www.paypal.com) to <micz.flor@web.de> using the *friends* option.

## Media

### Phoniebox Gallery

Visit the [Phoniebox Gallery](https://github.com/MiczFlor/RPi-Jukebox-RFID/wiki/GALLERY)

### Videos and Screenshots

<a href="https://youtu.be/7GI0VdPehQI" target="_blank"><img src="https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/iFun-YouTube.jpg" alt="Prototype of the RFID jukebox" width="800" height="450" border="1" /></a>

See the Phoniebox code in action, watch this video and read the blog post from [iphone-ticker.de](https://www.iphone-ticker.de/wochenend-projekt-kontaktlose-musikbox-fuer-kinder-123063/)

**We love Tech** published a video screencast on *how to build your Phoniebox* (in German), you can find all the steps and see the final product here:

| Installation und Hardware | Web App and Audio / Spotify | The finished Phoniebox in action |
| --- | --- | --- |
| [![Installation und Hardware](https://i.ytimg.com/vi/spHDGSxckmw/hqdefault.jpg)](https://youtu.be/spHDGSxckmw) | [![Web App and Audio / Spotify](https://i.ytimg.com/vi/9ZSKFoWr9WY/hqdefault.jpg)](https://youtu.be/9ZSKFoWr9WY) | [![The finished Phoniebox in action](https://i.ytimg.com/vi/YucQuwUD7XE/hqdefault.jpg)](https://youtu.be/YucQuwUD7XE) |

<!-- markdownlint-restore -->

**What makes this Phoniebox easy to install and use:**

* Runs on all Raspberry Pi models (1, 2, 3 and 4) and [Raspberry Zero](https://github.com/MiczFlor/RPi-Jukebox-RFID/issues/15).
* Just plug and play using USB, no soldering iron needed.
* Once the Phoniebox is up and running, add music from any computer on your home network.
* Register new RFID cards easily without having to connect to the RPi.
* Play single or multiple files, podcasts or web streams.
* Volume control is also done with RFID cards or key fobs.
* Connect to your Phoniebox via your wifi network or run the Phoniebox like an access point and connect directly without a router.
* **Bonus:** control the Phoniebox from your phone or computer via a web app.

![The web app allows you to change the volume level, list and play audio files and folders, stop the player and shut down the RPi gracefully.](https://raw.githubusercontent.com/wiki/MiczFlor/RPi-Jukebox-RFID/img/web-app-iphone-screens.jpg "The web app allows you to change the volume level, list and play audio files and folders, stop the player and shut down the RPi gracefully.")

The **web app** runs on any device and is mobile optimised. It provides:

* An audio player to pause, resume, shuffle, loop, stop and skip to previous and next track.
* Sub folder support: manage your collection in sub folders. Phoniebox has two play buttons: only this folder and eeeeverything in this folder.
* Manage files and folders via the web app.
* Register new RFID cards, manage Phoniebox settings, display system info and edit the wifi connection.
* Covers displayed in the web app (files called `cover.jpg`).

## Sustainability

You might be surprised how easy and affordable you can get a RaspberryPi or an "appropriate" housing for your Phoniebox **second hand**. Think about the planet before you buy a new one.
Creating a Phoniebox may be sustainable for the following reasons:

* You **buy things second hand** or **do upcycling on unused objects** and reduce newly produced products
* You built your Phoniebox yourself, so **maintaining and repairing is not a problem** (Additionally **a great community helps** you)
* Since the Phoniebox uses Linux as a base system it's **very unlikely that you run out of system and security updates** - so it can run and run and run...
* **RFID cards or tags can be reused - no need to buy new plastic elements** for changing the music or story linked to a card
* **Build it together with your kids** to show them that building things on their own is possible and in cooperation with others makes life easier and fun at the same time

## Contributing improvements

Read the [CONTRIBUTING.md](CONTRIBUTING.md) file for [more infos on how to contribute code](CONTRIBUTING.md).

## Acknowledgments

There are many, many, many inspiring suggestions and solutions on the web to bring together the idea of a jukebox with RFID cards. I want to mention a few of these that have inspired me.

* Thanks to all the [contributors](https://github.com/MiczFlor/RPi-Jukebox-RFID/graphs/contributors). Not only for the good code review and feature suggestions, but also for the good spirit I get each time a new Phoniebox comes to this world :)
* Thanks to Andreas aka [hailogugo](https://github.com/hailogugo) for writing and testing the script for the [GPIO buttons as controllers for the jukebox](docs/GPIO-BUTTONS.md).
* [Francisco Sahli's Music Cards: RFID cards + Spotify + Raspberry Pi](https://fsahli.wordpress.com/2015/11/02/music-cards-rfid-cards-spotify-raspberry-pi/) written in python, playing songs from Spotify. The code [music-cards](https://github.com/fsahli/music-cards) is on GitHub.
* [Jeremy Lightsmith's rpi-jukebox](https://github.com/jeremylightsmith/rpi-jukebox) written in Python, using the mpg123 player
* [Marco Wiedemeyer's Raspberry Pi Jukebox für Kinder (German)](https://blog.mwiedemeyer.de/post/Raspberry-Pi-Jukebox-fur-Kinder/) written in mono, using the MPD player
* [Marcus Nasarek's Kindgerechter Audioplayer mit dem Raspberry Pi (German)](http://www.raspberry-pi-geek.de/Magazin/2014/03/Kindgerechter-Audioplayer-mit-dem-Raspberry-Pi) triggered by QR codes via a camera instead of RFID cards, written in bash and using the xmms2 media player
* [Huy Do's jukebox4kids / Jukebox für Kinder](http://www.forum-raspberrypi.de/Thread-projekt-jukebox4kids-jukebox-fuer-kinder) written in Python, [the code is on github](https://github.com/hdo/jukebox4kids)
* [Willem van der Jagt's How I built an audio book reader for my nearly blind grandfather](http://willemvanderjagt.com/2014/08/16/audio-book-reader/) written in python and using the MDP player.

I also want to link to two proprietary and commercial projects, because they were an inspiration in the early days of the Phoniebox. Since the first release, the Phoniebox code has shown the power of open source development. Today, Phoniebox might be the most versatile project of its kind.

* [tonies® - das neue Audiosystem für mehr Hör-Spiel-Spaß im Kinderzimmer. (German)](https://tonies.de/) You buy a plastic figure which then triggers the audiofile - which is served over the web.
* [Hörbert - a MP3 player controlled by buttons](https://hoerbert.com) In Germany this has already become a *classic*. They also started selling a DIY kit.

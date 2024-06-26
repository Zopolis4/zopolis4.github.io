# Plasma 6 Thoughts

## Preamble

With the Plasma 6 release projected to arrive on February 28th, I thought now would be a good time to take a look at some of the upcoming changes that I've been looking forward to for some time now.

Note that I have not actually been running Plasma 6 and testing these changes, so the vast majority of what I say is probably wrong. I have, however, been reading [Planet KDE](https://planet.kde.org/), so I'm practically an expert.

(By the way, if you're interested in hearing about changes like these from people who actually know what they're talking about, [Planet KDE](https://planet.kde.org/) is the best place to do it.)

## I had no choice

Due to the nature of my issues, with some of them being hardware/distro specific, it makes sense to provide some information about my system.

I hired the world's greatest minds to figure out the best way to do this, and after months of research they presented their findings. Horrified, I immediately fired them all and set about assembling the world's second greatest minds to find me an answer. Unfortunately, they came to the same conclusion as their predecessors. The budget wouldn't support hiring the world's third greatest minds, so I am left with no choice but to do this.

[![neofetch.png]({{site.url}}/images/neofetch.png)]({{site.url}}/images/neofetch.png)

This hurts me more than it hurts you.

### Brightness

Previously, the settings for changing brightness were located in the battery applet. The problem is, the applet disappears at full charge, meaning that if I wanted to change my brightness I had to unplug my laptop and let the battery drop below full charge. This works, but is really not ideal. Thankfully, [in Plasma 6](https://invent.kde.org/plasma/plasma-workspace/-/issues/93), Brightness and Power Management are two separate applets, and I can manage brightness via the applet at all levels of charge.

Some of you may be asking the question: "Why can't you just use the dedicated brightness-changing keys on your keyboard to change brightness?"

You're right. I can. Except in one specific scenario, turning the brightness down as low as possible. As an [embedded software developer](https://octodon.social/@faho/111306878523313027), I encounter this scenario quite often, so this problem is very important to me.

The lowest brightness level the function keys can reach is 5%, with 0% brightness turning the screen off. The battery applet can set the brightness to 0% *without* turning the screen off. Behind the scenes, the function keys are setting the brightness level to 0, which on certain hardware (mine included) turns off the screen, while the battery applet is setting the brightness level to 1.

However, in Plasma 6, the behaviour of the [function keys](https://invent.kde.org/plasma/powerdevil/-/merge_requests/184) and [the applet](https://invent.kde.org/plasma/plasma-workspace/-/merge_requests/3166) has been changed to have a minimum brightness level of 1.

On the one hand, this is good! I can now select the lowest possible brightness through the function keys, and 0% brightness now has a consistent meaning. Although, I appreciated the ability to set brightness level 0 and turn the screen off. Additionally, hardware which does not turn the screen off at brightness level 0 is now unable to reach the lowest possible brightness.

### Discover

(From what I can tell, Discover does much better at this on other distros, particularly KDE Neon. Shockingly, it does not appear that rolling release Debian was considered as a common use case.)

This may sound like a criticism of Discover, but most of these issues aren't Discover's fault. Like a round peg in a square hole, Discover is technically able to fit into the role of a system package manager, but obviously was not designed to do so. Discover is perfectly good at doing what it is supposed to do, which does not include updating an entire system.

With that in mind, let's take a look at how Discover is terrible at automatically updating my system.

Firstly, automatically checking for updates can take up to 10 minutes. If you manually open Discover and tell it to check for updates, it'll be around the same speed as `apt update`, but that defeats the purpose of automatic updates. (At the very least, the progress bar when manually checking for updates has been [improved](https://invent.kde.org/plasma/discover/-/merge_requests/646))

When it comes to actually updating software, the only programs that get updated are "Applications" (i.e. Krita), with no luck when it comes to "System Software" (i.e. Git). Basically, if it doesn't have an icon in Discover, it won't get automatically updated.

Offline updates seem to be entirely broken, as enabling them results in no updates being applied at all.

Even when disabling all automatic update functionality, Discover Notifier remained in my System Tray regardless. This has now been [fixed](https://bugs.kde.org/show_bug.cgi?id=413053), and while it remains to be seen if I will be using automatic updates in the near future, it will be good to only have the notifier show up when relevant.

Discover has undergone a significant overhaul in Plasma 6, and I am interested to find out whether this leads to improvements in automatic updates.

### Defaults

When I switched from Windows to KDE, there were three main differences that I found hard to adjust to:
- Single-click to open
- Significantly different task switcher
- Opposite touchpad scroll direction

These (obviously) didn't stop me from using KDE, but the first thing I did was to change them (apart from scroll direction), and it wasn't an amazing first impression. Changing them back to what I was used to wasn't particularly hard, but I was already quite experienced with Linux at that point (~4 years of WSL) and determined to switch. Having helped a few people to switch to KDE, I have noticed that these differences often play a part in why they initially dislike Linux.

Which is why I am very excited that the new defaults in Plasma 6 include both [double-click to open](https://invent.kde.org/plasma/plasma-desktop/-/issues/72) and [Thumbnail Grid as the default task switcher](https://invent.kde.org/plasma/plasma-desktop/-/issues/53)! Sure, the touchpad scroll direction is still different, but that is still customizable, and thanks to the variations in touchpad scroll direction across other operating systems, adjusting is relatively easy.

It is important to strike a balance between being similar enough to Windows so that users can easily adjust, but different enough to not be seen as a Windows knockoff, but I see these changes as two steps in the right direction! (Even if [Windows is copying KDE](https://tech.lgbt/@tendstofortytwo/110656263284908250))

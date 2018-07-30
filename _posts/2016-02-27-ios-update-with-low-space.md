---
layout: "post"
title: "IOS update with low space"
categories: blog
tags: ['life']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

I have got an IPad4 and with IOS 6 installed. I would like to install some new apps which need IOS 7 or higher, which push me to do the upgrade.

In the meanwhile, I have an old laptop. Once I upgraded it with a new SSD, which has only 240GB capacity. I installed dual-boot for Win8 and Ubuntu. Since I always play under ubuntu, I assign low space to windows, less to C volume. Which results into the issue mentioned below.

In the beginning, I was going directly upgrade via IPad itself. But it raises error, keeping telling me I don't have enough space. The fact is, before I uninstalling my games I have more than 5GB free space. I also tried to reboot IPad but do no help though...


Reboot my laptop and enter windows using my _user account_ (acutally I have another _admin account_). I installed itunes, connect IPad to laptop, follow the steps, click on "next". Then error occurs, which telling me I have low space on this computer. I check and found really no space in C volume. The turth is I have installed itunes to D volume and also set "itunes media" path to D volume in preference setting. However, itunes update system still somehow using another path. 

What is happenning is something like:

* Itnues will first download the firmware to: `"%appdata%\Apple Computer\iTunes\ipad software updates"`
* The `appdata` environ variable is setting per user in this case, I'm _user account_
* After downloading, itunes will extract the firmware to some temp directory. At this point, no free space on my laptop.
* If nothing bad haapen, itunes will copy the extracted firmware to IPad and it will handle all the process left.

The solution is:

* Download  an application named `junction`, which seems to allow user to create some link file.
* Run `cmd` as _admin account_, run:
    * `> junction -d <path>`: to delete the existing link
    * `> junction <link> <target>`: to create a link to some directory else where. (__NOTE__: Don't use `%appdata%` directory since you are now _admin account_)
* Now, after rebooting itunes I can download the firmeware to volume D. But the remaning issue is itunes will extract firmware to somewhere in volume C. I am not trying to figure this out since it will actually not stop the process and will be deleted/moved afterwards.

Finally, I get IPad working and the Apps installed. Woo.. It takes me some time because I wasn't asking "google.com" at the very beginning but asking "baidu.comn" instead... Orz

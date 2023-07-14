---
title: "Mac Broken App"
date: 2020-10-14T16:12:55+08:00
draft: false
tags: ["OS X"]
categories: ["system"]
---

# Allow apps downloaded from anywhere
Fire up a terminal, type the command:
```
sudo spctl  --master-disable
Password: 
```
Noted that your password will not be shown. After you run the command, goto `System Preferences -> Security and Privacy -> General`  
you will see that there is a **Anywhere** option in `Allow apps downloaded from` section

# Modify Extended file attributes
If allowing app downloaded from everywhere doesn't work properly, you could try the following command in terminal:

```
xattr -r -d com.apple.quarantine /path/to/file
```
where:
```
--help  -h  -- display help information
-c          -- remove all attributes
-d          -- remove specified attribute
-l          -- display in long format
-p          -- display value of specified attribute
-r          -- act recursively
-s          -- act on symbolic links
-v          -- always display file name
-w          -- set value of specified attribute
-x          -- use hexademical format for value input and output
```
Noted that you need to drag the application to a **writable directory** first.

For those of you who are interested in the output of this command, see this post on [stackoverflow](https://stackoverflow.com/questions/46198557/understanding-output-of-xattr-p-com-apple-quarantine)
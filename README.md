# aUTo_scheduler
A tool to automate moving my UT course schedule to Google Calendar.

Dependencies: selenium, requests, bs4, time, os, pyautogui, platform
Must download webdriver for your desired browser. Chrome and Firefox are linked below:

  Chrome chromedriver: https://chromedriver.chromium.org/downloads
  Firefox geckodriver: https://github.com/mozilla/geckodriver/releases

After downloading, either add the driver to your PATH environment variable or update line 18 to take the driver path as an argument.

Then, paste the path to your desired browser into the pathToBrowser variable. This is used to open the browser from the command prompt since, for security purposes, Google does not allow browser automation scripts to log in to Gmail accounts. So instead of browser automation, this program uses GUI automation to control the keyboard directly.

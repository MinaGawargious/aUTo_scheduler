from selenium import webdriver
import requests
from bs4 import BeautifulSoup
from time import sleep
import os
import pyautogui
import platform

# Comments surrounded with "**********" indicate lines the user must update for his or her particular system. They are on lines 11 and 18.

pathToBrowser = "" # **********Fill this in with absolute or relative path to your desired browser. If you are not using Google Chrome, you will need to change line 18 to use your desired browser.**********

operatingSystem = platform.system()  # "Darwin" for MacOS, "Linux" for Linux, "Windows" for Windows
control = "command" if operatingSystem == "Darwin" else "ctrl" # Mac uses command. Everything else uses ctrl. May not work on Vitual Machines.

#-----Go to the page with the student's schedule-----
# Change webdriver.Chrome() to your desired browser (like webdriver.Firefox()) if you are not using Google Chrome.
browser = webdriver.Chrome() # ********** Add chromedriver to PATH environemt variable or put chromedriver path as argument (browser = webdriver.Chrome() becomes browser = webdriver.Chrome("path/to/chromedriver")).**********
browser.implicitly_wait(2) # Wait 2 seconds so new page has some time to load.
browser.get("https://utdirect.utexas.edu/registration/classlist.WBX")

# -----Wait for user to log in.-----
while("login" in browser.current_url.lower()):
   pass

# -----Get which semester and year this is for.-----
title = browser.find_element_by_id("pgTitle").text.split()
semesterString = title[2]
year = int(title[3][2:])

# ----- Go to UT's calendar.-----
semesterString = str(year if semesterString == "Fall" else year-1) + "-" + str(year+1 if semesterString == "Fall" else year)
longSession = requests.get("https://registrar.utexas.edu/calendars/" + semesterString)
try:
    longSession.raise_for_status()
except:
    print("Could not get start and end date for %s." %semesterString)
    longSession.raise_for_status() # Print useful error message, but crash program by rethrowing exception.

# -----Get the start and end days for this semester and yearfrom UT's calendar.-----
longSessionSoup = BeautifulSoup(longSession.text, features = "lxml")
ddElems = longSessionSoup.select("dd")
startDays = [elem for elem in ddElems if "classes begin" in elem.text.lower()] # Only get the elements that correspond to the days the fall and spring semesters begin.
endDays = [elem for elem in ddElems if "last class day" in elem.text.lower() and "last class day in the school of law" not in elem.text.lower()] # The Spring semester has 2 end days: one for the School of Law and one for everyone else. Get the one for everyone else.

semester = 0 if semesterString == "Fall" else 1
startDayOfWeek = startDays[semester].text.split()[0].lower() # This will give us "thursday" from "Thursday Classes Begin". I will use this since, when we go to set the days to repeat the reminder, Google will auto select the start day (so starting an event on Tuesday January 21 will make it repeat every Tuesday). But Tuesday is the first day of school, but not the first day EVERY class meets, so deselect the day Google selected.
startDayOfWeekAbbreviation = startDayOfWeek[:2 if startDayOfWeek == "thursday" else 1] # UT lists class days as abbreviations ("M" instead of "Monday"), but Thursday is the only one with 2 letters to not conflict with Tuesday.
startDay = startDays[semester].findPrevious().text 
endDay = endDays[semester].findPrevious().text

# ----- Get the user's schedule into a useful array -----
userScheduleElems = browser.find_elements_by_tag_name("tr")[1:] # User's schedule is an a table, where each row is a course (including class, recitation, lab...)/ userScheduleElems is an array of all the table rows for all the user's classes. Omit the top row, which just has headers for the table.
userScheduleInfo = [[td.text for td in userScheduleElems[i].find_elements_by_tag_name("td")][2:-1] for i in range(len(userScheduleElems))] # For each course, we have an array of the unique id, course number, title, building, room, days, time, and remarks. We don't care about unique id, course number, or remarks, so omit them. 

# -----Replace building codes with their full names ("EER" -> "Engineering Education and Research Center") so Google Calendar can auto suggest the exact address when filling it in. -----
for course in userScheduleInfo:
    try:
        buildingAcronymsList = course[1].split("\n") # Different buildings for different days are separated by new lines
        buildingNamesList = []
        for buildingAcronym in buildingAcronymsList:
            buildingCodes = requests.get("https://utdirect.utexas.edu/apps/campus/buildings/nlogon/facilities/UTM/%s/" %buildingAcronym)
            buildingCodes.raise_for_status() # Make sure building code request was valid.
            buildingCodesSoup = BeautifulSoup(buildingCodes.text, features = "lxml")
            divContainingName = buildingCodesSoup.select(".text-center")[0]
            buildingName = divContainingName.select("h2")[0].text
            buildingNamesList.append(buildingName[:buildingName.rfind("(")]) # Building names are given like "ENGR EDUCATION AND RESEARCH CENTER (EER - 0223)". I just want "ENGR EDUCATION AND RESEARCH CENTER", so strip off building acronym and number.
        # Each different meeting type (like class, recitation, lab) have a different building (course[1]), room (course[2]), days they meet (course[3]), and times (course[4]). Get them into lists so all the index 0's would, for example, be for class, and index 1 would be for labs... for the same course.
        course[1] = buildingNamesList 
        course[2] = course[2].split("\n")
        course[3] = course[3].split("\n")
        course[4] = course[4].split("\n")
    except:
        print("Failed on " + buildingAcronymsList)
        continue

browser.quit()

# ----- Parse a day format. A string like MWF should be separated into an iterable list ["M", "W", "F"], so when we click the days to repeat on, we can search a list instead of them all clustered together into one string. -----
def parseDayString(dayString):
    dayString = dayString.lower()
    currentIndex = 0
    days = []
    while currentIndex < len(dayString):
        if dayString[currentIndex:currentIndex+2] == "th": # Thursday is the only 2 character day we need to account for. We must account for it first since we don't want to look at one character first and have the 't' in "th" interpreted as Tuesday instead of Thursday.
            days.append(dayString[currentIndex:currentIndex+2])
            currentIndex = currentIndex + 2
        else:
            days.append(dayString[currentIndex])
            currentIndex = currentIndex + 1
    return days

# ----- Open Google Calendar -----.
if os.name == "posix": # Posix uses terminal. 
    os.system("open -n \"%s\" https://calendar.google.com/calendar/r" %pathToBrowser)
else: # Windows uses command prompt.
    os.system("start /d \"%s\" https://calendar.google.com/calendar/r" %pathToBrowser)
sleep(3) # Wait for Google Calendar to load before clicking the create button.            
    

# ----- Create class events -----
for currentClassIndex in range(len(userScheduleInfo)): # Loop through all courses
    currentClassInfo = userScheduleInfo[currentClassIndex]
    currentClassBuildings = currentClassInfo[1] 
    currentClassRooms = currentClassInfo[2]
    currentClassDays = currentClassInfo[3]
    currentClassTimes = currentClassInfo[4]

    for currentSessionIndex in range(len(currentClassBuildings)): # Loop through each meeting type for the class (class, recitation, lab, etc.)
        sleep(1)
        pyautogui.write(["c"]) # "c" is the keyboard shortcut to create a new event.
        sleep(2)
        # -----Type in class info----
        # Type class name.
        pyautogui.write(currentClassInfo[0].title())
        pyautogui.write(["tab"]*2)

        # Type class start day.
        pyautogui.write(startDay + ", 20" + str(year))
        pyautogui.write(["tab"])

        # Type class times.
        startAndEndTimes = currentClassTimes[currentSessionIndex].split("-") # Convert "12:30pm-2:00pm" into an array ["12:30pm", "2:00pm"] to send them to Google Calendar.
        pyautogui.write(startAndEndTimes[0].strip()) # Start time.
        pyautogui.write(["tab"])
        pyautogui.write(startAndEndTimes[1].strip()) # End time.
        pyautogui.write(["tab", "tab", "tab", "tab", "enter", "c", "enter"]) # Go to the "custom recurrence" view to say when this event should repeat.

        # Set repetition days
        sleep(1) # Wait for "Custom recurrence" view to appear.
        pyautogui.write(["tab"]*2)
        possibleClassDays = ["s", "m", "t", "w", "th", "f"] # The order Google Calendar has listed for possible repetition days
        actualClassDays = parseDayString(currentClassDays[currentSessionIndex])
        for day in possibleClassDays:
            if ((day in actualClassDays) ^ (day == startDayOfWeekAbbreviation)): # Same as (day in actualClassDays and day != startDayOfWeekAbbreviation) or (day not in actualClassDays and day == startDayOfWeekAbbreviation): Select the day if we have class that day and it's not already selected, deselect it if Google auto selected this day and we don't want it.
                pyautogui.write(["space"])
            pyautogui.write(["tab"])

        # Set end day to last class day of the semester
        pyautogui.write(["tab", "down", "tab"])
        pyautogui.write(endDay + ", 20" + str(year))
        pyautogui.write(["tab", "tab", "enter"]) 

        # Set location
        sleep(1) # Wait for "Custom recurrence" window to disappear.
        pyautogui.write(["tab"]*7)
        pyautogui.write(currentClassBuildings[currentSessionIndex])
        sleep(0.5) # Wait for Google recommendation
        pyautogui.write(["down"]) # Select first recommendation from Google
        sleep(0.25)
        pyautogui.write(["enter"])
        sleep(0.25)
        pyautogui.write(" Room %s" %currentClassRooms[currentSessionIndex])
        pyautogui.hotkey(control, "s") # Save the event.

#-----Close out browser tab. Program is done. -----
sleep(5)
pyautogui.hotkey(control, "w")

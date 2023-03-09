# Appium Web App Testing

*Going beyond native apps, learn how to use Appium to automate web browsers on mobile devices, just as if you were using Selenium for desktop browsers.*

Let's learn how to use Appium to test something other than a native mobile application. Mobile devices come with their own web browsers, enabling users to benefit from the full array of websites and web applications, and indeed much of the time we spend on our devices is probably in the browser. If you have a web app that needs testing, it goes without saying that it should be tested on mobile browsers, not just desktop browsers

Thankfully, Appium allows for the automation of the standard browsers on both Android and iOS, in other words Chrome and Safari respectively. It provides support for Safari and Chrome through different mechanisms unique to each platform, but the end result is the same: when you automate a mobile browser with Appium, it's just as if you're automating a desktop web browser with a Selenium driver.

The way to tell Appium that you want to automate a web browser instead of a native app is to use the <code>browserName</code> capability that's familiar from Selenium, rather than the <code>app</code> capability we've been using so far to launch native apps. The value of the <code>browserName</code> capability should be either <code>Safari</code> or <code>Chrome</code>. At the moment, automation of other mobile browsers is not supported.

When you start a session like this, you'll have access to all the standard WebDriver commands that you normally wouldn't be able to call from a native session. Here I'm talking about commands like getting the title of a webpage, or navigating to a certain URL. You'll also have access to all the web-based locator strategies, like CSS selector, that just wouldn't make sense in a native session.

Of course, you'll also have access to many of the mobile-related features that Appium provides. You won't be able to find native elements while automating a webpage, but you will be able to do many other things like change the device orientation, launch other apps or activities, and so on.

Let's move on to talk about how this web automation works specifically, with Safari first, and then Chrome.

How does Appium provide the ability to automate webpages within mobile Safari?

Basically, it's through the magic of something called the remote debugger. The debugger is relevant in the context of web automation tools like Puppeteer. Safari has its own version of the remote debugger, which allows external applications to connect on a special port that Safari opens on the host, and to send all kinds of commands through this port, that cause the browser to do various things. Mobile Safari is no exception. It also has a remote debugger port that allows for external connections. When we're talking about the iOS simulator, this port is open on the host machine, which makes it very easy to connect to. When we're talking about Safari on real devices, the port is open on the device's own network, so in order to connect to it we first need to run a little proxy server that can forward traffic from our local machine so Appium can connect to this port.

To make sure this port is open, a special setting needs to be turned on in the phone, as shown here under the advanced section of the Safari settings. This is true for simulators as well as real devices, but it is turned on by default for simulators, so we don't need to do anything special for them.

Here's how Appium supports regular WebDriver commands in this situation: when you send a WebDriver command, Appium first looks at it to see what it is. And if you're talking to mobile Safari, Appium turns that command into the appropriate kind of remote debugger command, and sends that off to the remote debugger port where Safari is listening for commands. This all happens under the hood, so you don't need to know or care that here Appium has translated your command into a different language, so to speak. But it's good to know that this is what happens, especially because it helps to explain some limitations in Appium's Safari support.

Because all we have access to is the remote debugger protocol for Safari, there are a few things that aren't supported very well. For example, when you try to click an element, at the end of the day we turn that into a JavaScript event which gets fired on the appropriate element. This is a bit different than what happens technically when a user touches the glass of the screen. Likewise, we can't use the remote debugger protocol's JavaScript functions to automate alerts, so we've had to think of some clever ways around that limitation. All this to say, Appium's Safari support isn't quite the same thing as what you would get with the official SafariDriver from Apple on Desktop, because that driver is able to hook into the Safari browser itself at a deeper level.

### Practical Example

Let's take s Safari session for a spin. We're going to be running what is essentially a web test. Ideally, we can reuse any web automation script by merely modifying the capabilities and session start code to run on Appium instead of Selenium. For example, we may have already run some web tests on desktop Firefox with <code>driver = webdriver.Firefox()</code>. 

    from selenium import webdriver
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.wait import WebDriverWait
    from selenium.webdriver.support import expected_conditions as EC

    driver = webdriver.Firefox()
    try:
        wait = WebDriverWait(driver, 10)
        driver.get('https://the-internet.herokuapp.com')

        form_auth_link = wait.until(EC.presence_of_element_located(
            (By.LINK_TEXT, 'Form Authentication')))
        form_auth_link.click()

        username = wait.until(EC.presence_of_element_located(
            (By.CSS_SELECTOR, '#username')))
        username.send_keys('tomsmith')
        password = driver.find_element(By.CSS_SELECTOR, '#password')
        password.send_keys('SuperSecretPassword!')

        driver.find_element(By.CSS_SELECTOR, 'button[type=submit]').click()

        wait.until(EC.presence_of_element_located(
            (By.LINK_TEXT, 'Logout'))).click()

        flash = wait.until(EC.presence_of_element_located(
            (By.CSS_SELECTOR, '#flash')))
        assert 'logged out' in flash.text
    finally:
        driver.quit()

We can remove the Firefox line and bring over some native iOS capabilities:

    CUR_DIR = path.dirname(path.abspath(__file__))
    APP = path.join(CUR_DIR, 'TheApp.app.zip')
    APPIUM = 'http://localhost:4723'
    CAPS = {
        'platformName': 'iOS',
        'platformVersion': '13.6',
        'deviceName': 'iPhone 11',
        'automationName': 'XCUITest',
        'app': APP,
    }

    driver = webdriver.Remote(
        command_executor=APPIUM,
        desired_capabilities=CAPS
    )

Since mobile browsers don't require a path to an app file, I delete the <code>CUR_DIR</code> and <code>APP</code> definitions. I also remove the <Code>app</code> capability line, and replace it with the <code>browserName</code> capability, setting it to <code>Safari</code>:

    'browserName': 'Safari',

Now, let's see if our script will run unmodified on the Safari browser on iOS. I've got my Appium server running and my simulator up as well, so I can head on over to my terminal and run the script.

As I watch the test run, I can see that Safari indeed loads, and we are walking through The Internet web app as expected. But an error during my test script:

        Traceback (most recent call last):
          File "web_ios.py", line 37, in <module>
            assert 'logged out' in flash.text
        AssertionError

t is an AssertionError, telling me that the string 'logged out' was not in the <code>flash.text</code> property as I expected. What could be going on here? This is a great opportunity to exercise some debugging skills. As I was watching the test, everything *looked* like it worked OK. I didn't see anything obviously wrong with the app. So my next step is to ask the question, if 'logged out' was not part of the flash element text property, what *is* that text property? So let's adjust the code to print out the property just so we can see what's happening:

        print(flash.text)

Putting a print statement just before the line that's causing the problem is a helpful way to have a sanity check. OK, let's run the script again. It looks exactly the same as before, but let's see what has been printed out.

        You logged into a secure area!
        ×
        Traceback (most recent call last):
          File "web_ios.py", line 39, in <module>
            assert 'logged out' in text
        AssertionError

We get the same error, but immediately prior to it, we see what was printed out. There's a line with an 'x', which we can ignore since that's just the little close button. But the line above that says, "You are logged into a secure area!". So now I know what's going on here. The issue is that Appium is looking for the text of the flash element, but it's so *fast* that it's actually finding the text of the flash element *before* the new text has loaded with the logged out text. So it's a race condition between the app and my test script. I'm expecting the app to win, but my test is winning instead, which is leading to a problem.

What should we do? Well, we need some way to make sure that we don't read the text value of the flash element until the new page has transitioned. We didn't have to worry about this with Selenium, since Selenium drivers know that when you click an element, a new page might load, and they're hooked deep into the browser so they can detect such events. With Appium's remote debugger based support for Safari, we don't have that luxury. Instead, we need to guard this assertion with another type of wait--a wait for the page itself to be what we expect. Let's add another expected condition right before we retrieve the flash element:

        wait.until(EC.url_to_be('https://the-internet.herokuapp.com/login'))

What we're doing here is making sure that the URL is what we expect before we try to find the new element. This is a wait that won't hurt anything in the Selenium case, but will hopefully help us in the Appium case here. So now let's go ahead and remove our print statement, and then let's rerun the script and see if it works.

So this was an excellent example of tracking down a race condition and using an explicit wait to help get rid of it, increasing our test stability all around. And all along the way of learning how Safari works. This is an illustration that in general you can just take Selenium tests and then run them without change on Appium, but you might run into situations where you have an opportunity to increase your test stability for *both* web and mobile by running it on different platforms. Alright, now let's turn our attention to Chrome on Android to figure out how to get *that* working as well.

Appium also supports automation of the Chrome web browser on Android. How does this work, and how is it made possible?

On Android we have something going for us which is pretty awesome, and it's called Chromedriver. You may recall Chromedriver from web testing. It's the piece of software released by Google which acts as a standalone webdriver server for Chrome. Luckily for us, it's designed in such a way that it works with Chrome on Android, not only Chrome on the desktop.

Appium enables Chrome automation, then, simply by finding Chromedriver, running it as a webdriver server, and then secretly passing all your test commands straight on to Chromedriver whenever you're actively automating a webpage.

When this is taking place, we say that Appium is in a "proxy" mode, meaning that it's not doing anything to your command requests or responses. It's just acting as a mere intermediary between your script and Chromedriver. The commands your script sends get forwarded unchanged to Chromedriver, and the responses that Chromedriver sends back get returned straight to your script unmodified as well. It's like for the moment Appium takes a break and says, OK Chromedriver, you handle everything related to web automation!

Because Chromedriver is actually written by the same teams that develop Chrome, it's quite a bit more powerful than the basic remote debugger support we have with Safari. For this reason, the little issue we ran into with Safari would not likely be an issue with Chrome.

All is not roses on the Chrome side of things, however. Chromedriver is both a blessing and a curse! It's amazing that it exists, and that we can use it to get web automation for Android for free. But it is an independent piece of software with its own requirements, and that adds some complexity.

The first thing to know is that each version of Appium, or more specifically the UiAutomator2 driver, comes with a certain version of Chromedriver. Typically this is whatever version was most recent when your version of the UiAutomator2 driver was released. This is great on one hand, since it means you don't have to worry about downloading Chromedriver yourself.

On the other hand, Chromedriver comes with some of its own requirements when it comes to automating Chrome. Each version of Chromedriver can only automate specific versions of Chrome, usually ones that were released at the same time as that version of Chromedriver. If you try to automate a version of Chrome that isn't supported with the Chromedriver that comes with Appium, you'll get an error message that looks like the one shown below:

...

As you can see, this is actually a pretty descriptive error. It tells you that there was no Chromedriver that could automate the version of Chrome which was found on your device, and gives you a link to look at for more information. And if you were to scroll further up in the Appium logs at this point, you would see even more helpful information. Any time Appium is used to automate a webpage, it will give you some information about the versions of Chromedriver which are available, and the minimum version of Chrome they require.

So to fix any problems, you *could* just go and update Chrome to the version specified. But usually, this is too much of a manual intervention, and so Appium actually provides a feature for handling this problem.

Essentially, Appium will happily download the version of Chromedriver for you, which is appropriate for the version of Chrome that Appium detects on the device. And in fact, if you automate lots of different devices with the same Appium server, Appium will download and keep a whole set of Chromedrivers for you, along with a mapping file that tells Appium which Chromedrivers it should use with which versions of Chrome.

Appium downloads Chromedrivers from Google's official repository, but it's always possible that this URL changes or is hijacked by a malicious actor. For that reason, the Appium team acknowledges that there's a potential security risk in allowing download and execution of files from the internet, and therefore the feature isn't turned on by default. It has to be explicitly opted in to by the administrator who's in charge of starting the Appium server.

The way we do this is by including a special flag when starting the Appium server, called <code>--allow-insecure</code>. This flag lets us specify any number of features we want to turn on that are not turned on by default because there's some security consideration to their use. So to turn on this specific Chromedriver autodownload feature, we need to know its feature name. These can be found in the Appium docs, but the specific feature name to use in this case is <code>chromedriver_autodownload</code>. So all we need to do to make things work for our case is simply to start the Appium server with this CLI parameter.

There are two capabilities that are related to this feature, that are worth knowing about as well. When Appium downloads Chromedrivers, by default it just puts them in the same directory where the UiAutomator2 driver is installed. This is fine, but if you update the driver, then all those Chromedrivers you've downloaded will disappear. So if you want them to persist in between driver updates, you can use the <code>chromedriverExecutableDir</code> capability and pass the path to a directory where Appium can put all its Chromedrivers. Obviously the user the Appium server runs as needs write permissions to this directory.

Likewise, there's a capability called <code>chromedriverChromeMappingFile</code> which you can use to set the path of the JSON file Appium creates to store a mapping between Chromedriver binaries and versions of Chrome they support.

Let's put all this to use with an example. I'm going to just keep all the test code from the above iOS script, but of course we need to update the capabilities to target an Android device and the Chrome browser instead of what's here now. To get those, I'll copy the <code>CAPS </code> definition from some existing Android script.

        CAPS = {
            'platformName': 'Android',
            'platformVersion': '13.0',
            'deviceName': 'Android Emulator',
            'automationName': 'UiAutomator2',
            'app': APP,
        }

The only thing left is to change the <code>app</code> capability to <code>browserName</code>, and set the value to <code>Chrome</code>, as we did before for Safari. Now we should be ready to go in terms of the script. But remember that the way the Appium server is currently running, the Chromedriver Autodownload feature won't be turned on. So I'll go over to the terminal where I have the server running, and stop it. Now, I'll restart it, but using the <code>--allow-insecure=chromedriver_autodownload</code> parameter. Now that the server is running with this feature enabled, I can go back and run the script.

After we wait a bit for the usual startup routine, Chrome pops up instead of the native app. And, we see it loading the page and walking through the flow. No errors were produced, so that's a passing test. And this is all you need to do to work with Chrome using Appium. The only complexity has to do with Chromedriver relating to Chrome versions, but you can let Appium deal with that headache for you automatically using the Chromedriver Autodownload feature. If for some reason you can't turn that feature on, then there are lots of other options, including telling Appium to use a different Chromedriver than the one that's installed, or updating Chrome itself. You can see all those options in the Appium documentation about managing Chromedriver.














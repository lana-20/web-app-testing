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








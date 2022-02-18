+++
date = "2016-02-15T17:03:05Z"
title = "Actionable security requirements using BDD"


+++

# Short intro to BDD

Checklists are useful for many security-related tasks. Whether we 're threat modeling, code reviewing or pentesting an application, at some point we all feel an urge to tick off some check boxes. Checklists are also a very compact way to communicate security requirements. However sometimes we take for granted that these requirements are understandable by all involved parties. This assumption may result to dialogues like the following:

> -Hey, you mention here that our app is vulnerable to session fixation.

> -That's correct.

> -OK...What's that?

If complexity is the enemy of security, then **visibility is security's most trusted ally**. When every team member, whether they 're security engineers, developers, testers, understands security specifications through a common vocabulary, chances of a bug slipping into production will get slimmer. **This is where Behavior-Driven Development, or BDD, comes in.**

In simple words, BDD is a process of verifying a product's behaviour using User Stories (for a more detailed, description see [here](http://guide.agilealliance.org/guide/bdd.html). For example, in order to clarify Session Fixation in the previous dialogue, we can use the following BDD scenario:
```
Scenario: Session Fixation
    Given a new browser instance
    And user testuser1 is logged in with password test1234
    When user logs out
    And user testuser1 logs in with password test1234
    Then session tokens should be different
```
What's not to understand, right? Using the [Gherkin](https://cucumber.io/docs/reference#gherkin) language, every scenario can be expressed using the Given, When, Then structure, which the author uses to describe the product's desired behaviour. Each sentence is called a Step (e.g user %s logs in) which is mapped to the actual testing code.

Each scenario belongs to a Feature which is actually a [user story](https://en.wikipedia.org/wiki/User_story). A Feature will usually contain multiple scenarios.

# Using Aloe to prevent burns
Currently, the dominant frameworks for Behavior-Driven Development are [Cucumber](https://cucumber.io/) (Ruby) and [JBehave](http://jbehave.org/) (Java). The idea of utilising BDD specifically for security testing has been around for sometime also, with [BDD-Security](http://www.continuumsecurity.net/bdd-intro.html), [Gauntlt](http://gauntlt.org/) and [mittn](https://github.com/F-Secure/mittn) being the prime examples. Since Python is my weapon of choice, I decided to try out [Aloe](http://aloe.readthedocs.org/en/latest/#), a rather new but promising Python BDD framework originating from [lettuce](http://lettuce.it/intro/overview.html).

Let's start by installing aloe:

```
pip install aloe
```



Create a directory for your project's tests (e.g. mywebapp) and place inside a directory named `features`, which will contain the following files:

  *   `steps.py` - Step definitions for our Features
  *   `session_management.feature` - The Feature we will be running
  *   `__init__.py` - Directories containing Features must contain packages (see [here](http://aloe.readthedocs.org/en/latest/features.html#feature-loading))

`session_management.feature` will define all our requirements for secure session management through multiple scenarios. Since this is an introductory tutorial I will implement only a few. A secure session management User Story can be described as:

```
Feature: Session Management

        As a user
        I want my session to be securely managed
        So that a malicious user cannot impersonate me
```

followed by all relevant scenarios.

# Writing Steps
BDD steps should describe the desired behaviour of your software in a way that everyone can understand, **so try to avoid gory implementation details as much as possible**. Borrowing an [example](http://aloe.readthedocs.org/en/latest/steps.html#writing-good-bdd-steps) from the official documentation, you should prefer this :

`When user testuser1 logs in with password test1234`

over this:
```
When I fill in username with "testuser"
And I fill in password with "test1234"
And I press "Log on"
And I wait for title to contain "Landing Page"
```

Back to our requirement, since we 're testing a web app, a web browser automation framework such as [Selenium](seleniumhq.org) is essential for verifying that the app is behaving in the desired way. Here's the implementation of our steps:

```python
from aloe import world, step, before, after
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC


URL = 'http://myinsecureapp.com'


@step
def compare_tokens(self):
    '''session tokens should be different'''
    try:
        assert world.session_tokens[0] != world.session_tokens[1]
    except IndexError:
        print "Not enough session tokens to compare."


@step
def user_login(self, user, password):
    '''user (\w+) with password (\w+) is logged in'''
    world.browser.get(URL)
    username_field = world.browser.find_element_by_name('Email')
    username_field.send_keys(user)
    pass_field = world.browser.find_element_by_name('Password')
    pass_field.send_keys(password)
    world.browser.find_element_by_id('submit-btn').click()
    try:
        WebDriverWait(world.browser, 30).until(EC.title_contains("Landing Page"))
    except:
        print "Unable to login"

    self.then('the value of the session token is saved')


@step
def user_login_2(self, user, password):
    ''' user (\w+) logs in with password (\w+)'''
    self.behave_as("Given user {} with password {} is logged in".format(user, password))


@step
def user_logout(self):
    '''user logs out'''
    world.browser.execute_script("return LogMeout();")


@step
def new_browser_instance(self):
    '''a new browser instance'''
    world.browser = webdriver.Firefox()


@step
def store_session_token(self):
    '''the value of the session token is saved'''
    for cookie in world.browser.get_cookies():
        if cookie['name'] == 'PHPSESSID':
            world.session_tokens.append(cookie['value'])


@before.all
def init():
    world.session_tokens = []


@after.each_example
def teardown_browser(scenario, outline, steps):
    world.session_tokens = []
    world.browser.delete_all_cookies()
    world.browser.quit()
```

Step code is pretty much self-explanatory. A few noteworthy points:

*   Step sentences can be defined in 3 different [ways](http://aloe.readthedocs.org/en/latest/steps.html#defining-steps) , I chose the function doc.
*   The `@before.all` and `@after.all` decorators are used to described actions that will execute before and after the scenario execution.
*   `world` is an Aloe [object](http://aloe.readthedocs.org/en/latest/hooks.html#world) that can be used to store information related to the test process
*   After each test a new browser instance is created for more deterministic results (it's a little slower though)

Test your app by running `aloe features/session_management.feature`. You can find this and a few other example scenarios [here](https://github.com/zuBux/aloe-security-features).

# Usable output
Since Aloe is a plugin for [nose](https://nose.readthedocs.org/en/latest/), a Python unit testing framework, `aloe` accepts the same flags as [nosetests](https://nose.readthedocs.org/en/latest/genindex.html#Symbols). This means that we can use nose's `--with-xunit` and `--xunit-file=<filename>.xml` to produce XUnit XML output for our tests, which is parseable by the majority of CI/CD systems (e.g Jenkins).

# Final Words
BDD is a way to convert user stories or requirements, in this case security-related, into automated tests. These and similar tests might help you protect your app against a large number of common XSS payloads **but it won't ensure that your sanitization filters are 100% bulletproof**. However I strongly believe that as a process it provides a clean, reusable and automated way to catch lots of low-hanging fruit while educating your team on secure coding practices.

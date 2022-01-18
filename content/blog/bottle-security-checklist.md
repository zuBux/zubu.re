+++
date = "2015-12-18T16:47:16Z"
title = "Bottle Security Checklist"

+++


[Bottle](http://bottlepy.org/docs/stable/index.html) is a micro web-framework for Python, strongly resembling Flask. I was attracted by its even greater simplicity (no dependencies!) and I gave it a try in a few projects (e.x spamcan). Although Bottle's documentation is very helpful, I noticed that it lacks detail in secure best practices.

Since micro-frameworks equal to small learning curves and small learning curves usually result to fast-paced development, I compiled a security checklist, covering the basics, that can be used as a reference when developing Bottle-based applications.
Update, update, update!

No matter how small your framework's codebase is, bugs will occur. Once in a while, they will be [serious](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-3137) too. Always keep an eye for security patches and update to the latest stable release when possible.

# SQL Injection
If your custom SQL queries accept user input you 're going to have a hard time securing every single one of them. Unless you have a good reason not to, use [Bottle-SQLalchemy](https://github.com/iurisilvio/bottle-sqlalchemy).

# Cross-Site Scripting (XSS)
Bottle uses the built-in SimpleTemplate Engine by default but also supports Mako, Cheetah and Jinja2 template engines.

When using SimpleTemplate, variables referenced inside curly brackets have their values automatically HTML-escaped. Escaping can be disabled by starting the expression with an exclamation mark. The following example is provided in the documentation:

```python
    >>>template('Hello {{name}}!', name='<b>World</b>')
    u'Hello &lt;b&gt;World&lt;/b&gt;!'
    >>>template('Hello {{!name}}!', name='<b>World</b>')
    u'Hello <b>World</b>!'
```
**Jinja2** has two [modes](http://jinja.pocoo.org/docs/dev/templates/#html-escaping) of HTML escaping, automatic and manual. You should prefer automatic where all values are escaped unless stated otherwise using the `safe` filter e.x `{{title | safe}}`. Automatic escaping can be enabled using an environmental [variable](http://jinja.pocoo.org/docs/dev/api/#jinja2.Environment) or a template [statement](http://jinja.pocoo.org/docs/dev/templates/#autoescape-extension)

**Mako** template engine uses the h built-in flag to escape HTML characters e.x `{{name | h }}`. More on expression filtering in Mako [here](http://docs.makotemplates.org/en/latest/filtering.html?highlight=escape#expression-filtering)

**Cheetah** provides a [WebSafe](http://www.cheetahtemplate.org/docs/users_guide_html_multipage/output.filter.html) filter for HTML escaping.

# CSRF
Bottle does not provide any built-in mechanism for CSRF protection. The easiest (and probably safest) solution is to use the Bottle Utils package which contains the [bottle-utils-csrf](http://outernet-project.github.io/bottle-utils/) (can be installed seperately). Usage is pretty [simple](http://outernet-project.github.io/bottle-utils/csrf.html).

# File Uploads
Bottle documentation contains a rather insecure [example](http://bottlepy.org/docs/stable/tutorial.html#file-uploads) for filetype validation in upload forms. File type validation should not rely only on file extensions, it should also validate the files' content. This can be easily performed using [python-magic](https://github.com/ahupp/python-magic). A better, more secure example would be:
```python
import magic

...

@app.post('/')
def upload():
    allowed_mimetypes = ['image/png']
    upload = request.files.get('uploadfile')
    filetype = magic.from_buffer(upload.file.read(), mime=True)
    if filetype in allowed_mimetypes:
        save_path = "uploads"
        upload.save(save_path)
        return 'OK'
    else:
        return "Filetype not allowed"
```
# Cookies
Bottle has no built-in mechanism for session management. The official documentation suggests using [beaker](http://beaker.readthedocs.org/en/latest/) or writing your own implementation. Anyway, don't forget to encrypt and mark all of your app's sensitive cookies with HttpOnly and Secure (in case of HTTPS-enabled sites) [flags](http://bottlepy.org/docs/stable/api.html?highlight=cookie#bottle.BaseResponse.set_cookie). **Both flags are disabled by default**. Modified [example](http://bottlepy.org/docs/stable/tutorial.html#cookies) code from documentation (assuming that login uses HTTPS):

```python
@route('/login')
def do_login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_login(username, password):
        response.set_cookie("account", username, secret='some-secret-key',\
                             secure=True, httponly=True)
        return template("<p>Welcome {{name}}! You are now logged in.</p>", name=username)
    else:
        return "<p>Login failed.</p>"
```
# Redirects
At the moment Bottle's `redirect()` method supports absolute URLs. In order to avoid any potential malicious redirects, do not pass user-defined values to this method. This behaviour will change in 0.13, which is still under development, to comply with [RFC7231](https://github.com/bottlepy/bottle/issues/749).

# Deployment
When you use Bottle code in production :

* Do not run Bottle as root
* Disable [debug](http://bottlepy.org/docs/stable/api.html?highlight=debug#bottle.debug) output
* Avoid the [built-in](http://bottlepy.org/docs/stable/tutorial_app.html#server-setup) web server.


I plan to keep this post updated so if you have any comments, suggestions or additions don't hesitate to contact me.

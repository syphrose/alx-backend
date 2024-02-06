# i18n
Introduction to Flask-Babel
As you can probably guess, there is a Flask extension that makes working with translations very easy. The extension is called Flask-Babel and is installed with pip:

(venv) $ pip install flask-babel
As part of this chapter, I'm going to show you how to translate the application into Spanish, as I happen to speak that language. I could also work with translators fluent in other languages and support those as well. To keep track of the list of supported languages, I'm going to add a configuration variable:

config.py: Supported languages list.

class Config(object):
    # ...
    LANGUAGES = ['en', 'es']
I'm using two-letter language codes for this application, but if you need to be more specific, a country code can be added as well. For example, you could use en-US, en-GB and en-CA to support American, British and Canadian English as different languages.

The Babel instance is initialized a locale_selector argument, which must be set to a function that will be invoked for each request. The function can look at the user request and pick the best language translation to use for that request. Here is the initialization of the Flask-Babel extension:

app/__init__.py: Initialize Flask-Babel.

from flask import request
# ...
from flask_babel import Babel

def get_locale():
    return request.accept_languages.best_match(app.config['LANGUAGES'])

app = Flask(__name__)
# ...
babel = Babel(app, locale_selector=get_locale)
# ...
Here I'm using an attribute of Flask's request object called accept_languages. This object provides a high-level interface to work with the Accept-Language header that clients send with a request. This header specifies the client language and locale preferences as a weighted list. The contents of this header can be configured in the browser's preferences page, with the default being usually imported from the language settings in the computer's operating system. Most people don't even know such a setting exists, but this is useful as users can provide a list of preferred languages, each with a weight. In case you are curious, here is an example of a complex Accept-Languages header:

Accept-Language: da, en-gb;q=0.8, en;q=0.7
This says that Danish (da) is the preferred language (with default weight = 1.0), followed by British English (en-GB) with a 0.8 weight, and as a last option generic English (en) with a 0.7 weight.

To select the best language, you need to compare the list of languages requested by the client against the languages the application supports, and using the client provided weights, find the best language. The logic to do this is somewhat complicated, but it is all encapsulated in the best_match() method of request.accept_languages, which takes the list of languages offered by the application as an argument and returns the best choice.

Marking Texts to Translate In Python Source Code
Okay, so now comes the bad news. The normal workflow when making an application available in multiple languages is to mark all the texts that need translations in the source code. After the texts are marked, Flask-Babel will scan all the files and extract those texts into a separate translation file using the gettext tool. Unfortunately this is a tedious task that needs to be done to enable translations.

I'm going to show you a few examples of this marking here, but you can get the complete set of changes from the GitHub repository link for this chapter shown above.

The way texts are marked for translation is by wrapping them in a function call that as a convention is called _(), just an underscore. The simplest cases are those where literal strings appear in the source code. Here is an example flash() statement:

from flask_babel import _
# ...
flash(_('Your post is now live!'))
The idea is that the _() function wraps the text in the base language (English in this case). This function will use the language selected by the get_locale() function to find the correct translation for a given client. The _() function then returns the translated text, which in this case will become the argument to flash().

Unfortunately not all cases are that simple. Consider this other flash() call from the application:

flash(f'User {username} not found.')
This text has a dynamic component that is inserted in the middle of the static text. The _() function has a syntax that supports this type of texts, but it is based on the older string substitution syntax from Python:

flash(_('User %(username)s not found.', username=username))
There is an even harder case to handle. Some string literals are assigned outside a web request, usually when the application is starting up, so at the time these texts are evaluated there is no way to know what language to use. An example of this are the labels associated with form fields. The only solution to handle those texts is to find a way to delay the evaluation of the strings until they are used, which is going to be under an actual request. Flask-Babel provides a lazy evaluation version of _() that is called lazy_gettext():

from flask_babel import lazy_gettext as _l

class LoginForm(FlaskForm):
    username = StringField(_l('Username'), validators=[DataRequired()])
    # ...
Here I'm importing this alternative translation function and renaming it to _l() so that it looks similar to the original _(). This new function wraps the text in a special object that triggers the translation to be performed later, when the string is used inside a request.

The Flask-Login extension flashes a message any time it redirects the user to the login page. This message is in English and comes from the extension itself. To make sure this message also gets translated, I'm going to override the default message and provide my own, wrapper with the _l() function for lazy processing:

login = LoginManager(app)
login.login_view = 'login'
login.login_message = _l('Please log in to access this page.')
Marking Texts to Translate In Templates
In the previous section you've seen how to mark translatable texts in Python source code, but that is only a part of this process, as template files also have text. The _() function is also available in templates, so the process is fairly similar. For example, consider this snippet of HTML from 404.html:

<h1>File Not Found</h1>
The translation enabled version becomes:

<h1>{{ _('File Not Found') }}</h1>
Note that here in addition to wrapping the text with _(), the {{ ... }} needs to be added, to force the _() to be evaluated instead of being considered a literal in the template.

For more complex phrases that have dynamic components, arguments can also be used:

<h1>{{ _('Hi, %(username)s!', username=current_user.username) }}</h1>
There is a particularly tricky case in _post.html that took me a while to figure out:

        {% set user_link %}
            <a href="{{ url_for('user', username=post.author.username) }}">
                {{ post.author.username }}
            </a>
        {% endset %}
        {{ _('%(username)s said %(when)s',
            username=user_link, when=moment(post.timestamp).fromNow()) }}
The problem here is that I wanted the username to be a link that points to the profile page of the user, not just the name, so I had to create an intermediate variable called user_link using the set and endset template directives, and then pass that as an argument to the translation function.

As I mentioned above, you can download a version of the application with all the translatable texts in Python source code and templates marked.

Extracting Text to Translate
Once you have the application with all the _() and _l() in place, you can use the pybabel command to extract them to a .pot file, which stands for portable object template. This is a text file that includes all the texts that were marked as needing translation. The purpose of this file is to serve as a template to create translation files for each language.

The extraction process needs a small configuration file that tells pybabel what files should be scanned for translatable texts. Below you can see the babel.cfg that I created for this application:

babel.cfg: PyBabel configuration file.

[python: app/**.py]
[jinja2: app/templates/**.html]
These lines define the filename patterns for Python and Jinja template files respectively. Flask-Babel will look for any files matching these patterns and scan them for texts that are wrapped for translation.

To extract all the texts to a .pot file, you can use the following command:

(venv) $ pybabel extract -F babel.cfg -k _l -o messages.pot .
The pybabel extract command reads the configuration file given in the -F option, then scans all the code and template files in the directories that match the configured sources, starting from the directory given in the command (the current directory or . in this case). By default, pybabel will look for _() as a text marker, but I have also used the lazy version, which I imported as _l(), so I need to tell the tool to look for those too with the -k _l. The -o option provides the name of the output file.

I should note that the messages.pot file is not a file that needs to be incorporated into the project. This is a file that can be easily regenerated whenever it is needed, simply by running the command above again. So there is no need to commit this file to source control.

Generating a Language Catalog
The next step in the process is to create a translation for each language that will be supported in addition to the base one, which in this case is English. I said I was going to start by adding Spanish (language code es), so this is the command that does that:

(venv) $ pybabel init -i messages.pot -d app/translations -l es
creating catalog app/translations/es/LC_MESSAGES/messages.po based on messages.pot
The pybabel init command takes the messages.pot file as input and writes a new language catalog to the directory given in the -d option for the language specified in the -l option. I'm going to be installing all the translations in the app/translations directory, because that is where Flask-Babel will expect translation files to be by default. The command will create a es subdirectory inside this directory for the Spanish data files. In particular, there will be a new file named app/translations/es/LC_MESSAGES/messages.po, that is where the translations need to be made.

If you want to support other languages, just repeat the above command with each of the language codes you want, so that each language gets its own repository with a messages.po file.

This messages.po file that created in each language repository uses a format that is a standard for language translations, the format used by the gettext utility. Here are a few lines from the beginning of the Spanish messages.po:

# Spanish translations for PROJECT.
# Copyright (C) 2021 ORGANIZATION
# This file is distributed under the same license as the PROJECT project.
# FIRST AUTHOR <EMAIL@ADDRESS>, 2021.
#
msgid ""
msgstr ""
"Project-Id-Version: PROJECT VERSION\n"
"Report-Msgid-Bugs-To: EMAIL@ADDRESS\n"
"POT-Creation-Date: 2021-06-29 23:23-0700\n"
"PO-Revision-Date: 2021-06-29 23:25-0700\n"
"Last-Translator: FULL NAME <EMAIL@ADDRESS>\n"
"Language: es\n"
"Language-Team: es <LL@li.org>\n"
"Plural-Forms: nplurals=2; plural=(n != 1)\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=utf-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Generated-By: Babel 2.5.1\n"

#: app/email.py:21
msgid "[Microblog] Reset Your Password"
msgstr ""

#: app/forms.py:12 app/forms.py:19 app/forms.py:50
msgid "Username"
msgstr ""

#: app/forms.py:13 app/forms.py:21 app/forms.py:43
msgid "Password"
msgstr ""
If you skip the header, you can see that what follows is a list of strings that were extracted from the _() and _l() calls. For each text, you get a reference to the location of the text in your application. Then the msgid line contains the text in the base language, and the msgstr line that follows contains an empty string. Those empty strings need to be edited to have the version of the text in the target language.

There are many translation applications that work with .po files. If you feel comfortable editing the text file, then that's sufficient, but if you are working with a large project it may be recommended to work with a specialized translation editor. The most popular translation application is the open-source poedit, which is available for all major operating systems. If you are familiar with vim, then the po.vim plugin gives some key mappings that make working with these files easier.

Below you can see a portion of the Spanish messages.po after I added the translations:

#: app/email.py:21
msgid "[Microblog] Reset Your Password"
msgstr "[Microblog] Nueva Contraseña"

#: app/forms.py:12 app/forms.py:19 app/forms.py:50
msgid "Username"
msgstr "Nombre de usuario"

#: app/forms.py:13 app/forms.py:21 app/forms.py:43
msgid "Password"
msgstr "Contraseña"
The download package for this chapter also contains this file with all the translations in place, so that you don't have to worry about that part for this application.

The messages.po file is a sort of source file for translations. When you want to start using these translated texts, this file needs to be compiled into a format that is efficient to be used by the application at run-time. To compile all the translations for the application, you can use the pybabel compile command as follows:

(venv) $ pybabel compile -d app/translations
compiling catalog app/translations/es/LC_MESSAGES/messages.po to
app/translations/es/LC_MESSAGES/messages.mo
This operation adds a messages.mo file next to messages.po in each language repository. The .mo file is the file that Flask-Babel will use to load translations for the application.


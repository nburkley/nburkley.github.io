---
layout: post
title: Localize Emails in Rails
category: ruby
tags: ruby rails localization I18n email languages devise
description: Localizing emails in Rails
comments: true
---

Right out of the box, Rails makes setting up your site for different languages very easy. Here's how to setup `I18n` localization for your emails as well.

## Introduction

Now that I'm working in Europe again, I've been getting used to building multi-lingual sites. Of course it's always best practice to your keep your site's static content in language specific files, but when I worked in the US, this rarely happened. Setting this up is quite straight forward, even for emails.

## Setting up the locales

Set up the the default and available locales for the application.

```ruby
# application.rb

# set the default locale to English
config.i18n.default_locale = :en
# if a locale isn't found fall back to this default locale
config.i18n.fallbacks = true
# set the possible locales to English and Brazilian-Portuguese 
config.i18n.available_locales = [:en, :'pt-BR'] 
```

When setting the available locales, we need to add the corresponding I18n `yml` files:

* `config/locale/en.yml`
* `config/locale/pt-BR.yml`

## Adding locale to the user

We'll need to add a `locale` attribute to the user object that can be used to store the user's preferred language - this is what we'll use to determine what language to pick for our email.

```ruby
# 20150722104103_add_locale_to_users.rb
class AddLocaleToUsers < ActiveRecord::Migration
  def change
    add_column :users, :locale, :string, null: false, default: :en
  end
end
```

## Setting the user locale

There's quite a few different ways get the user's locale, we can:

*  use the `HTTP_ACCEPT_LANGUAGE` HTTP header
*  use GeoIP database lookup on their IP address
*  allow the user to set their locale themselves

The best option is generally a mix of the above. Try to detect the users language using the Accept Language HTTP header or GeoIP lookup, and then allow the user to override this, setting their own preferred locale if they wish to change.

The rails guides has some more information here: [Setting the Locale from the Client Supplied Information](http://guides.rubyonrails.org/i18n.html#setting-the-locale-from-the-client-supplied-information), including some good [gems](https://github.com/iain/http_accept_language) for making the task easier.


## Translate your email

Let's say we've got a `UserMailer` to notify a user they've got a new follower.

Here's how our email template looks without any localization support:

```html
# app/views/user_mailer/new_follower.html.erb

<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>Hi <%= @user.name %></h1>
    <p>
       <%= @follower.name %> is now following you!
    </p>
  </body>
</html>

```

Instead of using static strings for our copy, we should pull the strings out and place them in `yml` files for their respective languages.

Here's what our `yml` files will look like:

```yaml
# config/locales/en.yml
en:
  user_mailer:
    new_follower:
      subject: 'You have a new follower'
      greeting: 'Hi %{name},'
      content: '%{name} is now following you!'
```

```yaml
# config/locales/pt-BR.yml
pt-BR:
  user_mailer:
    new_follower:
      subject: 'Você tem um novo seguidor'
      greeting: 'Olá %{name},'
      content: '%{name} está te seguindo!'
```

Then we can swap in those translated strings to the email template. If we name the keys correctly - `user_mailer` (after the `UserMailer`) and `new_follower` (after the mailer method and template name), the translation helper picks up the `greeting` and `content` keys nicely.

```html
# app/views/user_mailer/new_follower.html.erb

<!DOCTYPE html>
<html>
  <head>
    <meta content='text/html; charset=UTF-8' http-equiv='Content-Type' />
  </head>
  <body>
    <h1>
      <%= t('.greeting, name: @user.name) %>
    </h1>
    <p>
      <%= t('.content', name: @follow.name) %>
    </p>
  </body>
</html>

```

## Sending email

No that we've got our email template translated we can send the user an email translated into their preferred language.

We do this using the [I18n.with_locale](http://www.rubydoc.info/gems/activesupport/2.3.17/I18n.with_locale) method. It executes a block of code in a given locale. In this instance, our block of code is sending the user an email and the locale is stored on our `@user` instance.

```ruby
# app/mailers/user_mailer.rb

class UserMailer < ActionMailer::Base

  def new_follower(user, follower)
    @user = user
    @project = follower

    I18n.with_locale(@user.locale) do
      mail(
        to: @user.email,
        subject: I18n.t('user_mailer.new_follower.subject')
      )
    end
  end

end
```

## Sending email with Devise

If you use [devise](https://github.com/plataformatec/devise) for user authentication, then you know devise has it's own emails as well. To translate these you can override the methods that send these user emails.

Here's an example with the `send_confirmation_instructions` method, which sends the user's confirmation email. We override the method, set the locale to the user's locale, and then call the original method in the block using `super`.

```ruby
# app/models/user.rb

  def send_confirmation_instructions
    I18n.with_locale(self.locale) { super }
  end
```

You can find translations in many different locales for all the devise-related copy in the [Devise Wiki](https://github.com/plataformatec/devise/wiki/I18n).

## Conclusion

And there you have it, sending emails to users in their language of choice. While it make take a bit more work putting the I18n framework in place at the start, getting a project properly setup for localization at the beginning is definitely well worth it. As well as the obvious gains of making the site multi-lingual, all your copy lives one place, becoming much easier for non-tech folks to manage.

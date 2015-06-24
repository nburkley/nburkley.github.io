---
layout: post
title: Decouple ActiveRecord callbacks with some Pub/Sub
category: ruby
tags: ruby rails refactoring gems
comments: true
---

In a recent rails project I've been using a gem called [wisper](https://github.com/krisleech/wisper) that provides publish-subscribe capabilities to ruby objects. I've been very happy with how it de-cluttered my callbacks, tidied up my models and moved it all into background jobs.

## Introduction

As your application grows you often find you models growing along with it. I’m a big fan of keeping my controllers skinny and pushing the logic into the model layer. But, over time, the models get bigger and become entangled with logic that is less and less relevant to the object itself. Updates to a model can trigger a chain of callbacks which can slow requests requests down as you wait for other changes to be carried out.

One way to address these issues is to replace some callbacks and use a [publish/subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) pattern instead.

## Getting started

Let’s look an example `Post` model that has two on-create callbacks; one to notify editors a new post was created, and one to generate a feed item to add the new post to an activity feed.

```ruby
class Post < ActiveRecord::Base
  after_commit :notify_editors,     on: :create
  after_commit :generate_feed_item, on: :create

  private
  def notify_editors
    EditorMailer.send_notification(self).deliver_later
  end

  def generate_feed_item
    FeedItem.create(self)
  end

end
```

The `Post` objects should not have to know anything about editors or an activity feed. These callbacks are polluting the model - the editor and activity feed logic does not belong in the Post class. It’s also slowing our request down as we have to wait for the callbacks to complete before the request can return.

## Adding some basic Pub/Sub

Instead of using callbacks, we can use a pub/sub pattern to publish model events. We then listen to these events elsewhere, carrying out actions whatever actions need to be done as a result. This decouples the unrelated logic, pulling it out of the model into a listener where relevant actions can be taken.


```ruby
# app/models/post.rb
class Post < ActiveRecord::Base
  after_commit :publish_notification, on: :create

  private
  def publish_notification
      ActiveSupport::Notifications.instrument('post.created', post: self)
  end
end
```

```ruby
# config/initializers/subscribers.rb
ActiveSupport::Notifications.subscribe 'post.created' do |name, start, finish, id, payload|
  post = payload[:post]
  EditorMailer.send_notification(post).deliver_later
  FeedItem.create(post)
end
```

Using [ActiveSupport Instrumentation](http://edgeguides.rubyonrails.org/active_support_instrumentation.html) we can rewrite the above example. We replace our two callbacks with one, which just publishes a notification to broadcast that the post was created. Then a listener, subscribed to these notifications, looks after sending the editor notifications and creating the feed item.

With this setup, the `Post` model, has no knowledge of editors or feed items and the listener just needs to know that when a new post is created, it send an editor_notification and creates a feed item - a much cleaner approach, adhering much closer to the [single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle).

## Using Wisper

Active Support Instrumentation is built right it Active Support and is therefore already available in your application if you're using Active Support. However, as the name suggests, it is geared towards instrumenting your code rather than decoupling your callbacks.

[Wisper](https://github.com/krisleech/wisper) is an excellent gem by Kris Leech, that creates a very nice DSL for publishing and subscribing to notifications from your ruby models. I use this in conjunction with [wisper-activerecord](https://github.com/krisleech/wisper-activerecord), another gem produced by Kris, to transparently publish life-cycle events from your ActiveRecord models.

Rewriting the above example with wisper, and wisper-active recored, removes most of the boilerplate.

```ruby
# Gemfile
gem 'wisper'
gem 'wisper-activerecord'
```

```ruby
# config/initializers/subscribers.rb
Post.subscribe(PostSubscriber)
```

```ruby
class Post < ActiveRecord::Base
  include Wisper.model
end
```

```ruby
class PostSubscriber
  def self.after_create(post)
    EditorMailer.send_notification(post).deliver_later
    FeedItem.create(post)
  end
end
```

Using wisper, we've cleaned up the code and now have a clear separation of concerns. The `Post` model can go about it's business, publishing when it's created/updated/destroyed, and the `PostSubscriber` can listen the post events it wants. In this case, listening to the `after_create` event. 

## Move it into the background

What's more, is that this setup makes it even easier to move all this logic into the background. [wisper-activejob] (https://github.com/krisleech/wisper-activejob) let's us leverage the power of active-job and carry out the Subscriber actions with sidekiq, delayed_job or whatever tool we're using for background job processing.

The only thing we need to do is add the wisper-activejob gem:

```ruby
# Gemfile
gem 'wisper'
gem 'wisper-activerecord'
# at the time of publishing the version of wiper-activejob in rubygems.org as quite old
gem 'wisper-activejob', github: 'krisleech/wisper-activejob'
```

And change the subscriber to listen asynchronously:

```ruby
# config/initializers/subscribers.rb
Post.subscribe(PostSubscriber, async: true)
```
Now when our request comes in to create a post, the `after_create` notification is published and the request returns. The background processor looks after listening to the notifications and carrying out any actions for events it's listening to.

## Conclusion

Overall I feel we have gained a lot from these changes.

* it trims down our growing models
* helps avoid ugly chain of callbacks
* prevents polluting models with unrelated code
* moves our callbacks into the background, leaving our application free to receive more requests

There are may different ways to implement the pub/sub paradigm in ruby but I found [wisper](https://github.com/krisleech/wisper) does a great job of removing boilerplate and providing a nice clean DSL.

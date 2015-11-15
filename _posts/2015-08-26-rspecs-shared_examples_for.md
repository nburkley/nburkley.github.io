---
layout: post
title: DRY up your specs using RSpec's `shared_examples_for`
category: ruby
tags: ruby rails rspec testing specs dry
description: Using RSpec's `shared_examples_for` to remove duplication from your specs.
comments: true
---

[RSpec](http://rspec.info/) is great for writing tests in `ruby`,  providing a nice DSL for testing expected behaviour. However, as a test suite grows, you may find yourself writing a lot of boiler plate and even duplicating specs.  RSpec's `shared_examples_for` can be a good way to DRY up some of these cases. I couldn't find much documentation on using it, so wrote a short about it.

## Introduction

* 



I generally strive to use TDD/BDD when writing software, and for this RSpec is my library of choice. It's got a nice methodology for writing tests in a logical manner, resulting in easy to follow and easy to read specs:

1. _describe_ what you're doing
2. establish a _context_
3. _test_ some expected behavior

When writing application code, you to keep your code [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) by pulling out common functionality into modules and including the module rather than rewriting the common code. This is something you can do with your specs as well.

## Duplicated specs

Let's say our application has `User` and `Project` models and both of these items can be 'followed'. With the 'follow' logic extracted out into a `Followable` module our models might look something like this:

```ruby
# app/models/concerns/followable.rb
module Followable
  extend ActiveSupport::Concern

  included do
    has_many :followings, as: :followable, dependent: :destroy, class_name: 'Follow'
    has_many :followers, through: :followings, source: :user
  end

  def add_follower(user)
    followings.create(user: user)
  end
end

# app/models/user.rb
class User < ActiveRecord::Base
  include Followable
end

# app/models/project.rb
class Project < ActiveRecord::Base
  include Followable
end
```

Using [Factory Girl](https://github.com/thoughtbot/factory_girl) to set up our test data, our basic specs might look something like this:

```ruby
# spec/models/user_spec.rb
require 'rails_helper'

describe User do
  let(:user) { create(:user) }

  describe '#add_follower' do
    it 'should add a new follower' do
      expect { user.add_follower }.
        to change { user.followers.count }.from(0).to(1)
    end
  end
end

# spec/models/user_spec.rb
describe Project do
  let(:project) { create(:project) }

  describe '#add_follower' do
    it 'should add a new follower' do
      expect { project.add_follower }.
        to change { project.followers.count }.from(0).to(1)
    end
  end
end
```

While our `Followable` module keeps our model code nice and DRY, we still want to test that the 'follow' functionality works properly for both Users and Projects - we end up with duplication in our specs.

## Removing the duplication

Instead, we can pull the duplicated spec out into a reusable block using `shared_examples_for`:

```ruby
# spec/spec_helper.rb
Dir[Rails.root.join('spec/shared_examples/**/*.rb')].each { |f| require f }

# spec/shared_examples/followable.rb
shared_examples_for 'is_followable' do
  let(:resource) { create(described_class.name.underscore) }

  describe '#add_follower' do
    it 'should add a new follower' do
      expect { resource.add_follower }.
        to change { resource.followers.count }.from(0).to(1)
    end
  end
end
```
Then we can call it in our `User` and `Project` specs and call it using `it_behaves_like`.

```ruby
require 'rails_helper'

# spec/models/user_spec.rb
describe User do
  it_behaves_like 'is_followable'
end

# spec/models/project_spec.rb
describe Project do
  it_behaves_like 'is_followable'
end
```
The shared example has access to the `described_class`, which is the class being currently tested. It is passed in form the initial '`describe User do...`' at the start of the spec. From that it's easy to determine the factory name, and create a factory instance on which to perform our tests.

## Conclusion

Our tests are much cleaner and can easily be extended if other resources become followable. The `Followable` module may contain a lot more logic we need to test and if we decide at a later date that another resource is to be followable, it just takes one line of code to add tests for that.

The examples above are using rails `ActiveRecored` models, but this is by no means a rails-specific, any ruby tests using RSpec can make use of `shared_examples_for`.

Hope this helps.

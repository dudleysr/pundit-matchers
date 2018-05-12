# Pundit Matchers

[![Gem Version][version-badge]][rubygems]
[![Build Status][travis-badge]][travis]

A set of RSpec matchers for testing [Pundit](https://github.com/elabs/pundit)
authorisation policies. The matcher syntax was inspired by
[this excellent blog post](
  http://thunderboltlabs.com/blog/2013/03/27/testing-pundit-policies-with-rspec
)
from Thunderbolt Labs.

## Installation

Include `pundit-matchers` in your Rails application's Gemfile, inside the test
group:

```ruby
group :test do
  gem 'pundit-matchers', '~> 1.5.1'
end
```

And then execute the following command:

`bundle`

Pundit Matchers requires that both the
[rspec-rails](https://rubygems.org/gems/rspec-rails) and
[pundit](https://rubygems.org/gems/pundit) gems are also installed.

## Setup

Add the following to the top of your Rails application's `spec/spec_helper.rb`
file:

`require 'pundit/matchers'`

## Configuration

Pundit Matchers relies on your policies having a `user` attribute. If your app
checks against a differently named "user" model (such as `account`) you will
need to set a user alias. To add a user alias, add the following configuration
to your app's `spec/spec_helper.rb` or `spec/rails_helper.rb` file:

```ruby
Pundit::Matchers.configure do |config|
  config.user_alias = :account
end
```

The user alias is configured for the whole app and cannot be customised on a
policy by policy basis.

## Matchers

The following RSpec matchers can now be used in your the Pundit policy spec
files (by convention, saved in the `spec/policies` directory).

### Permit Matchers

* `permit_action(:action_name)` Tests that an action, passed in as a parameter,
  is permitted by the policy.
* `permit_action(:action_name, *arguments)` Tests that an action and any
  optional arguments, passed in as parameters, are permitted by the policy.
* `permit_actions([:action1, :action2])` Tests that an array of actions, passed
  in as a parameter, are permitted by the policy.
* `permit_new_and_create_actions` Tests that both the new and create actions
  are permitted by the policy.
* `permit_edit_and_update_actions` Tests that both the edit and update actions
  are permitted by the policy.
* `permit_mass_assignment_of(:attribute_name)` Tests that mass assignment of the
  attribute, passed in as a parameter, is permitted by the policy.

### Forbid Matchers

* `forbid_action(:action_name)` Tests that an action, passed in as a parameter,
  is not permitted by the policy.
* `forbid_action(:action_name, *arguments)` Tests that an action and any
  optional arguments, passed in as parameters, are not permitted by the policy.
* `forbid_actions([:action1, :action2])` Tests that an array of actions, passed
  in as a parameter, are not permitted by the policy.
* `forbid_new_and_create_actions` Tests that both the new and create actions
  are not permitted by the policy.
* `forbid_edit_and_update_actions` Tests that both the edit and update actions
  are not permitted by the policy.
* `forbid_mass_assignment_of(:attribute_name)` Tests that mass assignment of the
  attribute, passed in as a parameter, is not permitted by the policy.

## A Basic Example of a Policy Spec

The following example shows how to structure a Pundit policy spec (in this
example, the spec would be located in `spec/policies/article_policy_spec.rb`)
which authorises administrators to view and destroy articles, while visitors
are only authorised to access the show action and are forbidden to destroy
articles.

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:article) { Article.create }

  context 'being a visitor' do
    let(:user) { nil }

    it { is_expected.to permit_action(:show) }
    it { is_expected.to forbid_action(:destroy) }
  end

  context 'being an administrator' do
    let(:user) { User.create(administrator: true) }

    it { is_expected.to permit_actions([:show, :destroy]) }
  end
end
```

## A Testing Strategy

Pundit Matchers makes several assumptions about how you're going to structure
your policy spec file. First, you should declare a subject for the spec. The
subject should be a new instance of the policy class that you're testing. For
example:

```ruby
subject { described_class.new(user, article) }
```

The subject will be implicitly referenced inside of `it` blocks throughout the
spec, whenever a permit or forbid matcher is used. The new method of the policy
class should also contain the two objects that will be used to authorise each
action within the policy: the user who is attempting access to the record and
the record which is to be authorised.

Throughout the spec, you can use `let` statements to create objects for the
user/record pair which is being authorised. For example, the following
`permit_action` matcher would test that the user can destroy articles
containing a `user_id` attribute which matches the user's ID:

```ruby
let(:user) { User.create }
let(:article) { Article.create(user_id: user.id) }

it { is_expected.to permit_action(:destroy) }
```

The user and record objects used by the subject policy class will be
reassigned whenever you reassign these using the `let` keyword. This will
typically be done on a per context basis, as you will likely want to check the
outcome of an authorisation attempt using different configurations of
user/record pairs based on application state. These variations should be
organised into separate RSpec context blocks, as in the previous example.

## Testing Multiple Actions

To test multiple actions at once the `permit_actions` and `forbid_actions`
matchers can be used. Both matchers accept an array of actions as a parameter.
In the following example, visitors can only view articles, while administrators
can also create and update articles.

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:article) { Article.new }

  context 'being a visitor' do
    let(:user) { nil }

    it { is_expected.to permit_action(:show) }
    it { is_expected.to forbid_actions([:create, :update]) }
  end

  context 'being an administrator' do
    let(:user) { User.create(administrator: true) }

    it { is_expected.to permit_actions([:show, :create, :update]) }
  end
end
```

## Testing New/Create and Edit/Update Pairs

It common to write separate authorisation policies on a per action basis. A
notable exception to this is in the case of new/create and edit/update action
pairs. Generally speaking, you do not want to allow users to access a 'new'
form unless the user is also authorised to create the record associated with
that form. Similarly, you generally do not want the user to access an 'edit'
form unless that user can also update the associated record.

Pundit Matchers provides four shortcut matchers to account for these common
scenarios:

* `permit_new_and_create_actions`
* `permit_edit_and_update_actions`
* `forbid_new_and_create_actions`
* `forbid_edit_and_update_actions`

The following example tests a policy which grants administrators permission to
create articles, but does not authorise visitors to do the same.

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:article) { Article.new }

  context 'being a visitor' do
    let(:user) { nil }

    it { is_expected.to forbid_new_and_create_actions }
  end

  context 'being an administrator' do
    let(:user) { User.create(administrator: true) }

    it { is_expected.to permit_new_and_create_actions }
  end
end
```

## Testing Actions With Arguments

Sometimes you may have a custom policy action which accepts one or more
arguments. Pundit Matchers allows you to specify a number of optional arguments
to the `permit_action` and `forbid_action` matchers so that actions with
arguments can be tested. For example, you might have a policy with a
`create_comment?` method that takes the comment as an argument, like this:

```ruby
class ArticlePolicy < ApplicationPolicy
  def create_comment?(comment)
    true unless comment.spam
  end
end
```

To test this, we can simply pass the comment to the permit or forbid action
matcher.

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:user) { nil }
  let(:article) { Article.create }

  context 'comment is spam' do
    let(:comment) { Comment.new(spam: true) }

    it { is_expected.to forbid_action(:create_comment, comment) }
  end

  context 'comment is not spam' do
    let(:comment) { Comment.new(spam: false) }

    it { is_expected.to permit_action(:create_comment, comment) }
  end
end
```

## Testing the Mass Assignment of Attributes

For policies that contain a `permitted_attributes` method (to authorise only
particular attributes), Pundit Matchers provides two matchers to test for mass
assignment.

* `permit_mass_assignment_of(:attribute_name)`
* `forbid_mass_assignment_of(:attribute_name)`

Let's modify the earlier example which tests a policy where administrators are
granted permission to create articles, but visitors are not authorised to do so.
In this updated example, visitors *can* create articles but they cannot set the
publish flag.

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:article) { Article.new }

  context 'being a visitor' do
    let(:user) { nil }

    it { is_expected.to permit_new_and_create_actions }
    it { is_expected.to forbid_mass_assignment_of(:publish) }
  end

  context 'being an administrator' do
    let(:user) { User.create(administrator: true) }

    it { is_expected.to permit_new_and_create_actions }
    it { is_expected.to permit_mass_assignment_of(:publish) }
  end
end
```

## Testing the Mass Assignment of Attributes for Particular Actions

Pundit allows you to permit different attributes based on the current action
by adding a `permitted_attributes_for_#{action}` method to your policy.
Pundit Matchers supports testing of these methods via composable matchers.

* `permit_mass_assignment_of(:attribute_name).for_action(:action_name)`
* `forbid_mass_assignment_of(:attribute_name).for_action(:action_name)`

To illustrate this, we'll check for the mass assignment of a slug attribute in
our spec. The policy is expected to allow visitors to set the slug attribute
when creating an article, but not when updating it. Administrators will be
permitted to set the slug when either creating or updating the article.

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:article) { Article.new }

  context 'being a visitor' do
    let(:user) { nil }

    it { is_expected.to permit_actions([:create, :update]) }
    it { is_expected.to forbid_mass_assignment_of(:slug) }
    it { is_expected.to permit_mass_assignment_of(:slug).for_action(:create) }
    it { is_expected.to forbid_mass_assignment_of(:slug).for_action(:update) }
  end

  context 'being an administrator' do
    let(:user) { User.create(administrator: true) }

    it { is_expected.to permit_actions([:create, :update]) }
    it { is_expected.to permit_mass_assignment_of(:slug) }
    it { is_expected.to permit_mass_assignment_of(:slug).for_action(:create) }
    it { is_expected.to permit_mass_assignment_of(:slug).for_action(:update) }
  end
end
```

Warning: Currently, Pundit Matchers does *not* automatically check if the
attribute is permitted by a `permitted_attributes_for_#{action}` method, so even
if you include a `forbid_mass_assignment_of(:attribute)` expectation in the
policy spec, it's entirely possible that the attribute *is* being permitted
through a `permitted_attributes_for_#{action}` method that is tested separately.
For this reason, you should always explicitly test *all* implemented
`permitted_attributes_for_#{action}` methods, as demonstrated in the example.

## Testing Resolved Scopes

Another common scenario is to authorise particular records to be returned
in a collection, based on particular properties of candidate records for that
collection. To test for this you don't need to use any matchers. Instead, you
can test for the inclusion or exclusion of a record in the resolved scope by
using the `let` keyword to create a resolved scope based on the current user
and record objects used by a `Policy::Scope` class.

For example, to test that visitors can only view published articles in
a resolved scope you could write your policy spec as follows:

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:resolved_scope) do
    described_class::Scope.new(user, Article.all).resolve
  end

  context 'being a visitor' do
    let(:user) { nil }

    context 'accessing a published article' do
      let(:article) { Article.create(publish: true) }

      it 'includes article in resolved scope' do
        expect(resolved_scope).to include(article)
      end

      it { is_expected.to permit_action(:show) }
    end

    context 'accessing an unpublished article' do
      let(:article) { Article.create(publish: false) }

      it 'excludes article from resolved scope' do
        expect(resolved_scope).not_to include(article)
      end

      it { is_expected.to forbid_action(:show) }
    end
  end
end
```

The advantage of this approach is that it increases the readability of your
specs. It allows you to place all of your specifications for authorising a
particular context (user and record configuration) inside of a single context
block; attempts to access a particular record via both actions and
scopes are tested in the same part of the spec.

## Putting It All Together

The following example puts all of the techniques discussed so far together in
one policy spec that tests multiple user and record configurations within
different context blocks. Here visitors can view published articles and create
unpublished articles, while administrators have full access to all articles.
Visitors can only set the slug attribute when creating an article.

To avoid deeply nested context trees it is a good idea to split larger policy
specs up into multiple files. Here we divide the policy spec into seperate files
for the visitor and administrator contexts, but you could just as easily split
the files by published status or policy action.

`spec/policies/article_policy/visitor_context_spec.rb`:

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:resolved_scope) do
    described_class::Scope.new(user, Article.all).resolve
  end

  let(:user) { nil }

  context 'visitor creating a new article' do
    let(:article) { Article.new }

    it { is_expected.to permit_new_and_create_actions }
  end

  context 'visitor accessing a published article' do
    let(:article) { Article.create(publish: true) }

    it 'includes article in resolved scope' do
      expect(resolved_scope).to include(article)
    end

    it { is_expected.to permit_action(:show) }
    it { is_expected.to forbid_edit_and_update_actions }
    it { is_expected.to forbid_action(:destroy) }
  end

  context 'visitor accessing an unpublished article' do
    let(:article) { Article.create(publish: false) }

    it 'excludes article from resolved scope' do
      expect(resolved_scope).not_to include(article)
    end

    it { is_expected.to forbid_action(:show) }
    it { is_expected.to forbid_edit_and_update_actions }
    it { is_expected.to forbid_action(:destroy) }
  end

  describe 'permitted attributes for visitor' do
    it { is_expected.to forbid_mass_assignment_of(:publish) }
    it do
      is_expected.to forbid_mass_assignment_of(:publish).for_action(:create)
    end
    it do
      is_expected.to forbid_mass_assignment_of(:publish).for_action(:update)
    end
    it { is_expected.to forbid_mass_assignment_of(:slug) }
    it { is_expected.to permit_mass_assignment_of(:slug).for_action(:create) }
    it { is_expected.to forbid_mass_assignment_of(:slug).for_action(:update) }
  end
end
```

`spec/policies/article_policy/administrator_context_spec.rb`:

```ruby
require 'rails_helper'

describe ArticlePolicy do
  subject { described_class.new(user, article) }

  let(:resolved_scope) do
    described_class::Scope.new(user, Article.all).resolve
  end

  let(:user) { User.create(administrator: true) }

  context 'administrator creating a new article' do
    let(:article) { Article.new }

    it { is_expected.to permit_new_and_create_actions }
  end

  context 'administrator accessing a published article' do
    let(:article) { Article.create(publish: true) }

    it 'includes article in resolved scope' do
      expect(resolved_scope).to include(article)
    end

    it { is_expected.to permit_actions([:show, :destroy]) }
    it { is_expected.to permit_edit_and_update_actions }
  end

  context 'administrator accessing an unpublished article' do
    let(:article) { Article.create(publish: false) }

    it 'includes article in resolved scope' do
      expect(resolved_scope).to include(article)
    end

    it { is_expected.to permit_actions([:show, :destroy]) }
    it { is_expected.to permit_edit_and_update_actions }
  end

  describe 'permitted attributes for administrator' do
    it { is_expected.to permit_mass_assignment_of(:publish) }
    it do
      is_expected.to permit_mass_assignment_of(:publish).for_action(:create)
    end
    it do
      is_expected.to permit_mass_assignment_of(:publish).for_action(:update)
    end
    it { is_expected.to permit_mass_assignment_of(:slug) }
    it { is_expected.to permit_mass_assignment_of(:slug).for_action(:create) }
    it { is_expected.to permit_mass_assignment_of(:slug).for_action(:update) }
  end
end
```

[version-badge]: http://img.shields.io/gem/v/pundit-matchers.svg
[rubygems]: http://rubygems.org/gems/pundit-matchers
[travis-badge]: http://img.shields.io/travis/chrisalley/pundit-matchers/master.svg
[travis]: http://travis-ci.org/chrisalley/pundit-matchers

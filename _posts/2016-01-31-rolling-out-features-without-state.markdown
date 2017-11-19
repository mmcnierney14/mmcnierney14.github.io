---
layout: post
title:  "Rolling Out Features"
date:   2015-01-31
categories: ruby rails
---

Deploying code to production is scary. No matter how many edge cases you've added to specs, your users are virtually guaranteed to find one that you had not anticipated. The only question is how long it will take.

To combat this, all applications that have users should use feature flags. Feature flags allow you to deploy code that powers a new feature without immediately changing any behavior for users. _After_ the deploy, you can turn on the feature flag. Things started to break? No need to rollback all of the changes in the deploy — just turn off the offending feature flag. Then you can take your time to add a spec for the case you didn't anticipate, implement a fix, deploy the fix, and turn the feature back on. All with minimal user-facing impact.

There are many ways to implement feature flags in Rails. You could build a simple `FeatureFlag` model with ActiveRecord and store the state in MySQL. You could store the feature flags in Redis ([rollout][rollout-gh] is a popular gem that helps with this). You could use zookeeper — [the Yeller blog][yeller-blog] has a good post on the benefits and implementation of this approach. But in the end, you're basically writing an `if` statement that will prevent many headaches.

```ruby
if FeatureFlag.get('cool_new_feature')
  # Cool new feature implementation
else
  # Old implementation that's already been battle-tested
end
```

## Percentage Rollouts

Sometimes a simple on or off feature flag isn't enough. What if the feature behaves differently at scale? What if it's a feature that will immediately impact a large number of users? In those cases, we'd like to be able to specify a _percentage_ of users to show the new feature to.

```ruby
FeatureFlag.get('cool_new_feature')
# => 0.2
```

Now our application code is going to need to be a little more sophisticated than an `if` statement. Not by much though. Let's make an assumption that we have a `User` model, and a user is identified by a unique token.

```ruby
User.first.token
# => 'ABCD1234'
User.last.token
# => 'ABCD4321'
```

Then we can take a hash of the user token, mod it by 100, and compare that to the rollout percentage to determine which behavior to show. The algorithm is:

```ruby
def in_cool_new_feature_rollout?(user)
  (hash(user.token) % 100) > FeatureFlag.get('cool_new_feature') * 100
end
```

There's one remaining problem — how do we implement the `hash` function? The kneejerk reaction is to use `String#hash`. That won't give us a consistent rollout, however, because `String#hash` gives different results every time the Ruby process is restarted. (And that's [not a bug][hash-bug].)

Instead, we can use Ruby's [Digest library][digest-docs]. If we use the [hexdigest method][hexdigest], we can convert the hex result to an integer, and plug that right into our function above. The final result:

```ruby
require 'digest/sha1'

def in_cool_new_feature_rollout?(user)
  (hash(user.token) % 100) > FeatureFlag.get('cool_new_feature') * 100
end

def hash(user_token)
  Digest::SHA1.hexdigest(user_token).to_i(16)
end
```

[rollout-gh]: https://github.com/FetLife/rollout
[yeller-blog]: http://yellerapp.com/posts/2014-05-14-zookeeper-feature-flags.html
[hash-bug]: https://bugs.ruby-lang.org/issues/4103
[digest-docs]: http://ruby-doc.org/stdlib-2.1.0/libdoc/digest/rdoc/Digest.html
[hexdigest]: http://ruby-doc.org/stdlib-2.1.0/libdoc/digest/rdoc/Digest/Instance.html#method-i-hexdigest

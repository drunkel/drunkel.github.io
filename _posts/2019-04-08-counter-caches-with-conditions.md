---
layout: post
title:  "Counter caches in Rails, with conditions"
---

Recently I needed to update a counter cache to only count child elements meeting some criteria.
In this case a User has many Reviews, but we only want to count the 'approved' reviews in our 
counter cache that we use for display. [Rails comes with a nice counter cache](https://guides.rubyonrails.org/association_basics.html#options-for-belongs-to-counter-cache)
implementation which is what we have been using thus far.

## The Problem
[The built-in counter cache](https://api.rubyonrails.org/classes/ActiveRecord/CounterCache/ClassMethods.html#method-i-reset_counters)
does not allow for conditions to be set for incrementing and decrementing the
counter - the cache will always maintain the count of child records regardless of the properties of the child records.

In this case - I want to only increment the User 'reviews_count' if the review has been approved by a moderator.

## Solution
1.  Add `after_save` and `after_destroy` hooks to our Review model to call our cache-updating code.
```
class Review
      ...
      after_save :update_counter_cache
      after_destroy :update_counter_cache
      ...
    end
```
2.  Add the callback to be run each time our Review is saved or destroyed:
```
class Review
      private

      def update_counter_cache
        user.update_reviews_count
      end
    end
```
```
class User
  ...
  def update_reviews_count
        reviews_count = reviews.approved.count # Whatever condition you need here.
        self.update_column(:reviews_count, reviews_count)
  end
end
```
*Note:* I purposefully use `update_column` here to skip callbacks on our User model. This may/may not be appropriate for your situation.
3. You might want to add a job that refreshes the counters on a regular basis to to fix any data that may have been modified with SQL directly (skipping
our callbacks we made in (1)) or to call manually when you change the conditions of your counter. Rails has a
[built in function](https://api.rubyonrails.org/classes/ActiveRecord/CounterCache/ClassMethods.html#method-i-reset_counters) for this when using the stock counter_cache method ,
but in our case we will need to do make our own:
```
  namespace :counters do
      task update: :environment do
        User.select(:id).find_each do |user|
          user.update_reviews_count
        end
      end
  end
```
I opted for a rake task that gets called by a cron job once per week, or run manually if I change the condition.

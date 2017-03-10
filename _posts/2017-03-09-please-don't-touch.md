---
layout: post
title:  "touch: true, dependent: :destroy, $!"
date:   2016-02-09 17:15:11 +0000
categories: rails
image:  /preview.jpg
---

A little while ago our team was faced with an issue: certain DELETE requests were taking extremely long to process, to the tune of over 1 minute! This post explores what can happen when you have dependent: :destroy in a `has_many` relationship

For this example I've extracted any domain specific implementation to a hypothetical application which manages a library:

* A library has many books.
* A book belongs to 1 library.
* A book belongs to 1 donor (the person who donated the book)
* When we update a book, we should update the `updated_at` timestamp for the donor.
* When we *destroy* a library, we should *destroy* all of the books in the library.


In this example we have a library with around 2 000 books. One day I decide to delete the library. What happens in the background?

{% highlight sql %}
2.1.4 :019 > Library.find(6).destroy
  Library Load (0.2ms)  SELECT  `libraries`.* FROM `libraries` WHERE `libraries`.`id` = 6 LIMIT 1
   (0.1ms)  BEGIN
  Book Load (1.3ms)  SELECT `books`.* FROM `books` WHERE `books`.`library_id` = 6
  SQL (0.4ms)  DELETE FROM `books` WHERE `books`.`id` = 5001
  Donor Load (0.2ms)  SELECT  `donors`.* FROM `donors` WHERE `donors`.`id` = 1 LIMIT 1
  SQL (0.2ms)  UPDATE `donors` SET `donors`.`updated_at` = '2017-03-10 05:04:22' WHERE `donors`.`id` = 1

# And so on, for each of the 2000 books...
{% endhighlight %}

In total, this would require 4 db calls per book, plus 1 to load the Library, and 1 to delete it - a total of 8002 db calls for our example!

This is because of the dependent: :destroy from library to book - which will load and execute all callbacks for *each associated book individually*.

## Solution

How we ended up fixing this was by reaffirming our hatred of callbacks, and by moving our deletion logic to a service. This way you can perform your business logic without letting Rails load every associated record and do the logic one-by-one. This can be done by switching out `destroy` for `delete_all` on has_many associations, and moving any other callback logic into the service. In this example, it may look something like:

### Models:
{% highlight ruby %}
class Book < ActiveRecord::Base
  belongs_to :library
  belongs_to :donor
end

class Library < ActiveRecord::Base
  has_many :books, dependent: :delete_all
end


class Donor < ActiveRecord::Base
  has_many :books
end
{% endhighlight %}


### Service
{% highlight ruby %}
class LibraryDeletionService
  def destroy_library(library)
    Library.transaction do
      # Process the associations in whatever size batches is appropriate. Default 1000.
      library.books.find_in_batches do |batch|
        donor_ids = batch.map(&:donor_id).uniq
        Donor.where(id: donor_ids).update_all(updated_at: Time.now)
      end
      library.destroy
    end
  end
end

{% endhighlight %}

### SQL when destroying a library now:

{% highlight sql %}
2.1.4 :013 > LibraryDeletionService.new.destroy_library(Library.last)
  Library Load (0.2ms)  SELECT  `libraries`.* FROM `libraries`  ORDER BY `libraries`.`id` DESC LIMIT 1
   (0.1ms)  BEGIN
  Book Load (1.0ms)  SELECT  `books`.* FROM `books` WHERE `books`.`library_id` = 9  ORDER BY `books`.`id` ASC LIMIT 1000
  SQL (0.2ms)  UPDATE `donors` SET `donors`.`updated_at` = '2017-03-10 05:24:19' WHERE `donors`.`id` IN (/SOME BATCH OF DONORS/)
  Book Load (0.2ms)  SELECT  `books`.* FROM `books` WHERE `books`.`library_id` = 9 AND (`books`.`id` > 9000)  ORDER BY `books`.`id` ASC LIMIT 1000
  SQL (1.7ms)  DELETE FROM `books` WHERE `books`.`library_id` = 9
  SQL (0.2ms)  DELETE FROM `libraries` WHERE `libraries`.`id` = 9
   (0.7ms)  COMMIT
{% endhighlight %}

Much better! We're performing our business logic (updating the timestamp on donor) using manageable batches of books. This will cut down the number of db calls significantly, and since we're performing these operations on (what should be) primary keys, it will be pretty fast.

Hopefully this can be useful to someone on their next rails project.

Cheers

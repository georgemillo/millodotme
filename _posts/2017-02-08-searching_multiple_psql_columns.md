---
layout: post
title: Searching By Multiple Columns with ActiveRecord and PostgreSQL
permalink: search-by-multiple-columns-in-active-record
---

My Rails app has a model called `Account`, and I've been tasked with implementing an *account search* feature. Users type their search query into a form and see a list of accounts which match that query. The problem is that exactly what constitutes a 'match' is complicated. Each `Account` has multiple attributes (`email`, `first_name` etc.) which could match the search query, and even some *associated* models (e.g. an account `has_many :phone_numbers`) whose attributes should be searchable too.

The standard ActiveRecord API isn't going to cut it here. Maybe I should have just built this app using [Sequel](http://sequel.jeremyevans.net), but for now I'm stuck with ActiveRecord, so it looks like I'm going to have to get my hands dirty and write some raw SQL.

For the purposes of this article, I'm not going to worry about the front-end interface of my search feature. All I care about for now is creating the function that actually does the searching. Here's the shell of a class called `Account::Search` which I need to finish:

{% highlight ruby %}
class Account::Search
  def self.call(query)
    # TODO return a list of accounts
  end
end
{% endhighlight %}

I'm following the [Trailblazer](http://trailblazer.to) convention of encapsulating this logic within a single-purpose class that responds to `call`. The `call` method takes a string (the search query) as an argument, and it will return a collection of Accounts.

What should go in the body of the `call` method? I have no idea. The requirements are too complicated, so to get started I'm going to implement a simplified version of the feature and see where I can go from there.

Let's start by making Accounts searchable by just one attribute - their
`email`, which is a non-nullable string in the `accounts` table. Search should
be case-insensitive, and partial matches should return results (so for example
if the email address is `george@example.com`, you should be able to find it if
you just search for `'george'`).

This can be easily accomplished using the the PostgreSQL `ILIKE` keyword:

{% highlight ruby %}
def self.call(query)
  Account.where('email ILIKE ?', "%#{query}%")
end
{% endhighlight %}

Simple enough. Note that we have to add a `%` to the beginning and end of the query string to allow partial matches - you can think of `%` in PostgreSQL as being roughly equivalent to `.*` in a normal Ruby regex. Also note that the `ILIKE` keyword is case-insensitive - the case-sensitive equivalent is called `LIKE`.

That was too easy. An Account also has the attributes `first_name` (non-nullable) and `last_name` (nullable), and these should be searchable too.
How can we make our search function return accounts where the query matches *any* of these three attributes, not just `email`?

Normally I'd follow TDD and create some RSpec tests for my as-yet-unfinished `Account::Search` class, but for the sake of brevity I'll just provide some example data here that you can create in the rails console (remember that `last_name` can be `nil` but `email` and `first_name` can't):

{% highlight ruby %}
rob  = Account.create!(first_name: 'Robert', last_name: 'Brown', email: 'example@example.com')
paul = Account.create!(first_name: 'Paul', last_name: 'Robertson', email: 'whatever@example.com')
bob  = Account.create!(first_name: 'Bob', email: 'rob@example.com')
mike = Account.create!(first_name: 'Mike', last_name: 'Smith', email: 'notamatch@example.com')

# e.g. Account::Search.call('rob') should return rob, paul, and bob, but not mike.
{% endhighlight %}

Here's my first attempt:

{% highlight ruby %}
class Account::Search
  def self.call(query)
    Account.where('email ILIKE :query OR first_name ILIKE :query OR last_name ILIKE :query', query: "%#{query}%")
  end
end

Account::Search.call('paul')
# returns Paul Robertson
Account::Search.call('Rob')
# returns Robert Brown, Bob, and Paul Robertson
Account::Search.call('@example.com')
# returns all four people
{% endhighlight %}

This appears to be working... but what if we want to search for someone by their *full name*? Right now I can find Paul Robertson by searching for "Paul" or for "Robertson", but if I search for the full string "Paul Robertson" I get no results. That won't do. Back to the drawing board:

{% highlight ruby %}
def self.call(query)
  Account.where("email || ' ' || first_name || ' ' || last_name ILIKE ?", "%#{query}%")
end
{% endhighlight %}

Note that `||` is the PostgreSQL operator for *concatenation*, and doesn't mean 'or' like it does in Ruby.  So what I'm doing here is, for each account, concatenating its email, first name, and last name into a single string (with some whitespace in between, so e.g. for Rob we get `'example@example.com Robert Brown'`) and comparing the search query (case-insensitiv (case-insensitively)ely) against that full string. Now we can find Paul Robertson if we search for him by his full name.

Not so fast - something's still not right. Look what happens when we search for `"rob"` with our new method:

{% highlight ruby %}
Account::Search.call('rob')
# returns Robert Brown and Paul Robertson
{% endhighlight %}

These search results should also include Bob, since his email address is `rob@example.com`, but he's not showing up. What's going on? I notice that Bob is the only account in my test data that doesn't have a last name; maybe this is what's causing the problem.

To investigate, let's forget about Ruby for a second and look directly into the database via the PostgreSQL console, which you can fire up using `rails dbconsole` or just `psql name_of_database`. Let's see the 'searchable' string that's created for each one of our accounts:

{% highlight sql %}
SELECT first_name, email || ' ' || first_name || ' ' || last_name AS searchable FROM accounts;

 first_name |             searchable              
------------+-------------------------------------
 Robert     | example@example.com Robert Brown
 Paul       | whatever@example.com Paul Robertson
 Bob        | 
 Mike       | notamatch@example.com Mike Smith
{% endhighlight %}

Yerwhat? Why is the 'searchable' column blank for `Bob`?

It [turns out](https://stackoverflow.com/questions/8233746/concatenate-with-null-values-in-sql) that this is a quirk of concatenation in SQL: *anything concatenated with `NULL` returns `NULL`*. I sup suppose it kinda makes sense when you think about it: what does it even *mean* to concatenate a string with the null value? Trying to do it in Ruby (`'string' << nil`) will raise an error. So since Bob's last_name is `NULL`, his entire searchable string evaluates to `NULL` as well, and will match no queries.

Thankfully, with a bit more digging I discovered Postgres's `concat` operator, which treats `NULL`s like empty strings:

{% highlight sql %}
SELECT first_name, concat(email, ' ', first_name, ' ', last_name) AS searchable FROM accounts;
  first_name |             searchable              
------------+-------------------------------------
  Robert     | example@example.com Robert Brown
  Paul       | whatever@example.com Paul Robertson
  Bob        | rob@example.com Bob 
  Mike       | notamatch@example.com Mike Smith
{% endhighlight %}

Awww yeah! In fact, this can be made a bit prettier using a slightly different function called `concat_ws`:

`concat_ws(' ', email, first_name, last_name)`

You can think of this as being equivalent to `[email, first_name, last_name].join(' ')` in Ruby, noting that if `last_name` is `nil` then `join` will treat it like an empty string.

At last, I have a serviceable first approximation of my search function. I'll just make one more little improvement: before I pass the query string to ActiveRecord, I'll call `squish` on it, so that for example searching for `Mike &nbsp; Smith` will have the same results as searching for `Mike Smith`. Et voila:

{% highlight ruby %}
def self.call(query)
  Account.where("concat_ws(' ', email, first_name, last_name) ILIKE ?", "%#{query.squish}%")
end
{% endhighlight %}

There's still much more room for improvement. What if we search for someone with their names in the wrong order, like 'Smith Mike'? That seems like a case we should handle. We also need to test how long this search will take against a large database and if there are any ways we can speed it up - we'll almost certainly want to add some indexes to our `accounts` table to make the search run faster. And of course, in reality my `Account` model is much more complicated than this - there are more columns which have their own considerations, and there are some other models associated with `Account` which should also be considered in our search results.

I'll write about of all that in a future article. Until then, happy searching!

---

*Thanks to [this StackOverflow question and its answers](http://stackoverflow.com/a/22976269/1603071) for getting me on the right track when I was figuring all of this out.*

*The code examples in this article use:*

- *Ruby 2.3.0p0*
- *Rails + ActiveRecord 5.0.0.1*
- *PostgreSQL 9.5.2*
- *pg gem 0.19.0*

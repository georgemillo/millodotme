---
title: Make Capybara Raise, Not Warn, When It Detects Unused Parameters
layout: post
permalink: capybara-unused-parameters
published: true
---

I love Capybara but it has one little snag that keeps tripping me up. Have a look at the following RSpec code and see if you can spot the problem:

{% highlight ruby %}
example 'show user page' do
  user = FactoryGirl.create(:user)

  visit user_path(user)

  expect(page).to have_content user.name
  expect(page).to have_content user.email_address
  expect(page).to have_link 'Edit', edit_user_path(user)
end
{% endhighlight %}

Did you see it?  I'm using the `have_link` matcher incorrectly. The second argument is supposed to be a Hash, meaning the correct line looks like this:

{% highlight ruby %}
expect(page).to have_link 'Edit', href: edit_user_path(user)
{% endhighlight %}

(Note that extra `href:`.)

It's an easy mistake to make, especially when you consider the similarity with Rails's `link_to` helper, which *does* take a String (not a Hash) as its second argument. I *still* occasionally mess up `have_link` in this way, and I've been working with Capybara and RSpec for four years.

The sneaky thing here is that if you forget the `href:`, the matcher won't fail. It'll just ignore whatever URL you passed in, and only test that the link has the correct text, not that it has the correct `href` attribute. Meaning that if you got the `href` attribute wrong, you now have a false positive.

[Capybara does print a warning](https://github.com/teamcapybara/capybara/blob/2.7.1/lib/capybara/queries/selector_query.rb#L22) when you make this mistake, but it's easily missed, especially when the dodgy test is running alongside 1500 other tests in your suite. But since Capybara realises that something's amiss, why can't it just make the test fail? If I've passed a String as the second argument to `have_link`, that's a mistake and I want to know about it immediately, so I can fix it immediately.

To that end, here's a tiny little monkey-patch that will make the problem harder to miss (tested on Capybara 2.7.1). Stick this in `spec/support/raise_on_capybara_unused_parameters.rb`, and make sure the file is `require`d somewhere in your `rails_helper` or `spec_helper`:

{% highlight ruby %}
module Capybara
  module Queries
    class SelectorQuery
      def warn(*messages)
        if messages.first.start_with?('Unused parameters')
          raise messages.first
        else
          super
        end
      end
    end
  end
end
{% endhighlight %}

Now, the next time you mess up `have_link`, you'll realise straight away, because Capybara will raise an error and the test will fail. Fixing it is as easy as adding an `href:`, and now false positive test results are less likely to sneak through the net.

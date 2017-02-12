---
title: One Quick Trick to Make RSpec Play More Nicely
layout: post
permalink: 
published: true
---

When you run the `rspec` command without specifying a particular file or group of files that contains your specs, it assumes that your specs live in the `spec` directory. (You can override this default with the [--default-path](https://www.relishapp.com/rspec/rspec-core/v/3-0/docs/configuration/setting-the-default-spec-path) option, although I'm not sure why you'd want to.)

But that's not quite the ful story - `rspec` doesn't simply run *all files within `spec`*, it runs *all files within `spec` whose names end in `_spec.rb`*. If it doesn't end in `_spec.rb`, RSpec will assume that it doesn't contain any specs - which is a good thing, because your `spec` directory probably contains various things *other* that specs (such as shared examples, factory definitions, or custom RSpec matchers), and if RSpec tries to "run" them as if they were specs, things will break.

That's all fine, but the problem is that eventually - we've all done it - you're going to create a spec file within `spec` and forgot to put `_spec` on the end of the name. You might be creating a model called `Doohickey`, and you put it in `app/models/doohickey.rb`. Then you go to create some specs for your new model and you have a brain fart and forget the "_spec", putting your specs in `spec/models/doohickey.rb`. You might not even notice your mistake, because when you run the file individually with `rspec spec/models/doohickey.rb`, it'll run fine.

The problem is that now when you run your entire suite with `rspec`, you're going to run every test *except the ones in `spec/models/doohickey.rb`*. Meaning that those tests could start failing and you won't even notice, because you don't realise that you're not actually running them.

Well, after making the above mistake one too many times I resolved to not get fooled again, so I stuck the following code at the top of my `spec_helper.rb`:

{% highlight ruby %}
bad_file_names = spec_files.reject do |path|
  path.include?(File.join('spec', 'support')) ||
  path.include?(File.join('spec', 'factories')) ||
  path =~ /_spec.rb\z/ || path =~ /_helper.rb\z/
end

if bad_file_names.any?
  raise "Bad spec file names: #{bad_file_names.join(', ')}"
end
{% endhighlight %}

In other words, before my tests run, I perform a quick check that there are no files in `spec` (excluding `spec/support` and `spec/factories`; you'll need to tweak this depending on your own setup) that have names ending in something other than `_spec.rb` or `_helper.rb`, the last one being necessary so that `spec_helper.rb` and `rails_helper.rb` don't trigger the alarm. If this isn't the case, that means I've accidentally created a spec file with an incorrect name, so my `spec_helper` refuses to continue until I've fixed the error.

This minor addition to `spec_helper` lessens the chance of letting bugs and failing tests slip through the net, averting a potentially serious flaw with your test suite.

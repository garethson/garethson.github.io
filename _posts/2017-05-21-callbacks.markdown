---
layout: post
title:  "Callbacks"
date:   2017-05-21 10:18:00
categories: Rails
---

Although ActiveRecord callbacks are a legit way to provide consistent model behavior, I find myself pulling code back out of callbacks all the time (or really wishing I could). This generally has resulted in code that seems immediately easier to understand, debug, and reason about. Many situations that make heavy use of callbacks seem to follow the same pattern:
- Callbacks are great.
- This callback is getting tricky.
- 🔥Everything is on fire.🔥

I'd like to look at how this happens, how I've learned from my own mistakes, and hopefully try to encourage others to evaluate the true complexity cost of callbacks and consider duplicated(!) explicit code as a thing of beauty. :)

Here is an example of creating an order from a web request where we need to send an order notification email after the order is created. This is a pretty common scenario and is generally accepted to be a good candidate for using a callback:
{% highlight ruby %}
class OrdersController < ApplicationController
  def create
    order = Order.new(params)
    if order.save
      redirect_to order_path(order)
    else
      render status: :unprocessable_entity
    end
  end
end
{% endhighlight %}
{% highlight ruby %}
class Order < ActiveRecord::Base
  after_create :send_order_email
  
  def send_order_email
    OrderEmailSender.deliver(self)
  end
end
{% endhighlight %}

This callback does buy us a lot of simplicty and elegance, which is great. Lets see what happens as our app keeps growing. We now allow importing orders, for which we don't want to send emails. This isn't the end of the world, but we do need to inject some context into our callback.

If I *had* to do this, I would do it this way:
{% highlight ruby %}
class CsvOrderImporter
  def import_orders(orders)
    orders.each do |order|
      order.importing = true
      order.save!
    end
  end
end
{% endhighlight %}
{% highlight ruby %}
class Order < ActiveRecord::Base
  attr_acessor :importing
  after_create :send_order_email, unless: :importing
  
  def send_order_email
    OrderEmailSender.deliver(self)
  end
end
{% endhighlight %}
 While I'm very much ok with this solution because it's simple, some developers would say other objects shouldn't randomly be setting state on a model, so I've seen this:
{% highlight ruby %}
class CsvOrderImporter
  def import_orders(orders)
    orders.each do |order|
      order.with_importing do
        order.save
      end
    end
  end
end
{% endhighlight %}
{% highlight ruby %}
class Order < ActiveRecord::Base
  after_create :send_order_email
  
  def send_order_email
    OrderEmailSender.deliver(self) unless @importing
  end

  def with_importing(&block)
    @importing = true
    yield
    @importing = false
  end
end
{% endhighlight %}

*Sigh...* While it's cool that we're using some pretty neat Ruby language features and are adhering to a rule about encapsulation that we're pretty sure we should be following *([Law of Demeter maybe?](https://www.google.comhttps://en.wikipedia.org/wiki/Law_of_Demeter)  I actually have no idea.)*, things are now considerably more complicated.

Let's add one last scenario. We need to allow creating multiple orders via our API, but we want this to be an atomic operation so we're using a database transaction:

{% highlight ruby %}
class Api::OrdersController < ApplicationController
  def bulk_create
    orders = build_orders(params)
    Order.transaction do
      orders.each(&:save!)
    end
    render orders.to_json, status: :ok
  rescue Error
    render status: :unprocessable_entity
  end
end
{% endhighlight %}

On the surface this seems fine, but it's not going to work because our `OrderEmailSender` actually enqueues a background job, and those jobs may run before our database transaction is actually committed, meaning the `order_id` that they query will not exist in the database yet. For this reason we need to upate our callback to an `after_commit`. My gut says many people figure this out in production. I know I did.

This brings us to our final version:
{% highlight ruby %}
class OrdersController < ApplicationController
  def create
    order = Order.new(params)
    if order.save
      redirect_to order_path(order)
    else
      render status: :unprocessable_entity
    end
  end
end

class Api::OrdersController < ApplicationController
  def bulk_create
    orders = build_orders(params)
    Order.transaction do
      orders.each(&:save!)
    end
    render orders.to_json, status: :ok
  rescue Error
    render status: :unprocessable_entity
  end
end

class CsvOrderImporter
  def import_orders(orders)
    orders.each do |order|
      order.with_importing do
        order.save
      end
    end
  end
end

class Order < ActiveRecord::Base
  after_commit :send_order_email, on: :create
  
  def send_order_email
    OrderEmailSender.deliver(self) unless @importing
  end

  def with_importing(&block)
    @importing = true
    yield
    @importing = false
  end
end
{% endhighlight %}

Why this situation is not so great anymore:
 - **We've lost all context.**
 When order emails aren't sending and you have no idea why, the first step to solving the issue is finding out when they're actually supposed to be sent. Searching the code base for `send_order_email` only gives you the callback. Trying to search for `order.save` gives you 375 results. On any project of considerable size, if someone were to ask you *"In which interfaces into our platform does this code get run?"* the answer is unfortunately *"I have absolutely no idea."*
 - **Your test suite doesn't commit records.**
 Since your callback is in an `after_commit` and your test suite doesn't commit the data after each test is run and now [gems and more configuration](https://github.com/grosser/test_after_commit) are now required.
 - **Going back to explicit code is terrifying.**
   As your app gets larger, more and more interfaces open up to your data model, many of which you may be unaware of. Unless you're 100% confident that every code path that creates an order has a test for an email being sent, you'll probably miss some, and finding out where you missed them will be painful (ie: will be figured out in production). You now have a section of code that developers are afraid to touch and almost nothing is worse for a code base than that.
  
Let's try this without a callback:
{% highlight ruby %}
class OrdersController < ApplicationController
  def create
    order = Order.new(params)
    if order.save
      order.send_order_email
      redirect_to order_path(order)
    else
      render status: :unprocessable_entity
    end
  end
end

class Api::OrdersController < ApplicationController
  def bulk_create
    orders = build_orders(params)
    Order.transaction do
      orders.each(&:save!)
    end
    orders.each(&:send_order_email)
    render orders.to_json, status: :ok
  rescue Error
    render status: :unprocessable_entity
  end
end

class CsvOrderImporter
  def import_orders(orders)
    orders.each(&:save)
  end
end

class Order < ActiveRecord::Base
  def send_order_email
    OrderEmailSender.deliver(self)
  end
end
{% endhighlight %}

Why this is better?
 - **Searching the code base for `send_order_email` shows us *exactly* when emails get sent.** This seems like a trivial improvement but it's by far the most important one. Code bases are only as good as they make *humans* feel when working with them. The simple procedural calls to `send_order_email` in both places are easy to understand at first glance and handle database transactions properly with very little complexity added. Neither requires jumping in and out of context to understand what's going on.
 - **Importing an order doesn't need to deal with anything related to sending a confirmation email.** On large apps with many developers this simplicty is invaluable. 
 - **There is less code overall.** Although simply having less code doesn't guarantee less complexity, in this case we did end up with much less code and less to reason about.

The argument for callbacks is that they remove the burden on future developers to remember to place explicit code in every single interface for consistent application behavior. While the intention is good, you'll find out pretty quickly that your consistent application behavior is not so consistent after all, especially on largers apps. It's *much* easier to simply add a method call where one was previously missed than to pack a model full of assumptions and magic and try to pull it all apart when your assumptions are inevitably not correct.

Up next ... concerns!


---
layout: post
title: "In Rails reading can change state"
tags: rails ruby
comments: false
---

Reading can change the state of the object in Rails! Thats seems like an outrageous statement, and it kind of is, but it's true. Let's see how that can happen on an  example.

We have `tasks`, `employees` and `assignments` tables:

{% highlight ruby %}
class Task < ActiveRecord::Base
  has_many :assignments
  has_many :employees, :through => :assignments
end

class Employee < ActiveRecord::Base
  has_many :assignments
  has_many :tasks, :through => :assignments

class Assignment < ActiveRecord::Base
  belongs_to :task
  belongs_to :employee
end
{% endhighlight %}

So, a pretty straightforward example.

Let's say we're fetching all tasks for a certain user, something like: `a_user.tasks`. Under the hood, Rails constructs an SQL query similar to:

{% highlight sql %}
SELECT * FROM tasks, assignements
WHERE tasks.id = assignements.task_id
AND assignements.user_id = 8
{% endhighlight %}

The second time we ask for the user's tasks, while processing the same request, Rails, instead of running the above query against the database for the second time, pulls those tasks from the [Association cache](ass_cache). In most cases, it's a handy feature because it speeds up your model layer. But, it can also bite you on the ass.



---
[strong_entities](http://wofford-ecs.org/dataandvisualization/ermodel/material.htm#Figure 12)
[ass_cache]()


rails state

----------

We have `tasks`, `employees` and `assignments` tables. We're not using `has_and_belongs_to_many` because we have some additional columns in the `assignments` table.

Generally speaking, I [avoid]() `has_and_belongs_to_many` as much as I can. I do this because, more often than not, it turns out that you need more than just foreign keys in your join tables, meaning that your join table is actually a [strong entity](strong_entities), and migrating from `has_and_belongs_to_many` to `has_many :through` is a pain.


celluloid

----------- 

One of the first things that happens is that the class gets inheritable class-level properties. Normal Ruby class don't have that feature.

If you're not familiar with what I'm talking about here, read more about it in my blog post about [class-level inheritable properties](http://pltconfusion.dev/2015/04/15/class_level_inheritable_properties).

Since we need to bypass Ruby's regular inter-object messaging, we need a mailbox to store incoming messages. We'll cover the Mailbox class in detail a bit later.

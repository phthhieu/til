# Ruby DelegateClass

* For example we have _User_ model in our application
```ruby
class User < ActiveRecord::Base
end
```
* Now we need to add _trainer_ and _student_ role to _user_. We can initially use Single Table Inheritance pattern. STI is a common pattern in Rails applications to share responsibilities between classes and store all records in the same place.

* Above approach is only correct when _user_ if _user_ is _student_ or _trainer_, but what if _user_ is both. Now we use __delegation__

```ruby
class Trainer < DelegateClass(User)
  def education
  end

  def teaching_courses
  end
end
```

```ruby
class Learner < DelegateClass(User)
  def paid_courses
  end

  def learning_courses
  end
end
```

We define business logic now pretty easily, just think we have a new role for use, it is easy to add, since we are just follow __open to extension but closed for modification__ principle

* Now how we initialize a _Learner_ for eg:
```ruby
user = User.first
learner = Learner.new(user)
learner.paid_courses
# => PMP, SP
learn.teaching_courses
# => Exception
```

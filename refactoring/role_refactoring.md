# Role refactoring

__Before refactor__
```ruby
class Authorizable extend ActiveSupport::Concern
  def has_role?(role)
    case
    when :admin
      admin?
    when :trainer
      admin? || trainer?
    when :learner
      admin? || trainer? || learner?
    end
  end
end
```

__After refactor__
```ruby
class Authorizable extend ActiveSupport::Concern
  ROLES = %w[learner trainer admin]
  def has_role?(role)
    ROLES.index(self.role) >= ROLES.index(role)
  end
end
```

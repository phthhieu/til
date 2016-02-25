# Use and/or as control flow operators

Eg:

```ruby
# Make sense
File.exists? or raise "No config file!"
```

```ruby
# This doesn't make sense
raise "No config file!" unless File.exists?

# This is confusing
File.exists? || raise "No config file!"
```

__Logical operators (&& and ||) operate on values; control flow operators (and and or) operate on side effects.__

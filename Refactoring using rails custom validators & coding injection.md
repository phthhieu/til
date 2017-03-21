# Stage 1
## Requirements

Implement model `Student` with these attribute constraints:
- `id`:                         integer
- `target_language` : string, required, must be in the pre-defined list of languages
- `focuses`               : string, optional, in array format, must be in the pre-defined list of focuses
- `interests`              : string, optional, in array format, must be in the pre-defined list of interests

Notice `focuses`, and `interests` must be normalized before saving (trimmed space, downcase, removed empty value)
## What I implemented

``` ruby
class Student < ApplicationRecord
  ARRAY_ATTRS = %i( focuses interests )
  # LanguageChecker, FocusChecker and InterestChecker are services 
  #   to check a valid is supported or not, they were implemented
  CHECKER_MAP = {
    target_language: DataServices::LanguageChecker,
    focuses:         DataServices::FocusChecker,
    interests:       DataServices::InterestChecker
  }

  validates_presence_of :target_language
  validate :validate_array_attrs
  validate :validate_supported_attrs

  before_save :normalize_attrs

  private
    def validate_array_attrs
      ARRAY_ATTRS.each{ |attr| validate_array_format_of(attr) }
    end

    def validate_array_format_of(attr)
      # Use before_type_cast to validation array type
      # Ref: https://github.com/rails/rails/issues/14716
      attr_value            = self[attr]
      attr_before_type_cast = send("#{attr}_before_type_cast")

      if attr_value.blank? && attr_before_type_cast.present? && !attr_before_type_cast.is_a?(Array)
        errors.add(attr, 'must be in array format')
      end
    end

    def validate_supported_attrs
      CHECKER_MAP.each do |attr, checker_class|
        attr_value = self[attr]
        if attr_value.present? && !checker_class.instance.include?(attr_value)
          errors.add(attr, 'has unsupported values')
        end
      end
    end

    def normalize_attrs
      ARRAY_ATTRS.each{ |attr| normalize(attr) }
    end

    def normalize(attr)
      if changes[attr].present? && self[attr].is_a?(Array)
        self[attr] = self[attr].map{ |entry| entry.strip.downcase if entry.present? }.compact.uniq
      end
    end
end
```
## Explanation
- Using `before_save` callback to normalize the `ARRAY_ATTRS` (interests & focuses)
- Try to make a map `CHECKER_MAP` between the attribute and the service we gonna use to check.
## Technical debts
- The code is complex, not easy to read & understand.
- The principle: `Single Responsibility` wasn't maintained because rails `activerecord` is the right place to declare how model behaves or how data will be maintained the consistency via validation/normalization function. But it's not the right place to define the detail of validation, even I was using private method, the above code still has a bunch of shit.
# Stage 2:
## New requirement coming

A few weeks after the `stage 1`, new features were coming. We would have a `Trainer` model which will be defined with these constraints:
- `id`:                                   integer
- `native_language` :          string, required, must be in the pre-defined list of languages
- `additional_languages`:   string, in array format, must be in the pre-defined list of languages
- `interests`:                        string, in array format, must be in the pre-defined list interests
  And `additional_languages`, and `interests` also need to be normalized like what I did in `stage 1`
## What I thought
- I may copy the logic to new model, it takes like 5 mins to finish, but I will make a lot of duplication. Btw keep in mind, duplication is not only coding but also logic and thinking, when solving a single problem in many places, don't replicate your solution like launching a new KFC store for a new corner with the same menu and recipes.
- Time for refactoring, I remember this: https://github.com/phthhieu/til/issues/5
# Stage 3: Big refactoring
## Under the hood
- It's pretty clear to extract the private implemented methods to something like `helper` class, but how we refactor and beautify the solution btw. I got the idea from this gem https://github.com/mbleigh/acts-as-taggable-on, the work of injecting something always makes me excited. In that gem, if you want some data atrributes work with `tagging` behavior, we gonna implement like:

``` ruby
acts_as_taggable_on :skills, :interests
```
- It's pretty easy and make things beautiful!
- Well, I decided to do normalization process by code injection and validation using rails custom validator.
## What I did on refactoring
### Normalization data plug and play:
- _/models/concerns/array_normalizable.rb_

``` ruby
module ArrayNormalizable extend ActiveSupport::Concern
  included do
    def self.normalize_value_of(*attrs)
      @@normalizable_attrs = attrs

      class_eval do
        before_save :normalize_attrs

        private
          def normalize_attrs
            normalizable_attrs.each{ |attr| normalize_attr(attr) }
          end

          def normalize_attr(attr)
            if changes[attr].present? && self[attr].is_a?(Array)
              self[attr] = self[attr].map{ |entry| entry.strip.downcase if entry.present? }.compact.uniq
            end
          end

          def normalizable_attrs
            self.class.class_variable_get(:@@normalizable_attrs)
          end
      end
    end
  end
end
```
- It 's pretty easy to copy the code to another place, and change something, but the magic in above code is `plug&play` code, I try to replicate the idea from the gem `acts-as-taggable-on` which inspires me. If in the model we define `normalize_value_of` we will capture the input fields and do normalization using rails callback. The reason of using `class_eval` is we are injecting the runtime code to model, since we don't know exactly what are the fields to be normalized.
- One more small step to make above code work, we need to include the concern to `ApplicationRecord` which will be inherited by every model in Rails 5:

``` ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  include ArrayNormalizable
end
```
- If using Rails < 5, you can inject to code by adding these lines in the end of `ArrayNormalizable`. We call this technique `Open class & modify`

``` ruby
class ActiveRecord::Base
  include ArrayNormalizable
end
```
### Write custom validators
- I define 6 new validator classes to cover the idea:

**lib/validators/custom_data_validator.rb**

``` ruby
class CustomDataValidator < ActiveModel::Validator
  def validate(record)
    @record = record
    options[:fields].each{ |field| validate_field(field) }
  end

  private
    def validate_field(field)
      field_value = @record.public_send(field)
      if field_value.is_a? Array
        field_value.each{ |value| break unless valid_field_value?(field, value) }
      else
        valid_field_value?(field, field_value)
      end
    end
    # Override in subclass
    def valid_field_value?(field, value); end
end
```

**lib/validators/array_validator.rb**

``` ruby
class ArrayValidator < CustomDataValidator
  private
    def validate_field(field)
      attr_value            = @record[field]
      attr_before_type_cast = @record.send("#{field}_before_type_cast")

      if attr_value.blank? && attr_before_type_cast.present? && !attr_before_type_cast.is_a?(Array)
        @record.errors.add(field, 'must be in array format')
      end
    end
end
```

**lib/validators/checker_validator.rb**

``` ruby
class CheckerValidator < CustomDataValidator
  private
    def valid_field_value?(field, value)
      if value.present? && !checker.instance.include?(value)
        @record.errors.add(field, "has unsupported values")
        return false
      end
      true
    end

    # Override in subclass
    def checker; end
end
```

**lib/validators/focus_validator.rb**

``` ruby
class FocusValidator < CheckerValidator
  private
    def checker
      DataServices::FocusChecker
    end
end
```

**lib/validators/interest_validator.rb**

``` ruby
class InterestValidator < CheckerValidator
  private
    def checker
      DataServices::InterestChecker
    end
end
```

**lib/validators/language_validator.rb**

``` ruby
class LanguageValidator < CheckerValidator
  private
    def checker
      DataServices::LanguageChecker
    end
end
```
- The idea is simple, I just went through the `fields` arguments to try to perform the validations after that, the root class will be `CustomDataValidator` which will be inherited to represent the detailed validation.
### Integrate with current implementation, ensure unit test is in GREEN
- Now my model will be:

``` ruby
class Student < ApplicationRecord
  validates_presence_of :target_language

  validates_with LanguageValidator, fields: [:target_language]
  validates_with FocusValidator,    fields: [:focuses]
  validates_with InterestValidator, fields: [:interests]
  validates_with ArrayValidator,    fields: [:focuses, :interests]
  normalize_value_of :focuses, :interests
end
```
- It 's beautiful, right? it actually takes me 3 hours after the midnight to complete this, it doesn't make more value for business, but at least I was happy with what I did and I learned something news. :) :)
# Stage 4: Oops! Time for implementing new features, I nearly forgot it :(
- Now I implement the `Trainer` in 1 min

```
class Trainer < ApplicationRecord
  validates_presence_of :native_language

  validates_with LanguageValidator, fields: [:native_language, :additional_languages]
  validates_with InterestValidator, fields: [:interests]
  validates_with ArrayValidator,    fields: [:additional_languages, :interests]
end
```
- I may think it is like the same with `Student` model, but actually it has so many differences btw `Student` and `Trainer`, I just simplify them to make this article understandable.
- Perhaps, someone will say thanks to me for this nightmare work :). Even no one see,  I'm still fine.
# Summary
- Someday if thing or work is boring, I will dive into the code smell and find out my happiness.
- Try to learn new things, even might face to failure https://github.com/phthhieu/til/issues/42
- Pay my debts always https://github.com/phthhieu/til/issues/26
- Refactoring is art

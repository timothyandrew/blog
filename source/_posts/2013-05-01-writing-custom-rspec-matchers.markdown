---
layout: post
title: "Writing Custom RSpec Matchers"
date: 2013-05-01 11:56
comments: true
categories: [ruby, rspec, matcher, tdd, rails]
---

RSpec matchers let you abstract away common assertions in your test code.

For example, we recently had a spec file with a bunch of lines that looked like this:

```ruby
worksheet.rows[0].cells.map(&:value).should include "Foo"
```

Which tests if the excel file we're generating (using [axlsx](https://github.com/randym/axlsx)) includes `Foo` in the header row.  

That isn't very neat. What if we replace it with this?

```ruby
worksheet.should have_header_cell "Foo"
```

That looks a lot better. We can implement this kind of abstraction using custom RSpec matchers.

The matcher for this is as simple as:

```ruby
RSpec::Matchers.define :have_header_cell do |cell_value|
  match do |worksheet|
    worksheet.rows[0].cells.map(&:value).include? cell_value
  end
end
```
RSpec passes in the expected and actual values to these blocks, and our code has to return a boolean representing the result of the assertion.

Now what about assertions that look like this?

```ruby
worksheet.rows[1].cells.map(&:value).should include "Foo"
worksheet.rows[2].cells.map(&:value).should include "Bar"
worksheet.rows[3].cells.map(&:value).should include "Baz"
```

The row that we're checking changes for each assertion. Of course, we *could* create a different matcher for each of these cases, but there's a better way.

```ruby
worksheet.should have_cell("Foo").in_row 1
worksheet.should have_cell("Bar").in_row 2
worksheet.should have_cell("Baz").in_row 3
```

RSpec lets you *chain* custom matchers.

```ruby
RSpec::Matchers.define :have_cell do |expected|
  match do |worksheet|
    worksheet.rows[@index].cells.map(&:value).include? expected
  end

  chain :in_row do |index|
    @index = index
  end
  
  failure_message_for_should do |actual|
    "Expected #{actual} to include #{expected} at row #{@index}."
  end
end
```

We first store the argument passed in to `in_row` as an instance variable, and then access it in the main `have_cell` matcher.  


The example also includes a custom error message handler, which properly formats an error message if the assertion fails.



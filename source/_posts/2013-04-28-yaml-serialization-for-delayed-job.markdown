\\\\\\\\\---
layout: post
title: "YAML Serialization for Delayed Job"
date: 2013-04-28 08:08
comments: true
categories: [ruby, delayed job, yaml, serialization, rails]
---

When we first moved excel generation off to a delayed job on [survey-web](http://github.com/c42/survey-web), we had code that looked like this:

```ruby
responses = Response.where(:foo => bar)
Delayed::Job.enqueue(MyCustomJob.new(responses))
```

And this would bomb with an error like `Can't dump anonymous Module`.  
After some time getting nowhere, we solved it like this:

```ruby
response_ids = Response.where(:foo => bar).map(&:id)
Delayed::Job.enqueue(MyCustomJob.new(response_ids))
```

And in the job:

```ruby
responses = Response.where('id in (?)', response_ids)
```

While refactoring a lot of that code over the last few days, we ran into the same issue. But with one difference. A controller spec was failing, but a test for the job which also passed a bunch of responses into it passed.

We wondered if maybe it was because we were passing a relation into the job instead of an array.

So we tried:

```ruby
responses = Response.where(:foo => bar).all
Delayed::Job.enqueue(MyCustomJob.new(responses))
```

And that worked great.

(The files in question are [here][1] and [here][2]).

[1]: https://github.com/c42/survey-web/blob/master/app/controllers/responses_controller.rb#L16
[2]: https://github.com/c42/survey-web/blob/master/app/models/reports/excel/job.rb
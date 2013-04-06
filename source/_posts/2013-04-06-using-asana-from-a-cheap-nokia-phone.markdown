---
layout: post
title: "Using Asana From a Cheap Nokia Phone"
date: 2013-04-06 23:29
comments: true
categories: [asana, mobile, org]
---

I love [Asana](http://asana.com/). Its made me a lot more organized than I used to be.

I'd love to be able to check my tasks when I'm not at a computer, but my phone looks like this.

![Nokia C2-01](/images/c2-01.jpg)

There's no way the Asana app will load on its Opera Mini browser.  

It does have email, though. And Asana lets me [send tasks](http://asana.com/guide/tags-email/email-basics) in via email.

Unfortunately, there seems to be no way to actually *view* all your tasks from a phone like this.  

**Until now.**

I wrote a small [Sinatra](http://sinatrarb.com/) app that pulls my tasks from Asana and displays them on a simple HTML page. It's very rough around the edges, but works well enough for my needs. You can grab the code and set it up [here](http://github.com/timothyandrew/asana-light).

There is ***no* authentication** yet, so anyone with a link can view your tasks.

Read on for an explanation of how it works.

<!-- more -->

Luckily for us, there's a ruby interface to Asana's REST API in the form of the [asana gem](http://github.com/rbright/asana), which we can use to pull all our tasks from Asana.

First, we need to get the API key for our Asana account for authentication.

- Login to Asana
- Visit this link: [http://app.asana.com/-/account_api](http://app.asana.com/-/account_api)
- Copy the API key.

We use this to set up the gem.

```ruby
configure do
  Asana.configure do |client|
    client.api_key = "YOUR_API_KEY"
  end
end
```

Next, we need to get all your tasks. We get all the tasks assigned to you from each of your workspaces and put them all in one array. 

```ruby
user = Asana::User.me
workspaces = Asana::Workspace.all
tasks = workspaces.reduce([]) do |memo, workspace|
    memo << workspace.tasks(user.id).map(&:name)
    memo
end.flatten
```

We send all this to this ERB template:

```erb
<html><body>
	<ul>
	<% tasks.each do |task| %>
		<li><%= task %></li>
	<% end %>
	</ul>
</body></html>
```

Which renders the tasks in a format any browser can comfortably read.  
And that's pretty much all the code that's necessary.

---    

There are a lot of ways this can be improved, I'm sure. Off the top of my head:

- Group tasks by workspace
- Sort tasks by date
- Add authentication. Anyone who visits the URL shouldn't be able to see all your tasks. 


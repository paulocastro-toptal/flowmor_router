# FlowmorRouter

FlowmorRouter is a Rails::Engine that enables ActiveRecord Models to route themselves in Rails 4.x applications. For example:

```ruby
class Post < RoutableRecord
end

p = Post.create(title: "My First Post")
puts p.name # => "my-first-example"
puts p.path # => "/posts/my-first-example"
```

or in a view where PostCategory has_many :posts

```ruby
- PostCategory.all.each do |category|
  %h3= link_to category.title, category.path
  %ul
    - category.posts.recently_published.each do |post|
      %li= link_to post.title, post.path
```

FlowmorRouter also supports static pages.  All you have to do is create an app/views/static folder and place templates in that folder.  Routes are automatically generated and served by the StaticController.  Currently, this only goes one level deep.  That is, sub-directories of static not yet implemented.

If you're blogging with markdown or other file-based approaches, you'll appreciate how easy it is to reference your posts in those static views with linking:

```haml
=%h1 About
%p Please be sure to read my post, #{link_to "How to Use Flowmor Router", post_how_to_use_flowmor_router_path}
```

Every model instance's route is named after the model and name.

## State of the project

### 0.0.1

Its got enough functionality to work really well for me [(mwlang)](https://github.com/mwlang) in its current form.  Its a simple implementation with relatively few lines of code, adequately test covered.  It works and is used in production on a handful of sites.  You can see it in action on [my personal site](http://codeconnoisseur.org) and [business site](http://cybrains.net).

### Is it For You?

This isn't for everyone.  It does build routes from objects in the database.  It does provide functionality similar to [friendly_id](https://github.com/norman/friendly_id) or simply redefining the id of a model with AR's #to_param. If you run multiple instances of an application, you'll need to take care of syncing when the database is updated.

### To Install

Add to your Rails project Gemfile:

```
gem 'flowmor_router'
```

And then run the `bundle install` command.

## Convention over Configuration

I wanted a *simple* implementation and library to work with, so the convention is the model has a `title` field and a `name` field.  You (or your user) sets the title and the name field gets auto-populated with a routable name.  Hyphens are used instead of underscores because Google Webmaster Guidelines favors hyphens over underscores for SEO.

For example, "FlowmorRouter, the amazing little engine that could" will populate the name field with 'flowmor-router-the-amazing-little-engine-that-could'.  The controller by convention will have the same name as the model's name while the default action will be the #show action on that controller.  If you have a Post model, then its expected that your application has a PostController.  You're expected to provide the controller implementation.  Here's an example:

```ruby
class PostController < ApplicationController
  before_action :set_post, only: [:show]

  private
  
  def set_post
    @post = Post.find(params[:id])
  end
end
```

Note that you can find the record using the params[:id] which will be the actual id of the record because the routes are constructed specific to the ID for the object to be fetched.  This way we can skip the whole #to_param and params[:id].to_i non-sense or doing a more expensive SQL query and indexing on real titles, names, etc.

To make an ActiveRecord model routable, change the inheritance after generating the model from ActiveRecord::Base to RoutableRecord like so:

```ruby
class Post < RoutableRecord
  # ...
end
```

### Conventions Suck, I Really Want to Customize!

Ok, here's how to do it.  To change the field that the route name is derived from:

```ruby
class NewsArticle < RoutableRecord
  set_derived_name_field :caption # changes from :title
  set_name_field  :slug # changes from :name
end
```
To change the controller and action:

```ruby
class PostCategory < RoutableRecord 
  set_controller_action "blog#category"
end
```

To change how the route and route name are constructed (say you have Post that belongs_to PostCategory and need to avoid naming collision should two posts have same title, but belong to different categories):

```ruby
class PostCategory < RoutableRecord
  has_many :posts, foreign_key: "category_id"

  set_controller_action "blog#category"

  def route_model
    "category"
  end
end

class Post < RoutableRecord
  belongs_to :category, class_name: "PostCategory", counter_cache: true

  set_controller_action "blog#show"

  # Assuming you have a Post.create(title: "Some Title", category: PostCategory.create(title: "General"))
  # The names of the post route's name changes from post_some_title to post_general_some_title by
  # appending category name to the route name prefix
  def route_name_prefix
    super + "_#{category_name}"
  end
  
  # as a bonus, automatically "categorize" as "general" when no category assigned.
  def category_name
    category.try(:name) || 'general'
  end
  
  # The route is also changed in this example from /posts/some-title to /general/some-title
  def route
    "/#{category_name}/#{name}"
  end
end
```

If you need to get any fancier than that, then just about everything you need can be found in the app/models/routable_record.rb implementation.

### TODO and Contributing

This is largely an extraction of functionality from multiple Rails projects.  As such, it has the features I needed to fully extract to the engine.  However, some possible enhancements came to mind as I pulled this together:

* if a model belongs_to another model, then use ActiveRecord's Reflections to automatically build richer routes
* scan sub-directories under static to build nested pages that the static_controller can serve.
* instead of routing *all* RoutableRecord's, add "routable" scope that defaults to *all* but can be easily changed by redefining the :routable scope on the descendant model class.

Don't hold your breath for me, though.  Unless [I need 'em for a specific project](http://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it), they won't happen.  If you need it, implement and contribute back with pull request.  I'll take enhancements as long as they're test covered and don't break backwards compatibility.

## Testing

Testing makes use of a dummy Rails app, which can be found under test/dummy folder.  The test scripts for this app is under test/dummy/test and you'll find many of the examples presented above as working examples in this dummy app.

To test for the first time, you'll need to initialize the database with:

```
RAILS_ENV=test rake db:migrate
```

Following this, you can run the test suite with:

```
rake test
```

### LICENSE

This project uses MIT-LICENSE.  Please see the MIT-LICENSE file.
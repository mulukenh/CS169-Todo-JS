# TODO JS
Recall the [Todo App](https://github.com/saasbook/CS169-Todo) from an earlier discussion: a static todo list with checkboxes that can toggle task status. THe purpose of this app demo is to transition a static, page-by-page web application to a dynamic SPA Javascript-run app.
We will not be using any fancy Javascript frameworks such as Knockout or React; those are too complicated for the scope of this demo.

Provided is a slightly updated version of the Todo app. The only major difference from the previous version is the add Todo input. We can now create new Todo objects.
Setup is simple, as before. Simply run:
```bash
bundle install
rake db:migrate
rake db:seed
```

To run the app, type in your console:
```bash
rails s
```

We have removed tests to expedite the process; normally, however, TDD is still optimal development practice.

## Getting Away from ERB

### Setting Up Handlebars
True Javascript does not play very well with ERB. This is because ERB is handled server-side, while Javascript happens client-side. To simplify things, we'll maintain all of our templates on the client side. To do this, we will use a nice Javascript library called [Handlebars](http://handlebarsjs.com/), which helps us render HTML files from Javascript objects.

Let's imagine the two key components we want to replace with Handlebars templates. These should be the creation form and the task list itself. Get rid of the ERB code and insert placeholder divs that will represent our content for now. These will act as containers for our Handlebars code.
Ultimately, our `app/views/todos/index.html.erb` should look very simple:
```html
<h1>Task List</h1>

<div id="todo-form-create-container"></div>

<div id="todo-table-container"></div>
```

Now let's create the Handlebars templates. These should look similar to the ERB code we just replaced, with one tiny catch: we can't use embedded Ruby. The solution is use Handlebars syntax to populate the desired fields.

Create a file `app/assets/javascripts/templates/todos/todo_form_create.hbs`. It should include the following:
```html
<form id="todo-form-create" action="/todo/create" method="post">
  <div class="input-group">
    <input class="form-control" id="todo_description" name="todo[description]" type="text">
    <span class="input-group-btn">
      <button class="btn btn-default" name="button" type="submit">
        New Task
      </button>
    </span>
  </div>
</form>
```
This should look pretty similar to the old ERB code.

Likewise, create the Handlebars table template `app/assets/javascripts/templates/todos/todo_table.hbs`:
```html
<table class="table table-striped table-bordered" style="margin-top:50px;">
  <thead>
    <tr>
      <th>Completed</th>
      <th>Description</th>
    </tr>
  </thead>

  <tbody>
    {{#each this}}
      <form id="todo-form-create" action="/todo/create" method="post">
        <div class="input-group">
          <input class="form-control" id="todo_description" name="todo[description]" type="text">
          <span class="input-group-btn">
            <button class="btn btn-default" name="button" type="submit">
              New Task
            </button>
          </span>
        </div>
      </form>
    {{/each}}
  </tbody>
</table>
```

Other than Ruby vs Handlebars syntax, the only major difference between the ERB and HBS files is that we get rid of Rails' form tag helper. This means we can no longer use `PUT` and nice authenticity token management.
Let's just use a hackish rememdy. First, let's change all our routes in `config/routes.rb` to `POST`, which is supported by HTML.
```ruby
post 'todo/:id/toggle', :to => 'todos#toggle', :as => 'todo_toggle'
post 'todo/create', :to => 'todos#create', :as => 'new_todo'
```

Next (and even more hackish), let's just forgo authenticity tokens. This is probably fine for your dev environment - hopefully nobody tries to hack your silly task list. In `controllers/todos_controller.rb`, let's just add one line:
```ruby
class TodosController < ApplicationController
  skip_before_filter :verify_authenticity_token
```

### Making an AJAX Call

Voilà! The easy part is done. Before we do some real Javascript, let's add a minor but infinitely important feature to the controller.  
In `controllers/todos_controller.rb`, add the following to the `index` action:
```ruby
def index
  @todos = Todo.all

  respond_to do |format|
    format.html
    format.json { render :json => @todos }
  end
end
```
This allows us to respond to JSON requests, and give the corresponding `@todos` object in a JSON-friendly format. Let's see how we actually consume this information.

Create a file called `app/assets/javascripts/todos.coffee`. In this file, write the following code:
```coffee
$(document).ready( ->
  setupDocument()
)

setupDocument = ->
  $('#todo-form-create-container').html(HandlebarsTemplates['todos/todo_form_create']())
  loadTodos()

loadTodos = ->
  $.ajax({
    url: '/',
    dataType: 'json',
  }).success((data) ->
    $('#todo-table-container').html(HandlebarsTemplates['todos/todo_table'](data))
  )
```
This file is written in [CoffeeScript](http://coffeescript.org/), a preprocessor for Javascript. It makes it look more Pythonic and Ruby-like. Essentially, the `->` defines a function.  
Specifically, this file will load the `#todo-form-create-container` element with the Handlebars template we created for the create form. For the task list, it's a little more complicated. In `loadTodos`, we first make an AJAX call to the `/` path, which you'll recall is the index action for the Todo controller.
Thanks to the controller magic you just wrote, this AJAX call will grab the JSON data and pass it directly to the Handelbars templater. Inspect the `todo_table.hbs` file you created earlier and try to guess what happens. *Hint:* the outermost `this` refers to the context passed in: in this instance, `data`.  
After the templating is complete, we load the `#todo-table-container` element with our rendered HTML.

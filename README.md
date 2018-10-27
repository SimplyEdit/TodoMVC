# TodoMVC with SimplyEdit and SimplyView

TodoMVC is an example application implemented with many different javascript frameworks.
This version is implemented with SimplyEdit and SimplyView. Goto <https://simplyedit.io/> 
and <https://reference.simplyedit.io/> to learn more about SimplyEdit and SimplyView.

## Differences compared to other implementations

I've taken some small liberties compared to the original application, mostly to make it 
more mobile and keyboard friendly:

- Instead of double clicking to edit an item, hover over it and press the green pencil. Or
touch an item once and then press the green pencil. The CSS changes for this have been added
to the `css/base.css` file.
- When done editing, you can also press 'Esc' or 'Cancel', this reverts changes in the item
and hides the editing input.
- You can also drag/drop items using a 'long press'. The selected item will gain a shadow
when it is ready to be dragged.

## Structure

The whole implementation is contained within the `index.html` file. For larger applications
this is probably not a good idea, but SimplyEdit and SimplyView don't force a particular
structure by design. The idea is to grow your application and apply more complex structure
only when needed.

Also the code size is so small, that splitting the code into multiple files will only
obscure the implementation.

## The application shell

Most modern web application follow an application shell structure. This means that there is 
a html file which contains all the basic scaffolding HTML and usually one or more 'container'
elements. Then the javascript renders HTML inside these otherwise empty elements.

SimplyEdit uses a different strategy. All the HTML used is already there. At no point is HTML
generated in the javascript. Instead you change the view data and let SimplyEdit render it.

You specify how to render data directly in the HTML, using the `data-simply-field` and 
`data-simply-list` attributes.

All this can be found inside the `<section class="todoApp">` element. The header contains a
form with an input for new todo items:

```html
    <form id="form" data-simply-command="addTodo">
        <input class="new-todo" type="text" name="newItem" placeholder="What needs to be done?">
    </form>
```

The only thing specific to SimplyView here is ata-simply-command="addTodo". This ties the
form submit to a small function defined later in the javascript app:

```javascript
var todoApp = simply.app({
     commands: {
         addTodo: function(form, values) {
             if (values.newItem.trim()) {
                 todoApp.view.todo.push({
                     item : values.newItem.trim(),
                     completed : 0
                 });
             }
             form.newItem.value = '';
         },
         ...

```

Whenever the new item form gets submitted, by pressing enter, the `addTodo` command is called.
A command defined on a form will automatically gather all the forms values in an object and 
send it along as the second parameter to the command.

## Databinding

All you need to do now is to get the `newItem` from these values, trim it to remove white space
and push a new todo item on the list in the view area: `todoApp.view.todo`. This works, because
the body section of the app is defined as follows:

```html
<ul class="todo-list" data-simply-list="todo">
    <template>
        <li>
            <input type="checkbox" class="toggle" data-simply-field="completed" value="1">
            <div class="view">
                <label data-simply-field="item">Item</label>
                <button class="edit" data-simply-command="editTodo"></button>
                <button class="destroy" data-simply-command="delTodo"></button>
            </div>
            <input class="edit" data-simply-field="item" type="text" data-simply-command="showTodo">
        </li>
    </template>
</ul>
```

In the `<ul>` is an attribute `data-simply-list="todo"`. This means SimplyEdit knows that there
should be an array in `todoApp.view.todo`, and it creates a new empty array there if you don't tell it
otherwise.

The `<template>` element inside the list tells SimplyEdit how to render each entry in the `todo` list.
Inside the template, there are three tags with a `data-simply-field` attribute, one for `completed` and
two for `item`. This means that SimplyEdit knows that an entry in this list should be an object with
at least two properties: completed and item.

When you push a new entry into the todo array inside the addTodo command, SimplyEdit uses the template
as defined in the html and adds that to the dom as a new child of the `<ul>`. The elements with a
`data-simply-field` attribute automatically get filled with the contents of the specified properties.

This approach to rendering data is called databinding. You can just change your view data and SimplyEdit
will notice and update the DOM to match the data. But you can go the other way as well. Change the DOM
and your view data will automatically update as well.

## Two-way databinding

If you look closely in the code, you will see that there is no code that actually handles the `input`
value in each entry (the input with class `edit`). Yet if you press edit and change the inputs value
and then press enter, the todo item label is changed.

This is because both the label and the input have the same `data-simply-field="item"` attribute. This
means that SimplyEdit knows that they should always have the same value. And as you press enter in the
`input` the browser fires the `change` event, SimplyEdit sees that the value of the field has changed
and now automatically updates the value in the view data, e.g. `todoApp.view.todo[0].item`. Because
the view data has now changed, any other DOM element tied to that piece of data is updated as well. In
this case that means that the HTML inside the label is also updated.

But it doesn't stop there. If you change the list as renderen in the DOM, SimplyEdit will notice that
as well. The most obvious code to do this is the `destroy` button. It has a `data-simply-command="delTodo"`
that ties to a command implemented like this:

```javascript
delTodo: function(el, value) {
    el.closest('ul').removeChild( el.closest('li') );
}
```

As you can see, the only thing the `delTodo` command does, is to remove the `<li>` element in which
the destroy button is contained. Yet SimplyEdit also sees this change and understands that it means
that the corresponding entry in the `todo` list has been removed. So it updates the `todoApp.view.todo`
array and removes that entry. There is no need to update the DOM, as this is the only list tied to
that array, but if you added the list twice, the other DOM rendering of the array would automatically
update as well.

One of the nice benefits of this two-way databinding is that you can use any library that manipulates
the DOM and your data gets updated automatically. As an example of this I've added a javascript library
called Slip. This library allows you to drag and drop elements of a list. And all the code I need to
make that work is this:

```javascript
var todoList = document.querySelector('ul.todo-list');

todoList.addEventListener('slip:reorder', function(e){
    e.target.parentNode.insertBefore(e.target, e.detail.insertBefore);
});

todoList.addEventListener('slip:swipe', function(e) {
    e.target.parentNode.removeChild(e.target);
});

new Slip(todoList);
```

Which is almost exactly the demo code from the Slip manual. 

## Rendering fields as templates

A final trick used in this example is in how the item count is rendered:

```html
<span class="todo-count" 
    data-simply-field="count"
    data-simply-content="template"
    data-simply-default-template="other"
>
    <template data-simply-template="0">
        Nothing left to do!
    </template>
    <template data-simply-template="1">
        <strong>1</strong> item left
    </template>
    <template data-simply-template="other">
        <strong data-simply-field="count">n</strong> items left
    </template>
</span>
```

Here the `span.todo-count` element has a `data-simply-field="count"` attribute. Normally this would
just render the `count` variable inside this span, which is just a number. But because of the
`data-simply-content="template"` attribute, SimplyEdit knows it should use the value of this field
as an index for one of the templates defined inside this element. So if `count` is `0` then it uses
the template with the name `0`, which shows `Nothing left to do!`. Similar for a count of 1. But it
gets interesting for larger counts.

There is just one more template with the name 'other', which will never match the value of count.
But the `span.todo-count` element has one more attribute: `data-simply-default-template="other"`.
SimplyEdit will first try to find a template with a name equal to the cound field. If it doesn't
find such a template, it will now look for the default template, named 'other'. This template
can again contain the count field. As this field has no other attributes, SimplyEdit will simply
render the value of the count field inside it.

I hope you enjoyed this example and are ready to build your own web applications using SimplyEdit
and SimplyView. Good luck!

Auke van Slooten
auke@simplyedit.io
Oct. 27, 2018.
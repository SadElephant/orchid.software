---
title: Quick Start for beginners
description: The Quick Start Guide is a basic introduction to the Orchid infrastructure.
---

## Introduction

Admin panels and line applications are an essential part of many web applications. They provide a way for administrators to manage content, users, and other data.

To sample a basic selection of Orchid features, we will build a simple task list we can use to track all the tasks we want to accomplish (the typical “to-do list” example).

At this point you should have already installed [the framework and package](/en/docs/installation) and started the web server. 

> Before we begin, we strongly recommend that you don't copy and paste. Typing each piece of code will help you practice and remember it.


## Prepping The Database

### Database Migrations

First, let's use a migration to define a database table to hold all of our tasks. 

```php
php artisan make:migration create_tasks_table --create=tasks
```

Let's edit this file and add an additional string column for the `name` and boolean column for the `active` of our tasks:


```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('tasks', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->boolean('active')->default(1);
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('tasks');
    }
};
```

To run our migration, we will use the `migrate` Artisan command. 

```bash
php artisan migrate
```


### Eloquent Models


So, let's define a `Task` model that corresponds to our `tasks` database table we just created.


```bash
php artisan make:model Task
```

In the `app/Models` directory, a new file `Task.php` will be created, we will describe the fields as available for filling:

```php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Orchid\Screen\AsSource;

class Task extends Model
{
    use HasFactory, AsSource;
}
```

> **Note.** The model has the `AsSource` trait, for convenient handling via dot notation.


## Screen

Next, we're ready to add a first [screen](/en/docs/screens) to our application. 
The screen’s main difference from the controller is the structure defined in advance, which serves only one-page defining data and events.


```bash
php artisan orchid:screen TaskScreen
```

A new file `TaskScreen.php` will be created in the `app/Orchid/Screens` directory:

```php
namespace App\Orchid\Screens;

use Orchid\Screen\Screen;

class TaskScreen extends Screen
{
    /**
     * A method that defines all screen input data
     * is in it that database queries should be called,
     * api or any others (not necessarily explicit),
     * the result of which should be an array,
     * appeal to which his keys will be used.
     */
    public function query(): array
    {
        return [];
    }

    /**
     * The name is displayed on the user's screen and in the headers
     */
    public function name(): ?string
    {
        return "TaskScreen";
    }

    /**
     * The description is displayed on the user's screen under the heading
     */
    public function description(): ?string
    {
        return "TaskScreen";
    }
    
    /**
     * Identifies control buttons and events.
     * which will have to happen by pressing
     */
    public function commandBar(): array
    {
        return [];
    }

    /**
     * Set of mappings
     * rows, tables, graphs,
     * modal windows, and their combinations
     */
    public function layout(): array
    {
        return [];
    }
}
```

### Routing

Like the controller, the screen needs to register in the route file.
Define it in the file for the admin panel `routes/platform.php`:

```php
use App\Orchid\Screens\TaskScreen;

Route::screen('task', TaskScreen::class)->name('platform.task');
```

After we have registered a new route, you can go to the browser at `/admin/task`,
to look at the empty screen, fill it with the elements.

Add a name and description:

```php
/**
 * The name is displayed on the user's screen and in the headers
 */
public function name(): ?string
{
    return 'Simple To-Do List';
}

/**
 * The description is displayed on the user's screen under the heading
 */
public function description(): ?string
{
    return 'Orchid Quickstart';
}
```

## Navigation

### Menu

All this time, to display the screen, it was necessary to specify an explicit page in the browser, add a new item to the menu, for this
in the file at `app/Orchid/PlatformProvider.php` we add the declaration:

```php
use Orchid\Screen\Actions\Menu;

/**
 * @return Menu[]
 */
public function registerMainMenu(): array
{
    return [
        // Other items...
    
        Menu::make('Tasks')
            ->icon('envelope-letter')
            ->route('platform.task')
            ->title('Tools')
    ];
}
```

### Breadcrumbs

Now our utility is displayed on the left menu and is active when visiting. 
Navigation is carried out not only through transitions from the menu but also through breadcrumbs,
to add them to our screen, you need to append your route with `->breadcrumbs(...)` in `routes/platform.php`.

```php
use App\Orchid\Screens\EmailSenderScreen;
use Tabuna\Breadcrumbs\Trail;

Route::screen('email', EmailSenderScreen::class)
    ->name('platform.email')
    ->breadcrumbs(function (Trail $trail){
        return $trail
                ->parent('platform.index')
                ->push('Email sender');
    });
```


## Adding Tasks

### Creating The Task

The displayed elements of the workspace are declared in the method `layouts` let's
add a modal window that would contain a string with an input field for the name of the task:


```php
use Orchid\Screen\Fields\Input;
use Orchid\Support\Facades\Layout;

/**
 * The screen's layout elements.
 *
 * @return \Orchid\Screen\Layout[]|string[]
 */
public function layout(): iterable
{
    return [
        Layout::modal('create', Layout::rows([
            Input::make('task.name')
                ->title('Name')
                ->placeholder('Enter task name')
                ->help('Enter task name'),
        ]))
            ->title('Create Task')
            ->applyButton('Add Task'),
    ];
}
```

Let's check the browser, nothing has changed, right? Sure, because the modal window is hidden and must be called,
let's add a button to call it, to do this lets add it to the `commandBar` method which defines the basic actions on the screen:


```php
use Orchid\Screen\Actions\ModalToggle;

/**
 * The screen's action buttons.
 *
 * @return \Orchid\Screen\Action[]
 */
public function commandBar(): iterable
{
    return [
        ModalToggle::make('Create Task')
            ->modal('create')
            ->method('create')
            ->icon('plus'),
    ];
}
```

To do this, let's add a button `ModalToggle`. In which we will define:

— Button's text, 
— Name of the modal window that should open when you click on it
— Method of the screen, which will be called when sending.


Now, when we return to the browser, we see that there is a new button in the right corner named "Create Task". 
Let's define the method that should be executed when we send it:


```php
use App\Models\Task;
use Illuminate\Http\Request;

/**
 * @param \Illuminate\Http\Request $request
 *
 * @return void
 */
public function create(Request $request)
{
    $request->validate([
        'task.name' => 'required|max:255',
    ]);

    // Create The Task...
    $task = new Task();
    $task->name = $request->input('task.name');
    $task->save();
}
```

Great! We can now successfully create tasks. Next, let's continue adding to our screen by building a list of all existing tasks.

### Displaying Existing Tasks


## Deleting Tasks

### Adding The Delete Button
### Deleting The Task


Congratulations, you should now understand how the platform works!
It is an elementary example, but the development process will be identical in many aspects.
We recommend going to the [Screens](/en/docs/screens) section to learn more about the possibilities in your hands.
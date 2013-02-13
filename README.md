# TaskQueue.php [![Build Status](https://travis-ci.org/vis7mac/taskqueue.png?branch=master)](https://travis-ci.org/vis7mac/taskqueue)

> A background task queue for PHP.

For some tasks, the execution time limit is just not enough or we don't want the client to wait a long time until something has finished on the server.

Here comes TaskQueue - connecting your app to a background worker process doing all that stuff. 

# Usage

First of all: Require the library file (`lib/taskqueue.php`) in your app.

Make sure to copy the whole `lib` folder as it contains libraries `TaskQueue` requires to work.

## Task storage

Every app or collection of apps has it's own task storage containing task script files together with application states (variables, constants etc.), config and log files.

It's recommended to have at least a task storage for every app to make log reporting easier and to make everything run more smoothly.

You create a new task storage by creating a new instance of the `TaskQueue` class:

	$taskqueue = new TaskQueue("/path/to/storage", $behavior); // You can find more information on $behavior below

If there's already a valid task storage, it will use it.

But if the folder you point to exists, but is no task storage or an invalid one, `TaskQueue` will throw an Exception!

Otherwise, a blank task storage will be created. Make sure the path you are pointing at is writable!

### The structure of a task storage

	bin/
		runner              <-- The runner script, spawns a task runner (lib/runner.php) and handles logging
		lib/
			runner.php      <-- The runner script itself
			taskqueue.php   <-- The library file
	logs/
		YEAR-MONTH-DAY.txt  <-- Global log
	memory/
		<NAME>/
			task.json       <-- Information and settings for the memory task
			task.php        <-- The script file
			log.txt         <-- Usage of the memory task
	tasks/
		<STATE>_<READABLE_STATE>-<PRIORITY>-<UNIQUE_ID>-<NAME>/
			task.json       <-- Information and settings for the task
			task.php        <-- The script file
			state.json      <-- Application state (depending on settings)
			log.txt         <-- The task specific log
	taskqueue.json          <-- Information about the storage version (validity etc.) and configuration

## Defining tasks

Now that you have a task storage, you can start creating tasks.

Adding a task looks like this:

	$task = $taskqueue->add($name, $task, [$env], [$priority], [$path], [$state]);

The method returns a task object used to access the task later. You can find more information on this below.

### Name

If you want to get the task from another application or instance and don't want to save the unique ID, you can set a user-specific name to address the task.

This name should tell what the task does so you later know what the log contains after the script and information have been cleared (see below).

If the name already exists, the call will throw a `InvalidArgumentException`. Make sure to catch it!

### The task script

Every task has a PHP script to run. You provide this script as `$task`. It can either be a path to the file or the string of PHP.
Basically, the file will be included by the task runner.

This file must contain a return statement at the end. If `true` or `1` is returned, all worked, otherwise it failed.
Error codes are numbers from 4-∞ - they can have a different meaning in every app, so you must know what they mean.

Other error codes you can't use as return values but used internally are:

- `0`: Queued, waiting
- `~`: Currently running
- `2`: Failed running (paths not existing…)
- `3`: Task was invalid

Never return a string or any other data types! They will get changed to error code `3`.

### Other arguments

The other arguments are optional:

- `$env`: Information for the script what to do

	This is an associative array containing var names and values.
	Your script gets access to them using normal variable names.
	
		$env = array("variable" => "value", "anotherVariable" => "anotherValue");
		
		// Access using $variable and $anotherVariable

- `$priority`: The task priority

	This is a number between 0 (very high priority) and 9 (low priority) deciding when the task is going to run.
	
	Background: The tasks are run sequentially. If a task has finished, the task runner looks for the next task. If there's one with the priority 0, it will take it. If not, it will look for tasks with priority 1 and so on.

- `$path`: The working directory for the task

	If you want your script to run in a specific working directory, you can define the path using this variable. If not given, it will run in the current working directory.

- `$state`: The application state

	This is *kind of* the same than `$env`, but it's not for custom variables but for the current application state. This contains classes, class instances, variables, constants and functions.
	
	If you want to serialize the state, you can pass the result of `get_defined_vars()`. All other types are global and can be read by the method itself.
	
	Please note that this process produces damn big state files and is not recommended for every project. You can also include all required stuff using `$env` or in the task script file, so only required content is saved and available.

	But if your application is quite small, this can be a huge help, because you don't have to add all that stuff manually.

## The task memory

Sometimes, you use the same task again and again. Everytime you add a new task, it gets added to the queue and uses disk space.

But you can also define tasks to stay in memory of the task storage, so the script file is only there one time and is shared between future tasks.

	$success = $taskqueue->defineMemory($name, $task, [$priority]);

- `$name`: The internal name of the task.
- `$task`: See above
- `$priority`: The default priority - can be overridden.

### Add a task based on the task memory

This is basically the same as adding a normal task - but you pass the name of the memory task instead of the script file.

## The task object

Every task has it's own task object. You get it when adding the task.

Every property in the object will stay connected to the storage - so you don't have to get a new object each time you want to change something or get some information.

### Access the task later

#### Unique ID

You can get the unique task ID from a task object using

	$id = $task->uniqueID;

It is an integer and is unique in the whole task storage.

#### Name

	$name = $task->name;

### Pause a task

If you want to pause a task, you can do it like so:

	$success = $task->pause();

### Requeue a paused task

You can also add a paused task back in:

	$success = $task->queue();

### Remove a task

If you want to unqueue and delete a task completely, do it like so:

	$success = $task->unset();

### Instantly run a task

Please note that the task is then run in the current process and is blocking your application. Normally, you won't use that.

This also sets the state and logs. Quite the same thing the background runner does.

	$success = $task->run();

### Get the current status

You can poll if the task is done or if it worked.

Please don't use a while loop to do that, otherwise you don't have to use `TaskQueue` at all ;).

You can use it for client side polling using JavaScript or something like that.

	$status = $task->status;

The different values are:

- `TASKQUEUE_STATUS_IDLE`: Currently in the queue and waiting.
- `TASKQUEUE_STATUS_RUNNING`: Currently running.
- `TASKQUEUE_STATUS_DONE`: Done. No errors.
- < 0: Failed. (Status code from the task * -1)

### Get the run log

	$logString = $task->log;

If there's no log yet (the task is in the queue), this value will be an empty string, otherwise the whole output of the task.

## Get a task object by ID or name

This returns the same task objects like these you get when creating a task.

	$object = $taskqueue->findByID($uniqueID);
	$object = $taskqueue->findByName($name);

## Get the next task

This is used by the task runner. Don't use that.

	$object = $taskqueue->next;

## The task runner

To run something, you have to setup the background runner process.

It is a script running in an endless loop processing the tasks.

You can find the script in the respective task storage under `bin/runner`.
This can be run using `bash bin/runner` on the server.

But it is recommended to use a daemon to watch the process and restart it when it crashes.

You can find more information on this topic on the website of the project: <http://cr.yp.to/daemontools.html>

There's a very good (German) tutorial how to setup a daemon on Uberspace, a really good German hoster: <http://uberspace.de/dokuwiki/system:daemontools>

## Clearing behavior

After a task was run, the task script, the application state and the config will be deleted, only the log will be preserved.

If the task was paused, everything will stay in the folder until it gets deleted or started again.

You can configure the behavior using the initializer of the class:

	$taskqueue = new TaskQueue($path, $behavior);

It can be either:

- `TASKQUEUE_EVERYTHING`: Preserve everything (warning, can become very much over time!!)
- `TASKQUEUE_LOGS`: Preserve logs (default behavior)
- `TASKQUEUE_LOGS_FAIL`: Only preserve logs when the task failed, otherwise delete everything
- `TASKQUEUE_NONE`: Always delete the whole folder

# Feedback

This stuff is all beta - so if you have feedback or ideas, don't hesitate to contact me: <lukas@lu-x.me>.
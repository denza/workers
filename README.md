# Workers [![Build Status](https://travis-ci.org/chadrem/workers.svg)](https://travis-ci.org/chadrem/workers) [![Coverage Status](https://coveralls.io/repos/chadrem/workers/badge.svg?branch=master&service=github)](https://coveralls.io/github/chadrem/workers?branch=master)

Workers is a Ruby gem for performing work in background threads.
Design goals include high performance, low latency, simple API, customizability, and multi-layered architecture.
It provides a number of simple to use classes that solve a wide range of concurrency problems.
It is used by [Tribe](https://github.com/chadrem/tribe "Tribe") to implement event-driven actors.

## Contents

- [Installation](#installation)
- [Parallel Map](#parallel-map)
- [Tasks](#tasks)
- [Workers](#workers)
- [Pool](#pools)
- [Timers](#timers)
- [Schedulers](#schedulers)
- [Concurrency and performance](#concurrency-and-performance)
- [Contributing](#contributing)

## Installation

Add this line to your application's Gemfile:

    gem 'workers'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install workers

## Parallel Map

Parallel map is the simplest way to get started with the Workers gem.
It is similar to Ruby's built-in Array#map method except each element is mapped in parallel.

    Workers.map([1, 2, 3, 4, 5]) { |i| i * i }

Any exceptions while mapping with cause the entire map method to fail.
If your block is prone to temporary failures (exceptions), you can retry it.

    Workers.map([1, 2, 3, 4, 5], :max_tries => 100) do |i|
      if rand <= 0.5
        puts "Sometimes I like to fail while computing #{i} * #{i}."
        raise 'sad face'
      end

      i * i
    end

## Tasks

Tasks and task groups provide more flexibility than parallel map.
The main benefit is that you get to decide how you want to handle exceptions.

    # Create a task group (it contains a pool of worker threads).
    group = Workers::TaskGroup.new

    # Add tasks to the group.
    10.times do |i|
      10.times do |j|
        group.add do
          group.synchronize { puts "Computing #{i} * #{j}..." }
          i * j # Last statement executed is the result of the task.
        end
      end
    end

    # Execute the tasks (blocks until the tasks complete).
    # Returns true if all tasks succeed.  False if any fail.
    group.run

    # Return an array of all the tasks.
    group.tasks

    # Return an array of the successful tasks.
    group.successes

    # Return an array of the failed tasks (raised an exception).
    group.failures

    # Review the results.
    group.tasks.each do |t|
      t.succeeded? # True or false (false if an exception occurred).
      t.failed?    # True or false (true if an exception occurred).
      t.input      # Input value.
      t.result     # Output value (the result of i * i in this example).
      t.exception  # The exception if one exists.
      t.max_tries  # Maximum number of attempts.
      t.tries      # Actual number of attempts.
    end

Note that instances of TaskGroup provide a 'synchronize' method.
This method uses a mutex so you can serialize portions of your tasks that aren't thread safe.

#### Options

    group = Workers::TaskGroup.new(
      :logger => nil,                   # Ruby logger instance.
      :pool => Workers.pool             # The workers pool used to execute timer callbacks.
    )

    task = Workers::Task.new(
      :logger => nil,                   # Ruby logger instance.
      :perform => proc {},              # Required option.  Block of code to run.
      :args => [],                      # Array of arguments passed to the 'perform' block.
      :finished => nil,                 # Callback to execute after attempting to run the task.
      :max_tries => 1,                  # Number of times to try completing the task (without an exception).
    )

## Workers

#### Basic

The main purpose of the Worker class is to add an event system on top of Ruby's built-in Thread class.
This greatly simplifies inter-thread communication.
Workers are fairly low level and don't handle exceptions for you.
Failing to handle exceptions will result in dead workers so you must rescue them in your application code.

    # Initialize a worker pool.
    pool = Workers::Pool.new

    # Perform some work in the background.
    100.times do
      pool.perform do
        begin
          sleep(rand(3))
          puts "Hello world from thread #{Thread.current.object_id}"
        rescue Exception => e
          puts "Oh no, my hello world failed!"
        end
      end
    end

    # Tell the workers to shutdown.
    pool.shutdown do
      puts "Worker thread #{Thread.current.object_id} is shutting down."
    end

    # Wait for the workers to shutdown.
    pool.join

#### Advanced

The Worker class is designed to be customized through inheritence and its event system:

    # Create a subclass that handles custom events.
    class CustomWorker < Workers::Worker
      private
      def process_event(event)
        case event.command
        when :my_custom
          begin
            puts "Worker received custom event: #{event.data}"
            sleep(1)
          rescue Exception => e
            puts "This is a very sad program."
          end
        end
      end
    end

    # Create a pool that uses your custom worker class.
    pool = Workers::Pool.new(:worker_class => CustomWorker)

    # Tell the workers to do some work using custom events.
    100.times do |i|
      pool.enqueue(:my_custom, i)
    end

    # Tell the workers to shutdown.
    pool.shutdown do
      puts "Worker thread #{Thread.current.object_id} is shutting down."
    end

    # Wait for the workers to shutdown.
    pool.join

#### Without pools

In most cases you will be using a group of workers (a pool) as demonstrated above.
In certain cases, you may want to use a worker directly without the pool.
This effectively gives you direct access to a single event-driven thread.
Note that you must handle exceptions yourself since you are working directly with the worker class:

    # Create a single worker.
    worker = Workers::Worker.new

    # Perform some work in the background.
    25.times do |i|
      worker.perform do
        begin
          sleep(0.1)
          puts "Hello world from thread #{Thread.current.object_id}"
        rescue Exception => e
          puts "Oh no, my hello world failed!"
        end
      end
    end

    # Tell the worker to shutdown.
    worker.shutdown

#### Options

    worker = Workers::Worker.new(
      :logger => nil,                   # Ruby Logger instance.
      :input_queue => nil               # Ruby Queue used for input events.
    )

## Pools

Pools allow a group of workers to share a work queue.
The Workers gem has a default pool (Workers.pool) with 20 workers so in most cases you won't need to create your own.
Pools can be adjusted using the below methods:

    # Create a pool.
    pool = Workers::Pool.new

    # Return the number of workers in the pool.
    pool.size

    # Increase the number of workers in the pool.
    pool.expand(5)

    # Decrease the number of workers in the pool.
    pool.contract(5)

    # Resize the pool size to a specific value.
    pool.resize(20)

#### Options

    pool = Workers::Pool.new(
      :size => 20,                      # Number of threads to create.
      :logger => nil,                   # Ruby Logger instance.
      :worker_class => Workers::Worker  # Class of worker to use for this pool.
    )

## Timers

Timers provide a way to execute code in the future.
You can easily use them to tell a Worker or it's higher level classes (Task, TaskGroup, etc) to perform work in the future.

    # Create a one shot timer that executes in 1.5 seconds.
    timer = Workers::Timer.new(1.5) do
      puts 'Hello world'
    end

    # Create a periodic timer that loops infinitely or until 'cancel' is called.
    timer = Workers::PeriodicTimer.new(1) do
      puts 'Hello world many times'
    end

    # Let the timer print some lines.
    sleep(5)

    # Shutdown the timer.
    timer.cancel

#### Options

    timer = Workers::Timer.new(1,
      :logger => nil,                   # Ruby logger instance.
      :repeat => false,                 # Repeat the timer until 'cancel' is called.
      :scheduler => Workers.scheduler,  # The scheduler that manages execution.
      :callback => nil                  # The proc to execute (provide this or a block, but not both).
    )

    timer = Workers::PeriodicTimer.new(1,
      :logger => nil,                   # Ruby logger instance.
      :scheduler => Workers.scheduler,  # The scheduler that manages execution.
      :callback => nil                  # The proc to execute (provide this or a block, but not both).
    )

## Schedulers

Schedulers are what trigger Timers to fire.
The Workers gem has a default scheduler (Workers.scheduler) so in most cases you won't need to create your own.
Schedulers execute timers using a pool of workers so make sure your timer block is thread safe.
You can create additional schedulers as necessary:

    # Create a workers pool with a larger than default thread count (optional).
    pool = Workers::Pool.new(:size => 100)

    # Create a scheduler.
    scheduler = Workers::Scheduler.new(:pool => pool)

    # Create a timer that uses the above scheduler.
    timer = Workers::Timer.new(1, :scheduler => scheduler) do
      puts 'Hello world'
    end

    # Wait for the timer to fire.
    sleep(5)

    # Shutdown the scheduler.
    scheduler.dispose

#### Options

    scheduler = Workers::Scheduler.new(
      :logger => nil,                   # Ruby logger instance.
      :pool => Workers::Pool.new        # The workers pool used to execute timer callbacks.
    )

#### Bucket Schedulers

The Bucket scheduler class is a specialized scheduler designed to work around lock contention.
This is accomplished by using many pools (100 by default) each with a small number of workers (1 by default).
Timers are assigned to a scheduler by their ````hash```` value.
Most users will never need to use this class, but it is documented here for completeness.
Both the number of buckets and the number of workers assigned to each bucket are configurable.

    # Create a bucket scheduler.
    scheduler = Workers::BucketScheduler.new

## Concurrency and performance

Workers is tested with both JRuby and MRI (C Ruby).
Below are some notes specific to each Ruby implementation.
In summary, JRuby is the recommended Ruby to use with Workers since it provides the highest performance with multiple CPU cores.

#### JRuby (recommended)

JRuby is designed for multi-threaded apps running on multiple cores.
When used with Workers, you will be able to saturate all of your CPU cores with little to no tuning.
It is highly recommended you increase the number of workers in your pool if you have a large number of cores.
A good starting point is 1x - 2x the number of cores for CPU bound apps.
For IO bound apps you will need to do some benchmarking to figure out what is best for you.
A good starting point is 4x - 10x the number of cores for IO bound apps.

#### MRI 1.9.x or newer (supported)

MRI 1.9 and above use real operating system threads with a global interpreter lock (GIL).
The bad news is that due to the GIL, only one thread can execute Ruby code at a given point in time.
This means your app will be CPU bound to a single core.
The good news is that IO bound applications will still see huge benefits from Workers.
Examples of such IO are web service requests, web servers, web crawlers, database queries, writing to disk, etc.
Threads performing such IO will temporarily release the GIL and thus let another thread execute Ruby code.

#### MRI 1.8.x or older (not supported)

These old versions of Ruby used green threads (application layer threads) instead of operating system level threads.
I recommend you upgrade to a newer version as I haven't tested Workers with them.
They also aren't officially supported by the Ruby community at this point.

#### Rubinius

I haven't tested Workers with Rubinius, but in theory it should just work.
The above JRuby notes should apply.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

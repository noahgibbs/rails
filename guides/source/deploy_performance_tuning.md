**DO NOT READ THIS FILE ON GITHUB, GUIDES ARE PUBLISHED ON https://guides.rubyonrails.org.**

Tuning Performance for Deployment
=================================

This guide covers performance and concurrency configuration for deploying your production Ruby on Rails application.

After reading this guide, you will know:

* Whether to use Puma, the default application server
* How to configure important performance settings for Puma

More information about how to configure your application can be found in the [Configuration Guide](configuring.html).

--------------------------------------------------------------------------------

Choosing an Application Server
------------------------------

An application server uses a particular concurrency method. For example Unicorn uses processes, Puma is hybrid process- and thread-based concurrency, Thin uses EventMachine and Falcon uses Ruby Fibers.

A full discussion of Ruby's concurrency methods is beyond the scope of this document. If you want to use a method other than processes or threads, you will need to use a different application server. There are other features are only available with another server, such as Pitchfork's preforking.

The most common application servers are used by removing the Puma gem from your Gemfile and including the gem for the server. Consult the appropriate application server documentation for details.

Ruby Concurrency
----------------

Ruby has many kinds of concurrency. Puma supports a hybrid process-based and thread-based concurrency model.

This guide is for [CRuby](https://ruby-lang.org), the original implementation of Ruby. If you're using a Ruby without a GVL such as JRuby or TruffleRuby, the configuration will be very different. If needed, check other sources specific to your Ruby implementation.

### Process-Based Concurrency

Puma calls its multi-process concurrency "clustered mode". In this mode it forks new worker processes from a master process and each one separately processes requests. Each worker is a fully-capable Ruby process, doing everything Ruby normally does.

Creating new processes uses a lot of memory. Each worker contains all data from the master process. Ruby uses [copy-on-write memory](https://en.wikipedia.org/wiki/Copy-on-write) to avoid duplicating most master-process data that doesn't change. But process-based workers often use a lot of memory, especially long-running workers.

Processes are resilient. Killing a single process doesn't affect other processes much. They are also slower to start up than threads. Loading your application in the master thread instead of the workers is called preloading. It helps startup time, but may increase memory use.

### Thread-Based Concurrency

Multiple threads can run in the same process. This avoids copying master data to each worker. Thread-based workers usually use much less memory than process-based workers.

[CRuby](https://www.ruby-lang.org/en/) has a [Global Interpreter Lock](https://en.wikipedia.org/wiki/Global_interpreter_lock), often called the GVL or GIL. The GVL prevents multiple threads from running Ruby code at the same time in a single process. A thread can be waiting on network data, database operations or some other non-Ruby work, but only one can actively run Ruby code at a time. This means thread-based concurrency is more efficient for applications that use a lot of I/O such as database operations or network APIs. The more I/O your application uses, the more threads it would benefit from.

With the GVL, using a lot of threads has limited value. A Rails app rarely benefits from more than 6. To have a large number of workers, some other concurrency method must be used.

Threads are less resilient than processes. Certain errors like segmentation faults can destroy the entire process and all threads inside.

Thread-based workers have much faster startup time than process-based workers. This isn't an issue for most long-running Rails servers, but can be important in some cases.

### Hybrid Concurrency

Puma allows forking multiple processes, each of which uses multiple threads. This provides a compromise between process-based and thread-based concurrency. Using multiple threads per process helps memory usage. But multiple processes are required for CRuby to run more Ruby code at the same time.

Hybrid concurrency limits the damage from a segmentation fault or other error that kills a process. A single process dying will kill that process's threads but not threads in other processes.

Choosing Default Settings
-------------------------

Rails' default settings are chosen for small applications. You can improve performance for your large application that serves a lot of requests by changing them.

This section contains common sense defaults based on the type and size of your application and the hosts on which it runs. You can improve performance more by testing your application specifically. See "Performance Testing" for details.

If you're using a Ruby implementation without a GVL such as JRuby or TruffleRuby the number of threads and processes will be completely different. These recommendations are only accurate for CRuby. These are also production recommendations. Your development application will have different needs from a production application, and should use a different configuration.

### Threads Per Process

Rails uses 3 threads per process by default. A well-optimized I/O-heavy Rails application should specify 5 or 6 threads per process at maximum. Discourse, for example, benefits from about 5 threads per process. Small applications with less I/O benefit from around 3 threads per process.

From the default Puma configuration:

    As a rule of thumb, increasing the number of threads will increase how much
    traffic a given process can handle (throughput), but due to CRuby's
    Global VM Lock (GVL) it has diminishing returns and will degrade the
    response time (latency) of the application.

### Number of Processes

[Puma's deployment documentation](https://github.com/puma/puma/blob/master/docs/deployment.md) recommends running at least 1.5 processes per available processor core. Automatic methods to determine the number of cores are unreliable. You should specify the number of processes manually.

### Preloading



Performance Testing
-------------------

Settings from "Choosing Default Settings" are much better than using the initial Rails defaults. But your specific application may have unusual needs or benefit from different configuration options.

The best way to choose your application's settings is to test the performance of your application with a simulated production workload.

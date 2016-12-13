---
layout: post
title: The Open/Closed Principle
---

I am a big proponent of the [SOLID principles](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)). But one of the principles - the open/closed principle - is often misunderstood.

> software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification

This is how Bertrand Meyer stated it first in his book "Object-Oriented Software Construction" in 1988. The problem with it is that some people see the word `extension` and they think that it is talking about inheritance (because PHP uses the `extend` keyword for inheritance).

I recently came across this quote on reddit:

![Reddit quote]({{ site.url }}/public/open_closed_reddit.png)

The author clearly misunderstood the principle and now he is advocating against dependency injection with the reason that it makes extending things a pain.

From his viewpoint his argument makes sense. I have experienced this pain myself a few years ago, before I had a better grasp of object oriented programming.

Let us assume that we have a logger that writes to the filesystem:


```php

<?php declare(strict_types=1);

interface Logger
{
    public function log(string $message) : void;
}
```


```php

<?php declare(strict_types=1);

class FilesystemLogger implements Logger
{
    protected $filesystemWriter;
    protected $config;

    public function __construct(
        FilesystemWriter $filesystemWriter,
        Config $config
    ) {
        $this->filesystemWriter = $filesystemWriter;
        $this->config = $config;
    }

    public function log(string $message) : void
    {
        $this->filessystemWriter->append(
            $this->config->getLogfilePath(),
            $message
        );
    }
}
```

Now let's try to add some functionality through extension. Let's say that we also want to receive an email when we log something in some cases. So we create a separate class for that.

```php

<?php declare(strict_types=1);

class EmailAndFilesystemLogger extends FilesystemLogger implements Logger
{
    protected $emailer;

    public function __construct(
        FilesystemWriter $filesystemWriter,
        Config $config,
        Emailer $emailer
    ) {
        parent::__construct($filesystemWriter, $config);

        $this->emailer = $emailer;
    }

    public function log(string $message) : void
    {
        parent::log($message);

        $this->emailer->send(
            $this->config->getLogEmail(),
            $message
        );
    }
}
```

Chances are that you have seen code like that before. If something changes in the base class, all the child classes break. Clearly not a good way to write maintainable code. If you want to annoy your coworkers even more, add multiple levels of inheritance.

This can often be seen in frameworks where they use some form of a `BaseController`. They put the most commonly used things in the base controller (templating, session, ...) and then only add additional dependencies in the specific controllers.

Then one day you try to add something new to the base controller and everything breaks down. So they came up with a solution: they injected the dependency injection container into the base controller.

By injecting a dependency injection container they create a service locator. A single class that provides access to a lot of other classes.

So this solves the problem? Kind of, but not really.

It works in that you will get working code that can solve business problems. But that approach has a lot of drawbacks that will bite you in the ass once your project grows. Your code hides its dependencies and has become very hard to test. You have a godlike object (the service locator) that your whole application depends on.

Let us take a step back and see if there is a better way to solve this problem. How do we follow the open/closed principle while using dependency injection at the same time?

We use the same start, but this time we prevent inheritance.

```php

<?php declare(strict_types=1);

interface Logger
{
    public function log(string $message) : void;
}
```

```php

<?php declare(strict_types=1);

final class FilesystemLogger implements Logger
{
    private $filesystemWriter;
    private $config;

    private function __construct(
        FilesystemWriter $filesystemWriter,
        Config $config
    ) {
        $this->filesystemWriter = $filesystemWriter;
        $this->config = $config;
    }

    public function log(string $message) : void
    {
        $this->filessystemWriter->append(
            $this->config->getLogfilePath(),
            $message
        );
    }
}
```

Instead of inheritance, we will limit ourselves to composition this time. First let us try to emulate what we did earlier with inheritance.

```php

<?php declare(strict_types=1);

final class EmailAndFilesystemLogger implements Logger
{
    private $filesystemLogger;
    private $config;
    private $emailer;

    public function __construct(
        FilesystemLogger $filesystemLogger,
        Config $config,
        Emailer $emailer
    ) {
        $this->filesystemlogger = $filesystemLogger;
        $this->config = $config;
        $this->emailer = $emailer;
    }

    public function log(string $message) : void
    {
        $this->filesystemLogger->log($message);

        $this->emailer->send(
            $this->config->getLogEmail(),
            $message
        );
    }
}
```

Now instead of extending the `FilesystemLogger` to access the functionality, we just used it as another dependency. This is a much better approach than the one above.

But I still don't like this, whenever you use `and` in a class name then it's a good indicator that the class is doing too much. There is no way to use the `EmailLogger` by itself. So why not decouple everything and make the whole thing easily extensible in the spirit of the open/closed principle?

```php

<?php declare(strict_types=1);

final class EmailLogger implements Logger
{
    private $config;
    private $emailer;

    public function __construct(
        Config $config,
        Emailer $emailer
    ) {
        $this->config = $config;
        $this->emailer = $emailer;
    }

    public function log(string $message) : void
    {
        $this->emailer->send(
            $this->config->getLogEmail(),
            $message
        );
    }
}
```


```php

<?php declare(strict_types=1);

final class AggregatorLogger implements Logger
{
    private $loggers = [];

    public function add(Logger $logger) : void
    {
        $this->loggers[] = $logger;
    }

    public function log(string $message) : void
    {
        foreach ($this->loggers as $logger) {
            $logger->log($message);
        }
    }
}
```

Now we just need to set up the `AggregatorLogger` with both the `FilesystemLogger` and `EmailLogger`. We had to write a little more code and more classes, but in the end we ended up with very simple code and little mental overhead compared to using inheritance.

The code that we wrote is open for extension and closed for modification. We followed the open/closed principle while also following the other SOLID principles.

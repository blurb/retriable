Retriable
============

[![Build Status](https://secure.travis-ci.org/kamui/retriable.png)](http://travis-ci.org/kamui/retriable)

Retriable is an simple DSL to retry a code block if an exception should be raised.  This is especially useful when interacting external api/services or file system calls.

Installation
------------
Via command line:

```ruby
gem install retriable
```

In your ruby script:

```ruby
require 'retriable'
```

In your Gemfile:

```ruby
gem 'retriable'
```

Usage
---------------

Code in a retriable block will be retried if an exception is raised. By default, Retriable will rescue any exception inherited from `StandardError` (and `Timeout::Error`, which does not inherit from `StandardError` in ruby 1.8) and make 3 retry attempts before raising the last exception.

```ruby
require 'retriable'

class Api
  # Use it in methods that interact with unreliable services
  def get
    retriable do
      # code here...
    end
  end
end
```

Here are the available options:

`tries` (default: 3) - Number of attempts to make at running your code block

`interval` (default: 0) - Number of seconds to sleep between attempts

`timeout` (default: 0) - Number of seconds to allow the code block to run before raising a Timeout::Error

`on` (default: [StandardError, Timeout::Error]) - `StandardError` and
`Timeout::Error`, array of exceptions to rescue for each attempt, or
and array of exception & regex pairs.

`on_return` (default: nil) - Allows the return value of the code block
to be examined and retried if the Proc returns true

`on_retry` - (default: nil) - Proc to call after each attempt is rescued

You can pass options via an options `Hash`. This example will only retry on a `Timeout::Error`, retry 3 times and sleep for a full second before each attempt.

```ruby
retriable :on => Timeout::Error, :tries => 3, :interval => 1 do
  # code here...
end
```

You can also specify multiple errors to retry on by passing an array of exceptions.

```ruby
retriable :on => [Timeout::Error, Errno::ECONNRESET] do
  # code here...
end
```

You can also provide a regex pattern to further narrow down the range of
exceptions. In this example, it will retry if the exception is a Timeout::Error
or if it's a StandardError containing "HTTP":

```ruby
retriable :on => [Timeout::Error, [StandardError, /HTTP/]] do
  # code here...
end
```

Note, that when you're specifying a single item, you need to nest the
exception-regex pair in an array:

```ruby
retriable :on => [[StandardError, /HTTP/]] do
  # code here...
end
```

You can also specify a timeout if you want the code block to only make an attempt for X amount of seconds. This timeout is per attempt.

```ruby
retriable :timeout => 1 do
  # code here...
end
```

If you need millisecond units of time for the sleep or the timeout:

```ruby
retriable :interval => (200/1000.0), :timeout => (500/1000.0) do
  # code here...
end
```

You can pass a proc for the sleep interval:

```ruby
retriable :interval => Proc.new { |attempt| (2**attempt-1)/2 } do
  # code here...
end # backoff exponentially
```

You can retry based on the return value of the code block:

```ruby
retriable :on_return => Proc.new { |return_value, attempt_count| return_value != OK } do
  # code here...
end
```

The Proc provided to the :on_return option should return true if the code block should be executed again. If all retry attempts have been exhausted, the last return value of the block is returned in the case there was no exception thrown.

```ruby
return_values = [1,2,3]
result = retriable :tries => 3, :on_return => Proc.new { |return_value, attempt_count| return_value < 10 } do
  return_values.shift
end

result==3 # true  
```


Retriable also provides a callback called `:on_retry` that will run after an exception is rescued. This callback provides the number of `tries`, and the `exception` that was raised in the current attempt. As these are specified in a `Proc`, unnecessary variables can be left out of the parameter list.

```ruby
do_this_on_each_retry = Proc.new do |exception, tries|
  log "#{exception.class}: '#{exception.message}' - #{tries} attempts."}
end

retriable :on_retry => do_this_on_each_retry do
  # code here...
end
```

What if I want to execute a code block at the end, whether or not an exception was rescued ([ensure](http://ruby-doc.org/docs/keywords/1.9/Object.html#method-i-ensure))? Or, what if I want to execute a code block if no exception is raised ([else](http://ruby-doc.org/docs/keywords/1.9/Object.html#method-i-else))? Instead of providing more callbacks, I recommend you just wrap retriable in a begin/retry/else/ensure block:

```ruby
begin
  retriable do
    # some code
  end
rescue => e
  # run this if retriable ends up re-rasing the exception
else
  # run this if retriable doesn't raise any exceptions
ensure
  # run this no matter what, exception or no exception
end
```

Non Kernel version
------------------
By default, `require 'retriable'` will include the `#retriable` method into the `Kernel` so that you can use it everywhere. If you don't want this behaviour, you can load a non-kernel version:

```ruby
gem 'retriable', require => 'retriable/no_kernel'
```

Or in your ruby script:

```ruby
require 'retriable/no_kernel'
```

In this case, you'll just execute a retriable block from the `Retriable` module:

```ruby
Retriable.retriable do
  # code here...
end
```


Credits
-------

Retriable was originally forked from the retryable-rb gem by [Robert Sosinski](https://github.com/robertsosinski), which in turn originally inspired by code written by [Michael Celona](http://github.com/mcelona) and later assisted by [David Malin](http://github.com/dmalin). The [attempt](https://rubygems.org/gems/attempt) gem by Daniel J. Berger was also an inspiration. [Lamont Nelson] (http://github.com/lamontnelson) contributed the code for retrying based on the return value of the code block.

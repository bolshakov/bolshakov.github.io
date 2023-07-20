---
layout: post
title: "Navigating Ruby Autoloading: From Chaos to Order"
date: 2023-07-19 21:53:35 +0300
tags: ruby
excerpt: Discover how to navigate the complexities of managing dependencies in Ruby. This article explores different techniques for handling autoload in Ruby, along with their respective challenges, such as the ordering of <code>require</code> statements and the thread safety of <code>Kernel.autoload</code>. Delve into the world of autoloading, learn about its pitfalls, and see how to effectively use it in your codebase. The highlight of the discussion is an introduction to Zeitwerk, a modern solution to these challenges.
---

Consider a scenario where you're developing a Ruby gem that offers a range of domain models 
for an audio library. This gem encompasses a common set of classes typical to this 
domain, such as `Album`, `Artist`, `Track`, `Genre`, and so forth.

```ruby
# lib/audio.rb
require 'dry-struct'

require 'audio/album'
require 'audio/artist'
require 'audio/country'
require 'audio/track'
require 'audio/types'

module Audio
  VERSION = 0.1
end
```

```ruby
# lib/audio/types.rb
module Audio
  module Types
    include Dry.Types()
  end
end
```

```ruby
# lib/audio/album.rb
module Audio
  class Album < Dry::Struct
    attribute :title, String
    attribute :tracks, Types::Array.of(Track)
    attribute :country, Country
  end
end
```

```ruby
# lib/audio/track.rb
module Audio
  class Track < Dry::Struct
    attribute :title, String
    attribute :performers, Types::Array.of(Artist)
  end
end
```

```ruby
# lib/artist.rb
module Audio
  class Artist < Dry::Struct
    attribute :name, String
    attribute :country, Country
  end
end
```

```ruby
# lib/country.rb
module Audio
  class Country < Dry::Struct
    attribute :name, String
  end
end
```

This is regular ruby library with all of the components loaded in the main `Audio` module.
Here goes the simplest smoke test:

```ruby
require 'audio'

RSpec.describe Audio do
  it 'has a version number' do
    expect(Audio::VERSION).not_to be nil
  end
end

```

Trying to run it fails with the following error.

```ruby
$ bundle exec rspec
NameError:
  uninitialized constant Audio::Album::Types
# ./lib/audio/album.rb:4:in `<class:Album>'
# ./lib/audio/album.rb:2:in `<module:Audio>'
# ./lib/audio/album.rb:1:in `<top (required)>'
# ./lib/audio.rb:3:in `<top (required)>'
# ./spec/audio_spec.rb:1:in `require'
# ./spec/audio_spec.rb:1:in `<top (required)>'
```

Here's what's happening: Our `audio.rb` file contains a series of required files. The Ruby interpreter 
reads and evaluates these files in sequence.

Let's examine the `album.rb` file. When the interpreter comes across line #4 (`attribute :country, Types::Array.of(Track)`), 
it attempts to locate the `Types` constant in the constants table. However, at this point, the `Types` constant 
hasn't been loaded yet.

Remember the `audio.rb` file? The `Types` module loading occurs after the `Album` class loading! To overcome this 
issue, you must correctly arrange your dependencies order.

The `Type` module should be loaded first. The `Country` class should be loaded prior to the `Artist` class, 
and the `Track` class should be loaded before the `Album` class:

```ruby
# lib/audio.rb
require 'dry-struct'

require 'audio/types'
require 'audio/country'
require 'audio/artist'
require 'audio/track'
require 'audio/album'
```

This solution, while effective, is not particularly user-friendly. It leads to a number of carefully  
ordered `require` statements. Another drawback of this approach is that it results in an increase in startup 
time and slower feedback from tests.

# Explicitly Required Dependencies

An alternative solution to this problem is to explicitly require dependencies within each individual file.

```ruby
# lib/audio.rb
module Audio < Dry::Struct
end
```

```ruby
# lib/audio/album.rb
require 'dry-struct'

require 'audio/types'
require 'audio/track'
require 'audio/country'

module Audio
  class Album < Dry::Struct
    attribute :title, String
    attribute :tracks, Types::Array.of(Track)
    attribute :country, Country
  end
end

```

```ruby 
# lib/audio/artist.rb
require 'dry-struct'

require 'audio/types'

module Audio
  class Artist < Dry::Struct
    attribute :name, String
    attribute :country, Country
  end
end
```

```ruby
# lib/audio/coountry.rb
require 'dry-struct'

module Audio
  class Country < Dry::Struct
    attribute :name, String
  end
end
```

```ruby
# lib/audio/track.rb
require 'dry-struct'

require 'audio/types'
require 'audio/artist'

module Audio
  class Track < Dry::Struct
    attribute :title, String
    attribute :performers, Types::Array.of(Artist)
  end
end
```

This solution appears quite attractive. Each class has explicitly specified dependencies, allowing for 
straightforward code writing in a Test-Driven Development (TDD) style, as each individual test runs at an impressive speed.

After some progress, you decide to incorporate this gem into a Rails application. The following code demonstrates 
the creation of a new `Artist` form:

```ruby
require 'audio/artist'

class ArtistController < ApplicationController
  def new
    @artist = Audio::Artist.new
  end
end
```

Impressive, isn't it? This approach gives you the convenience of loading only the required components. 
For instance, if you only need the `Artist`, there's no need to load the entire `Audio` module. Now, 
let's proceed with running the spec.

```ruby
RSpec.describe ArtistController, type: :controller do
  describe 'GET #new' do
    it 'returns http success' do
      get :new
      expect(response).to have_http_status(:success)
    end
  end
end
```
It appears you've once again fallen into the dependency trap:

```bash
ninitialized constant Audio::Artist::Country (NameError)
	from audio/lib/audio/artist.rb:8:in `<class:Artist>'
	from audio/lib/audio/artist.rb:6:in `<module:Audio>'
	from audio/lib/audio/artist.rb:5:in `<top (required)>'
```

Despite the unit tests in the Audio gem passing, the controller tests are still failing! Your 
unit tests functioned incidentally because the tests for the `Country` class were loaded before 
the tests for the `Artist` class.

Only after attempting to load the `Artist` class without loading the rest of the library, we 
discover that the `Country` class was not required. This oversight was the root cause of the issue.

# Autoloading to the Rescue

Can we navigate out of this dependency labyrinth? Absolutely! The solution is known as `autoload`. 
The `Kernel.autoload` method accepts two arguments: the class name and the file where this class is defined. 
By invoking `autoload`, you register a filename to be loaded the first time the class (or module) is accessed.

```ruby
# lib/audio.rb

module Audio
  autoload :Artist, 'audio/artist.rb'
  autoload :Album, 'audio/album.rb'
  autoload :Country, 'audio/country.rb'
  autoload :Track, 'audio/track.rb'
end
```

By registering all constants to be autoloaded, Ruby will handle the loading order for you. 
This means you no longer need to explicitly require dependencies for each file.

All you need to do is require the top-level module with the registered autoloads.

# When Does Autoload Not Work?

Autoload would not function as expected when you define a namespaced class in a single line:

```ruby
class Audio::Artist
  # ...
end
```

If you attempt to run the specs for `Artist`, they fail with a familiar error:

```bash
uninitialized constant Audio::Artist::Country (NameError)
Did you mean?  Audio::Country
```

Why does this occur? To understand this, you need to delve into how Ruby loads constants. Within any given section 
of code, Ruby defines a _nesting_. [Nesting](http://ruby-doc.org/core-2.3.0/Module.html#method-c-nesting) is an array 
of the classes and modules within which the current context is nested. The `Module.nesting` method can be used to 
inspect the nesting.

```ruby
module Audio
  class Artist
    puts Module.nesting.inspect #=> [Audio::Artist, Audio]
  end
end

class Audio::Artist
  puts Module.nesting.inspect #=> [Audio::Artist]
end
```

Did you notice? When you use a compact form, the nesting does not include the top-level module. 
Let's see what happens when you mix both styles.

```ruby
class Audio::Artist
  class Bio
    puts Module.nesting.inspect #=> [Audio::Artist::Bio, Audio::Artist]
  end
end
```
`Module.nesting` is used by Ruby to determine where to search for constants. It follows the sequence below:

1. If the nesting is not empty, the constant is searched within its elements, in order.
2. If the constant is not found, the algorithm moves up the ancestor chain.
3. If still not found and the first element in the nesting chain is a module, the constant is searched for in `Object`.
4. If still not found, `const_missing` is invoked on the first element. The default implementation of `const_missing` raises a `NameError`.

For our first example, when you refer to the `Country` constant from within the `Artist` class, the search happens as follows:

1. The nesting contains two elements: `Audio::Artist` and `Audio`.
2. `Audio::Artist` does not contain the `Country` definition.
3. `Audio` contains the `Country` constant, which is registered for autoload. The autoload magic occurs, and Ruby resolves the constant to `Audio::Country`.

In the second example, the nesting contains only a single element. `Audio::Artist` does not contain the `Country` definition, so `Audio::Artist.const_missing` is invoked and a `NameError` exception is raised.

You can read more about constants loading in the [Rails guides](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html#nesting).

# Autoloading and Thread Safety

Unfortunately, the `Kernel.autoload` function in Ruby is [known to not be thread-safe](https://bugs.ruby-lang.org/issues/5653). 
This lack of thread-safety can result in the same file being loaded twice if your application uses threads.

# Zeitwerk to the Rescue

[Zeitwerk](https://github.com/fxn/zeitwerk) is a Ruby library that provides a mechanism for code loading and autoloading 
in an efficient, thread-safe manner. Zeitwerk implements an autoloading mechanism that leverages Ruby's `Module#const_missing` method. Zeitwerk hooks into 
this `#const_missing` method: it infers the file path from the name of the missing constant, then loads the 
corresponding file to define the constant.

For instance, if you reference `Audio::Artist` and it's not yet defined, Zeitwerk will infer the file 
path `audio/artist.rb` from the constant name, and then load that file.

This process removes the need for explicit `require` or `autoload` statements. Instead, files are loaded on-demand 
when their constants are referenced. This approach not only makes code cleaner and more manageable, 
but it is also thread-safe, avoiding the potential issues associated with Ruby's built-in `autoload` mechanism.

Let's explore how we can leverage Zeitwerk for our "audio" gem. Firstly, you need to eliminate all explicit `require` 
and `autoload` statements. Additionally, ensure all file names adhere to Zeitwerk's conventions.

```diff
# lib/audio.rb

module Audio
-  autoload :Artist, 'audio/artist.rb'
-  autoload :Album, 'audio/album.rb'
-  autoload :Country, 'audio/country.rb'
-  autoload :Track, 'audio/track.rb'
end
```

After these steps, we need to configure Zeitwerk to perform its autoloading magic in the gem's main file.

```ruby 
# lib/audio.rb

require "zeitwerk"
loader = Zeitwerk::Loader.for_gem
loader.setup

module Audio
end
```

# Conclusion

In this article, we explored a common challenge in Ruby: managing dependencies in a modular codebase. We looked at a
hypothetical "audio" gem and saw how the order of `require` statements can lead to issues.

We first attempted to solve this problem by carefully ordering `require` statements. However, this approach was not
user-friendly and increased the startup time.

Next, we tried to explicitly require dependencies for each file. While this approach allowed faster loading, it 
reintroduced the same dependency issue when a dependency was inadvertently overlooked.

We then turned to Ruby's `autoload` method. Autoload registers a file to be loaded the first time a class or module 
is accessed, thus seemingly solving our problem. However, we found that `autoload` does not always work as 
expected when defining a namespaced class in a single line and it is also not thread-safe.

Finally, we found a solution in Zeitwerk, a Ruby library for efficient, thread-safe code loading. By leveraging 
Ruby's `Module#const_missing` method, Zeitwerk allows files to be loaded on-demand when their constants are referenced, 
eliminating the need for explicit `require` or `autoload` statements. This makes the code cleaner, more manageable, and thread-safe.

In conclusion, managing dependencies in Ruby can be a tricky task. However, with the right tools and understanding, 
it can become much more manageable. Zeitwerk offers a powerful solution for autoloading dependencies, making it an 
excellent choice for any Rubyist looking to streamline their codebase.

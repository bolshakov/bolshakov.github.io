---
layout: post
title: Loading ruby code
date: 2016-05-21 21:53:35 +0300
tags: ruby
---

Imagine you are working on a gem which provides a list of domain models for audio library.
It contains typical for this domain set of classes: `Album`, `Artist`, `Track`, `Genre`, etc.

```ruby
# lib/audio.rb
require 'audio/artist'
require 'audio/album'
require 'audio/country'
require 'audio/track'

module Audio
end

# lib/audio/album.rb
module Audio
  class Album
    include Virtus.model

    attribute :title, String
    attribute :tracks, Array[Track]
    attribute :country, Country
  end
end

# lib/audio/track.rb
module Audio
  class Track
    include Virtus.model

    attribute :title, String
    attribute :performers, Array[Artist]
  end
end

# lib/artist.rb
module Audio
  class Artist
    include Virtus.model

    attribute :name, String
    attribute :country, Country
  end
end

# lib/county.rb
module Audio
  class Country
    include Virtus.model

    attribute :name, String
  end
end
```

This is regular ruby library with all of the components loaded in the main `Audio` module.
Here goes the simplest smoke test:

```ruby
RSpec.describe Audio do
  it 'has a version number' do
    expect(Audio::VERSION).not_to be nil
  end
end
```

Trying to run it fails with the following error.

```ruby
$ bundle exec rspec
.../virtus-1.0.5/lib/virtus/const_missing_extensions.rb:14:in `const_missing': uninitialized constant Audio::Artist::Country (NameError)
  from .../audio/lib/audio/artist.rb:6:in `<class:Artist>'
  from .../audio/lib/audio/artist.rb:2:in `<module:Audio>'
```

What happened here? There is a bunch of required files in our `audio.rb` file.
Ruby interpreter reads their content one by one and evals it.
Lets look at `artist.rb` file. When interpreter encounters line #6
(`attribute :country, Country`), it tries to find `Country` constant in a constants table, but the
 constant not yet loaded.

Do you remember `audio.rb` file? Loading `Country` class goes after loading
of `Artist` class! To resolve the problem you should properly order your dependencies.

`Country` should be loaded before `Artist`, and `Track` before `Album`.

```ruby
require 'audio/country'
require 'audio/artist'
require 'audio/track'
require 'audio/album'
```

This solution looks unfriendly, and leads to dozens of carefully placed requires.
Another downside of this approach is increase of start up time, and slower feedback time from tests.

# Explicitly required dependencies

Another way to address this issue is to explicitly require dependencies for each file.

```ruby
# lib/audio.rb
module Audio
end

# lib/audio/album.rb
require 'audio/track'
require 'audio/country'

module Audio
  class Album
    # ...
  end
end

# lib/audio/track.rb
require 'audio/artist'

module Audio
  class Track
    # ...
  end
end
```

This solution looks very tempting. Each class have explicitly specified dependencies. You can
easily write your code in TDD style, since a single test runs blazing fast.

After some work you decide to use this gem in a Rails app. This code shows new Artist form:

```ruby
require 'audio/artist'

class ArtistController < ApplicationController
  def new
    @artist = Audio::Artist.new
  end
end
```

Pretty cool, right? You don't have to load all the Audio, if you need only an Artist.
Lets try to run spec.

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
You are caught into the dependency trap again.

```bash
../virtus-1.0.5/lib/virtus/const_missing_extensions.rb:14:in `const_missing': uninitialized constant Audio::Artist::Country (NameError)
  from .../audio/lib/audio/artist.rb:6:in `<class:Artist>'
  from .../audio/lib/audio/artist.rb:2:in `<module:Audio>'
  from .../audio/lib/audio/artist.rb:1:in `<top (required)>'
```

Despite of passing unit tests in an Audio gem, controller test are still failing! Your unit tests
worked accidentally, just because occasionally tests for `Country` class had been loaded before tests
for `Artist` class.

We found that you have forgotten to require `Country`, only after loading `Artist` class
without loading the rest of the library.

# Autoloading for rescue

Is there a way to avoid this dependency hell? Yes it is! And it's called `autoload`.
`Kernel.autoload` receives two arguments -- class name and the file where this class is defined. By calling
`autoload`, you register a filename to be loaded the first time the class (or module) is accessed.

Our final solution looks the following way:

```ruby
# lib/audio.rb

module Audio
  autoload :Artist, 'audio/artist.rb'
  autoload :Album, 'audio/album.rb'
  autoload :Country, 'audio/country.rb'
  autoload :Track, 'audio/track.rb'
end
```

You register all constants to be autoloaded, and ruby will manage loading order for you.
Thus, you don't have to explicitly require dependencies for each file.

You just need to require top level module with registered autoloads.

# When autoload does not work?

When you define a namespaced class in a single line:

```ruby
class Audio::Artist
  # ...
end
```

If you run Artist'a specs, it fails with familiar error:

```bash
../virtus-1.0.5/lib/virtus/const_missing_extensions.rb:14:in `const_missing': uninitialized constant Audio::Artist::Country (NameError)
Did you mean?  Audio::Country
```

Why does it happen? To answer this question you have to know how ruby loads constants.

For any given place in the code, it defines _nesting_. [Nesting](http://ruby-doc.org/core-2.3.0/Module.html#method-c-nesting) is an
array of enclosed classes and modules. Nesting may be inspected using `Module.nesting` method.

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

Have you noticed? When you use a compact form, the nesting does not contain top level module.
Let see what happens if you'll mix both styles.

```ruby
class Audio::Artist
  class Bio
    puts Module.nesting.inspect #=> [Audio::Artist::Bio, Audio::Artist]
  end
end
```

`Module.nesting` is used by ruby to determine where to search for constants. It searches in the following order:

1. If the nesting is not empty the constant is looked up in its elements and in order.
2. If not found, then the algorithm walks up the ancestor chain.
3. If not found and the first element in the nesting chain is a module, the constant is looked up in `Object`.
4. If not found, `const_missing` is invoked the first element. The default implementation of `const_missing` raises `NameError`.

For our's first example when you refer to `Country` constant from withing `Artist` class, it's searched this way:

1. Nesting contains two elements `Audio::Artist` and `Audio`
2. `Audio::Artist` does not contain `Country` definition.
3. `Audio` contains registered for an autoload `Country` constant. The autoload magic happens and ruby resolves
the constant to `Audio::Country`.

In the second example nesting contains only a single element. `Audio::Artist` does not
contain `Country` definition, so `Audio::Artist.const_missing` is invoked and `NameError` exception raised.

You can read more about constants loading in [rails guides](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html#nesting).

# Conclusion

1. To manage internal dependencies always use autoload.
2. Always use the same style when defining nested classes and modules. I prefer not to use compact style,
since it leads to more verbose constants references.

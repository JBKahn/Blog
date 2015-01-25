---
layout: post
title: "Vagrant Plugin: Back 2 Tha Hood"
description: "Writing a simple vagrant plugin to wrap an ansible command"
modified: 2015-01-25
category: articles
tags: [post]
comments: true
share: true
---

# Introduction

Once again, writing a post after a long break from updating my blog. I found my free time began to disappear when I started taking class at night after work and didn't have much time for blogging. When my co-op finished in September, I then worked part time between my classes and finished up the last of my requirements. This has been my first month back to work and getting used to having some free time again.

In school I covered a variety of topics from databases and PHP to the SOLID principles. I might try to make a post about something I covered in school over the next few weeks. I’ve also gotten back into reading and just finished [Geek Sublime: The Beauty of Code, the Code of Beauty](http://www.amazon.ca/Geek-Sublime-The-Beauty-Code/dp/1555976859) as it was a gift from a friend. The book was pretty interesting buy, yet it did seem a bit dry at times and certain passages seemed to go on forever. Anyway, on to the good stuff.

One thing I was attempting to do this week at work was to write a custom Vagrant command. For those of you unfamiliar with Vagrant, it’s a wrapper around tools like VirtualBox. It makes configuring, creating, and managing virtual machines for development a lot easier. It also provides some useful commands to make starting and using VMs easier.

When I first started researching how to make custom commands, it seemed rather daunting. The work to gain ratio seemed to be a lot higher than I'd imagined. I had thought it would be a few lines of configuration to map command X with function Y, but that was just not the case.

# Woes With Creating The Plugin

As far as I can tell, in the current version of Vagrant, there is only one way to build these and that is through their plugin system. Once again I was hopeful as I dashed into the [documentation](https://docs.vagrantup.com/v2/plugins/index.html) and found this:

> Warning: Advanced Topic! Developing plugins is an advanced topic that only experienced Vagrant users who are reasonably comfortable with Ruby should approach.

As far as I was aware, I was neither an experienced Vagrant user or reasonably comfortable with Ruby…so I was preparing for the worst. Apparently, all plugins are to defined as gems, regardless of complexity. My only experience with gems was to run `bundle install` to be able to work on this blog. I figured it couldn't be too hard to do and they must have examples of how to build them. I was sadly mistaken and confused as began researching. I found confusing documentation and half-examples with decent explanation or full examples with none. I was flummoxed.

I thought, how can there not be a complete (and easy to follow) Hello World type example for those who know little of Ruby? The documentation was the closest I could find but instead of giving you the full story, they leave that warning. I hope this will serve as an example to the next person who just wants a simple custom vagrant command.

# It Begins

The first thing you need for a gem file is the structure. Apparently you can auto-generate the skeleton, I opted to take a simple gem and make my changes. Here’s the file structure of my gem:

{% highlight text %}
Gemfile                         # The is where I put my dependancies.
Rakefile                        # The is somehow tied to building the gem.
vagrant-custom-command.gemspec  # The information/build data, like a `setup.py` file.
lib/
--- vagrant-custom-command.rb   # This is the top level file of the command.
--- custom-command/
--- --- command.rb              # The logic of the command and the argument parsing.
--- --- plugin.rb               # Defining the command and point it at `command.rb`.
{% endhighlight %}

There aren't too many files here , it was reasonably easy to make a custom gem. I'm not making use of any other ruby gems or doing anything complex.

# The Files

# Gemfile

{% highlight ruby linenos=table %}
gemspec

group :development do
  gem 'vagrant',         github: 'mitchellh/vagrant'
end
{% endhighlight %}

As you can see, I've just defined vagrant as a requirement for development, that's all. Simple file.

Rakefile
--------

{% highlight ruby linenos=table %}
require "bundler/gem_tasks"
{% endhighlight %}

I don't know why this is here, at some point I'd like to delve more into Ruby. For the moment I just accept that this works.

vagrant-custom-command.gemspec
------------------------------

{% highlight ruby linenos=table %}
# coding: utf-8
lib = File.expand_path('../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)

Gem::Specification.new do |spec|
  spec.name          = "vagrant-custom-command"
  spec.version       = 1
  spec.authors       = ["Joseph Kahn"]
  spec.email         = ["JosephBKahn@gmail.com"]
  spec.description   = %q{Awesome custom command to do something}
  spec.summary       = spec.description
  spec.homepage      = "<repo url>"

  spec.files         = `git ls-files`.split($/)
  spec.executables   = spec.files.grep(%r{^bin/}) { |f| File.basename(f) }
  spec.require_paths = ["lib"]

  spec.add_development_dependency "bundler", "~> 1.3"
  spec.add_development_dependency "rake"
end
{% endhighlight %}

That's pretty straight forward. I think I used bundler to install rake and rake to build the module so that's what this gem required for development.

vagrant-project-setup.rb
------------------------

{% highlight ruby linenos=table %}
require_relative 'custom-command/plugin'
{% endhighlight %}

This file as just going to use the following `plugin.rb` file and exists in here to make the directory structure clean.

plugin.rb
---------

{% highlight ruby linenos=table %}
module VagrantPlugins
  module CustomCommand
    class Plugin < Vagrant.plugin(2)
      name 'test'
      description 'testing'

      command "hello" do
        require_relative "command"
        Command
      end

    end
  end
end
{% endhighlight %}

All this file does it to register a command which can be called using `vagrant hello` and then will execute the logic in the `command.rb` file.

command.rb
----------

{% highlight ruby linenos=table %}
module VagrantPlugins
  module CustomCommand
    class Command < Vagrant.plugin(2, :command)
      def self.synopsis
        "Prints Hello World"
      end

      def execute
        puts "Hello World!"
        return 0
      end
    end
  end
end
{% endhighlight %}

The execute function is the code that is executed when it's called and the synopsis is how the command is described under `vagrant list-commands`.

The above snippet prints `Hello World!` to the screen and exits with a 0 status.

More Complex `command.rb`
-------------------------

{% highlight ruby linenos=table %}
module VagrantPlugins
  module CustomCommand
    class Command < Vagrant.plugin(2, :command)
      def self.synopsis
        "Calls an Ansible script to do something."
      end

      def execute
        options = {}
        options[:optionA] = false
        options[:optionB] = false

        opts = OptionParser.new do |o|
          o.banner = "Usage: vagrant blah [item]"
          o.separator ""

          o.on("-a", "--optionA", "do something conditional.") do |a|
            options[:optionA] = a
          end
          o.on("-b", "--optionB", "do something conditional.") do |b|
            options[:optionB] = b
          end
        end

        argv = parse_options(opts)

        if argv.length == 0
          puts "You must specify at least one item."
          return 1
        end
        items = argv.join(',')

        host_arg = "-i ./provisioning/hosts_file"
        private_key_arg = "--private-key=./.vagrant/machines/default/virtualbox/private_key"
        user_arg = "-u vagrant"
        playbook_path = "./provisioning/some_playbook.yml"
        extra_vars = "--extra-vars \"items=#{items} option_a=#{options[:optionA]} option_b=#{options[:optionB]}\""

        result = system("ansible-playbook #{host_arg} #{private_key_arg} #{user_arg} #{playbook_path} #{exatra_vars}")

        if result.nil?
          puts "Error was #{$?}"
          return 1
        end
        puts "Success!!"
        return 0
      end
    end
  end
end
{% endhighlight %}

In this example, I'm using the built-in option parsing to pass a list of items and two conditionals to an ansible playbook. That snippet is pretty straight forward under we get to the system call. I had to specify the vagrant user and give vagrant’s private key to have sufficient privileges to run this playbook. This makes it easier for the other developers to run these commands. They don’t need to remember the full command or where the `private_key` file is.

# Building, Testing & Usage

In order to build the file, I downloaded ruby (I use rvm) and bundler following their documentation. After installing rvm, I ran `rvm all do gem install bundler` (I only have one version of Ruby installed, otherwise I would have specified which). Once I had both of those, I ran `rake build` to build my plugin. That created a `.gem` file in a `pkg` directory it created at the root of the gem.

Once I had the gem file, I went to my vagrant folder for the project and ran `vagrant plugin install <path to gem file>`. Once it's installed, a simple `vagrant hello` will test whether the plgin works and will return an error if there was an issue. During development I would continuously run `vagrant plugin uninstall vagrant-custom-command && vagran plugin install <path to gem>`.

# Conclusion

I feel that plugin creation is a bit more complicated than it should have been for a `Hello World` case. Hiding behind a warning makes the initial plunge into vagrant plugins that much scarier and harder. While I had a few hiccups during development, it was more confusing because I thought it was more difficult than it turned out to be. Writing a gem for a simple command seems a bit like overkill. I didn't see a way to define something like `Hello World` in the `Vagrantfile` or something equally simple. I hope this post helps the next confused person overcome those hurdles and make a great plugin.

If you have any suggestions on an easier way to do this, information on the gem that I missed, or any part of this than please leave a comment below.

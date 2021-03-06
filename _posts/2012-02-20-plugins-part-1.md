---
layout: post
title: Plugins in Adhearsion 2.0 - Part 1
tags: [plugins, adhearsion]
last_updated: 2012-02-20
---

The ability to easily add reusable functionality to a framework is one of the most important features.
Plugins in Adhearsion 2.0 have been completely rebuilt to better suit the new structure and allow them to provide a wider variety of features.
Controller methods, initializer code, specific configuration, rake tasks and included generators all are possible with the new plugin classes.
In this first post we will be generating a plugin using the CLI and looking through the resulting code.

### Components in Adhearsion 1.x
The previous release of Adhearsion was deeply tied to the concept of a dialplan and related contexts, like the dialplan itself or event handling code.
That brought the component architecture to simply define a series of methods that were available in some or all of the contexts.
Components, however easy to create and use, were limited in scope and very difficult to properly test.
Adhearsion 2.0 completely removes support for components in favor of plugins.

### What is an Adhearsion 2.0 plugin?
A plugin, in Adhearsion as in many other Ruby frameworks, simply represents a collection of code, usually but not exclusively in the form of modules used as mixins.
The library is packaged as a gem to facilitate its use, reuse, and sharing with the community. 
In addition to providing classes and modules, a plugin can bring a series of extra functionalities that will be demonstrated through the post series.

Anatomy of a plugin
-------------------
The easiest way to create a skeleton plugin is to use the Adhearsion command "ahn generate".
By running the following:
{% highlight bash %}
ahn generate plugin GreetPlugin
{% endhighlight %}
a directory named greet_plugin will be created in your current location, containing a sample plugin ready to be used.
The plugin itself, being a gem, can reside anywhere, unlike components that needed to be inside the application directory.
Output for the generate command should be the following:
{% highlight bash %}
      create  greet_plugin
      create  greet_plugin/lib
      create  greet_plugin/lib/greet_plugin
      create  greet_plugin/spec
      create  greet_plugin/greet_plugin.gemspec
      create  greet_plugin/Rakefile
      create  greet_plugin/README.md
      create  greet_plugin/Gemfile
      create  greet_plugin/lib/greet_plugin.rb
      create  greet_plugin/lib/greet_plugin/version.rb
      create  greet_plugin/lib/greet_plugin/plugin.rb
      create  greet_plugin/lib/greet_plugin/controller_methods.rb
      create  greet_plugin/spec/spec_helper.rb
      create  greet_plugin/spec/greet_plugin/controller_methods_spec.rb
{% endhighlight %}

We will now go through the generated structure to highlight the important parts.

## Gem structure and plugin information
The .gemspec file and the Gemfile contain information on your plugin, required dependencies and other necessary data.
You can simply enter your personal information in greet_plugin.gemspec to have a fully functional gem.
The README is formatted as Markdown but that is by no means mandatory.
The Rakefile contains tasks that pertain to the plugin gem itself, as tasks that are to be provided to an Adhearsion application are defined elsewhere.

## Plugin files
The entry point for the plugin, as usual with gems, resides in lib/greet_plugin.rb.
It is mainly composed of requires for the plugin classes and modules.
When adding functionality to a plugin, it will need to be required here to be available.
{% highlight ruby %}
# lib/greet_plugin.rb
module GreetPlugin
  require "greet_plugin/version"
  require "greet_plugin/plugin"
  require "greet_plugin/controller_methods"
end
{% endhighlight %}
Plugins are namespaced using a module to avoid conflicts.

version.rb is used to provide the required version number for gem and general use.

### plugin.rb
Most of the plugin magic resides in plugin.rb, that holds the definitions for many important functionalities.
{% highlight ruby %}
# lib/greet_plugin/plugin.rb
module GreetPlugin
  class Plugin < Adhearsion::Plugin
    # Actions to perform when the plugin is loaded
    #
    init :greet_plugin do
      logger.warn "GreetPlugin has been loaded"
    end

    # Basic configuration for the plugin
    #
    config :greet_plugin do
      greeting "Hello", :desc => "What to use to greet users"
    end

    # Defining a Rake task is easy
    # The following can be invoked with:
    #   rake plugin_demo:info
    #
    tasks do
      namespace :greet_plugin do
        desc "Prints the PluginTemplate information"
        task :info do
          STDOUT.puts "GreetPlugin plugin v. #{VERSION}"
        end
      end
    end

  end
end
{% endhighlight %}
The plugin is automatically registered with Adhearsion though a subclassing hook.

#### Initialization with #init
The #init method defines code that is run when the plugin is loaded. Every plugin goes through two separate phases before it is ready to run.

It first gets initialized through #init, then the #run method is called.
Both are optional, but if they are defined, the mandatory arguments are the name of the plugin as a symbol and a block to provide the code to be run.

a plugin can request to be initialized before or after another, by name, using the :before and :after options passed as an hash to #init or #run.
{% highlight ruby %}
  init :greet_plugin, :after => :translation_plugin do ... #assuming we need a translation plugin to correctly greet users
{% endhighlight %}
ideally, #init will load and prepare resources, while #run would run associated processes.

#### Plugin configuration
the #config block allows a plugin to define configuration values in a customizable and self-documenting way.
Every configuration line has the key, followed by a default value, optionally followed by a :desc key to allow for a description.

"rake adhearsion:config:show" in an application directory will display all config keys provided by the core and the plugins, with their descriptions.
It will also display the environment variable that will be automatically used to load the value if it is present.

For example, the above configuration entry will be loaded from AHN_GREET_PLUGIN_GREETING if it is defined in the running environment.
A config line can also specify a transform as a Proc:
{% highlight ruby %}
   debug_level :debug, :desc => "Debug level to use when running", :transform => Proc.new { |v| v.to_sym }
{% endhighlight %}
The :transform will be used to modify the configuration value after it is read from the environment variable.
All ENV values are strings and our configuration might need a different type, or some kind of processing before using the value.

#### Rake tasks
The :tasks method allows the plugin developer to define Rake tasks to be available inside an application directory.

Task definitions follow Rake conventions.

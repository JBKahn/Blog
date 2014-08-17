---
layout: post
title: "For a Few Ansible Modules More"
description: "Writing my own ansible modules to make the most of the tool"
modified: 2014-08-17
category: articles
tags: [post]
comments: true
share: true
---

# Introduction

Once again, school has kept me busy. I've since finished taking [CSC 373 - Algorithm Design, Analysis, and Complexity](http://www.cs.utoronto.ca/~lalla/csc373/index.html), completing a final assignment and subsequent exam. I've just gotten back to working on the Ansible script that I started a month ago. In case you missed my post from last time, [Ansible](http://www.ansible.com/home) is a tool used to deploy and update applications in an easy to use language, using SSH, with no agents to install on remote systems. The specifications for Ansible are all written in YAML, making them well structured and generally easy to follow. Back at Wave, Nathan wrote [An Ansible primer](http://engineering.waveapps.io/post/80595462671/an-ansible-primer) blog post on the [Wave Engineering blog](http://engineering.waveapps.io/). A few weeks ago, I wrote a post about using ansible to setup my local machine: [Ansible or: How I Learned to Stop Wasting Time Setting Up My Computer and Script It]({% post_url 2014-07-27-ansible %}). Here, I'll be taking you through the creation of custom modules, starting with the why.

While writing the YAML for my task files, I found there were a few common tasks I wanted to do that did not have a module wrapper. The easy way to deal with those tasks is to use the [Command module](http://docs.ansible.com/command_module.html). That way I can tell Ansible to execute an arbitrary command on the machine. There are some downsides to doing this, which depend on the specific commands in question. One thing I  like about the output of Ansible is that it can tell me how many commands Ansible executed, previously executed and caused errors  (if I allow them). But yet, arbitrary commands are always run because Ansible has no way to know if they have been executed.

There are two options that I saw to address this, as I wanted to lower the number of changed tasks on repeated runs. I also wanted to prevent some commands from running multiple times. The first is to add another task and use conditional execution. I did that for two of the tasks, however it still had to register a changed task in order to do the check. This led me to look into writing my own modules and I wasn't able to find many great examples. I should have started with looking at the modules in the Ansible library, but that did not occur to me. In the end I wrote the modules with the help of the docs, the ansible IRC channel and ansible library modules. Below I'll outline writing my own module as well as conditional execution.

# Conditional Execution

There isn't a whole lot to write here, the code is  self explanatory. I wrote this to fix a screen dimming issue on my sager laptop. This particular set of tasks is also conditionally executed, covered by my last blog post.

{% highlight YAML %}
---
- name: screen dimming - alter grub file
  lineinfile: dest=/etc/default/grub regexp=^GRUB_CMDLINE_LINUX_DEFAULT= line='GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video.use_native_backlight=1"'
  sudo: yes
  register: grubfile

- name: screen dimming - update grub
  command: sudo update-grub
  sudo: yes
  when: grubfile|changed
{% endhighlight %}

# Module Writing

The first thing to work out for a new module is what arguments it will take and how to leverage Ansible to do most of the heavily lifting around it. Here's the start of my Ansible module for gconftool-2:

{% highlight python %}
from ansible.constants import mk_boolean
from ansible.module_utils.basic import *


def main():

    module = AnsibleModule(
        argument_spec={
            'key': {'required': True},
            'bool': {'type': 'bool'},
            'int': {'type': 'int'},
            'string': {'type': 'str'},
            'float': {'type': 'float'},
            'list': {'type': 'list'},
            'pair': {'type': 'list'},
            'pair-cdr-type': {'choices': ['int', 'bool', 'float', 'string']},
            'pair-car-type': {'choices': ['int', 'bool', 'float', 'string']},
            'list-type': {'choices': ['int', 'bool', 'float', 'string']}
        },
        mutually_exclusive=[
            ['bool', 'string', 'int', 'float', 'list', 'pair'],
            ['bool', 'string', 'int', 'float', 'list-type', 'pair'],
            ['bool', 'string', 'int', 'float', 'list', 'pair-car-type'],
            ['bool', 'string', 'int', 'float', 'list', 'pair-cdr-type']
        ],
        required_one_of=[['bool', 'string', 'int', 'float', 'list', 'pair']],
        required_together=[
            ['pair', 'pair-car-type', 'pair-cdr-type'],
            ['list', 'list-type']
        ],
        supports_check_mode=True
    )
{% endhighlight %}

Most of that is pretty easy to follow and it shows a few of the options Ansible handles.  It handles mutual exclusions, requiring one of a set items as well as items which are required together. Additionally, in the `argument_spec` you can also define defaults (and more, although this was all that I needed). The `supports_check_mode` is a boolean which represents that the module supports dry runs. Now that I've defined my argument structure, it's time to get onto the next piece of code.

# The Getter and Setter

{% highlight python %}
def _set_value(module, key, value, argument_type, additional_args):
    ''' Set value of setting, under `key`, using gconftool-2 to `value` of type `argument_type`'''
    return module.run_command(" ".join(['/usr/bin/gconftool-2 --set --type {} {} "{}" {}'.format(argument_type, key, value, additional_args)]))


def _get_value(module, key):
    ''' Return value of setting, under `key`, from gconftool-2'''
    return module.run_command('gconftool-2 --get {}'.format(key))[1].strip()
{% endhighlight %}

The `module` set in the above code block contains some circuital functions, one of which is `run_command`. This wraps the native python system library for running commands. It has very good error handling and provides a plethora or input options. Most of those aren't needed for simple tasks like running `gconftool-2`. All I'm doing in the code block is building the command that I'd previously supplied to the Command Module. I've already shown how to define the arguments, but now I'll show how to access them.

# Argument Accessing

{% highlight python %}
key = module.params['key']
boolean_value = module.params['bool']
string_value = module.params['string']
integer_value = module.params['int']
float_value = module.params['float']
list_value = module.params['list']
pair_value = module.params['pair']
{% endhighlight %}

Ansible makes arguments extremly easy to access. Now I'll show you a sample of how I worked with the input.

# Argument Parsing

{% highlight python%}
additional_args = ''
instance_type_mapping = {'int': int, 'string': str, 'float': float, 'bool': mk_boolean}
if boolean_value is not None:
    argument_type = 'bool'
    value = str(mk_boolean(boolean_value)).lower()
    old_value = str(mk_boolean(old_value)).lower()
...
elif float_value is not None:
    argument_type = 'float'
    value = float_value
...
elif pair_value is not None:
    if len(pair_value) != 2:
        module.fail_json(msg='A pair must be a list of length 2, {} items found.'.format(len(pair_value)))
    argument_type = 'pair'
    try:
        car_value = pair_value[0]
        car_type = module.params['pair-car-type']
        if car_type == 'bool':
            module.boolean(car_value)
            car_value = mk_boolean(car_value)
        elif not str(instance_type_mapping.get(car_type)(car_value)) == car_value:
            raise ValueError

        cdr_value = pair_value[1]
        cdr_type = module.params['pair-cdr-type']
        if cdr_type == 'bool':
            module.boolean(cdr_value)
            cdr_value = mk_boolean(cdr_value)
        elif not str(instance_type_mapping.get(cdr_type)(cdr_value)) == cdr_value:
            raise ValueError
    except ValueError:
        module.fail_json(msg='pair type `{}` or `{}` does not match the type of the contents.'.format(module.params['pair-car-type'], module.params['pair-cdr-type']))
    additional_args = '--car-type={} --cdr-type={}'.format(car_type, cdr_type)
    value = '({},{})'.format(car_value, cdr_value)
{% endhighlight %}

There are a few interesting things to note in this code block. The type specifications works for individual items but there is no equivalent for lists. When I set `pair` to type `list` it does not support type checking of particular elements or the length of the list. As such, I have to do my own error checking. Fortunately, Ansible provides a `module.fail_json(msg='blah')` to use for error reporting back to the console. There is a lot more you can provide than just a message but for my purposes that was perfect. Otherwise I'm just formatting the code to send off to my getter and setter. Now I'll provide you with the skeleton for the rest of the module.

# The Skeleton

{% highlight python %}
def main():
    #  module specification
    old_value = _get_value(module, key)
    # argument parsing

    changed = old_value != str(value)

    if changed and not module.check_mode:
        _set_value(module, key, value, argument_type, additional_args)

    module.exit_json(
        changed=changed,
        key=key,
        type=argument_type,
        value=value,
        old_value=old_value
    )

main()
{% endhighlight %}

As you can see, there isn't much more to do. That conditional is how I support check mode. All I'm doing here is calling my getter and setter and then using Ansible's `module.exit_json()` to handle to integration with Ansible. It requires the `changed` argument and the rest is custom and allows me to decide what gets printed on the screen when things change. It's also important to note that at the bottom of the file I'm calling the `main` function, which is required to run the module. The only other section of the file, which I've done solely to that I may contribute to Ansible, is the documentation at the top of the time.

# Documentation

The documentation includes ussage information as well as examples, here is an exerpt from the file:

{% highlight python %}

DOCUMENTATION = '''
---
module: gconftool-2
version_added: "post 1.7.1"
author: Joseph Kahn
short_description: alter gconftool-2 controlled settings.
description:
   - Set the value of a gconftool-2 controlled setting using a key and a string, an integer, a boolean, a pair or a list.
options:
  key:
    description:
      - The key of the gconftool-2 setting to change.
    required: true
  bool:
    description:
      - The boolean value to set the key to.
    required: false
...
'''

EXAMPLES = '''
# Set string value
- gconftool-2: key=/apps/gnome-terminal/global/default_profile string=base-16-monokai-dark

# Set bool value
- gconftool-2: key=/apps/gnome-terminal/profiles/base-16-monokai-dark/use_system_font bool=false

# Set pair value
- gconftool-2: key=/path/to/something pair-car-type=int pair-cdr-type=string pair=1,'Joseph Kahn'
'''
{% endhighlight %}

# Usage

Using your custom modules is easy to do. Once you create a `custom_modules` directory, you can use them within Ansible. All you need to do is provide the `module-path` argument like so `--module-path custom_modules`.

Once you've done that, you can use it like any other Ansible module in your task files. i.e.
{% highlight YAML %}
- name: base16 - set default terminal profile
  gconftool-2: key=/apps/gnome-terminal/global/default_profile string=base-16-monokai-dark
{% endhighlight %}

# Testing

The Ansible repo provides an executable that allows for easy module testing from the command line. It can be invoked with a command like:

`~/ansible/hacking/test-module -m ./gconftool-2 -a "key=/path/to/stuff pair-car-type=int pair-cdr-type=string pair=1,'Joseph Kahn'"`

You can also use breakpoints if you provide the debugger used, i.e.

`~/ansible/hacking/test-module -m ./gconftool-2 -a "key=2 list=1,1,true,1 list-type=bool" -D ipdb`

Running the testing module provides to full generated source file, as well as the standard output information. The output file can be run manually without the use of additional arguments.

# Conclusion

Writing a simple Ansible module wasn't very difficult and requires little code to get the basics up and running. Even with all the error handling and verification done above, the module code is only 117 lines. While I did this for convenience, it's easy to see how making modules simple to write can help make great provisioning scripts. These modules are relatively easy to follow and integrate seamlessly with the rest of the modules.

# That's It

That's all you need to get a custom module up and running and reporting changes. I'll leave you with a few relevant links:

*  [The Repository](https://github.com/JBKahn/provisioning-local) (The custom modules are in the `ansible modules` folder).
*  [Ansible Modules](http://docs.ansible.com/list_of_all_modules.html).

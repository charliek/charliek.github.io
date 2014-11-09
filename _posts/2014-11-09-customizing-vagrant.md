---
layout: post
title:  "Allowing Users to Customize Vagrant"
date:   2014-11-09 15:03:30
categories: vagrant
---

Vagrant is an incredible tool to help get developers setup quickly, and provide repeatable environment similar to production. However everybody has different tools and tricks to make themselves more productive, and not having these within the vagrant environment can slow them down. These things include installing programs (`vim`, `emacs`, `htop`), setting up your custom cofiguration (`.vimrc`, `.emacs`), or tweaking vagrant's memory or CPU settings. To allow people to make these customizations I've successfully used the below techniques.  I've found enabling these items allow people to be more comfortable with doing a vagrant destroy and not feeling like they will lose any custom setup, and can also allow new features to be tested before pushing them to the larger team.

None of the techniques here are real advanced, but they were non-obvious to me when setting up my first vagrant file so I figured I would share.

## Customization via Environment Variables

If you want to allow users to customize parts of a virtual machine, or have different flags that can be passed into your provisioning scripts using environment variables is a great way to do it. With environment variables you can allow your users to customize the number of CPUs, the amount of memory to allocate, and even provide feature flags to the provisioning script to enable experimental behavior. You can do this as shown below:

{% highlight ruby linenos %}
Vagrant.configure("2") do |config|

    # Environment variable customizations can be setup this way. Vagrantfiles
    # are just ruby files so you can do powerful things.
    num_cpus = ENV['VAGRANT_CPU']
    mem_size = ENV['VAGRANT_MEM']
    beta_tester = ENV['VAGRANT_BETA'] == 'true'

    if beta_tester
        # Setup beta features here
    else
        ENV['VAGRANTMANAGER_BETA'] = 'false'
    end

    config.vm.box = 'ubuntu/trusty64'

    config.vm.provider :virtualbox do |vb|
    	# The vm setup can be customized based on the environment variables
    	# as could mount points, ip addresses, or anything else.
        vb.customize ["modifyvm", :id, "--memory", mem_size ? mem_size : 2048]
        vb.customize ["modifyvm", :id, "--cpus",  num_cpus ? num_cpus : 2]
    end

    config.vm.provision :shell, :inline => <<-SH
      # By using inline scripts you can also pass values into the provisioning
      # step where they can be leveraged by your provisioning scripts. Below
      # you see an example of how I've leveraged this to allow features to be
      # rolled out slowly using an opt in beta tester environment variable.
      set -eux
      export VAGRANTMANAGER_BETA=#{ENV['VAGRANT_BETA']}

      # This will call the startup.sh file to run the provisioning scripts.
      # I prefer doing provisioning using scripts since production will almost
      # always do this as well. Then these scripts can call into chef or some
      # other config management tool if advanced things are necessary.
      source /vagrant/startup.sh
    SH
end
{% endhighlight %}

As you can see above you can enable powerful customizations using environment variables. You can also use this technique to pass in user credentials that you might not want to hard code into your vagrant box.

## Customization via Mount Points

Environment variables are great but they don't allow you do things like installing custom packages or configuration.  To allow for this one trick is to check for a custom local path, and if that path is there you can leverage it as an extension point.  To do this you might do something like:

{% highlight ruby linenos %}
Vagrant.configure("2") do |config|

  # ... Snip ...

  # mount a custom script if the specific folder is found on the host disk.
  # Note that you could make this more flexible using environment variables
  # if desired.
  bootstrap_dir = File.expand_path("~/vmbootstrap")
  if File.directory?(bootstrap_dir)
    # This will mount the users ~/vmbootstrap directory into the /opt/vmbootstrap
    config.vm.synced_folder bootstrap_dir, "/opt/vmbootstrap"
  end

  config.vm.provision :shell, :inline => <<-SH
    set -eux
    source /vagrant/startup.sh
  SH
end
{% endhighlight %}

Then in your `startup.sh` file you can provide a custom execution hook into the provisioning script by doing something like:

{% highlight bash linenos %}
#!/bin/bash -eux

# Do what ever provisioning you want here
# ...

# Call into your users custom setup script if it exists. This allows your
# users to have scripts that run at provisioning time, which allows them
# them install editors, user config files, debugging tools, or whatever
# makes them happy. Since the whole vmbootstrap folder is mounted multiple
# files can be put into that file and copied around as well.
bootstrap_script='/opt/vmbootstrap/setup'
if [ -x $bootstrap_script ] ; then
  echo 'running custom setup script'
  sudo $bootstrap_script
fi

{% endhighlight %}

Naturally this technique can be done an infinite number of ways, but the example above is a simple approach that has served me well.

## Conclusion

The two techniques above can be composed and extended to provide very complex customizations, but they can also be abused to make Vagrantfiles that are hard to follow and environments that don't work. At first I hesitated building these hooks into the team's vagrant images, but I've found that trusting people with these hooks has worked out well and made everybody happier. Most people probably don't know the hooks exist which is just fine, but these customizations allow the power users and tinkerers to be more productive. I find these customization points allow Vagrant to work better for me.



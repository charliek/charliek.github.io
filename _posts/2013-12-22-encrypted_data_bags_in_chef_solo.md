---
layout: post
title:  "Encrypted Data Bags in Chef Solo"
date:   2013-12-22 05:35:54
categories: chef
---

## Why

Generally attributes should be used to customize environments in chef, however for sensitive information encrypted data bags are a great tool. The built in support for encrypted data bags in chef solo is not super well documented so I figured I'd document my findings for myself and others.

## Setup

First you will need to install the necessary `knife` plugins that will make things easier.

{% highlight bash %}
sudo /opt/chef/embedded/bin/gem install knife-solo
sudo /opt/chef/embedded/bin/gem install knife-solo_data_bag
{% endhighlight %}

Then you will need to generate a key that will be used for encryption. Make sure to keep this secret.

{% highlight bash %}
openssl rand -base64 512 | tr -d '\r\n' > /tmp/encrypted_data_bag_secret
{% endhighlight %}

Make sure you have the `EDITOR` environment value set to the editor of your choice.  Set this in your .bash_profile or just set it for your current shell.

{% highlight bash %}
export EDITOR=vim
{% endhighlight %}

## Creating and Editing Databags

Now that everything is installed you will need to tell chef where to find it by adding this info into your `solo.rb` file.

{% highlight ruby %}
file_cache_path '/opt/chef-solo/cache'
cookbook_path [ '/opt/chef-solo/website/chef/cookbooks']
role_path root '/opt/chef-solo/website/chef/roles'
data_bag_path '/opt/chef-solo/website/chef/data_bags'
encrypted_data_bag_secret '/tmp/encrypted_data_bag_secret'
log_level :info
{% endhighlight %}

Now you can use this file to create and edit your encrypted data bags.  For example to create a _users_ data bag in the _site_ folder use the below command. When run an editor window will launch where you can edit your json, which will be saved as an encrypted data bag on close.

{% highlight bash %}
knife solo data bag create site users -c /opt/chef-solo/solo.rb
{% endhighlight %}

Once created to edit data bags issue the edit command:

{% highlight bash %}
knife solo data bag edit site users -c /opt/chef-solo/solo.rb
{% endhighlight %}

Now you should be able to easily create and edit encrypted data bags easily.

## Using Encrypted Data Bags in Chef

To use the data in the data bag from chef you can use the standard encrypted data bag apis. In the example below the `user_db` variable will be the decrypted json data from the data which can be looped over as standard json.

{% highlight bash %}
user_db = Chef::EncryptedDataBagItem.load("site", "users")
users = user_db["users"]

users.each do |u|
 username = u['username']
 Chef::Log.info("Setting up user '#{username}'")
end
{% endhighlight %}

I hope this helps.

Also thanks to the below sites which helped me track this information down:

* [distinctplace.com](http://distinctplace.com/infrastructure/2013/08/04/secure-data-bag-items-with-chef-solo/)
* [ed.victavision.co.uk](http://ed.victavision.co.uk/blog/post/4-8-2012-chef-solo-encrypted-data-bags)


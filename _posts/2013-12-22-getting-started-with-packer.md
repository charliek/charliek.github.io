---
layout: post
title:  "Getting Started with Packer"
date:   2013-12-22 07:39:41
categories: packer
---

## Introduction

[Packer][packer] is a very specialized tool for packaging cloud images. The concepts used in packer are simple but powerful, and seems to be the best tool for building images in an automated fashion. Basically the tool takes an existing cloud image, starts up a cloud vm with that image, runs various provisioners that you define on the vm, burns and image, and then shuts everything down. The end result is a new image that can be used for cloud deployments.

The documentation on the [site][packer] is better than what I could write here, so I will focus on a working example and the issues I ran into to supplement the official documentation.

[packer]: http://www.packer.io/

## Chef Example with Multiple Builder

The example I'm going to show will upload a tarball of chef code, install chef, and then do a chef-solo run to build up the image. Note that packer does have a [native chef-solo provisioner](http://www.packer.io/docs/provisioners/chef-solo.html) so you may choose to use that instead. I chose this method of running chef because I wanted to know exactly how everything was installed and run to ease debugging.

One feature of packer that may be interesting to you is that it can build multiple types of machine images. In this example I'll show how I build the following image types: Amazon (instance based), Amazon (ebs based), Rackspace (Open Stack), and Digital Ocean. Support for other image types are available (VMWare, Virtual Box) and Google Compute Engine appears to be added in the next version.

Packer is a simple binary that takes in a json file with build instructions and then builds an image.  Unlike Chef or Puppet Packer really just acts on this one file and assumes you will be uploading assets and running scripts via provisioners. So the below packer json is all that packer needs to run, and you are expected to write the files to be executed on the machine.

The basic working example that I have started with is below:

{% highlight json linenos %}
{
  "builders": [
    {
      "access_key": "{{user `aws_access_key`}}",
      "account_id": "{{user `aws_account_id`}}",
      "ami_name": "blog-instance-{{timestamp}}",
      "bundle_upload_command": "sudo EC2_AMITOOL_HOME=/opt/ec2-ami-tools/ec2-ami-tools-1.4.0.9 -n /opt/ec2-ami-tools/ec2-ami-tools-1.4.0.9/bin/ec2-upload-bundle -b {{.BucketName}} -m {{.ManifestPath}} -a {{.AccessKey}} -s {{.SecretKey}} -d {{.BundleDirectory}} --batch --retry",
      "bundle_vol_command": "sudo EC2_AMITOOL_HOME=/opt/ec2-ami-tools/ec2-ami-tools-1.4.0.9 -n /opt/ec2-ami-tools/ec2-ami-tools-1.4.0.9/bin/ec2-bundle-vol -k {{.KeyPath}} -u {{.AccountId}} -c {{.CertPath}} -r {{.Architecture}} -e {{.PrivatePath}}/* -d {{.Destination}} -p {{.Prefix}} --batch --no-filter",
      "instance_type": "m1.small",
      "region": "us-east-1",
      "s3_bucket": "{{user `aws_s3_bucket`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "source_ami": "ami-955b79fc",
      "ssh_username": "ubuntu",
      "tags": {
        "Application": "blog",
        "OS_Version": "Ubuntu",
        "Release": "13.04"
      },
      "type": "amazon-instance",
      "x509_cert_path": "{{user `aws_x509_cert_path`}}",
      "x509_key_path": "{{user `aws_x509_key_path`}}",
      "x509_upload_path": "/tmp"
    },
    {
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "us-west-2",
      "source_ami": "ami-8c0663bc",
      "instance_type": "m1.small",
      "ssh_username": "ubuntu",
      "ami_name": "blog-ebs-{{timestamp}}"
    },
    {
      "type": "openstack",
      "username": "{{user `rackspace_user`}}",
      "password": "{{user `rackspace_password`}}",
      "provider": "rackspace-us",
      "region": "ORD",
      "ssh_username": "root",
      "image_name": "blog-{{timestamp}}",
      "source_image": "80fbcb55-b206-41f9-9bc2-2dd7aac6c061",
      "flavor": "2"
    },
    {
      "type": "digitalocean",
      "client_id": "{{user `digitalocean_client_id`}}",
      "api_key": "{{user `digitalocean_api_key`}}",
      "image_id": 350076,
      "region_id": 4,
      "size_id": 66,
      "droplet_name": "blog-{{timestamp}}",
      "snapshot_name": "blog-image-{{timestamp}}"
    }
  ],
  "provisioners": [
    {
      "inline": [
        "sleep 30"
      ],
      "type": "shell"
    },
    {
      "destination": "/tmp/chef-setup.tar.gz",
      "source": "chef-setup.tar.gz",
      "type": "file"
    },
    {
      "destination": "/tmp/encrypted_data_bag_secret",
      "source": "{{user `chef_key_location`}}",
      "type": "file"
    },
    {
      "inline": [
        "sudo mkdir -p /opt/chef-solo/website",
        "cd /opt/chef-solo/website",
        "sudo tar -zxf /tmp/chef-setup.tar.gz -C /opt/chef-solo/website",
        "sudo rm /tmp/chef-setup.tar.gz",
        "sudo /opt/chef-solo/website/util/vagrant-bootstrap.sh /opt/chef-solo/website/chef/solo-nodes/blog-prod.json",
        "sudo docker stop $(sudo docker ps -q)",
        "sudo service docker stop",
        "rm /tmp/encrypted_data_bag_secret"
      ],
      "type": "shell"
    },
    {
      "inline": [
        "sudo apt-get install -y ruby1.8 ruby1.8-dev make unzip build-essential rsync",
        "sudo mkdir -p /opt/ec2-ami-tools",
        "sudo chown -R ubuntu /opt/ec2-ami-tools",
        "wget -r 3 -O /opt/ec2-ami-tools/ec2-ami-tools-1.4.0.9.zip http://charliek-tools.s3.amazonaws.com/ec2-ami-tools-1.4.0.9.zip",
        "cd /opt/ec2-ami-tools",
        "sudo unzip ec2-ami-tools-1.4.0.9.zip",
        "sudo rm ec2-ami-tools-1.4.0.9.zip",
        "sudo chown -R root /opt/ec2-ami-tools",
        "sudo sync"
      ],
      "only": ["amazon-instance"],
      "type": "shell"
    }
  ],
  "variables": {
    "chef_key_location": "",

    "aws_access_key": "",
    "aws_account_id": "",
    "aws_s3_bucket": "",
    "aws_secret_key": "",
    "aws_x509_cert_path": "",
    "aws_x509_key_path": "",

    "rackspace_user": "",
    "rackspace_password": "",

    "digitalocean_client_id": "",
    "digitalocean_api_key": ""
  }
}
{% endhighlight %}

Executing the example can be done on the command line using the command:

{% highlight bash linenos %}
packer build -var-file="$HOME/.packer/env.json" -only=amazon-ebs build.json
{% endhighlight %}

The above command assumes you have an `env.json` file at the specified path, and that you only want to run the `amazon-ebs` builder for this execution.  See the [packer documentation](http://www.packer.io/docs/command-line/build.html) for all the command line details.

The focus for this post is the packer build and not the chef, but to understand what the chef code is doing you may want to dig further. If so the core of the work is done in the [vagrant-bootstrap.sh](https://github.com/charliek/website-docker/blob/6bfd0eaf9d0ddf0019b26f3eed855eca418107f2/util/vagrant-bootstrap.sh) script which installs and runs chef, and the chef-setup.tar.gz tarball which is basically my [website-docker](https://github.com/charliek/website-docker/) repo. This should hopefully look like a pretty basic chef solo run.  Note that all of this code is really more for my learning and you should probably use with caution.

Included in this small example are some items I'll call out further since it took me a bit of time to get right.

## Variables

In the example above you will see many references to variables. In the example I use variables to allow the packer builder be checked into source control with out exposing any sensitive information. The `env.json` file referenced on the command line contains all the sensitive information and looks something like this:

{% highlight json linenos %}
{
  "chef_key_location": "/Users/cknudsen/projects/.../chef.key",
  "aws_access_key": "*************",
  "aws_secret_key": "*************",
  "aws_account_id": "*************",
  "aws_x509_cert_path": "/Users/cknudsen/.../aws_cert.pem",
  "aws_x509_key_path": "/Users/cknudsen/.../aws_key.pem",
  "aws_s3_bucket": "*************",

  "rackspace_user": "*************",
  "rackspace_password": "*************",

  "digitalocean_client_id": "*************",
  "digitalocean_api_key": "*************"
}
{% endhighlight %}

When packer is run all the items that look like ```{{user `aws_access_key`}}``` will be substituted at run time with the values in the env.json file.  Also note that the values shown in this file must match the values in the variables section of the file used for building.

## Digital Ocean Values

The Digital Ocean packer documentation is pretty good, however the main issue I had was understanding what to enter for the `image_id`, `region_id`, and `size_id` values. The documentation references the [API](https://cloud.digitalocean.com/login) and the [Tugboat gem](https://github.com/pearkes/tugboat) to find these values, however I found it much easier to pull these ids out of the html of the Digital Ocean site. Basically go to build a machine manually, inspect the html for the different button options, and then us the IDs found there as the values for the respective packer values. In my tests this matched up the to the expected values every time and was much faster to lookup.

## Rackspace (openstack) Values

The openstack documenation is actually not very good in describing what the different values should be. Once you get the correct values in place everything works great, but getting to that point can be frustrating.

Currently the openstack builder [only works for rackspace](https://github.com/mitchellh/packer/issues/480) and is not a general purpose openstack builder yet. This was what I wanted, but is obviously not great for other open stack deploys. Once you know this knowing the values to enter is easier.

To figure out what the `source_image` and `flavor` values should be I had to use [the rackspace api](http://docs.rackspace.com/servers/api/v2/cs-devguide/content/ch_general_api_information.html), and inspect the raw json returned. To start you need to get a auth token which can be done with a post to the below url:

Post json to: https://identity.api.rackspacecloud.com/v2.0/tokens

Post data:

{% highlight json linenos %}
{
  "auth": {
     "passwordCredentials":
        { "username":"USERNAME", "password":"PASSWORD"}
  }
}
{% endhighlight %}

This should give you a `access.token.id` and an `access.token.tenant.id` needed for following calls, as well as a bunch of urls that should be used for future api calls. This initial call also shows all the different `region` values than can be used. I'm going to use `ORD` which is the Chicago DC. Next you will look up the `source_image` and `flavor` via the below api calls:

* For the `source_image` use the the url: https://ord.servers.api.rackspacecloud.com/v2/`access.token.tenant.id`/images
* For the `flavor` use the the url: https://ord.servers.api.rackspacecloud.com/v2/`access.token.tenant.id`/flavors

For both calls use the headers:

* X-Auth-Project-Id == `access.token.tenant.id`
* X-Auth-Token      == `access.token.id`

The responses these calls will show you what `source_image` and `flavor` values that can be used.

Note that the [Rackspace API documentation](http://docs.rackspace.com/servers/api/v2/cs-devguide/content/ch_general_api_information.html) can be used if the above calls don't work for you.

## AWS Instance Backed Builder

The instance based AWS builder is much tricker than the other builder listed here. As you can see in the example you will need to install your own ec2-ami-tools for the builder to work. This can be done by uploading the zip and then installing the packages required to run the tools.  These tools will then be used to package your AMI. Note that you will need Ruby 1.8 as newer versions are not yet supported by the ami-tools, and I also had to override the `bundle_upload_command` and `bundle_vol_command` values of the builder so that the proper `EC2_AMITOOL_HOME` environment variable was set. Once all this was done I was able to successfully build an instance backed AMI.

One big caveat to the above is that I still have some issues with this builder depending on what is being installed. I'm still trying to track down what exactly breaks it, but this builder has been the slowest and most fragile for me. I've resorted to using ebs backed images for now due to some of these issues. I hope to update this post with a fix once I am consistently building instance back AMIs.

## Conclusion

Packer is definitely a tool worth checking out if you want to build cloud images. It is being rapidly developed, has pretty good documentation, and is an easy to understand tool with a focused goal.

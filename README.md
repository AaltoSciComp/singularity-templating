# Singularity templating

This repository contains scripts that:

1. Allow creation and filling of templates describing 
   [Singularity](http://singularity.lbl.gov) definitions
2. Allow creation and filling of templates describing Lmod module files
   (not yet implemented)
3. Allow creation and filling of templates describing Jenkins jobs that build 
   images and modules based on these definitions.

By utilizing inheritance of templates and inclusion of 'snippets' - small 
instructions that install specific software - complex images can be templated
with minimal instructions.

There are two possible use case supported:

1. Creating a full deployment environment, where pushing specialized
   fill.yml information and templates will send a web hook to Jenkins
   that will clone the repository and initiate a full build of the
   image.
2. Creating a limited deployment, where Singularity images are built
   on local machine.

## Installation

Requirements for singularity-templating are:

1. [Singularity installation](http://singularity.lbl.gov/install-linux)
2. Jinja2 and PyYaml for Python (packages python-jinja2 and python-yaml for 
   Ubuntu)
3. sudo rights

This is all, if you want to create local images.

If you want to use the Jenkins setup, you need:
1. Jenkins server on a machine where you have access through SSH keys.
2. SSH deployment key for Jenkins.
3. For web hooks, the server should be visible from your GitLab instance.
4. This repository should be cloned to /opt/singularity-templating in order to
   create a trusted source for 'create-singularity-image'-command.
5. Jenkins should be given sudo rights to commands 
   '/opt/singularity-templating/bin/create-singularity-image' and 'singularity'.
6. (Optional) Install
   [build timestamp plugin](https://wiki.jenkins.io/display/JENKINS/Build+Timestamp+Plugin).

You should create a Jenkins project that:
1. Uses Git for source code management with previously described deployment key.
2. Has a build trigger for source code changes in arbitrary template repository
   with secret token X.
3. Has these job steps:
   1. ```sh
      !/bin/bash
      echo 'Creating singularity image...'
      sudo /opt/singularity-templating/bin/create-singularity-image -s 6144 "$JOB_NAME"-"$BUILD_TIMESTAMP".img
      ```
   2. ```sh
      #!/bin/bash

      git clone git@github.com:AaltoScienceIT/singularity-templating.git
      ```
   3. ```sh
      #!/bin/bash

      echo 'Creating definitions...'

      singularity-templating/bin/fill-template > "$JOB_NAME".def

      echo 'Bootstrapping the image...'

      sudo singularity bootstrap "$JOB_NAME"-"$BUILD_TIMESTAMP".img "$JOB_NAME".def
      ```
4. (Optional) Have a post-build action that archives *.img files.

This needs to be done once. After that one can copy the same project structure 
with bin/init-jenkins-project for each new template repository.

## Usage for local builds

Usage is quite simple. Clone this repository to some location
e.g. ~/singularity-templates:
```sh
cd ~
git clone git@github.com:AaltoScienceIT/singularity-templating.git
```

Add bin-folder to path:
```sh
export PATH=~/singularity-templating/bin:$PATH
```

You need a fill.yml that describes has parameters for templates and tells where
to look for further templates. There exists a fill.yml.example that you can copy
for testing:
```sh
cp fill.yml.example fill.yml
```
If you cloned singularity-templating to some other folder, change 
lookup\_dirs-list in fill.yml accordingly.

After this you can fill the template into Singularity definitions file with
```sh
fill-template > Aalto-Ubuntu.def
```

Then you can build the Singularity image normally:
```sh
sudo singularity create -s 2048 Aalto-Ubuntu.img
sudo singularity bootstrap Aalto-Ubuntu.img
```

## Usage for Jenkins builds

Clone this repository to some location
e.g. ~/singularity-templates:
```sh
cd ~
git clone git@github.com:AaltoScienceIT/singularity-templating.git
```

Add bin-folder to path:
```sh
export PATH=~/singularity-templating/bin:$PATH
```

Then you should create a git repository in your GitLab instance and 
clone it to a folder.

Go to the repo folder and run 
```sh
init-jenkins-project
```

This script will ask you question like the address of the Jenkins instance
and the name of Jenkins project you want to use as a template for this
project. After you have answered these question a new Jenkins project
should appear with same kind of web hooks as the previous project.

After you have set in GitLab that Jenkins deployment key should be granted
read access and there is a web hook with secret key, you can simply add the
fill.yml to repository and commit changes. This should initialize the
web hook that instructs Jenkins to build a singularity image based on
instructions given in fill.yml and possible templates.

# Templating scheme

## Writing templates

Templates are written using the templating language provided by 
[Jinja2](http://jinja.pocoo.org/). This repo contains some sample
templates that can be used as a starting point.

Base idea behind the scheme is that lower level instructions (e.g.
operating system and package installation, mountpoint creation) are shared 
across multiple different software in base templates.

These base templates (e.g. aalto-ubuntu-base.template) are then extended
with other templates, which fill certain 'block' areas in lower level
templates. For example, aalto-ubuntu-docker.template extends 
aalto-ubuntu-base.template by filling the bootstrap block with instructions
on how to do Singularity bootstrap starting from Docker image. Meanwhile,
aalto-ubuntu-image.template extends aalto-ubuntu-base.template to have
instruction on how to do the same with debbootstrap.

One can then continue this inheritance by writing an another template that can
extend aalto-ubuntu-image.template with 'run','setup' and 'post' instructions.

The inheritance is single inheritance, so each template needs to follow
the next. Thus complex software suites should be written through 'snippets'.
Snippets are small parts of code that do a small installation of some kind, when included with
```
{% include "<snippet name>.snippet" %}
```
For example, python.snippet install commonly used python packages and optional
packages through pip. Do note extra intendation in snippets. As these snippets
are straight up included into their corresponing blocks, there needs to be 
correct amount of intendation.

When you call fill-template, fill.yml is opened and read into memory. There
are three blocks:
1. 'singularity_parameters' for singularity template writing.
2. 'module_parameters' for module template writing.
3. 'jenkins_parameters' for Jenkins job template writing.
For all parameters, see 
```sh
fill-template --help
```
An additional secrets.yml is also read in if present. This file can contain
parameters you do not want to include in your repository like licence keys,
personal information, ssh-keys etc.

The template named by `template_name` is read and `lookup_dirs` are searched
for other templates and snippets. After this parameters are filled in. Jinja2
has its own coding structure with possibility of if-else-logic, for loops and
access to dictionaries. These can be utilized freely. You can always use
`fill-template` to check what the output is and see if the result looks fine.

Projects that utilize this scheme should be organized as follows:
- fill.yml - contains parameters for all templates
- secrets.yml - secret parameters that should not be added to the repository
- snippets - folder for snippets
- templates - folder containing Singularity definition templates
- modtemplates - folder containing Lmod module templates
- jobtemplates - folder containing Jenkins jobs templates

Most projects most likely do not require secrets.yml nor jobtemplates.

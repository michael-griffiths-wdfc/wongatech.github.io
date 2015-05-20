---
title: Jenkins Job Builder Ruby DSL
author: huyle
---

Wonga has pretty complex continous integration system in which we employ Jenkins to build SOAP components and run different test suites. As we favor infrastructure as code, we want the creation of Jenkins jobs configuration done using code instead of using Web UI.

Initially we used the [Openstack Job Builder](http://ci.openstack.org/jenkins-job-builder/), which allows us to create a job's configuration in YAML. Maintain jobs configuration in YAML is huge step forward comparing to Web UI, however we soon discovered that YAML has several limitations. It lacks both composition and abstraction causing tedious repetitive work and maintenance issues. This is driving factor for the birth of Ruby Job Builder DSL.

Ruby Jobs Builder DSL is designed as Ruby internal DSL, so it offers full fledged programming experience at the same time concise, focused vocabularies for job's creation. It generates XML job configuration files and deploy directly into Jenkins Server. The library is easy to use and has minimal dependencies.

**Hello World Job**

An example below shows how to create and deploy a simple job.

Step 1 - Create a ruby file e.g. `hello-world.rb` with the following content

    $ cat hello_world.rb
    require 'rubyjobbuilderdsl'
    builder = JenkinsJob::Builder.new
    builder.freestyle 'hello-world' do
      shell 'echo hello world'
    end
    JenkinsJob::Deployer.new(builder).run

Step 2 - Deploy it

    $ cat localhost.ini
    [jenkins]
    url=http://localhost:8080/
    user=jenkins-jobs
    password=7bb352e4f3da683c17247f3abf88f47c

    $ ruby hello_world.rb --deploy --config-file=localhost.ini
    deploying hello-world

Ruby Jobs Builder DSL is extensively used within Wonga. Its DSL supports a handful number of plugins that meet the need of different teams. It is used to create jobs ranging from simplest one to the most complex pipeline involving hundreds of jobs.

**Flexibility**

Ruby Jobs Builder DSL quite easy to extends, in Wonga we build an other library on top of it to provide template for creation of jobs that are specific to our technology stacks and work workflow. The following example shows how to create a template then subsequently use it for create a job.

    $ cat hello_moon.rb
    require 'rubyjobbuilderdsl'
    builder = JenkinsJob::Builder.new do
      def say(what)
        freestyle "say-#{what}" do
          shell "echo #{what}"
        end
      end
    end

    builder.say 'world'
    builder.say 'moon'

    JenkinsJob::Deployer.new(builder).run

Ruby Jobs Builder DSL goes beyond job's creation, it can delete a job, wipe out workspace, trigger a build or even run a given groovy script on Jenkins server. This allows us to have a script that not only create jobs but also configure Jenkins as well as perform other administrative activities bringing automation to different level.

    $ cat hello_mars.rb
    require 'rubyjobbuilderdsl'
    JenkinsJob::Deployer.new.run do
      disable_job 'hello_mars'
      wipeout_workspace 'hello_mars'

      groovysh <<-EOS
        import hudson.tasks.Mailer
        desc = Jenkins.instance.getDescriptorByType(Mailer.DescriptorImpl.class)
        desc.smtpHost = 'mail.wonga.com'
        desc.defaultSuffix = '@wonga.com'
        desc.save()
      EOS

      enable_job 'hello_mars'
      trigger_job 'hello_mars'
    end


**References**

* http://ci.openstack.org/jenkins-job-builder/
* https://github.com/jenkinsci/jenkins.rb
* https://github.com/jenkinsci/job-dsl-plugin/wiki/Job-DSL-Commands

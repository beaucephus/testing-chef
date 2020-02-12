# testing-chef
## How to combine Test Kitchen, Docker, and InSpec to make testing Chef code easy

Chef is a very powerful infrastructure-as-code (IAC) configuration management tool that can enable our teams to build
and configure systems quickly, precisely, and in parallel. That said, writing automated tests for Chef code can get
tricky when we consider the fact that we need to stand up an entire operating system with storage, networking, etc...
just to get a good test. There is an in-memory solution to this problem called ChefSpec which is an extension of RSpec,
but we can only mimic so much in memory. This is where Test Kitchen comes in. Test Kitchen allows us to seamlessly
deploy our Chef code to a containerized instance of our OS of choice and test how our code really affects the state
of our instance.

## Install the tools

We'll need Docker and the Chef Workstation (previously Chef Development Kit) to get started. We can follow the
instructions on the respective tool's website or use our package manager of choice to install these tools in our
development environment.

In addition, this example using several reference files from the `beaucephus_nginx` cookbook in this repository. We'll
clone this repository to our development environment to access those files.

## Create the cookbook

The cookbook is a grouping of directories, metadata, and Chef code that the Chef run-time will load, interpret, and
execute. We'll start by generating a new empty cookbook using standard naming conventions, usually "\<company\>_\<tool\>".
This cookbook is going to install nginx using the nginx community cookbook. This pattern of wrapping a community
cookbook with our organization-specific cookbook is very common in the Chef community. We generate a new cookbook with
the Chef Workstation command `chef`.

#### Note: Use a different company name for the sake of this example.

```
chef generate cookbook beaucephus_nginx
```

This generates the scaffolding, all the directories, metadata, and files that we need to author a cookbook. However,
before we write any Chef code to install and configure nginx, we should finish setting up our test environment and
write some tests.

## Spin up the environment

We'll be using Test Kitchen which comes as part of the Chef Workstation package to initiate our tests. We'll want to
configure our Test Kitchen to use dokken which is a fast, lightweight driver that uses Docker containers for the
environment in which to run our code.

### Configure Test Kitchen

The main configuration file for Test Kitchen is the kitchen.yml, and there is plenty to unpack in this file that we
won't get into in this example. For now, we'll simply copy the kitchen.yml from the `beaucephus_nginx` cookbook into the
root directory of our cookbook. Afterward, we should now be able to use `kitchen list` to view the instances that we
have available to spin up.

```
$ kitchen list
Instance          Driver  Provisioner  Verifier  Transport  Last Action  Last Error
default-centos-7  Dokken  Dokken       Inspec    Dokken     Created      <None>
```

### Spin up the container instance

Now lets spin up our centos instance with the following command.

```
kitchen create
```

This might take a few moments as the necessary files are downloaded from the Docker community hub. They'll be cached
locally so that subsequent runs will finish much faster. Once finished you can log into your container over SSH and
verify every thing looks good with the following command.

```
kitchen login
```

Very cool! This is the equivalent of attaching to your container using `docker run` to run an interactive shell.

## Test-driven development

It's time to write and run some tests for our Chef code. We know we're going to install nginx, and just to get started
we'll probably want it to bind and listen on port 80. We can always add other configurations like HTTPS, keep alive,
worker numbers, etc... later, but for now we'll start simple.

### Write the tests

Open the `test/integration/default/default_test.rb` file and replace the existing contents
with the contents below.

```ruby
describe port(80) do
  it { should be_listening }
end
```

This is a very simple InSpec test that will test if port 80 on our container is listening for network requests.

### Watch the tests fail

Lets deploy and run our cookbook, then run the test using the following commands.

```
kitchen converge
```
```
kitchen verify
```

We should see in the output that the test failed since we haven't written the code yet that will install nginx.

```
Port 80
   ×  is expected to be listening
   expected `Port 80.listening?` to return true, got false
```

### Using community code

One of the best features of using Chef is the vast repository of community-maintained cookbooks. If you want to
configure something, anything, chances are there's already a community cookbook for it. We'll include the nginx
community cookbook by declaring it as a dependency in our metadata.rb file. Add the following line to metadata.rb.

```ruby
depends 'nginx'
```

Since we're using the nginx community cookbook we can simply include the install resource from that cookbook in our
default recipe. Open the file at `recipes/default.rb` and replace the contents with the line below.

```ruby
nginx_install 'epel'
```

Newer versions of Chef require us to update our Policyfile if we add a dependency. Since we've added nginx as a
dependency, we need to update our Policyfile using the following command.

```
chef update
```

### Watch the tests pass (with fingers crossed)

Lets deploy, run, and test our cookbook again.

```
kitchen converge
```
```
kitchen verify
```

This time you should see the state of your container change. Chef is installing and starting up nginx with a basic set
of configurations. Also the tests should pass.

```
✔  is expected to be listening

Test Summary: 1 successful, 0 failures, 0 skipped
```

## Conclusion

Finally we've gotten our cookbook going with a solid testing environment and automated tests. Now we can add and modify
configurations to our heart's or company's content using TDD best practices.

## Next Steps

Our next step is to create a Continuous Integration pipeline for our cookbook. Thankfully Test Kitchen makes the build
and test stages of our pipeline as simple as running the following

```
kitchen converge
```

and

```
kitchen verify
```

respectively.






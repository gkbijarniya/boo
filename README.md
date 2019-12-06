# Boo  [![Maven Central][maven-svg]][maven-url]

The purpose of Boo is to provide a simple command-line tool for the OneOps developers and OneOps users to deploy,
update, destroy and operate their OneOps assemblies.

For the OneOps pack developers, it provides a fast and repeatable way to run tests against complex OneOps packs. For
example, it allows a pack developer to use a short, one-line bash command to verify a parameter or code changes in an
OneOps pack by attemping to instantiate the updated pack and executing tests on the resulting functional cluster. For an
OneOps user, it provides an command-line tool to deploy, update, destroy and operator their OneOps assemblies. The boo
tool is an addition to the OneOps UI interfaces.

It requires a [YAML](http://yaml.org/) configuration file and uses the OneOps client API to primarily interact with OneOps via `https:443`.

## Prerequisite

- [OneOps](http://oneops.com/user/).
- Cloud computing fundamentals.

## Usage

To use Boo you download the executable JAR and place it in your `$PATH`. Use the latest version available from
[http://repo1.maven.org/maven2/com/oneops/boo/boo](http://repo1.maven.org/maven2/com/oneops/boo/boo) e.g.:

```
$ curl -o boo http://repo1.maven.org/maven2/com/oneops/boo/boo/1.0.19/boo-1.0.19-executable.jar
$ chmod +x boo
$ boo <options>
```

For debug use:
```
java -Dorg.slf4j.simpleLogger.defaultLogLevel=debug -jar boo <options>
```


To see what options are available for use:

```
boo -h
```

## Configuration

Boo YAML templates are processed with [Mustache][1] to allow variable interpolation when the standard `~/.boo/config` file
contains a `default` profile. If you have a `default` profile that looks like the following:

```
[default]
host=https://localhost:9090
organization=devtools
api_key=oneops_api_key # copy from OneOps UI->profile->authentication->API Token
email=jvanzyl@walmart.com
cloud=prod-cdc6
```

With a Boo YAML template that looks like this:

```
boo:
  oneops_host: '{{host}}'
  organization: '{{organization}}'
  api_key: '{{api_key}}'
  email: '{{email}}'
  environment_name: 'dev'
  ip_output: 'json'
  clouds:
    {{cloud}}:
       priority: '1'
       dpmt_order: '1'
       pct_scale: '100'

...
```

It will yield the following:

```
boo:
  oneops_host: 'https://localhost:9090'
  organization: 'devtools'
  api_key: 'oneops_api_key'
  email: 'jvanzyl@walmart.com'
  environment_name: 'dev'
  ip_output: 'json'
  clouds:
    prod-cdc6:
       priority: '1'
       dpmt_order: '1'
       pct_scale: '100'

...
```
If you want to see what your Boo YAML template will look like with interpolated values you can use the following command:

```
boo -f boo.yml -v
```

## Inlining File Content

You can use the `file` directive to inline content into a Boo template. The following are examples of how you can inline
the content of files:

```
{{file(./id_rsa.pub)}}
{{file(~/.ssh/id_rsa.pub)}}
{{file(/Users/jvanzyl/id_rsa.pub)}}
```

So, for example, if you wanted to inline the content of your public SSH key into your Boo template before execution you
can do the following:

```
user:
  user-jvanzyl:
    system_account: true
    sudoer: true
    username: 'jvanzyl'
    description: "JvZ"
    authorized_keys: '["{{file(~/.ssh/id_rsa.pub)}}"]'
```

The result will look something like:

```
user:
  user-jvanzyl:
    system_account: true
    sudoer: true
    username: 'jvanzyl'
    description: "JvZ"
    authorized_keys: '["ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7g6cjv8yxH0pbo..."]'
```

Again, you can see what the result will be using `boo -f your.yml -v`.

When using `file()`, multiple lines will be combined together when the YAML file is processed.  In order to preserve
multiple lines, use `multilineFile()`.

For example, the following can be used to create an INI file template from an external file.  If you have:

sample.ini
```
[general]
environment=dev
user=bob
```

then this configuration:

```
file:
  file-config:
    path: '/etc/sample.ini'
    content: '{{multilineFile(sample.ini)}}'
```

will expand to:

```
file:
  file-config:
    path: '/etc/sample.ini'
    content: '[general]

environment=dev

user=bob
'
```

The blank lines are inserted so that when the YAML is processed, the resulting string will contain the line breaks.

# Sample Boo Operations

- **Create an assembly, environment and deploy your instance to OneOps.**

     `boo -f boo.yml -create`
     
         Creating the environment DEV.
          32% **************** 
         
          42% ********************* 
         
          52% ************************** 
         
         Updating the compute size in DEV - empty-vm
          72% ************************************ 
         
         Starting the deployment now.
         100% ************************************************** 
     
-  **Check the status of your deployment**

    `boo -f boo.yml -status`
    
    ``DEV deployment status:active``
    
- **Update a configuration and deploy your changes to OneOps**  

  `boo -f boo.yml -update`      
                              
          Updating component daemon for empty-vm ...
          Updating component lb for empty-vm ...
          Updating component secgroup for empty-vm ...
          Updating component os for empty-vm ...
          Updating component java for empty-vm ...
          Updating component user-test for empty-vm ...
           15% ******* 
          
           20% ********** 
          
          Environment exist, skip create environment DEV.
           32% **************** 
          
           42% ********************* 
          
           52% ************************** 
  
          Starting the deployment now.
          100% ************************************************** 
 
- **Delete an assembly & platform**

  First `-remove` destroys the resource. Second `-remove` destroys the orphaned assembly. 
    
    One can use `-force` to destroy all at once without the `y/n` prompt.

    `boo -f boo.yml -remove`
    
        WARNING! There are 1 assemblies using the my-first-assembly-dev configuration. Do you want to destroy all of them? (y/n)
         y
         Destroying OneOps assembly my-first-assembly-dev 
         
         A deployment has been started to remove active nodes. Please execute this command again once the deployment is complete to finish deleting remaining elements.

- **Update a configuration and do not deploy it to OneOps**

    `boo -f boo.yml -update --no-deploy`
    
       Environment exist, skip create environment DEV.
       32% **************** 
      
       42% ********************* 
      
       52% ************************** 
      
      Updating the compute size in DEV - empty-vm
       72% ************************************ 
      
      100% ************************************************** 
      
      Created/updated assembly without deployments.

# Development

## Build the source code to an executable command:

```
mvn clean package

chmod +x ./target/boo-1.0.10-executable.jar

mv ./target/boo-1.0.10-executable.jar boo

boo <options>
```

## Build to an RPM

```
mvn clean package -Pbuild-rpms

Find the rpm at .//target/rpm/boo/RPMS/noarch/boo-1.0.10*.noarch.rpm
```

## Install the RPM

```
On Linux: sudo rpm -ivh boo-1.0.10*.noarch.rpm
On Mac: sudo rpm -ivh boo-1.0.10*.noarch.rpm --nodeps
```

## Running integration tests

The integration tests take some time to run as they spin up real assemblies, validate them and spin them down. To run the integration tests along with the tests use the following:

```
mvn clean verify -Pits
```

Note that in order to run the integration tests you also need a `default` profile in your `~/.boo/config` that looks like the following:

```
[default]
host=https://localhost:9090
organization=oneops_org_name
api_key=oneops_api_key
email=me@email.com
cloud=cloud_name_in_oneops
```

## Deploying SNAPSHOT

```
mvn clean deploy -P release
```

## Releasing

Provided you have permissions to deploy to OSSRH with a serverId `ossrh` in your settings and the needed gpg setup you
can build and deploy a release with

```
mvn release:prepare
mvn release:perform
```

And then just check and release the staging repo on [https://oss.sonatype.org](https://oss.sonatype.org)


## Code style

Boo uses the Google code style. The formatter for Eclipse you can find here:

https://raw.githubusercontent.com/google/styleguide/gh-pages/eclipse-java-google-style.xml


-----------------
 <sup><b>**</b></sup>Require [Java 8 or later][java-download]
 
 <!-- Badges -->
[maven-url]: http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22com.oneops.boo%22%20AND%20a%3A%22boo%22
[maven-svg]: https://img.shields.io/maven-central/v/com.oneops.boo/boo.svg?label=Maven%20Central&style=flat-square
[1]: https://github.com/spullara/mustache.java

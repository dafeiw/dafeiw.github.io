---
title: Gradle note
date: 2019-10-19 18:34:07
tags:
- build
categories: Build
---

# Gradle Basics

Projects and tasks are two fundamental concepts

Every gradle build is made up of one or more projects. A project represents either a thing to be built or a thing to be done.

Each project is made up of one or more tasks. A task represents some atomic piece of work to be performed. 
<!--more-->
```groovy
task hello {
    doLast {
        println 'Hello world!'
    }
}

task intro(dependsOn: hello) {
    doLast {
        println "I'm Gradle"
    }
}

// via API (Each task is available as a property of the build script)
intro.dependsOn hello

intro.doLast {
    println 'Second doLast'
}

// dynamically create tasks
4.times { counter ->
    task "task$counter"  {
        doLast {
            println "I'm task number $counter"
        }
    }
}

// add extra property (project, task, source set can hold extra property)
task myTask {
    ext.myProperty = 'myValue'
}

task printTaskProperties {
    doLast {
        println myTask.myProperty
    }
}

```

# Working with Tasks

## Define tasks
Defining tasks using strings for task names
```groovy
task('hello') {
    doLast {
        println "hello"
    }
}

task('copy', type: Copy) {
    from(file('srcDir'))
    into(buildDir)
}
```

Defining tasks using the tasks container
```groovy
tasks.create('hello') {
    doLast {
        println "hello"
    }
}

tasks.create('copy', Copy) {
    from(file('srcDir'))
    into(buildDir)
}
```

Defining tasks using a DSL specific syntax
```groovy
task(hello) {
    doLast {
        println "hello"
    }
}

task(copy, type: Copy) {
    from(file('srcDir'))
    into(buildDir)
}
```

## Locating Tasks

Accessing tasks using a DSL specific syntax:
```groovy
task hello
task copy(type: Copy)

// Access tasks using Groovy dynamic properties on Project

println hello.name
println project.hello.name

println copy.destinationDir
println project.copy.destinationDir
```

Accessing tasks via tasks collection:
```groovy
task hello
task copy(type: Copy)

println tasks.hello.name
println tasks.named('hello').get().name

println tasks.copy.destinationDir
println tasks.named('copy').get().destinationDir
```

Accessing tasks by path:
```groovy
project(':projectA') {
    task hello
}

task hello

println tasks.getByPath('hello').path
println tasks.getByPath(':hello').path
println tasks.getByPath('projectA:hello').path
println tasks.getByPath(':projectA:hello').path
```


## Configure tasks

```groovy
task myCopy(type: Copy)
```
The name of this task is “myCopy”, but it is of type “Copy”. You can have multiple tasks of the same type, but with different names.

Configuring a task using the API:
```groovy
Copy myCopy = tasks.getByName("myCopy")
myCopy.from 'resources'
myCopy.into 'target'
myCopy.include('**/*.txt', '**/*.xml', '**/*.properties')
```

Configuring a task using a DSL specific syntax:
```groovy
myCopy {
   from 'resources'
   into 'target'
}
myCopy.include('**/*.txt', '**/*.xml', '**/*.properties')
```

You can also use a configuration block when you define a task. Defining a task with a configuration block:
```groovy
task copy(type: Copy) {
   from 'resources'
   into 'target'
   include('**/*.txt', '**/*.xml', '**/*.properties')
}
```

## Passing argumetns to a task constructor
Prefer creating a task with constructor arguments using the TaskContainer

```groovy
class CustomTask extends DefaultTask {
    final String message
    final int number

    @Inject
    CustomTask(String message, int number) {
        this.message = message
        this.number = number
    }
}

tasks.create('myTask', CustomTask, 'hello', 42)

task myTask(type: CustomTask, constructorArgs: ['hello', 42])
```


## Skipping tasks

Use a predicate
```groovy
hello.onlyIf { !project.hasProperty('skipHello') }
```
Skipping tasks with StopExecutionException
```groovy
compile.doFirst {
    // Here you would put arbitrary conditions in real life.
    // But this is used in an integration test so we want defined behavior.
    if (true) { throw new StopExecutionException() }
}
```

Enabling and disabling tasks
```groovy
task disableMe {
    doLast {
        println 'This should not be printed if the task is disabled.'
    }
}
disableMe.enabled = false
```

Use timeout property. When a task reaches its timeout, its task execution thread is interrupted.
```groovy
task hangingTask() {
    doLast {
        Thread.sleep(100000)
    }
    timeout = Duration.ofMillis(500)
}
```

## Incremental Build

Incremental build won’t work unless a task has at least one task output, although tasks usually have at least one input as well.

As a build author, you need to specify which are inputs and which are outputs. Also be careful of non-deterministic tasks that may generate different output for exactly the same inputs.


## External dependencies for the build script

If your build script needs to use external libraries, you can add them to the script’s classpath in the build script itself. You do this using the buildscript() method. The block passed to the buildscript() method configures a ScriptHandler instance

```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath group: 'commons-codec', name: 'commons-codec', version: '1.2'
    }
}
```

## The script API

When Gradle executes a Groovy build script (.gradle), it compiles the script into a class which implements Script. This means that all of the properties and methods declared by the Script interface are available in your script.



# Working with Plugin

Two types of plugins
- script plugin
- binary plugin (classes that implement the Plugin interface)

A plugin often starts out as a script plugin (because they are easy to write) and then, as the code becomes more valuable, it's migrated to a binary plugin that can be easily tested and shared betwen multiple projects or organizations.

To use a plugin has two steps (resolving and applying). Applying a plugin means actually executing the plugin’s Plugin.apply(T) on the Project you want to enhance with the plugin.

## Script Plugins
```groovy
apply from: 'other.gradle'
```

## Binary Plugins

A plugin is simply any class that implements the Plugin interface. Non-core binary plugins need to be resolved before they can be applied. 

- Including the plugin from the plugin portal or a custom repository using the plugins DSL
- Including the plugin from an external jar defined as a buildscript dependency
- Defining the plugin as a source file under the buildSrc directory in the project
- Defining the plugin as an inline class declaration inside a build script.

## Applying plugins

The plugins DSL block configures an instance of PluginDependenciesSpec.
```groovy
plugins {
    id 'java'
}

plugins {
    id 'com.jfrog.bintray' version '0.4.1'
}

apply from: rootProject.file('gradle/dependencies.gradle')
```

## Applying plugins to subprojects

The default behaviour of the plugins{} block is to immediately `resolve` and `apply` the plugins. 

```groovy
plugins {
    id 'org.gradle.sample.hello' version '1.0.0' apply false
    id 'org.gradle.sample.goodbye' version '1.0.0' apply false
}

subprojects {
    if (name.startsWith('hello')) {
        apply plugin: 'org.gradle.sample.hello'
    }
}
```

## Write a simple plugin

There are several places where you can put the source for the plugin.
- Build script
- `buildSrc` project
- Standalone project

To create a Gradle plugin, you need to write a class that implements the Plugin interface. When the plugin is applied to a project, Gradle creates an instance of the plugin class and calls the instance’s Plugin.apply() method. The project object is passed as a parameter, which the plugin can use to configure the project however it needs to. 


```groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast {
                println 'Hello from the GreetingPlugin'
            }
        }
    }
}

// Apply the plugin
apply plugin: GreetingPlugin
```

## Make the plugin confiugration

To make the plugin configurable, you need add an extension object (represents the configuration) to the ExtensionContainer.

```groovy
class GreetingPluginExtension {
    String message
    String greeter
}

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        def extension = project.extensions.create('greeting', GreetingPluginExtension)
        project.task('hello') {
            doLast {
                println "${extension.message} from ${extension.greeter}"
            }
        }
    }
}

apply plugin: GreetingPlugin

// Configure the extension using a DSL block
greeting {
    message = 'Hi'
    greeter = 'Gradle'
}
```


# Building Java project

```groovy
plugins {
    id 'java'
}

sourceCompatibility = '1.8'
targetCompatibility = '1.8'
version = '1.2.1'
```

```groovy
apply plugin: 'java'
```

```
project  
    +build  
    +src/main/java  
    +src/main/resources  
    +src/test/java  
    +src/test/resources  
```

java plugin adds some tasks. `gradle tasks`


By applying the Java plugin, you get a whole host of features: compileJava, compileTestJava, test, jar, javadoc

```groovy
task show  {
    doLast{
        println relativePath(compileJava.destinationDir)
        println relativePath(processResources.destinationDir)
    }
}  
```

You can change the destinationDir of compireJava.
```groovy
apply plugin: 'java'
compileJava.destinationDir = file("$buildDir/output/classes")
task show  {
    doLast{
        println relativePath(compileJava.destinationDir)
        println relativePath(processResources.destinationDir)
    }
}  
```

But compileJava is not the only task that needs to know destinationDir. Gradle has sourceSet

```groovy
apply plugin: 'java'
sourceSets.main.output.classesDir = file("$buildDir/output/classes")
task show << {
    println relativePath(compileJava.destinationDir)
}  
```

## Source Set

java plugin defines main source set and test source set. 

Custom source dir

```groovy
sourceSets {
    main {
         java {
            srcDirs = ['src']
         }
    }

    test {
        java {
            srcDirs = ['test']
        }
    }
}
```

## Changing compiler options

Most of the compiler options are accessible through the corresponding task, such as `compileJava` and `compileTestJava`.

```java
compileJava {
    options.incremental = true
    options.fork = true
    options.failOnError = false
}
```

# Build Phases

Initializaiton -> Configuration -> Execuiton

# Composite build

A composite build is simply a build that includes other builds. In many ways a composite build is similar to a Gradle multi-project build, except that instead of including single projects, complete builds are included

Two concepts:
* composite build
* included build

A build that is included in a composite build is referred to, naturally enough, as an "included build". Included builds do not share any configuration with the composite build, or the other included builds. Each included build is configured and executed in isolation.

Included builds interact with other builds via dependency substitution. If any build in the composite has a dependency that can be satisfied by the included build, then that dependency will be replaced by a project dependency on the included build.

## Defining a separate composite build

```groovy
rootProject.name = 'adhoc'

includeBuild '../my-app'
includeBuild '../my-utils'
```

# Best Practice

## Name your project

In `seetings.gradle` file

```groovy
include ':api', ':impl', ':client'
 
// Rename the project early in the kinematics (after that the names are read-only) 
rootProject.name = 'XXX'
project(':api').= 'XXX'
project(':impl').= 'XXX'
project(':client').= 'XXX'
```

After renaming a project, all references must be updated to accomodate the new name. For instance, the existing references "project(':impl')" must now be replaced by "project(':XXX')" 

## Use the wrapper

Run `gradle wrapper` to generate two files:

- `gradle/wrapper/gradle-wrapper.jar`
- `gradle/wrapper/gradle-wrapper.properties`
- `gradlew`
- `gradlew.bat`

## Apply DRY

```groovy

apply plugin: 'java'
  
def jerseyVersion = '1.19'
  
dependencies {
    // Note that we replaced the single quotes by double quotes to let Groovy evaluate the expression "${jerseyVersion}"
    compile "com.sun.jersey:jersey-core:${jerseyVersion}"
    compile "com.sun.jersey:jersey-servlet:${jerseyVersion}"
    compile "com.sun.jersey:jersey-json:${jerseyVersion}"
}
```

## Segregate the code from the configuration

Ideally, your Gradle scripts should just contain configuration to keep them as simple / readable as possible. When there's too much code in your scripts, you should consider moving this custom code outside of the Gradle scripts.

Gradle automatically compiled the "buildSrc" project prior running the main build. 
# Shamrock

Shamrock is framework that allows you process Java EE and Microprofile metadata at build time,
and use it to create low overhead jar files, as well as native images using Graal.

At the moment is has the following features:

- Clean build/runtime separation of components
- Bytecode recorders to allow for the generation of bytecode without knowledge of the class file format
- An API to easily enable reflection support in Graal
- Support for injection into build time processors
- Support for build and runtime config through MP config
- 'Instant Start' support on Graal through the use of static init to perform boot
- Various levels of support for Resteasy, Undertow, Weld, MP Config and MH Health (e.g. Weld works, but only if you specify -H:+ReportUnsupportedElementsAtRuntime)
- A maven plugin to run the build, and create native images
- A JUnit runner that can run tests, and supports IDE usage
- A JUnit runner that can test a native image produced by the maven plugin


## How to build Shamrock

* Install GraalVM (tested on RC4)
* set `GRAALVM_HOME` to your GraalVM Home directory e.g. `/Users/emmanuel/JDK/GraalVM/Contents/Home`
* `mvn install`

The default build will create two different native images, which is quite time consuming. You can skip this
by disabling the `native-image` profile: `mvn install -P\!native-image`.

Wait. Success!


## Architecture Overview

Shamrock runs in two distinct phases. The first phase is build time processing, which is done by instances of ResourceProcessor:

https://github.com/protean-project/shamrock/blob/master/core/deployment/src/main/java/org/jboss/shamrock/deployment/ResourceProcessor.java

These processors run in priority order, in general they will read information from the Jandex index, and either directly output bytecode for use at runtiime, or provide information for later processors to write out. 

These processors write out bytecode in the form of implementations of StartupTask:

https://github.com/protean-project/shamrock/blob/master/core/runtime/src/main/java/org/jboss/shamrock/runtime/StartupTask.java

When these tasks are created you can choose if you want them run from a static init method, or as part of the main() method execution. This has no real effect when running on the JVM, however when building a native image anything that runs from the static init method will be run at build time. This is how shamrock can provide instant start, as all deployment processing is done at image build time. It is also why Weld can work without modification, as proxy generation etc is done at build time.

As part of the build process shamrock generates a main method that invokes all generated startup tasks in order.

## Runtime/deployment split

In general there will be two distinct artifacts, a runtime and a deployment time artifact. 

The runtime artifact should have a `dependencies.runtime` file in the root of the jar. This is a file that is produced
by the maven dependencies plugin:

```
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>resolve</goal>
            </goals>
            <phase>compile</phase>
            <configuration>
                <silent>true</silent>
                <outputFile>${project.build.outputDirectory}/dependencies.runtime</outputFile>
            </configuration>
        </execution>
    </executions>
</plugin>
```

This file tells the build plugin which dependencies are actually needed at runtime.

### Packaging and dependencies

The deployment time artifact should have a dependency on the runtime artifact. When using shamrock you just declare a
dependency on the deployment time artifacts for the features you want. This dependency *must* be scope `provided`.

The shamrock plugin will not copy provided artifacts to the lib directory (and hence they will not be included in the
native image). The exception to this is artifacts that contain a `dependencies.runtime` files (as described above).

If this artifact has this file, or its maven coordinates were listed in another artifacts file then it will be included
(this match does not take version into account, so if dependency resolution has resulted in a different version being
selected it will still be included).

This mechanism means that you only need to declare a single dependency per feature, but shamrock still has enough information
to only include necessary runtime components.


## Bytecode recording

These startup tasks are generated by the bytecode recorder. This recorder works by creating proxies of classes that contain deployment logic, and then recoding the invocations. 

It will only be necessary to use this if you are integrating a framework that requires custom code to bootstrap it. If you only need simple integrations like adding a bean or a Servlet then this is not necessary. For example the health check integration requires no bytecode generation, as it just adds a servlet and some beans. The integration basically just adds them to a list, and a later processor handles writing out the bytecode:

https://github.com/protean-project/shamrock/blob/master/health/deployment/src/main/java/org/jboss/shamrock/health/HealthProcessor.java#L30

This is the Weld deployment time processor:

https://github.com/protean-project/shamrock/blob/master/weld/deployment/src/main/java/org/jboss/shamrock/weld/deployment/WeldAnnotationProcessor.java#L28

Using this runtime template:

https://github.com/protean-project/shamrock/blob/master/weld/runtime/src/main/java/org/jboss/shamrock/weld/runtime/WeldDeploymentTemplate.java

The first thing that happens is the processor gets a bytecode recorder by calling addStaticInitTask. The priority number that is passed in here controls the order in which the tasks are executed. It then gets a copy of the template by calling getRecordingProxy.

The next call to createWeld() will start recording bytecode. The proxy will record the invocation and write it out to bytecode when the recorder is closed. The return value of this method is also a proxy. This proxy cannot be invoked on, but can be passed back into template methods, and the recorded will automatically write out bytecode that does the same. 

We see an example of this with the template.addClass call, which adds a class to the deployment. The first parameter is the proxy from the createWeld call. This method also shows how Class objects are handled. Because this method takes a 'Class' object, and the actual classes are not loadable at build time we use the classProxy method to create a Class object to pass in. 

In general these invocations on the templates should look very similar to the same sequence of invocations you would actually make to start weld.

The next example is the Undertow processor, which is a bit more complex:

https://github.com/protean-project/shamrock/blob/master/undertow/deployment/src/main/java/org/jboss/shamrock/undertow/ServletAnnotationProcessor.java#L41

https://github.com/protean-project/shamrock/blob/master/undertow/runtime/src/main/java/org/jboss/shamrock/undertow/runtime/UndertowDeploymentTemplate.java#L27

This example uses the @ContextObject annotation to pass parameters around, instead of relying on proxies.

If this annotation is applied to a method then the return value of that method will be placed in the StartupContext under the provided key name. If the annotation is applied to a parameter then the value of that parameter will be looked up from the startup context. This allows processors to interact with each other, e.g. an early processor can create the DeploymentInfo and store it in the StartupContext, and then a later processor can actually boot undertow.

In this case there are four processors, one that creates the DeploymentInfo, another that actually adds all discovered Servlets, one that performs the actual deployment, and one that actually starts undertow (which runs from the main method instead of static init).

The last of these also has an example of using MP config. If you inject ShamrockConfig into your application the bytecode recorded will treat String's returned from config in a special manner. If the recorder detects that a String has come from config then instead of just writing the value it will write some bytecode that loads the value from MP config, and defaulting to the build time value if it is not present. This means configuration can be applied at both build and runtime.

## Testing

In order to support IDE usage Shamrock also has a 'runtime mode' that performs the build steps at runtime. This will also be needed for Fakereplace support, to allow the new metadata to be computed at runtime. This mode works by simply creating a special ClassLoader, and writing all bytecode into an in memory map that this class loader can use to load the generated classes. When running the tests this process is performed once, and then all tests are run against the resulting application.

There is also a graal based runner that builds a native image and then runs tests against it, although this is quite primitive. 

## Reflection

In order to make reflection work Shamrock provides an API to easily register classes for reflection. If you call ProcessorContext.addReflectiveClass then a Graal AutoFeature will be written out that contains the bytecode to register this class for reflection. 

The reason why bytecode is used instead of JSON is that this does not require any arguments to the native-image command, it 'just works'. It would also work better in a multiple jar scenario, as each jar could just have its own reflection wiring baked in.

I think that pretty much covers most of what is in there. As it has just been a bit of an experiment some of the code is not that great, but if we did decide to use this as the basis for the PoC it would be easy enough to clean up. 


## Plugin Output

The shamrock build plugin generating wiring metadata for you application. The end result of this
is:

*   ${project.build.finalName}-runner.jar 
    The jar that contains all wiring metadata, as well as a manifest that correctly sets up the classpath. This jar can be executed directly using `java -jar`, or can be turned into a native image in the same manner.
     
*   ${project.build.finalName}.jar 
    The unmodified project jar, the shamrock plugin does not modify this.
    
*   lib/*
    A directory that contains all runtime dependencies. These are referenced by the `class-path` manifest entry in the runner jar.
    
## Shamrock Run Modes

Shamrock supports a few different run modes, to meet the various use cases. The core of how it works is the same in each
mode, however there are some differences. The two basic modes are:

*   Built Time Mode
    This mode involves building the wiring jar at build/provisioning time, and then executing the resulting output as a
    separate step. This mode is the basis for native image generation, as native images are generated from the output
    of this command. This is the only mode that is supported for production use.

*   Runtime Mode
    Runtime mode involves generating the wiring bytecode at startup time. This is useful when developing apps with Shamrock,
    as it allows you to test and run things without a full maven. This is currently used for the JUnit test runner, 
    and for the `mvn shamrock:run` command.
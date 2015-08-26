# NetBeans Module Maven plugin

[ ![Codeship Status for mojohaus/nbm-maven-plugin](https://codeship.com/projects/827d5e10-2da6-0133-ca32-7e50f7676400/status?branch=master)](https://codeship.com/projects/98910)

This Apache Maven plugin is able to create NetBeans module(plugin) artifacts. It registers a new packaging type `nbm`. Any project with this packaging will be automatically turned into a NetBeans module project. Additionally it allows to create clusters of modules, generate an autoupdate site content or build and assemble an application on top of NetBeans platform.

Note: The `nbm:populate-repository` goal has been moved to it's own plugin at [nb-repository-plugin](https://github.com/mojohaus/nb-repository-plugin/).

To get access to a repository with NetBeans.org module artifacts and metadata, add [http://bits.netbeans.org/maven2/](http://bits.netbeans.org/maven2/) repository to your project POM or the repository manager you are using. The repository hosts binaries of NetBeans 6.5 and later.

Also see: [Maven NBM development FAQs](http://wiki.netbeans.org/NetBeansDeveloperFAQ#Mavenized_Builds)

Sample pom.xml excerpts for creation of a NetBeans module:
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <artifactId>example-netbeans-module</artifactId>
  <groupId>org.mycompany.myproject</groupId>
  <!--here is the packaging and lifecycle defined-->
  <packaging>nbm</packaging>

....
  <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>nbm-maven-plugin</artifactId>
                <version>3.8.1</version>
                <extensions>true</extensions>
            </plugin>
            <plugin> <!-- required since nbm-plugin 3.0-->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <useDefaultManifestFile>true</useDefaultManifestFile>
                </configuration>
            </plugin>
        </plugins>
  </build>

 ....
    <!-- this section is important only to access the binaries of NetBeans that you use as dependencies -->
    <repositories>
        <repository>
            <id>netbeans</id>
            <name>repository hosting netbeans.org api artifacts</name>
            <url>http://bits.netbeans.org/maven2/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
```    
To build the project then, just type
```
mvn install
```
## Maven Dependency vs. NetBeans runtime dependency
There are important differences between Maven's dependency mechanism and NetBeans runtime dependencies. Maven's dependencies are transitive, so at compile time you get not only direct dependencies you declared, but also dependencies of dependencies etc. In NetBeans, the module dependencies are non-transitive by nature, you have to explicitly declare all at runtime. Additionally next to module dependencies there are also library jars attached and shipped with the module's main artifact. In the NetBeans terminology there is a special sort of modules called "library wrappers". These library wrappers add the libraries on the module's classpath and allow other modules to depend on the libraries within the IDE's runtime.

The ways in which the nbm-maven-plugin tries to adress these issues has changed over time.

The plugin walks the dependency tree to detect and identify module dependencies and classpath libraries.

A maven dependency is turned into a NetBeans runtime dependency when:

* For NetBeans module dependencies (dependency jars that have the NetBeans specific entries in META-INF/MANIFEST.MF)
  * It's a direct dependency (non-transitive) and is a NetBeans module itself. Preferred way of declaring module dependencies.
  * It's defined in existing (though optional and deprecated) module.xml file in dependencies section. Try to avoid this, but still useful if one wants to put an explicit dependency value on the module, or use implementation dependency.
  * When the dependency is of type nbm. Deprecated in 3.0.x, only helpful in older versions. Such dependencies don't include their transitive deps on compilation classpath. That should allow one to simulate the rumtime dependencies at compilation time in maven, however there's one major drawback. Not only are the nbm's module dependencies hidden, but the libraries associated with the given nbm module are also hidden. So you can end up with less stuff on classpath as opposed to more stuff with jar typed dependencies.
* For module libraries (jars that are packed together with the module and appear on it's classpath directly, not by a dependency relationship.)
  * It's a direct dependency and is not of provided scope.
  * It's a transitive dependency, pulled in by a direct dependency (only non-module one - see first bullet) This is new in 3.0+
  * It's defined in existing (though optional) module.xml file in libraries section. Consider this deprecated in 3.0+.

The complete nbm descriptor format documentation, and example descriptors are also available. Please note that since 3.8 version, the descriptor is deprecated and replaced by plugin configuration parameters.

Additionally we perform dependency analysis in order to warn the user when runtime dependencies are wrong. So project's own classes and it's classpath libraries' classes are checked against the module dependencies (with appropriate filtering for public packages/private packages). If the classes depend on declared module dependency's private classes or on transitive module dependency's classes, the build fails. That should prevent ClassNotFoundException's later at runtime, when the NetBeans module system constructs the classpath for the module based on our metadata generated.

## Using OSGi bundles in NetBeans platform based applications
Starting with version 3.2, it's possible for the NetBeans modules to depend on OSGi bundles. A proper module dependency section will be generated. To include the bundle in the application, add dependency on the bundle from nbm-application. There are a few prerequisites.

It works only in NetBeans 6.9 and later which support the embedding of bundles at runtime
Add `<useOSGiDependencies>true</useOSGiDependencies>` configuration entry to all the modules depending on OSGi bundles. Existing applications/modules need to check modules wrapping external libraries for library jars that are also OSGi bundles. Such modules will no longer include the OSGi bundles as part of the module NBM but will include a modular dependency reference on the bundle only. Modules depending on these old wrapper modules shall depend directly on the bundle, eventually rendering the old library wrapper module obsolete.
in the distribution, all bundles will be included in the default cluster (extra if not configured otherwise), in 3.10 and later the plugin will attempt to guess the cluster based on modules depending on it.
Before version 3.10 all bundles will be autoload, thus requiring at least one depending regular module to enable them. In 3.10 and later, developers of the OSGi bundles can influence the autoload vs regular behaviour by adding Nbm-Maven-Plugin-Autoload attribute to the bundle's manifest with "true" or "false" values. False means the module will be enabled on start, even without any other modules depending on it.

## Multi module setup
If you have a set of NetBeans modules, or are building on top of NetBeans Platform, you will make use of the additional goals provided by the plugin.

If you are building a Platform-based application, use a project with `nbm-application` packaging to perform the final application assembly. This packaging type (defined in nbm-maven-plugin) should have your module projects and all dependencies of the target NetBeans Platform included as dependencies.

For the NetBeans Platform/IDE modules, there are artifacts that aggregate modules in clusters. These are put in the org.netbeans.clusters groupId (on bits.netbeans.org or in your own repository). The following snippet will include the basic NetBeans platform cluster and your own module in the application. You can use standard dependency exclusion lists to cut out modules from the Platform that you don't need.

```
    <artifactId>application</artifactId>
    <packaging>nbm-application</packaging>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <dependency>
            <groupId>org.netbeans.cluster</groupId>
            <artifactId>platform8</artifactId>
            <version>${netbeans.version}</version>
            <type>pom</type>
        </dependency>
        <dependency>
            <groupId>com.mycompany</groupId>
            <artifactId>module1</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
```    
The nbm-application project/packaging defines a build lifecycle that creates a final application from the NBM files in local/remote repotories and bundles them in a ZIP file (also uploadable to the repository). In addition to that you can configure the project to generate an autoupdate site and/or webstartable binaries of the applications (typically in a deployment profile):

```
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>nbm-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>extra</id>
                        <goals>
                            <goal>autoupdate</goal>
                            <goal>webstart-app</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```            
See the `autoupdate` and `webstart-app` goals for more details.

### mvn nbm:cluster
This goal aggregates output of multiple NetBeans module projects and creates one or more clusters in the current project. So usually one runs this goal on the parent POM project, which aggregates the content of all its modules. The resulting cluster structure can later be used for running the application, creating an installer or similar. A variant of this goal is also included in the nbm-application project's default lifecycle.

### mvn nbm:branding
Branding is to used when one builds an application based on NetBeans Platform (as opposed to creating set of modules for the IDE). Branding contains all the resources that are to be changed in the platform binaries (resource bundles, images, HTML files etc.) to give the application its unique look.

This goal can be attached to one of the nbm module projects that will be part of the NetBeans Platform-based application.

For more detailed tutorial, check the Screencast: Maven and the NetBeans Platform video recorded by Fabrizio Giudici. It describes to Fabrizio's open source project ForceTen which can be used as reference setup for Maven NetBeans Platform based apps.

The branding is included as part of a regular nbm subproject and cannot be attached to a pom packaged root project.

### mvn nbm:run-ide nbm:run-platform
These two goals do almost the same, they allow you to execute your projects content within the IDE or NetBeans platform.

nbm:run-platform only makes sense to execute on projects with nbm-application packaging.

For more information on plugin configuration and customization, see goal documentation.

## Public packages declaration
By default all your module's packages (and classes) and private to the given module. If you want to expose any API to other modules, you will need to declare those public packages in your pom.xml. This includes not only your own classes but also any other 3rd party library classes that are packaged with your module and are to be exposed for reuse by other modules.

For example:
```
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>nbm-maven-plugin</artifactId>
                <version>3.8.1</version>
                <extensions>true</extensions>
                <configuration>
                   <publicPackages>
                       <publicPackage>org.foo.api</publicPackage>
                       <publicPackage>org.apache.commons.*</publicPackage>
                   </publicPackages>
                </configuration>
            </plugin>
```            
there is a package org.foo.api made public (but not org.foo.api.impl package) and any package starting with org.apache.commons, so both org.apache.commons.io and org.apache.commons.exec packages are exposed to the outside

## Archetypes anyone?
There are two basic archetypes:

The first once creates a single project preconfigured to be a NetBeans module. Use this one if you are developing a NetBeans IDE module, or a module for a NetBeans Platform-based application.
```
mvn -DarchetypeGroupId=org.codehaus.mojo.archetypes -DarchetypeArtifactId=nbm-archetype -DarchetypeVersion=... \
  -DgroupId=org.kleint -DartifactId=milos -Dversion=1.0 archetype:generate
```  
The second one creates a parent POM project containing configuration and application branding for your NetBeans Platform-based application.
```
mvn -DarchetypeGroupId=org.codehaus.mojo.archetypes -DarchetypeArtifactId=netbeans-platform-app-archetype \
  -DarchetypeVersion=... -DgroupId=org.kleint -DartifactId=milos -Dversion=1.0 archetype:generate
```  
## IDE support
The NetBeans IDE has Maven support. Among other features, it contains additional support for working with NetBeans module projects. The support includes file templates, important nodes in projects view, running module(s) in the IDE or Platform.

## Sample real life application
Check the Screencast: Maven and the NetBeans Platform video recorded by Fabrizio Giudici. It describes to Fabrizio's open source project ForceTen which can be used as reference setup for Maven NetBeans Platform based apps.

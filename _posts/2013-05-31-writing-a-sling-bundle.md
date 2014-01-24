---
layout: post
title:  "Writing a Sling bundle"
date:   2013-05-31
---

Sling is a web framework built on top of OSGi. Every addition to the framework, be it your own application or an extension to the framework itself, is packaged and installed as a bundle. This article tries to explain the basic steps to go up and running with your first Sling bundle.

## Set up the Maven project

The build system of choice for Sling itself is Maven. Some Sling-related Maven plugins are maintained by the Sling community, and a lot of other plugins exist to create OSGi bundles or to generate descriptors for OSGi components.

The minimal POM file for a bundle is the following.

{% highlight xml %}
<project>
    <modelVersion>4.0.0</modelVersion>
     
    <groupId>com.test</groupId>
    <artifactId>bundle</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>bundle</packaging>
 
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    </properties>
 
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.felix</groupId>
                <artifactId>maven-bundle-plugin</artifactId>
                <version>2.3.7</version>
                <extensions>true</extensions>
            </plugin>
        </plugins>
    </build>
</project>
{% endhighlight %}

This POM loads Maven lifecycle extensions of the [Maven Bundle Plugin](http://felix.apache.org/site/apache-felix-maven-bundle-plugin-bnd.html) to provide a new packaging, bundle. If you use this packaging you will ask the Maven Bundle Plugin to invoke [Bnd](http://www.aqute.biz/Bnd/Bnd), provide a minimal configuration via default Bnd rules, and build an OSGi bundle. The default set of Bnd rules provided by the Maven Bundle Plugin can be found in the Maven Bundle Plugin documentation.

Default Bnd rules are reasonable enough, but you can always override them by adding an `<instructions>` section in the plugin configuration, as show in this snippet.

{% highlight xml %}
<plugin>
    <groupId>org.apache.felix</groupId>
    <artifactId>maven-bundle-plugin</artifactId>
    <version>2.3.7</version>
    <extensions>true</extensions>
    <configuration>
        <instructions>
            <!-- Add your rules here -->
        </instruction>
    </configuration>
</plugin>
{% endhighlight %}

## Write an OSGi component

Sling development is based on OSGi. Extending Sling often involves the creation of new OSGi components. The process of declaring new components and including them in your OSGi bundle can be automated by using the [Maven SCR Plugin](http://felix.apache.org/documentation/subprojects/apache-felix-maven-scr-plugin.html) and the [SCR annotations](http://felix.apache.org/documentation/subprojects/apache-felix-maven-scr-plugin/scr-annotations.html).

The minimal code for a component is the following.

{% highlight java %}
@Component
public class MyComponent {
 
    private final Logger logger = LoggerFactory.getLogger(MyComponent.class);
 
    @Activate
    public void activate() throws Exception {
        logger.info("Component activated");
    }
 
    @Deactivate
    private void deactivate() throws Exception {
        logger.info("Component deactivated");
    }
 
}
{% endhighlight %}

This component has some interesting characteristics. First, the only mandatory annotation is `@Component`. This qualifies the class as a new component to install into the OSGi container. Second, the `@Activate` and `@Deactivate` annotations mark methods that will be called during the component lifecycle.

Also note that in Sling using the built-in logging support is as easy as creating a new SLF4J Logger object.

If your component implements an OSGi service, you will use the `@Service` class annotation to declare the implemented interface. To reference services provided by other OSGi bundles (or by your own bundle, too), you will use the @Reference annotation. To get further information about providing and referencing services, have a look at the [SCR Annotations](http://felix.apache.org/documentation/subprojects/apache-felix-maven-scr-plugin/scr-annotations.html) project.

To make the OSGi components available to the underlying framework, make sure to add to your POM the [Maven SCR Plugin](http://felix.apache.org/documentation/subprojects/apache-felix-maven-scr-plugin.html). This plugin will do all the tedious work required to declare and package your OSGi components. It will scan your source code for SCR annotations, generate XML component descriptors and add manifest headers to declare the generated descriptors inside the bundle.

To compile your code you have to include a dependency to the SCR annotations project.

{% highlight xml %}
<dependency>
    <groupId>org.apache.felix</groupId>
    <artifactId>org.apache.felix.scr.annotations</artifactId>
    <version>1.9.2</version>
</dependency>
{% endhighlight %}

To generate descriptors and link them to the manifest file, you have to run the scr goal of the Maven SCR Plugin.

{% highlight xml %}
<plugin>
    <groupId>org.apache.felix</groupId> 
    <artifactId>maven-scr-plugin</artifactId>
    <version>1.12.0</version>
    <executions>
        <execution>
            <goals>
                <goal>scr</goal>
            </goals>
        </execution>
    </executions>
</plugin>
{% endhighlight %}

## Install the bundle

To test your bundle you have to deploy it to a running Sling instance. The whole process can be automated by using the [Sling Maven Plugin](http://sling.apache.org/site/sling.html). More precisely, you have to execute the install goal and provide information about the running instance’s URL, and the credentials for the deployer user. An example of the usage of the plugin is shown below.

{% highlight xml %}
<plugin>
    <groupId>org.apache.sling</groupId>
    <artifactId>maven-sling-plugin</artifactId>
    <version>2.1.0</version>
    <executions>
        <execution>
            <phase>install</phase>
            <goals>
                <goal>install</goal>
            </goals>
            <configuration>
                <slingUrl>${sling.url}</slingUrl>
                <user>${sling.deployer.user}</user>
                <password>${sling.deployer.password}</password>
            </configuration>
        </execution>
    </executions>
</plugin>
{% endhighlight %}

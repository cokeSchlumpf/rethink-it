The [IBM WebSphere Application Server Liberty Buildpack](https://github.com/cloudfoundry/ibm-websphere-liberty-buildpack) is used to deploy Liberty applications to Bluemix and other Cloud Foundry-based platforms. The Liberty buildpack deploys compiled artifacts such as .war or .ear files, server packages (.zip files), or server directories. In combination with the [cloudfoundry-maven-plugin](https://github.com/cloudfoundry/cf-java-client/tree/master/cloudfoundry-maven-plugin) and the [liberty-maven-plugin](https://github.com/WASdev/ci.maven) you're able to integrate all build, test and deployment steps with Maven. 

This blog post will show how you can setup the Liberty Profile Server, run and test your application on it, create a packaged server and deploy it to Bluemix - All done with Maven.

An alternative approach would be to deploy the source and execute a Maven build on the Bluemix server using the [Heroku buildpack for Java applications](https://github.com/heroku/heroku-buildpack-java) as described [here](https://developer.ibm.com/wasdev/docs/deploying-bluemix-liberty-maven-plug/).

The sample project which will be used is hosted on [GitHub](https://github.com/cokeSchlumpf/liberty-maven-bluemix-sample). It's a simple application which exposes a JAX-RS WebService to flip a coin:

```prettyprint lang-java
@Stateless
@Path("coin")
public class CoinService {
  @GET
  public String flip() {
    return Math.random() > 0.5 ? "Heads" : "Tail";
  }
}
```
The usage of the service is quite easy:
```
curl http://localhost:9080/ws/coin
> Tail
curl http://localhost:9080/ws/coin
> Head
```

## Configure Liberty Maven Plugin
At first the Liberty Maven Plugin must be added to the POM file.

```prettyprint lang-xml
<plugin>
   <groupId>net.wasdev.wlp.maven.plugins</groupId>
   <artifactId>liberty-maven-plugin</artifactId>
   <version>1.1-SNAPSHOT</version>
</plugin>
```

To use the snapshot versions you need to add the Sonatype OSS Maven snapshots repository. Only major releases are published to Maven Central.

```prettyprint lang-xml
<pluginRepositories>
   <pluginRepository>
     <id>sonatype-nexus-snapshots</id>
     <name>Sonatype Nexus Snapshots</name>
     <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
     <snapshots>
       <enabled>true</enabled>
     </snapshots>
     <releases>
       <enabled>false</enabled>
     </releases>
   </pluginRepository>
</pluginRepositories>
```

In the configuration section of the plugin you can set several properties to get things done right.

```prettyprint lang-xml
<configuration>
  <install>
    <!-- license code specified in ~/.m2/settings.xml -->
    <licenseCode>${ibm.liberty.license}</licenseCode>
  </install>

  <configFile>src/main/wlp/server.xml</configFile>
  <packageFile>${project.build.directory}/${project.name}-server.zip</packageFile>
  <include>usr</include>

  <bootstrapProperties>
    <Port>9080</Port>
  </bootstrapProperties>

  <features>
    <acceptLicense>true</acceptLicense>
    <whenFileExists>ignore</whenFileExists>
    <feature>ejbLite-3.1</feature>
    <feature>jaxws-2.2</feature>
  </features>
</configuration>
```

Let's have a closer look to the configuation:

* In the install section it's necessary to set the Liberty license code. This indicates that you accept the license code during Liberty Profile Server installation. The license code can be found in the [license agreement for the current version](http://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/downloads/wlp/8.5.5.5/lafiles/runtime/en.html) (The code is at the end after "D/N"). It's a best practice to configure a maven property within your Maven settings.xml.

* The `configFile` property points to the location of your ```server.xml```. This is the place where the configuration for your WLP server is defined (e.g. database configuration, endpoints, security, enabled features,  etc.). Have a look on [GitHub](https://github.com/cokeSchlumpf/liberty-maven-bluemix-sample/blob/master/src/main/wlp/server.xml) for the example ```server.xml```.

* The `packageFile` and the `include` properties are used during the packaging phase of the Maven build. The `include` property is set to `usr` to package only the application and the configuration - Not the whole runtime, since this will be provided by the CF Liberty Buildpack on Bluemix.

* Within the `bootstrapProperties` section you can configure properties for Liberty which will be used to replace variables within the `server.xml` during the startup of the WLP server with the Maven plugin. Keep in mind that you should only use variables which are also replaced by the Liberty Profile Buildpack on Bluemix ([details](https://www.ng.bluemix.net/docs/starters/liberty/index.html#optionsforpushinglibertyapplications)).

* Additional features which should be installed on top of the Liberty core runtime can be defined within the `features` section.

Finally the plugin executions must be configured. The `package-server` goal can be bound to the `packaging` phase of the Maven build. To install the liberty profile when it's necessary, the `create-server` goal should be triggered within a profile which checks the presence of the server:

```prettyprint lang-xml
<profiles>
  <profile>
    <id>install-liberty</id>

    <activation>
      <file>
        <missing>target/liberty/wlp/bin</missing>
      </file>
    </activation>

    <build>
      <plugins>
        <plugin>
          <groupId>net.wasdev.wlp.maven.plugins</groupId>
          <artifactId>liberty-maven-plugin</artifactId>
          <executions>
            <execution>
              <phase>compile</phase>
              <goals>
                <goal>create-server</goal>
                <goal>install-feature</goal>
              </goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

A pitfall of this approach is that the server won't be installed when executing `mvn clean install`, since the profile is not activated at the beginning due to the presence of the server. So you have to execute `mvn clean` and `mvn install` separately to work properly.

Now the Maven Liberty plugin is configured to download and install the server during compile time. One more step is needed to deploy the WAR file during the build process, so that you could start the server within the `test` phase to run component tests agains the running server. Just modify the target dir of the Maven WAR plugin:

```prettyprint lang-xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-war-plugin</artifactId>
  <version>2.6</version>
  <configuration>
    <outputDirectory>${project.build.directory}/liberty/wlp/usr/servers/defaultServer/apps</outputDirectory>
    <warName>${project.name}</warName>
  </configuration>
</plugin>
```

## Configure Cloud Foundry Maven Plugin
The cloud foundry maven plugin enables Maven to execute all commands you would usually do with the Cloud Foundry CLI manually. Let's have a look on its configuration:

```prettyprint lang-xml
<plugin>
  <groupId>org.cloudfoundry</groupId>
  <artifactId>cf-maven-plugin</artifactId>
  <version>1.1.1</version>

  <executions>
    <execution>
      <phase>deploy</phase>
      <goals>
        <goal>push</goal>
      </goals>
    </execution>
  </executions>

  <configuration>
    <!-- Disable the following lines if you already created the app on Bluemix -->
    <buildpack>https://github.com/cloudfoundry/ibm-websphere-liberty-buildpack.git</buildpack>
    <env>
      <IBM_JVM_LICENSE>${ibm.jvm.license}</IBM_JVM_LICENSE>
      <IBM_LIBERTY_LICENSE>${ibm.liberty.license}</IBM_LIBERTY_LICENSE>
    </env>
    <!-- End -->
    
    <path>${project.build.directory}/${project.name}-server.zip</path>
    <server>bluemix</server>
    <target>https://api.ng.bluemix.net</target>
    <org>your@org</org>
    <space>dev</space>
    <appname>wellnr-${project.name}</appname>
    <memory>512</memory>
    <diskQuota>1024</diskQuota>
    <healthCheckTimeout>60</healthCheckTimeout>

    <services>
      <service>
        <name>sqldb</name>
        <label>sqldb</label>
        <plan>sqldb_free</plan>
      </service>
    </services>
  </configuration>
</plugin>
```

The plugin enables you to setup all necessary options to deploy the app on Bluemix.

* The `buildpack` can be referenced via a Git repo. If this option is set, Bluemix will use the defined buildpack. **Note:** I've recognized that the published Buildpack on GitHub is a little bit different to the buildpack which is installed on Bluemix. The GitHub buildpack will not automatically configure services within your `server.xml` as described in the [Bluemix Documentation](https://www.ng.bluemix.net/docs/#starters/liberty/index.html#liberty). If you want to use these features you should create the app manually via the Bluemix UI and remove the buildpack configuration from the POM file.

* The `server` tag identifies the server id. The authentication properties for the server should be configured within your Maven settings.xml.

```prettyprint lang-xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>bluemix</id>
      <username>{username}</username>
      <password>{password}</password>
    </server>
  </servers>
</settings>
```

* Within the `services` section you can configure services which should be created (if not present) and bind to your app.

The example binds the CF `push` operation to the `deploy` phase of Maven.

## Conclusion

The presented configuration allows you to run `mvn install` to compile your application and install Liberty Profile to your target directory, if not present. Finally it will deploy (copy) the created WAR file to the WLP server.

Running `mvn liberty:run-server` would start the WLP server with your application on the context root.

`mvn deploy` packages the server configuration and pushes the app to your Bluemix account.

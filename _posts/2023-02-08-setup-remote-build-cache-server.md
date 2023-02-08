---
title: Setting up a remote build cache server for Maven
date: 2023-02-08
tags: Maven, GitHub, Docker
categories: [Eclipse]
---

Starting with Maven 3.9.0, you can use the [maven-build-cache-extension](https://maven.apache.org/extensions/maven-build-cache-extension/) to store build caches both locally and remotely. However the documentation for setting up a remote cache server is [fairly vague](https://maven.apache.org/extensions/maven-build-cache-extension/remote-cache.html#setup-http-server-to-store-artifacts).

But, after looking at the [integration tests](https://github.com/apache/maven-build-cache-extension/blob/master/src/test/java/org/apache/maven/buildcache/its/RemoteCacheDavTest.java#L91-L114) of `maven-build-cache-extension`, I figured out how to set up a Docker-run NginX cache server.


## Launching the remote cache server
Long story short, you need to execute a Docker command that looks like:

```
docker run -d \
           -e WEBDAV_USERNAME=admin \
           -e WEBDAV_PASSWORD=admin \
           -p 9080:80 \
           -v /path/to/cache/:/var/webdav/public \
           xama/nginx-webdav

```
This will start a [WebDAV server on NGinX](https://hub.docker.com/r/xama/nginx-webdav), running in a Docker container, accessible from `http://localhost:9080`. The username/password are `admin`/`admin` for authentication.

It's important that your local `/path/to/cache/` directory, mounted as a volume in the Docker container, contains a subdirectory (let's say `maven`) that will be used as the WebDAV collection holding your build caches.

## Configuring your build
The same username and password should then be stored in your Maven's `settings.xml`, like

```xml
  <servers>
    <server>
      <id>remote-cache</id>
      <username>admin</username>
      <password>admin</password>
    </server>
  </servers>
```

Your `.mvn/maven-build-cache-config.xml` should then look like:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<cache xmlns="http://maven.apache.org/BUILD-CACHE-CONFIG/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/BUILD-CACHE-CONFIG/1.0.0 https://maven.apache.org/xsd/build-cache-config-1.0.0.xsd">
  <configuration>
    <enabled>true</enabled>
    <remote enabled="true" id="remote-cache" saveToRemote="true">
      <url>http://localhost:9080/maven</url>
    </remote>
  </configuration>
</cache>
```

where `http://localhost:9080/maven` points at the `maven` collection living under your `/path/to/cache/` directory.


## Running Maven builds

Your 1st Maven build will show there's no cache found on the remote server, but the build artifacts will be stored remotely at the end of the build.

```
➜  mvnd-cache-test git:(main) ✗ ./mvnw clean verify
[INFO] Loading cache configuration from /Users/fbricon/Dev/souk/mvnd-cache-test/.mvn/maven-build-cache-config.xml
[INFO] Using XX hash algorithm for cache
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------------< foo.bar:mvn-cache-test >-----------------------
[INFO] Building mvn-cache-test 0.0.1-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- clean:3.2.0:clean (default-clean) @ mvn-cache-test ---
[INFO] Deleting /Users/fbricon/Dev/souk/mvnd-cache-test/target
[INFO] Going to calculate checksum for project [groupId=foo.bar, artifactId=mvn-cache-test]
[INFO] Scanning plugins configurations to find input files. Probing is enabled, values will be checked for presence in file system
[INFO] Found 2 input files. Project dir processing: 5, plugins: 2 millis
[INFO] Project inputs calculated in 19 ms. XX checksum [996f4a3b35403f08] calculated in 13 ms.
[INFO] Attempting to restore project foo.bar:mvn-cache-test from build cache
[INFO] Downloading http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/buildinfo.xml
[INFO] Cannot download http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/buildinfo.xml
org.apache.http.client.HttpResponseException: status code: 404, reason phrase: Not Found (404)
    at org.eclipse.aether.transport.http.HttpTransporter.handleStatus (HttpTransporter.java:544)
    at org.eclipse.aether.transport.http.HttpTransporter.execute (HttpTransporter.java:368)
    at org.eclipse.aether.transport.http.HttpTransporter.implGet (HttpTransporter.java:298)
    at org.eclipse.aether.spi.connector.transport.AbstractTransporter.get (AbstractTransporter.java:72)
    at org.apache.maven.buildcache.RemoteCacheRepositoryImpl.getResourceContent (RemoteCacheRepositoryImpl.java:165)
    at org.apache.maven.buildcache.RemoteCacheRepositoryImpl.findBuild (RemoteCacheRepositoryImpl.java:114)
    at org.apache.maven.buildcache.LocalCacheRepositoryImpl.findBuild (LocalCacheRepositoryImpl.java:183)
    at org.apache.maven.buildcache.CacheControllerImpl.findCachedBuild (CacheControllerImpl.java:212)
    at org.apache.maven.buildcache.CacheControllerImpl.findCachedBuild (CacheControllerImpl.java:179)
    at org.apache.maven.buildcache.BuildCacheMojosExecutionStrategy.execute (BuildCacheMojosExecutionStrategy.java:114)
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:160)
    at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject (LifecycleModuleBuilder.java:105)
    at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject (LifecycleModuleBuilder.java:73)
    at org.apache.maven.lifecycle.internal.builder.singlethreaded.SingleThreadedBuilder.build (SingleThreadedBuilder.java:53)
    at org.apache.maven.lifecycle.internal.LifecycleStarter.execute (LifecycleStarter.java:118)
    at org.apache.maven.DefaultMaven.doExecute (DefaultMaven.java:260)
    at org.apache.maven.DefaultMaven.doExecute (DefaultMaven.java:172)
    at org.apache.maven.DefaultMaven.execute (DefaultMaven.java:100)
    at org.apache.maven.cli.MavenCli.execute (MavenCli.java:821)
    at org.apache.maven.cli.MavenCli.doMain (MavenCli.java:270)
    at org.apache.maven.cli.MavenCli.main (MavenCli.java:192)
    at jdk.internal.reflect.DirectMethodHandleAccessor.invoke (DirectMethodHandleAccessor.java:104)
    at java.lang.reflect.Method.invoke (Method.java:578)
    at org.codehaus.plexus.classworlds.launcher.Launcher.launchEnhanced (Launcher.java:282)
    at org.codehaus.plexus.classworlds.launcher.Launcher.launch (Launcher.java:225)
    at org.codehaus.plexus.classworlds.launcher.Launcher.mainWithExitCode (Launcher.java:406)
    at org.codehaus.plexus.classworlds.launcher.Launcher.main (Launcher.java:347)
    at jdk.internal.reflect.DirectMethodHandleAccessor.invoke (DirectMethodHandleAccessor.java:104)
    at java.lang.reflect.Method.invoke (Method.java:578)
    at org.apache.maven.wrapper.BootstrapMainStarter.start (BootstrapMainStarter.java:47)
    at org.apache.maven.wrapper.WrapperExecutor.execute (WrapperExecutor.java:156)
    at org.apache.maven.wrapper.MavenWrapperMain.main (MavenWrapperMain.java:72)
[INFO] Remote cache is incomplete or missing, trying local build for foo.bar:mvn-cache-test
[INFO] Local build was not found by checksum 996f4a3b35403f08 for foo.bar:mvn-cache-test
[INFO]
[INFO] --- resources:3.3.0:resources (default-resources) @ mvn-cache-test ---
[INFO] skip non existing resourceDirectory /Users/fbricon/Dev/souk/mvnd-cache-test/src/main/resources
[INFO]
[INFO] --- compiler:3.10.1:compile (default-compile) @ mvn-cache-test ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/fbricon/Dev/souk/mvnd-cache-test/target/classes
[INFO]
[INFO] --- resources:3.3.0:testResources (default-testResources) @ mvn-cache-test ---
[INFO] skip non existing resourceDirectory /Users/fbricon/Dev/souk/mvnd-cache-test/src/test/resources
[INFO]
[INFO] --- compiler:3.10.1:testCompile (default-testCompile) @ mvn-cache-test ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/fbricon/Dev/souk/mvnd-cache-test/target/test-classes
[INFO]
[INFO] --- surefire:3.0.0-M7:test (default-test) @ mvn-cache-test ---
[INFO] Using auto detected provider org.apache.maven.surefire.junit4.JUnit4Provider
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running foo.bar.AppTest
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.055 s - in foo.bar.AppTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO]
[INFO] --- jar:3.2.2:jar (default-jar) @ mvn-cache-test ---
[INFO] Building jar: /Users/fbricon/Dev/souk/mvnd-cache-test/target/mvn-cache-test-0.0.1-SNAPSHOT.jar
[INFO] Saved to remote cache http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/buildinfo.xml
[INFO] Saved to remote cache http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/mvn-cache-test.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.358 s
[INFO] Finished at: 2023-02-08T15:49:28+01:00
[INFO] ------------------------------------------------------------------------
[INFO] Saving cache report on build completion
[INFO] Saved to remote cache http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/ea8602ee-e325-459e-b114-b272eccc1d88/build-cache-report.xml

```

If you replay the same command, you'll see the local cache is now used, the remote cache is skipped, the last request failure dating from less than 1h:

```
➜  mvnd-cache-test git:(main) ✗ ./mvnw clean verify
[INFO] Loading cache configuration from /Users/fbricon/Dev/souk/mvnd-cache-test/.mvn/maven-build-cache-config.xml
[INFO] Using XX hash algorithm for cache
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------------< foo.bar:mvn-cache-test >-----------------------
[INFO] Building mvn-cache-test 0.0.1-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- clean:3.2.0:clean (default-clean) @ mvn-cache-test ---
[INFO] Deleting /Users/fbricon/Dev/souk/mvnd-cache-test/target
[INFO] Going to calculate checksum for project [groupId=foo.bar, artifactId=mvn-cache-test]
[INFO] Scanning plugins configurations to find input files. Probing is enabled, values will be checked for presence in file system
[INFO] Found 2 input files. Project dir processing: 4, plugins: 2 millis
[INFO] Project inputs calculated in 17 ms. XX checksum [996f4a3b35403f08] calculated in 12 ms.
[INFO] Attempting to restore project foo.bar:mvn-cache-test from build cache
[INFO] Skipping remote lookup, last unsuccessful lookup less than 1h ago.
[INFO] Remote cache is incomplete or missing, trying local build for foo.bar:mvn-cache-test
[INFO] Local build found by checksum 996f4a3b35403f08
[INFO] Found cached build, restoring foo.bar:mvn-cache-test from cache by checksum 996f4a3b35403f08
[INFO] Skipping plugin execution (cached): resources:resources
[INFO] Skipping plugin execution (cached): compiler:compile
[INFO] Skipping plugin execution (cached): resources:testResources
[INFO] Skipping plugin execution (cached): compiler:testCompile
[INFO] Skipping plugin execution (cached): surefire:test
[INFO] Skipping plugin execution (cached): jar:jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.383 s
[INFO] Finished at: 2023-02-08T15:50:30+01:00
[INFO] ------------------------------------------------------------------------
[INFO] Saving cache report on build completion
[INFO] Saved to remote cache http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/b0a16886-0913-4f12-aa08-0361240252a6/build-cache-report.xml
```

You can delete the local cache to force Maven to connect to the cache server, and then relaunch a build. This time the remote cache will be hit and reused:

```
➜  mvnd-cache-test git:(main) ✗ rm -rf ~/.m2/build-cache/v1/
```
```
➜  mvnd-cache-test git:(main) ✗ ./mvnw clean verify
[INFO] Loading cache configuration from /Users/fbricon/Dev/souk/mvnd-cache-test/.mvn/maven-build-cache-config.xml
[INFO] Using XX hash algorithm for cache
[INFO] Scanning for projects...
[INFO]
[INFO] -----------------------< foo.bar:mvn-cache-test >-----------------------
[INFO] Building mvn-cache-test 0.0.1-SNAPSHOT
[INFO]   from pom.xml
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- clean:3.2.0:clean (default-clean) @ mvn-cache-test ---
[INFO] Deleting /Users/fbricon/Dev/souk/mvnd-cache-test/target
[INFO] Going to calculate checksum for project [groupId=foo.bar, artifactId=mvn-cache-test]
[INFO] Scanning plugins configurations to find input files. Probing is enabled, values will be checked for presence in file system
[INFO] Found 2 input files. Project dir processing: 4, plugins: 3 millis
[INFO] Project inputs calculated in 18 ms. XX checksum [996f4a3b35403f08] calculated in 12 ms.
[INFO] Attempting to restore project foo.bar:mvn-cache-test from build cache
[INFO] Downloading http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/buildinfo.xml
[INFO] Build info downloaded from remote repo, saving to: /Users/fbricon/.m2/build-cache/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/remote-cache/buildinfo.xml
[INFO] Found cached build, restoring foo.bar:mvn-cache-test from cache by checksum 996f4a3b35403f08
[INFO] Downloading http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/996f4a3b35403f08/mvn-cache-test.jar
[INFO] Skipping plugin execution (cached): resources:resources
[INFO] Skipping plugin execution (cached): compiler:compile
[INFO] Skipping plugin execution (cached): resources:testResources
[INFO] Skipping plugin execution (cached): compiler:testCompile
[INFO] Skipping plugin execution (cached): surefire:test
[INFO] Skipping plugin execution (cached): jar:jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.483 s
[INFO] Finished at: 2023-02-08T15:53:58+01:00
[INFO] ------------------------------------------------------------------------
[INFO] Saving cache report on build completion
[INFO] Saved to remote cache http://localhost:9080/maven/v1/foo.bar/mvn-cache-test/c53dbf84-8c16-44d2-8841-c229ed496e34/build-cache-report.xml
```

## Conclusion

Docker makes everything simpler! Setting up a remote build cache server to be used by the [maven-build-cache-extension](https://maven.apache.org/extensions/maven-build-cache-extension/) is no different. However keep in mind remote cache servers are *mostly* relevant for CI environments, so those instructions simply have an educational purpose.

You can check a self-contained demo project showcasing the usage of such remote cache server: [https://github.com/fbricon/mvn-cache-test](https://github.com/fbricon/mvn-cache-test).

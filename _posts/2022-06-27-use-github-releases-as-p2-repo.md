---
title: Use GitHub Releases to host an Eclipse P2 Repository
date: 2022-06-27
tags: Eclipse, P2, JBang, GitHub
categories: [Eclipse]
---

Hosting an Eclipse P2 repository is a great way to share your awesome Eclipse plugins with the rest of the world. However, finding a host can be a challenge. Who wants to pay for hosting nowadays? For some time, people (including me) have (ab)used Github pages as a hosting solution. Lorenzo Bettini has a pretty thorough [blog post](https://www.lorenzobettini.it/2021/03/publishing-an-eclipse-p2-composite-repository-on-github-pages/) about this. Even though Git is not meant to host binaries, it'll be good enough for most users.

But what if we could use [GitHub Releases](https://docs.github.com/en/repositories/releasing-projects-on-github) as a hosting solution instead? GitHub Releases is a free service specifically tailored to host and serve binaries on GitHub. Why hasn't anyone in the Eclipse community done this yet?

## The repository structure problem

Well the problem is that GitHub Releases hosts all files under a flat directory structure. Eclipse p2 repositories however, are hierarchical by default, like this:

```
p2 repo
│
│ p2.index
│ artifacts.jar
│ artifacts.xml.xz
│ contents.jar
│ contents.xml.xz
│
└───features/
│   │  feature1.jar
│   │  ...
│   └─ featureN.jar
│
└───plugins/
    │  plugin1.jar
    │  ...
    └─ pluginN.jar
```

There's no way (that I have found) to keep a hierarchical structure in GitHub Releases, so the only way to host a p2 repository there, would be to flatten said repository. After validating with teammate [Roland Grunberg](https://github.com/rgrunber) that flat p2 repos can actually work after some (manual) manipulations, all we had to do was to tweak our Maven build process to automate the flattening.

## JBang to the rescue
Unfortunately, [Eclipse Tycho](https://projects.eclipse.org/projects/technology.tycho) ([p2](https://www.eclipse.org/equinox/p2/) actually), doesn't seem to expose a way to configure the p2 repository structure. Fixing p2 and Tycho would be the obvious, long-term solution, but I'm both lazy and impatient, so went the scripting way to achieve our goal.

The script needs to perform 2 things:
 - move every file under `plugins/` and `features/` to the root of the p2 repository,
 - replace all references to `/plugins/` and `/features/` with `/`, in the `artifacts.xml` file compressed in `artifacts.jar` and `artifacts.xml.xz`.

There are probably a quasi-infinite number of ways to script this, but I chose the awesome [JBang](https://www.jbang.dev/) to do it. It's just Java, and is super easy to integrate into a Maven build.


> In Eclipse, the [`JBang / Eclipse Integration`](https://marketplace.eclipse.org/content/jbang-eclipse-integration) plugin makes it super easy to edit JBang scripts.
{: .prompt-tip }

Let's assume your p2 repository directory is `my.p2.repo.dir`. So here's the script, that you can store as `my.p2.repo.dir/jbang/repoflattener.java`:

```java
///usr/bin/env jbang "$0" "$@" ; exit $?
/**
 * Copyright 2022 Fred Bricon
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

//DEPS commons-io:commons-io:2.11.0
//DEPS org.tukaani:xz:1.9
//JAVA 11
import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.net.URI;
import java.nio.file.FileSystem;
import java.nio.file.FileSystems;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.Collections;
import java.util.Map;
import java.util.jar.JarInputStream;
import java.util.stream.Collectors;

import org.apache.commons.io.FilenameUtils;
import org.apache.commons.io.file.PathUtils;
import org.tukaani.xz.LZMA2Options;
import org.tukaani.xz.XZOutputStream;

public class repoflattener {

  public static void main(String... args) throws IOException {
    Path baseDir = (args == null || args.length == 0) ? Path.of("") : Path.of(args[0]);

    Path originalRepo = baseDir.resolve("target").resolve("repository");
    System.out.println("🛠 flattening " + originalRepo.toAbsolutePath());
    Path flatRepo = originalRepo.resolveSibling("flat-repository");
    if (Files.exists(flatRepo)) {
      PathUtils.deleteDirectory(flatRepo);
    }
    Files.createDirectory(flatRepo);

    var files = Files.walk(originalRepo).filter(path -> {
      if (!Files.isRegularFile(path)) {
        return false;
      }
      var fileName = FilenameUtils.getName(path.toString());
      return !fileName.startsWith("artifacts");
    }).collect(Collectors.toList());

    for (Path file : files) {
      PathUtils.copyFileToDirectory(file, flatRepo);
    }
    Path artifactsXml = extractAndRewriteArtifactXml(originalRepo.resolve("artifacts.jar"));
    createXZ(artifactsXml, flatRepo);
    createJar(artifactsXml, flatRepo);

    System.out.println("🙌 repository was flattened to " + flatRepo.toAbsolutePath());
  }

  private static Path extractAndRewriteArtifactXml(Path archive) throws IOException {
    var extracted = Files.createTempFile("artifacts", ".xml");
    try (JarInputStream archiveInputStream = new JarInputStream(
        new BufferedInputStream(Files.newInputStream(archive)))) {
      // we assume only 1 entry
      archiveInputStream.getNextJarEntry();
      streamRewrite(archiveInputStream, extracted);
    }
    if (Files.size(extracted) == 0) {
      throw new IOException("💥 Failed to extract/rewrite artifacts.xml");
    }
    return extracted;
  }

  private static void streamRewrite(InputStream src, Path dst) throws IOException {
    try (BufferedReader br = new BufferedReader(new InputStreamReader(src));
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(Files.newOutputStream(dst)))) {
      String line;
      while ((line = br.readLine()) != null) {
        line = line.replace("/plugins/", "/").replace("/features/", "/");
        bw.write(line);
        bw.newLine();
      }
    }
  }

  private static void createXZ(Path artifactsXml, Path flatRepo) throws IOException {
    Path artifactsXmlXZ = flatRepo.resolve("artifacts.xml.xz");
    try (BufferedInputStream in = new BufferedInputStream(Files.newInputStream(artifactsXml));
        XZOutputStream xzOut = new XZOutputStream(
            new BufferedOutputStream(Files.newOutputStream(artifactsXmlXZ)), new LZMA2Options());) {
      byte[] buffer = new byte[4096];
      int n = 0;
      while (-1 != (n = in.read(buffer))) {
        xzOut.write(buffer, 0, n);
      }
    }
  }

  private static void createJar(Path artifactXml, Path flatRepo) throws IOException {
    Path artifactsJar = flatRepo.resolve("artifacts.jar").toAbsolutePath();
    var env = Collections.singletonMap("create", "true");// Create the zip file if it doesn't exist
    URI uri = URI.create("jar:file:" + artifactsJar.toString().replace('\\', '/'));
    try (FileSystem zipfs = FileSystems.newFileSystem(uri, env)) {
      Path pathInZipfile = zipfs.getPath("artifacts.xml");
      Files.copy(artifactXml, pathInZipfile, StandardCopyOption.REPLACE_EXISTING);
    }
  }
}
```

So, assuming you're building your Eclipse p2 repository with Eclipse Tycho, you can add a `flat-repo` Maven profile to your `my.p2.repo.dir/pom.xml` file, invoking the [`jbang-maven-plugin`](https://github.com/jbangdev/jbang-maven-plugin) to execute `jbang/repoflattener.java` during the `package` phase, after Tycho generated the `my.p2.repo.dir/target/repository` directory:

```xml
  <profiles>
    <profile>
      <id>flat-repo</id>
      <build>
        <plugins>
          <plugin>
            <groupId>dev.jbang</groupId>
            <artifactId>jbang-maven-plugin</artifactId>
            <version>0.0.7</version>
            <executions>
              <execution>
                <id>run</id>
                <phase>package</phase>
                <goals>
                  <goal>run</goal>
                </goals>
                  <configuration>
                    <script>${project.basedir}/jbang/repoflattener.java</script>
                    <args>
                      <arg>${project.basedir}</arg>
                    </args>
                  </configuration>
              </execution>
            </executions>
          </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>
```
The `${project.basedir}` arg is the path to the root of your Eclipse p2 repository project, from which all computed paths will be relative to.

This script being saved as a [GitHub Gist](https://gist.github.com/fbricon/3c718d03f55c3ceba5dea570af4af5f8), we can reference its URL directly in our build  (yes JBang is that convenient), so that would give you:

```xml
<configuration>
  <script>https://gist.github.com/fbricon/3c718d03f55c3ceba5dea570af4af5f8</script>
  <args>
    <arg>${project.basedir}</arg>
  </args>
  <trusts>
    <trust>https://gist.github.com</trust>
  </trusts>
</configuration>
```

Either way, when calling `mvn verify -Pflat-repo`, the `repoflattener.java` script will be executed, and you shoud find your flat p2 repo under the `my.p2.repo.dir/target/flat-repository` directory. eg:

![Build output](https://user-images.githubusercontent.com/148698/176004440-8cd2750e-d312-479f-8a1e-7fff4fbeed9f.png)


## Releasing from a GitHub action:
Now, all we have left to do, is to "release" the `flat-repository` contents to GitHub Releases, and we're done. You could do this manually, or via any CI system, GitHub Actions is just a great way to do it.

The following is a GitHub action (you can save as `.github/worflows/CI.yaml`) that will do this on each push to the `main` branch, using the `latest` tag as a rolling release, thanks to [marvinpinto/action-automatic-releases](https://github.com/marvinpinto/action-automatic-releases):

```yaml
name: Build P2 Update Site

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: "11"
          distribution: "temurin"
          cache: "maven"

      - name: Build with Maven
        run: mvn --batch-mode --update-snapshots verify -Pflat-repo

      - name: Upload flat p2 update site
        if: github.ref == 'refs/heads/main'
        uses: marvinpinto/action-automatic-releases@latest
        with:
          repo_token: {% raw %}"${{secrets.GITHUB_TOKEN}}"{% endraw %}
          automatic_release_tag: "latest"
          prerelease: true
          title: "Development Build"
          files: |
            my.p2.repo.dir/target/flat-repository/*
```

Any push to the `main` branch will trigger a new release, overwriting the previous `latest` tag. The update site will be available as:

> https://github.com/your.org/your.repo/releases/download/latest/

This is perfect for snapshot builds or continous deployment. You can also configure your release process for immutable specific versions, as long as you upload `my.p2.repo.dir/target/flat-repository/*`. An exercise left to the reader.

## Conclusion
GitHub Releases is actually a great way to release your Eclipse plugins as a p2 repository, all you have to do is ensure that repository is configured to serve files from a flat structure. JBang makes it super easy to script those changes.

As a bonus you get [download statistics](https://tooomm.github.io/github-release-stats/?username=sidespin&repository=jre-discovery) for your plugins.

You can check an existing project leveraging this approach: [https://github.com/sidespin/jre-discovery](https://github.com/sidespin/jre-discovery).

Hope you find it useful!

# Dockerfile Maven

[![Build Status](https://travis-ci.com/spotify/dockerfile-maven.svg?branch=master)](https://travis-ci.com/spotify/dockerfile-maven)
[![Maven Central](https://img.shields.io/maven-central/v/com.spotify/dockerfile-maven.svg)](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.spotify%22%20dockerfile-maven)
[![License](https://img.shields.io/github/license/spotify/dockerfile-maven.svg)](LICENSE)

## 状态: 成熟
**目前插件比较成熟，基于这一点，我们不再开发，添加新功能，修复非关键性错误。**

本Maven插件将 Maven 与 Docker 集成在一起。

插件设计目标是：

  - 本插件需通过 `Dockerfile` 构建 docker 项目。
  - 使 Docker 构建过程 集成在 Maven 构建过程。默认当输入 `mvn
    package` 你就会得到 docker 镜像。 当输入 `mvn deploy`，发布 docker 镜像。
  - 清晰的构建目标，你可以输入 `mvn dockerfile:build` 然后 `mvn dockerfile:tag` 然后 `mvn dockerfile:push` 。这样消除了一些不明确类似 `mvn dockerfile:build -DalsoPush` 命令; 可以用 `mvn dockerfile:build dockerfile:push` 作为替换。
  - 集成 Maven build reactor（Maven 多模块构建）。你可以在另一个项目中依赖一个项目的 Docker 映像，Maven 将以正确的顺序构建项目。这对于多服务测试相当有用。

参与项目的维护，希望可以遵守以下协议 [Open Code of Conduct][code-of-conduct]

更新日志 [changelog for a list of releases][changelog]

[code-of-conduct]: https://github.com/spotify/code-of-conduct/blob/master/code-of-conduct.md
[changelog]: CHANGELOG.md

## 要求

本插件需要 Java 7 或者更高版本， 和 Apache Maven 3 活更高 (dockerfile-maven-plugin <=1.4.6 需要
Maven >= 3, 其他实例需要， Maven >= 3.5.2). 还有 在集成测试或在实践中使用插件，docker 也是必须的。

## 示例

更多例子，请看 [integration test](./plugin/src/it) 目录。

尤其 [advanced](./plugin/src/it/advanced) 测试展示了一项完整的服务，该服务包含两个使用 `helios-testing` 测试进行了集成测试的微服务 .

打包使用 `mvn package` 发布用 `mvn deploy`.  当然也可以简单点用 `mvn dockerfile:build` 。

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>dockerfile-maven-plugin</artifactId>
  <version>${dockerfile-maven-version}</version>
  <executions>
    <execution>
      <id>default</id>
      <goals>
        <goal>build</goal>
        <goal>push</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <repository>spotify/foobar</repository>
    <tag>${project.version}</tag>
    <buildArgs>
      <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
    </buildArgs>
  </configuration>
</plugin>
```

相对应的 `Dockerfile` :

```
FROM openjdk:8-jre
MAINTAINER David Flemström <dflemstr@spotify.com>

ENTRYPOINT ["/usr/bin/java", "-jar", "/usr/share/myservice/myservice.jar"]

# Add Maven dependencies (not shaded into the artifact; Docker-cached)
ADD target/lib           /usr/share/myservice/lib
# Add the service itself
ARG JAR_FILE
ADD target/${JAR_FILE} /usr/share/myservice/myservice.jar
```

**重要提示**

 通常 在 Maven artifact 里 我们引用 `project.build.directory` 变量说明
'target'-目录. 然后，这是绝对路径, 不支持 Dockerfile 的 ADD 命令. 因为任何此类源都必须在 Docker 构建的 *上下文* 中，因此必须由 *相对路径* 引用。（这句话意思是 你电脑里的路径 在 docker 里是没有的，所以要用相对路径）
查看 https://github.com/spotify/dockerfile-maven/issues/101

*不要使用 `${project.build.directory}` 作为引用构建目录*

## 它给我什么？

使用此插件进行构建有很多优点。

### 更快的构建时间

该插件可让您更一致地利用 Docker 缓存，通过在镜像中缓存 Maven 依赖项来极大地加快构建速度。 建议为了加快速度，不再使用 `maven-shade-plugin`。

### 一致的构建生命周期

你无需再使用:

    mvn package
    mvn dockerfile:build
    mvn verify
    mvn dockerfile:push
    mvn deploy

仅需要:

    mvn deploy

这将确保使用基本配置在正确的时间构建并推送镜像。

### 依赖于其他服务的 Docker 镜像

您可以依赖另一个项目的 Docker 信息，因为此插件在构建 Docker 镜像时会附加项目元数据。只需将此信息添加到任何项目中：

```xml
<dependency>
  <groupId>com.spotify</groupId>
  <artifactId>foobar</artifactId>
  <version>1.0-SNAPSHOT</version>
  <type>docker-info</type>
</dependency>
```

现在，您可以阅读有关您所依赖的项目的 Docker 镜像的信息：

```java
String imageName = getResource("META-INF/docker/com.spotify/foobar/image-name");
```

这对于集成测试非常有用，在集成测试中，您需要另一个项目的 Docker 镜像的最新版本。

请注意，您必须在POM（或父POM）中注册Maven扩展，以支持docker-info类型：

```xml
<build>
  <extensions>
    <extension>
      <groupId>com.spotify</groupId>
      <artifactId>dockerfile-maven-extension</artifactId>
      <version>${version}</version>
    </extension>
  </extensions>
</build>
```

## 使用其他依赖 Dockerfile 的 Docker 工具

你的项目看起来像这样：
```
a/
  Dockerfile
  pom.xml
b/
  Dockerfile
  pom.xml
```

现在，您可以将这些项目与 Fig 或 docker-compose 或与 Dockerfiles 一起使用的其他系统一起使用。例如，docker-compose.yml 可能类似于：

```yaml
service-a:
  build: a/
  ports:
  - '80'

service-b:
  build: b/
  links:
  - service-a
```

现在 `docker-compose up` 和 `docker-compose build` 将按预期工作。


## 用法

见 [usage docs](https://github.com/spotify/dockerfile-maven/blob/master/docs/usage.md).

## 认证方式

见 [authentication docs](https://github.com/spotify/dockerfile-maven/blob/master/docs/authentication.md).

## 发布

剪贴 Maven release:

```
mvn clean [-B -Dinvoker.skip -DskipTests -Darguments='-Dinvoker.skip -DskipTests'] \
  -Dgpg.keyname=<key ID used for signing artifacts> \
  release:clean release:prepare release:perform
```

我们使用 [`gren`](https://github.com/github-tools/github-release-notes#installation) 发版

```
gren release
```

# Chapter 3 — Maven

## What is Maven?

Maven automates the full build lifecycle of a Java project: creating, compiling, testing, packaging, and distributing.

**Core principle:** Convention over configuration — sensible defaults mean you only need to configure what differs from the standard.

Reference: https://cguntur.me/2020/05/23/understanding-apache-maven-part-1/

---

## Project Structure

```
project/
├── src/
│   ├── main/
│   │   ├── java/        # source files
│   │   └── resources/   # additional required files
│   └── test/
│       └── java/        # unit tests
├── target/              # generated on build — compiled classes and jar archives
└── pom.xml              # project configuration
```

---

## Standard Build Lifecycle

| Phase | What happens |
|-------|-------------|
| `validate` | Project structure is verified |
| `compile` | Source code is compiled |
| `test-compile` | Test code is compiled |
| `test` | Tests are executed |
| `package` | Compiled code is packaged (e.g. into a `.jar`) |

Each phase can be extended or overridden via plugins configured in `pom.xml`.

---

## `pom.xml` — Project Object Model

The `pom.xml` is the configuration file for the project. It inherits from the **Super POM** (Maven's built-in defaults). Anything you write here overrides the Super POM.

To inspect the full resolved configuration, open the **Effective POM** (`ProjectName-effective-pom.xml`) in IntelliJ.

### Coordinates — GAV

Every project, plugin, and dependency is identified by its **GAV**:

```xml
<groupId>com.example</groupId>       <!-- reverse domain name of the organisation -->
<artifactId>my-project</artifactId>  <!-- product name -->
<version>1.0-SNAPSHOT</version>      <!-- -SNAPSHOT means still in development -->
```

### Encoding (recommended baseline)

```xml
<properties>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

---

## Plugins

A plugin is a small program that carries out one phase of the lifecycle. Each phase has a default plugin bound to it (defined in the Super POM).

Plugins are downloaded from the public Maven repository (`http://maven.apache.org`).

### Plugin structure in `pom.xml`

```xml
<build>
    <plugins>
        <plugin>
            <!-- GAV identifies the plugin -->
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>

            <!-- configuration for the plugin -->
            <configuration>
                <release>11</release>
            </configuration>

            <!-- bind the plugin to a specific phase/goal -->
            <executions>
                <execution>
                    <id>default-compiler</id>   <!-- existing ID = replaces the Super POM entry -->
                    <phase>compile</phase>
                    <goals>
                        <goal>compile</goal>
                    </goals>
                </execution>
                <execution>
                    <id>default-testCompile</id>
                    <phase>test-compile</phase>
                    <goals>
                        <goal>testCompile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

- **Existing ID** (e.g. `default-compiler`) — replaces the Super POM's version of that execution.
- **New ID** — runs as an additional execution alongside the existing one.
- For the compiler and surefire plugins, you usually only need to update the GAV version; the Super POM already handles the phase binding.

After changing `pom.xml`, click the **Reload** button in the Maven tool window.

---

## JUnit — Adding a Test Dependency

Required libraries are declared as dependencies — Maven downloads the `.jar` from the public repository and sets all classpaths automatically.

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.3.2</version>
        <scope>test</scope>   <!-- only available during the test phase -->
    </dependency>
</dependencies>
```

GAV coordinates can be looked up at https://mvnrepository.com

### Running Tests — `maven-surefire-plugin`

The default surefire version in the Super POM is outdated and won't run JUnit 5 tests. Override it:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0-M5</version>
</plugin>
```

Run the `test` lifecycle phase in the Maven tool window — successful output looks like:

```
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```

---

## Running a Specific Class as a Goal

Use the JavaFX Maven plugin (or any run-capable plugin) to execute a specific main class:

Reference: https://github.com/openjfx/javafx-maven-plugin

```xml
<plugin>
    <groupId>org.openjfx</groupId>
    <artifactId>javafx-maven-plugin</artifactId>
    <version>0.0.4</version>
    <configuration>
        <mainClass>com.example.MyMainClass</mainClass>
    </configuration>
</plugin>
```

Run it via the double-Control shortcut in IntelliJ and type:

```
mvn javafx:run
```

To bind it to a lifecycle phase (e.g. `test`):

```xml
<executions>
    <execution>
        <id>run-app</id>
        <phase>test</phase>
        <goals>
            <goal>run</goal>
        </goals>
    </execution>
</executions>
```

---

## Packaging — Creating a JAR

Run the `package` lifecycle phase. Maven places the archive in the `target/` folder:

```
target/MyProject-1.0-SNAPSHOT.jar
```

---

## Executable JAR with Dependencies

The default `package` plugin does not bundle external libraries. Use `maven-assembly-plugin` to create a fat JAR that includes all dependencies and a manifest pointing to the main class:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>2.2-beta-5</version>
    <configuration>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <archive>
            <manifest>
                <mainClass>com.example.MyMainClass</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <!-- Use "make-assembly" to ADD alongside the default jar -->
            <!-- Use "default-jar" to REPLACE the default jar        -->
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Run the resulting archive:

```bash
java -jar MyProject-1.0-SNAPSHOT-jar-with-dependencies.jar
```

---

## Generating Documentation (JavaDoc)

### 1. Site plugin

The `maven-site-plugin` generates a project website (into `target/site/index.html`). The default version is outdated — override it:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-site-plugin</artifactId>
    <version>3.9.1</version>
</plugin>
```

### 2. JavaDoc plugin

JavaDoc is not generated automatically by the site plugin. Add `maven-javadoc-plugin` under the `<reporting>` tag (not `<build>`):

```xml
<reporting>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>3.2.0</version>
            <configuration>
                <javadocExecutable>${java.home}/bin/javadoc</javadocExecutable>
                <show>package</show>   <!-- includes package-visible members; private is excluded -->
            </configuration>
        </plugin>
        <!-- suppresses "empty version" warning for project-info-reports -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-project-info-reports-plugin</artifactId>
            <version>3.1.1</version>
        </plugin>
    </plugins>
</reporting>
```

Run the `site` lifecycle phase. The generated JavaDoc is accessible under **Project Reports** on the site's index page.

---

## Adding an External Library (Example: Guava)

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>23.5-jre</version>
</dependency>
```

---

## Glossary

| Term | Definition |
|------|-----------|
| **Artifact** | The build output — usually a `.jar` or `.war` archive |
| **POM** | Project Object Model; the `pom.xml` configuration file; inherits from the Super POM |
| **Dependency** | An external library required by the project |
| **GAV** | GroupID + ArtifactID + Version — uniquely identifies any plugin, dependency, or project |
| **Lifecycle** | A build process — an ordered sequence of phases, each with plugins bound to it |
| **Plugin** | A program that carries out one phase of the build process, or adds extra actions |
| **Goal** | A command sent to a plugin telling it what to do |
| **Super POM** | Maven's built-in default POM that every project inherits from |
| **Effective POM** | The fully resolved POM after merging the project's `pom.xml` with the Super POM |

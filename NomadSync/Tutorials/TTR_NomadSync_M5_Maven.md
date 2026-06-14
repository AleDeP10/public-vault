# Maven — Cheatsheet & Tutorial

## Lifecycle di base

```powershell
mvn clean          # elimina target/
mvn compile        # compila src/main/java
mvn test-compile   # compila src/test/java
mvn test           # compila + esegue tutti i test
mvn package        # compila + test + produce il JAR
mvn install        # package + copia nel repository locale (~/.m2)
```

I goal sono cumulativi: `mvn package` esegue anche `compile` e `test`. Per saltare i test: `mvn package -DskipTests`.

---

## Esecuzione selettiva dei test

```powershell
# singola suite
mvn test -Dtest=GitignoreServiceTest

# più suite
mvn test -Dtest=GitignoreServiceTest,LogServiceTest

# pattern wildcard
mvn test -Dtest="*ServiceTest"

# singolo metodo
mvn test -Dtest=GitignoreServiceTest#load_gitignoreNotPresent_createsFromScratch

# più metodi della stessa suite
mvn test -Dtest="GitignoreServiceTest#load_gitignoreNotPresent+load_emptyFile_returnsDefaultPatternsOnly"

# escludere una suite
mvn test -Dexclude=SocketServerTest
```

---

## Output e diagnostica

```powershell
# output dettagliato (stack trace completo)
mvn test -e

# debug massimo
mvn test -X

# silenziare i WARNING di build
mvn test -q

# report HTML dei test (in target/surefire-reports)
# generato automaticamente dopo ogni mvn test
```

I report XML/HTML sono in:

```
target/surefire-reports/
    *.xml           ← parsabile da CI (Jenkins, GitHub Actions)
    *.txt           ← leggibile a mano
```

---

## Dipendenze

```powershell
# lista tutte le dipendenze risolte
mvn dependency:tree

# cerca conflitti di versione
mvn dependency:tree -Dverbose

# scarica sorgenti e javadoc (utile in IntelliJ)
mvn dependency:sources
mvn dependency:resolve -Dclassifier=javadoc

# forza il re-download delle dipendenze corrotte
mvn dependency:purge-local-repository
```

---

## Build info e versioni

```powershell
# versione Maven installata
mvn --version

# versione effettiva del progetto
mvn help:effective-pom

# proprieta del progetto
mvn help:evaluate -Dexpression=project.version
mvn help:evaluate -Dexpression=project.groupId
```

---

## FAT JAR con maven-assembly-plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.6.0</version>
    <configuration>
        <archive>
            <manifest>
                <mainClass>io.aledep10.nomadsync.Main</mainClass>
            </manifest>
        </archive>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
        <finalName>NomadSync</finalName>
        <appendAssemblyId>false</appendAssemblyId>
    </configuration>
    <executions>
        <execution>
            <id>assemble-all</id>
            <phase>package</phase>
            <goals><goal>single</goal></goals>
        </execution>
    </executions>
</plugin>
```

```powershell
mvn package                         # produce target/NomadSync.jar
java -jar target/NomadSync.jar      # esecuzione standalone
```

---

## Risorse nella build

```xml
<!-- copia file extra in target/ -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>3.3.1</version>
    <executions>
        <execution>
            <id>copy-resources</id>
            <phase>package</phase>
            <goals><goal>copy-resources</goal></goals>
            <configuration>
                <outputDirectory>${project.build.directory}</outputDirectory>
                <resources>
                    <resource>
                        <directory>src/main/resources</directory>
                        <includes>
                            <include>NomadSync.bat</include>
                            <include>config.dev.properties</include>
                        </includes>
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## WARNING comuni e come eliminarli

**`'build.plugins.plugin.version' for maven-compiler-plugin is missing`**

Aggiungere la versione esplicita nel pom:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <release>21</release>
    </configuration>
</plugin>
```

**Mockito dynamic agent warning (Java 21)**

Aggiungere alla sezione `<properties>`:

```xml
<argLine>-XX:+EnableDynamicAgentLoading</argLine>
```

Oppure nel plugin surefire:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <argLine>-XX:+EnableDynamicAgentLoading</argLine>
    </configuration>
</plugin>
```

---

## Profili (environments)

```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation><activeByDefault>true</activeByDefault></activation>
        <properties>
            <config.file>config.dev.properties</config.file>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <config.file>config.prod.properties</config.file>
        </properties>
    </profile>
</profiles>
```

```powershell
mvn package -Pprod    # attiva profilo prod
```

---

## Struttura standard del progetto

```
pom.xml
src/
  main/
    java/           ← sorgenti produzione
    resources/      ← .properties, .bat, .json copiati in target/classes
  test/
    java/           ← sorgenti test
    resources/      ← risorse test (opzionale)
target/
  classes/          ← .class compilati da main
  test-classes/     ← .class compilati da test
  surefire-reports/ ← report XML/TXT dei test
  NomadSync.jar     ← fat JAR prodotto da package
```

---

## Variabili built-in nel pom.xml

|Variabile|Valore|
|---|---|
|`${project.version}`|versione dichiarata nel pom|
|`${project.groupId}`|groupId|
|`${project.artifactId}`|artifactId|
|`${project.build.directory}`|`target/`|
|`${project.build.sourceDirectory}`|`src/main/java`|
|`${java.version}`|property personalizzata (non built-in)|
|`${maven.compiler.release}`|versione Java per il compilatore|
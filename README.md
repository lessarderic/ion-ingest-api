# ION Ingest API

## Prerequisites
* Java 11

## Building
```
mvn clean install
```

## Usage

### Dependency Check (OWASP)

Clients using the Ingest API generated client code, or server code using the generated stubs,
can add the following to their `dependency-check-maven` configuration to inherit the
suppressions and avoid having to repeat them in their own project.

_Maven_

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.owasp</groupId>
            <artifactId>dependency-check-maven</artifactId>
            <configuration>
                <suppressionFiles>
                    <!-- Suppression file from ingest-api-cxf-client dependency -->
                    <suppressionFile>owasp/ingest-api-cxf-client/suppressions.xml</suppressionFile>
                    <!-- Local suppression file -->
                    <suppressionFile>${basedir}/owasp/suppressions.xml</suppressionFile>
                </suppressionFiles>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>com.connexta.ingest</groupId>
                    <artifactId>ingest-api-cxf-client</artifactId>
                    <version>${ingest-api.version}</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

_Gradle_

**TODO**

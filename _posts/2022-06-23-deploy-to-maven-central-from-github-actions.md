---
layout: post
title: Deploy Your GitHub Project to Maven Central from GitHub actions
date: 2022-06-23
tags: [ Java, GitHub ]
---

It's the day you want to publish your framework/tool/API/library, to sum up, your artifacts, into Maven Central, so the community can start using it. So, how to do it? You'd need to follow the official instructions from [here](https://central.sonatype.org/publish/publish-guide/#deployment). 

So, every time you want to release a new version of your artifacts, you'd need to perform the steps from the link. But what about if we'd have a repetitive release process that anybody could start? Here we are! We'll explain how to achieve exactly this using Maven and the Maven Release Plugin, and how to trigger the release from GitHub actions.

### Step 1: Request access to the Sonatype OSSRH JIRA

As we're going to be pushing artifacts to the OSSRH repository, we first need to request access. 
To do so, you need to signup to [the OSSRH JIRA site](https://issues.sonatype.org/) and then create a ticket to request permission to publish your project. You can use [this ticket](https://issues.sonatype.org/browse/OSSRH-81782) as an example.

**IMPORTANT:** The group-id must follow Maven naming conventions and be the reverse of a domain you own. For projects hosted on GitHub, it can start with com.github or io.github.

### Step 2: Prepare your Maven configuration

Before continuing with the release process, let's start preparing the Maven profile we'll use to start the release. We will only need to add this profile in your pom.xml file:

```xml
<profile>
    <id>release</id>
    <build>
        <plugins>
        <!-- we will add plugins here in the following steps --> 
        </plugins>
    </build>
</profile>
```

We'll need also to fulfill some information that Sonatype requires:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>YOUR GROUP ID</groupId>
  <artifactId>YOUR ARTIFACT ID</artifactId>
  <packaging>jar</packaging>
  <version>0.0.1-SNAPSHOT</version>
  <name>THE NAME</name>
  <description>THE DESCRIPTION</description>
  <url>https://github.com/org/repo</url>

  <!-- The license is mandatory -->
  <licenses>
    <license>
      <name>The Apache License, Version 2.0</name>
      <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
    </license>
  </licenses>

  <!-- The list of maintainers -->
  <developers>
    <developer>
      <id>github user</id>
      <name>user name</name>
      <email>user email</email>
    </developer>
  </developers>

  ... 
</project>
```

And finally, let's add the distribution management configuration that will be needed for the Maven Release plugin:

```xml
  <distributionManagement>
    <snapshotRepository>
      <id>ossrh</id>
      <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
    </snapshotRepository>
    <repository>
      <id>oss.sonatype</id>
      <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
  </distributionManagement>
```

See a full example of the pom.xml [here](https://github.com/yaml-path/YamlPath/blob/main/pom.xml).

### Step 3: Sign your artifacts

One of the requirements is that the artifacts are signed with GPG.

You first need to install [the GPG tool](http://www.gnupg.org/download/), and generate [a new key-par](https://central.sonatype.org/publish/requirements/gpg/#generating-a-key-pair) using "gpg --gen-key". 

When you have generated your key-par, you should see it using:

```
> gpg --list-keys

pub   rsa2048 2022-06-16 [SC]
      7CB022E3CFC857AEE78C81F6D480140178B9D7D8
uid        [  absoluta ] Your Name <xxx@gmail.com>
sub   rsa2048 2022-06-16 [E]
```

Now, you need to synchronize this key to the keyservers:

```
gpg --keyserver keyserver.ubuntu.com --send-keys 7CB022E3CFC857AEE78C81F6D480140178B9D7D8
gpg --keyserver keys.openpgp.org --send-keys 7CB022E3CFC857AEE78C81F6D480140178B9D7D8
gpg --keyserver pgp.mit.edu --send-keys 7CB022E3CFC857AEE78C81F6D480140178B9D7D8
```

Note that you don't need to send the key to all the servers as, eventually, they all are going to be synchronized with each other. However, if you don't want to wait, you can send the keys individually to each one.

And, finally, you need to append the GPG Maven plugin in the Maven release profile we created in step 2.:

```xml
    <profile>
      <id>release</id>
      <build>
        <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <version>3.0.1</version>
            <configuration>
              <gpgArguments>
                <arg>--pinentry-mode</arg>
                <arg>loopback</arg>
              </gpgArguments>
            </configuration>
            <executions>
              <execution>
                <id>sign-artifacts</id>
                <phase>verify</phase>
                <goals>
                  <goal>sign</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
        </plugins>
      </build>
    </profile>
```

### Step 4: Add the Source and the JavaDoc Maven plugins

Another requirement is to add the sources and the JavaDoc artifacts, so we need to add these two plugins as part of the Maven release profile as well:

```xml
<profile>
      <id>release</id>
      <build>
        <plugins>
          <!-- The maven-gpg-plugin configuration from step 3. --> 
          <!-- ... -->
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>3.2.1</version>
            <executions>
              <execution>
                <id>attach-sources</id>
                <goals>
                  <goal>jar-no-fork</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>3.4.0</version>
            <executions>
              <execution>
                <id>attach-javadocs</id>
                <goals>
                  <goal>jar</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
        </plugins>
      </build>
    </profile>
```

### Step 5: Configure the Maven Release plugin

The GitHub job to perform the release will use the Maven release plugin to do the actual work. There are a lot of benefits of using the Maven Release plugin:
- It ensures your artifact version conforms to the standards
- It will automatically tag your repository
- It will update your POM versions by removing the SNAPSHOT tag and by incrementing the version

So, let's add it to our Maven release profile as well:

```xml
<profile>
      <id>release</id>
      <build>
        <plugins>
          <!-- The maven-gpg-plugin configuration from step 3. --> 
          <!-- ... -->
          <!-- The maven-source-plugin, maven-javadoc-plugin configurations from step 4. --> 
          <!-- ... -->
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-release-plugin</artifactId>
            <version>2.5.3</version>
            <configuration>
              <autoVersionSubmodules>true</autoVersionSubmodules>
              <tagNameFormat>@{project.version}</tagNameFormat>
              <pushChanges>false</pushChanges>
              <localCheckout>true</localCheckout>
              <remoteTagging>false</remoteTagging>
              <arguments>-DskipTests=true</arguments>
            </configuration>
          </plugin>
          <plugin>
            <groupId>org.sonatype.plugins</groupId>
            <artifactId>nexus-staging-maven-plugin</artifactId>
            <version>1.6.13</version>
            <extensions>true</extensions>
            <configuration>
              <nexusUrl>https://s01.oss.sonatype.org/</nexusUrl>
              <serverId>ossrh</serverId>
              <autoReleaseAfterClose>true</autoReleaseAfterClose>
              <stagingProgressTimeoutMinutes>60</stagingProgressTimeoutMinutes>
            </configuration>
          </plugin>
        </plugins>
      </build>
    </profile>

    <scm>
        <connection>scm:git:git@github.com:org/repo.git</connection>
        <developerConnection>scm:git:git@github.com:org/repo.git</developerConnection>
        <url>https://github.com/org/repo</url>
        <tag>HEAD</tag>
    </scm>
```

The Maven release plugin will invoke the Nexus staging plugin and also use the configuration under the `scm`.

### Step 6: Log In to the OSSRH repository

Next, you need to configure Maven with the user you created in step 1. to connect with the OSSRH repository. To do so, you need to configure the Maven settings file called `maven-settings.xml` with:

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>ossrh</id>
      <username>YOUR OSSRH USER</username>
      <password>YOUR OSSRH PASSWORD</password>
    </server>
  </servers>
</settings>
```

Since you're not going to perform releases from your local machine, but from a GitHub job, you need somehow to provide the `YOUR OSSRH USER` and `YOUR OSSRH PASSWORD` values to be read by the GitHub job. 

Here, we could provide these values via repository or organization secrets. However, we decide to encrypt the Maven settings file we previously create using the key-par you generated in step 3.:

```
> gpg --sign --default-key YOUR_EMAIL maven-settings.xml
```

This command will generate the encrypted file `maven-settings.xml.gpg`. Let's copy it to `.github/release`. 

**Note:** We can decrypt back this file using the command "gpg --quiet --batch --yes --decrypt --passphrase="YOUR PASSPHARSE from step 3" --output maven-settings.xml .github/release/maven-settings.xml.gpg".

### Step 7: GitHub Secrets

We now need [to create the secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) we'll use during the release process:
- GPG_PRIVATE_KEY

We can get the private key of the key-par you generated in step 3 using one of [these commands](https://github.com/crazy-max/ghaction-import-gpg#prerequisites).

The private key will look like this:

```
-----BEGIN CERTIFICATE-----
MIID0DCCArigAwIBAgIBATANBgkqhkiG9w0BAQUFADB/MQswCQYDVQQGEwJGUjET
MBEGA1UECAwKU29tZS1TdGF0ZTEOMAwGA1UEBwwFUGFyaXMxDTALBgNVBAoMBERp
bWkxDTALBgNVBAsMBE5TQlUxEDAOBgNVBAMMB0RpbWkgQ0ExGzAZBgkqhkiG9w0B
CQEWDGRpbWlAZGltaS5mcjAeFw0xNDAxMjgyMDM2NTVaFw0yNDAxMjYyMDM2NTVa
MFsxCzAJBgNVBAYTAkZSMRMwEQYDVQQIDApTb21lLVN0YXRlMSEwHwYDVQQKDBhJ
bnRlcm5ldCBXaWRnaXRzIFB0eSBMdGQxFDASBgNVBAMMC3d3dy5kaW1pLmZyMIIB
IjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvpnaPKLIKdvx98KW68lz8pGa
RRcYersNGqPjpifMVjjE8LuCoXgPU0HePnNTUjpShBnynKCvrtWhN+haKbSp+QWX
SxiTrW99HBfAl1MDQyWcukoEb9Cw6INctVUN4iRvkn9T8E6q174RbcnwA/7yTc7p
1NCvw+6B/aAN9l1G2pQXgRdYC/+G6o1IZEHtWhqzE97nY5QKNuUVD0V09dc5CDYB
aKjqetwwv6DFk/GRdOSEd/6bW+20z0qSHpa3YNW6qSp+x5pyYmDrzRIR03os6Dau
ZkChSRyc/Whvurx6o85D6qpzywo8xwNaLZHxTQPgcIA5su9ZIytv9LH2E+lSwwID
AQABo3sweTAJBgNVHRMEAjAAMCwGCWCGSAGG+EIBDQQfFh1PcGVuU1NMIEdlbmVy
YXRlZCBDZXJ0aWZpY2F0ZTAdBgNVHQ4EFgQU+tugFtyN+cXe1wxUqeA7X+yS3bgw
HwYDVR0jBBgwFoAUhMwqkbBrGp87HxfvwgPnlGgVR64wDQYJKoZIhvcNAQEFBQAD
ggEBAIEEmqqhEzeXZ4CKhE5UM9vCKzkj5Iv9TFs/a9CcQuepzplt7YVmevBFNOc0
+1ZyR4tXgi4+5MHGzhYCIVvHo4hKqYm+J+o5mwQInf1qoAHuO7CLD3WNa1sKcVUV
vepIxc/1aHZrG+dPeEHt0MdFfOw13YdUc2FH6AqEdcEL4aV5PXq2eYR8hR4zKbc1
fBtuqUsvA8NWSIyzQ16fyGve+ANf6vXvUizyvwDrPRv/kfvLNa3ZPnLMMxU98Mvh
PXy3PkB8++6U4Y3vdk2Ni2WYYlIls8yqbM4327IKmkDc2TimS8u60CT47mKU7aDY
cbTV5RDkrlaYwm5yqlTIglvCv7o=
-----END CERTIFICATE-----
```

- GPG_PASSPHRASE

The passphrase you used when creating your key-par in step 3.

### Step 8: The GitHub Release workflow

At this point, we should have:
- our OSSRH user,
- our OSSRH JIRA ticket should have been resolved
- the `.github/release/maven-settings.xml.gpg` file created with our OSSRH user credentials
- the `pom.xml` file properly updated

Let's now add the GitHub Release workflow that will do the actual release!

```yaml
name: Release

on:
  pull_request:
    types: [closed]
    paths:
      - '.github/project.yml'

jobs:
  release:
    runs-on: ubuntu-latest
    name: release
    if: ${{github.event.pull_request.merged == true}}

    steps:
      - uses: radcortez/project-metadata-action@master
        name: Retrieve project metadata
        id: metadata
        with:
          github-token: ${{secrets.GITHUB_TOKEN}}
          metadata-file-path: '.github/project.yml'

      - uses: actions/checkout@v2

      - name: Import GPG key
        id: import_gpg
        uses: crazy-max/ghaction-import-gpg@v3
        with:
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          passphrase: ${{ secrets.GPG_PASSPHRASE }}

      - name: Install JDK 11
        uses: joschi/setup-jdk@e87a7cec853d2dd7066adf837fe12bf0f3d45e52
        with:
          distribution: 'temurin'
          java-version: 11
          check-latest: true

      - name: Configure Git author
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
      - name: Maven release ${{steps.metadata.outputs.current-version}}
        run: |
          gpg --quiet --batch --yes --decrypt --passphrase="${{secrets.GPG_PASSPHRASE}}" --output maven-settings.xml .github/release/maven-settings.xml.gpg
          git checkout -b release
          mvn -X -Prelease -B release:clean release:prepare -DreleaseVersion=${{steps.metadata.outputs.current-version}} -DdevelopmentVersion=${{steps.metadata.outputs.next-version}} -s maven-settings.xml
          git checkout ${{github.base_ref}}
          git rebase release
          mvn -X -Prelease -B release:perform -DskipTests -s maven-settings.xml
      - name: Push changes to ${{github.base_ref}}
        uses: ad-m/github-push-action@v0.6.0
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          branch: ${{github.base_ref}}

      - name: Push tags
        uses: ad-m/github-push-action@v0.6.0
        with:
          branch: ${{github.base_ref}}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          tags: true
```

And, finally, let's trigger the release process! 

To trigger the release process, we need to open a pull request (pull requests from forked repositories are not allowed) and create/update the following file at `.github/project.yml` which contains:

```yml
name: PROJECT NAME
release:
  current-version: 0.0.1
  next-version: 0.0.2-SNAPSHOT
```

This means that you'll be releasing the version `0.0.1` and then will set the version to `0.0.2-SNAPSHOT`. 

## Conclusions

This is a very easy process that once in place, all the team members within a team can trigger. 

Note that we have chosen to trigger the process after pull requests are closed, but we could have also chosen to do this when pushing in the `main` branch and the `project.yml` file is modified, so this is really up to the team to decide. 

To see a complete example, go [here](https://github.com/yaml-path/YamlPath). 
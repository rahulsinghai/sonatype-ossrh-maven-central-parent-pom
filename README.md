# sonatype-ossrh-maven-central-parent

Parent `pom.xml` to ease deployment to Maven Central via Sonatype Nexus.

## Getting started

Add dependencies and plugins from the `pom.xml` of this project.

Once this project is available in Maven repo, you can simply add this parent to your `pom.xml`:

```xml
<parent>
  <groupId>io.github.rahulsinghai</groupId>
  <artifactId>sonatype-ossrh-maven-central-parent-pom</artifactId>
  <version>1.0.0</version>
</parent>
```

## [Deploying on OSS/Maven](https://central.sonatype.org/pages/ossrh-guide.html)

-   [Create your JIRA account](https://issues.sonatype.org/secure/Signup!default.jspa)

-   [Create a New Project ticket](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134)

-   Modify project `pom.xml` with the [requirements](https://central.sonatype.org/pages/requirements.html)

-   Generate GPG keys (detailed below) to sign your artifacts to be uploaded.

-   Modify Maven settings.xml with details of OSS credentials. Details [here](https://central.sonatype.org/pages/apache-maven.html).

-   Perform the release.

## [Working with PGP Signatures](https://central.sonatype.org/pages/working-with-pgp-signatures.html)

**NOTE**: Prefer to do this exercise on Linux.

-   Download and install [Windows installer](https://www.gnupg.org/download/) (you can go with Simple installer for the current GnuPG).

-   If you are working on Windows, then add a new environment variable `GNUPGHOME` and set it to `C:\Users\<username>\.gnupg`

-   If it is Linux, then add `export GNUPGHOME=/home/<username>/.gnupg` in `~/.bashrc`
    
-   Open Console and verify installation:

    ```shell
    echo $GNUPGHOME
    gpg --version
    ```

-   Create a file `C:\Users\<username>\.gnupg\start-gpg.sh` or `/home/<username>/.gnupg/start-gpg.sh` with following line:

    ```shell
    gpg --homedir=/c/Users/<username>/.gnupg/GnuPG "$@"
    # Or for Linux:
    gpg --homedir=/home/<username>/.gnupg/GnuPG "$@"
    ```

-   Add git config for gpg to use `start-gpg.sh` as command:

    ```shell
    git config --global gpg.program "C:/Users/<username>/.gnupg/start-gpg.sh"
    # Or for Linux:
    git config --global gpg.program "/home/<username>/.gnupg/start-gpg.sh"
    ```

-   Generate GPG key pair:

    ```shell
    # Preferred
    gpg --full-generate-key
    # OR
    gpg --gen-keygpg
    ```

-   List keys:

    ```shell
    gpg --list-keys
    # OR
    gpg --homedir=/c/Users/<username>/.gnupg --list-keys

    gpg --list-secret-keys --keyid-format=long
        /Users/<username>/.gnupg/secring.gpg
        ------------------------------------
        sec   4096R/3FF5C34371567BD2 2021-08-10 [expires: 2021-09-10]
        uid                          username
        ssb   4096R/4FF317FD4BA89E7A 2021-08-10
    ```

-   You might want to set your default GPG signing key in Git. For this, paste the text below, substituting in the GPG key ID you'd like to use. In this example, the GPG key ID is 3FF5C34371567BD2:

        git config --global user.signingkey 3FF5C34371567BD2

-   Printing keys. Here `keyname` is the last 16 HEX characters of key that you get from `list-keys` command above.

        gpg --armor --export <keyname>

-   Add this key to your Github/Bitbucket/Gitlab, where you will be sending your signed commits: <https://github.com/settings/keys>
-   Distribute public key <https://central.sonatype.org/publish/requirements/gpg/#distributing-your-public-key>:

        export GPG_KEY=<last 16 HEX characters of key that you get from `list-keys` command above>
        export GPG_PASSPHRASE=**************

        gpg --keyserver hkp://keyserver.ubuntu.com --send-keys $GPG_KEY
        gpg --keyserver hkp://keyserver.ubuntu.com --recv-keys $GPG_KEY

-   Sign files using the private key:

    ```shell
    gpg -ab temp.jar
    gpg --verify temp.java.asc
    ```

-   Dealing with expired keys:

        gpg --edit-key <keyname>
        gpg> 1 
        gpg> expire
        gpg> save

-   Delete keys:

        gpg --delete-secret-keys <keyname>
        gpg --delete-keys <keyname>

-   Migrating Keys:

    -   Move the .gnupg directory: If the target computer does not have gnupg setup yet, or you do not mind replacing the existing keys and settings, then just copying over the key rings is the quickest way to make the move. One just needs to copy over the entire `~/.gnupg` directory, which contains all of your keys and key rings.

        ```shell
        tar -cvzf gnupg_backup_yyyymmdd.tgz /c/Users/<username>/.gnupg
        ```

    -   Individually export/import Public and Private Keys:

        To export all of your public php keys and save them to a file, run the command:

        ```shell
        gpg --export > gpg_public_keys.pgp
        gpg --export-secret-keys > gpg_private_keys.pgp
        ```

        To import on the target computer, run the following command.

        ```shell
        gpg --import < gpg_public_keys.pgp
        gpg --import < gpg_private_keys.pgp
        ```

## The Release Process

You have 2 options:

1.  You can do build directly from current version and release it using following command:

    ```bash
    mvn -Dmaven.test.skip=true -Dadditionalparam=-Xdoclint:none -Dpackaging=jar clean package javadoc:jar source:jar verify gpg:sign deploy
    ```

2.  Otherwise, you can use **maven-release-plugin**.

### maven-release-plugin

Let’s break apart the release process into small and focused steps.
We are performing a Release when the current version of the project is a SNAPSHOT version – say 1.0.0-SNAPSHOT.

**Tip!** Leave the `-SNAPSHOT` in your `pom.xml`. The **release** plugin removes it and increments to the next snapshot version for you automatically. It also tags the commit for you too.

#### release:clean

Cleaning a Release will:

-   delete the release descriptor (release.properties)
-   delete any backup POM files

#### release:prepare

Next part of the Release process is Preparing the Release; this will:

-   perform some checks – there should be no uncommitted changes and the project should depend on no SNAPSHOT dependencies
-   change the version of the project in the pom file to a full release number (remove SNAPSHOT suffix) – in our example – 1.0.0
-   run the project test suites
-   commit and push the changes
-   create the tag out of this non-SNAPSHOT versioned code
-   increase the version of the project in the pom – in our example – 1.0.1-SNAPSHOT
-   commit and push the changes

#### release:perform

The latter part of the Release process is Performing the Release; this will:

-   checkout release tag from SCM
-   build and deploy released code
-   This second step of the process relies on the output of the Prepare step – the `release.properties` file.

#### Steps

-   First Do a Dry Run

    **Note:** replace "sonatype-ossrh-maven-central-parent-pom" with your own artifactId, and replace 1.0.0 with your current version.

    ```bash
    export RELEASE_VERSION=0.2.0
    export DEV_VERSION=0.2.1

    mvn clean
    mvn release:clean
    mvn --batch-mode release:prepare -Dtag=sonatype-ossrh-maven-central-parent-pom-v"$RELEASE_VERSION" -DreleaseVersion=$RELEASE_VERSION -DdevelopmentVersion="$DEV_VERSION"-SNAPSHOT -DdryRun=true
    mvn release:rollback
    ```

-   Then Do the Real Thing:

    ```bash
    mvn release:clean
    mvn --batch-mode release:prepare -Dtag=sonatype-ossrh-maven-central-parent-pom-v"$RELEASE_VERSION" -DreleaseVersion=$RELEASE_VERSION -DdevelopmentVersion="$DEV_VERSION"-SNAPSHOT -DautoVersionSubmodules=true "-Darguments=-Dgpg.keyname=$GPG_KEY -Dgpg.passphrase=$GPG_PASSPHRASE"
    mvn release:perform "-Darguments=-Dgpg.keyname=$GPG_KEY -Dgpg.passphrase=$GPG_PASSPHRASE"
    ```

-   Publishing release in Github:

    Next thing to do is to upload the JAR file against the tag created by Maven release plugin above in Github.
    
    -   Open your project in Github, and click on **Releases** link.
    
    -   Click on **Tags** tab.
    
    -   Against the tag created by maven release plugin, click on the **...** > **Create release**
    
    -   Add the changelog in the description section.
    
    -   Upload the JAR file created for the tag and click on the **Publish release** button.

-   The Final Stretch

    The last thing to do now is to use the Sonatype web interface to release the artifacts:

    -   Log in to [Sontaype Nexus](https://oss.sonatype.org/) and follow the instructions [here](http://central.sonatype.org/pages/releasing-the-deployment.html).

    -   Search for `iogithubrahulsinghai` to find your repository.

    -   A full deployment to Maven Central requires two main steps: closing and releasing.

    -   This can be automated by making `autoReleaseAfterClose` as `true` in your `nexus-staging-maven-plugin` plugin configuration in pom file.

    -   If you don't want automatic release, then you will have to do it manually:

        -   First check <https://oss.sonatype.org/#stagingRepositories> for staged artifacts.
        -   You close the project’s "repository" and then you release the artifact that you want to publish to Maven Central repository.
        -   Check for Activity tab, in case closing fails. Drop the staging repositories that fail.
        -   Between the two steps you are allowed and encouraged to manually download the staged artifacts and test them.
        -   After closing the repository, you’ll need to hit the refresh button next to the "close" button in order for the "release" button to become activated.
        -   It may take a minute for the system to allow releasing so be patient.

    -   After a couple minutes of waiting, check [The Central Repository](https://search.maven.org/search?q=g:io.github.rahulsinghai) or [Maven Repository](https://mvnrepository.com/artifact/io.github.rahulsinghai/) for your released artifacts to be available on Maven Central.
    
    -   It might take longer for them to be available on [search.maven.org](http://search.maven.org/), 
        but as soon as they show up in the [repository](https://oss.sonatype.org/content/groups/public/io/github/rahulsinghai/), 
        one can access the artifacts by adding a dependency to the pom.xml file in a Maven project.

#### Releasing

Once you have your `settings.xml` sorted, and your gpg keys as per <http://central.sonatype.org/pages/apache-maven.html>, you can release in one command line to Maven Central:

```bash
mvn release:prepare && mvn release:perform
```

You will be prompted for versions.

My preference is to use this bash function in `.bashrc`:

```bash
function release() {
  RELEASE_VERSION=$1
  GPG_PASSPHRASE=$2
  mvn --batch-mode release:prepare \
    -Dtag=$RELEASE_VERSION \
    -DreleaseVersion=$RELEASE_VERSION \
    -DdevelopmentVersion=$RELEASE_VERSION.1 \
    -DautoVersionSubmodules=true \
    -Darguments=-Dgpg.passphrase=$GPG_PASSPHRASE && \
  mvn --batch-mode release:perform \
    -Darguments=-Dgpg.passphrase=$GPG_PASSPHRASE
}
```

which is called like this:

    release 1.0.0 <GPG_PASSPHRASE>

## Encrypting environment variables in travis.yml:

Encrypt environment variables with the public key attached to your repository using the travis gem:
If you do not have the travis gem installed, run `gem install travis`.

In your repository directory, run:

```bash
travis encrypt MY_SECRET_ENV=super_secret --add env.matrix
travis encrypt CODECOV_TOKEN=********************** --add env.global
```

Commit the changes to your .travis.yml.

## Code Styling

-   Please find instructions [here](https://github.com/HPI-Information-Systems/Metanome/wiki/Installing-the-google-styleguide-settings-in-intellij-and-eclipse) on how to configure your IntelliJ or Eclipse to format the source code according to Google style.
    Once configured in IntelliJ, format code as normal with `Ctrl + Alt + L`.

    Adding the XML file alone and auto-formatting the whole document could replace imports with wildcard imports, which isn't always what we want.

    -   To stop this from happening, Go to `File` → `Settings` → `Editor` → `Code Style` → `Java` and select the `Imports` tab.
    -   Set `Class Count to use import with '*'` and `Names count to us static import with '*'` to a higher value; anything over `999` should be fine.

    You can now reformat code throughout your project without imports being changed to Wildcard imports.

-   You also need to use `maven-git-code-format` plugin in `pom.xml` to auto format the code according to Google code style before any Git commit.

## [Markdown formatting](https://github.com/remarkjs/remark/tree/master/packages/remark-cli)

Use **remark-cli** to format markdown files.
It ensures a single style is used: list items use one type of bullet (_, -, +), emphasis (_ or \_) and importance (\_\_ or \*\*) use a standard marker, table fences are aligned, and more.

-   Install `remark-cli` and `remark-preset-lint-recommended`

    ```bash
    npm install remark-cli -g
    npm install remark-preset-lint-recommended -g

    # Add a table of contents to `README.md`
    remark README.md --use toc --output

    # Lint markdown files in the current directory
    # according to the markdown style guide.
    remark README.md --use remark-preset-lint-recommended -o

    # Rewrite all applicable files
    remark . -o
    ```

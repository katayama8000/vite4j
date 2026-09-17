# Publishing

Releases go to [Maven Central](https://central.sonatype.com) through the Central Portal. The pom
carries everything the build needs; what is left is an account, a namespace and a signing key, and
those cannot be scripted.

## One-time setup

### 1. A Central Portal account

Sign in at [central.sonatype.com](https://central.sonatype.com) with the GitHub account that owns
this repository.

### 2. Claim the namespace

Add the namespace `io.github.katayama8000`. The Portal hands back a verification key and asks for a
public repository named exactly that, on the same GitHub account. Create it, press **Verify
Namespace**, and delete the repository afterwards.

The namespace has to match the `groupId` in the pom.

### 3. A signing key

Central will not accept an unsigned artifact.

```sh
gpg --gen-key                                  # RSA, no expiry is fine
gpg --list-keys --keyid-format short           # note the key id
gpg --keyserver keyserver.ubuntu.com --send-keys <KEY_ID>
```

The public key has to be on a keyserver before the upload is validated, and it can take a few minutes
to propagate.

### 4. Credentials in `~/.m2/settings.xml`

Generate a user token in the Portal (**Account → Generate User Token**). It gives a username and a
password, neither of which is your GitHub login.

```xml
<settings>
  <servers>
    <server>
      <id>central</id>
      <username>TOKEN_USERNAME</username>
      <password>TOKEN_PASSWORD</password>
    </server>
  </servers>
</settings>
```

The `id` has to read `central`, matching `publishingServerId` in the pom.

## Releasing

Central does not take snapshots, so the version has to be a release one.

```sh
mvn versions:set -DnewVersion=0.1.0
mvn clean deploy -Prelease
```

That builds the jar, the sources jar and the javadoc jar, signs all three, and uploads the bundle.
`autoPublish` is off, so the Portal validates it and then waits: nothing is public until you press
**Publish** in the deployment view. Check the coordinates and the contents there first, because a
version that has been published cannot be replaced or withdrawn.

Then tag it and put the repository back on a snapshot:

```sh
git tag v0.1.0 && git push origin v0.1.0
mvn versions:set -DnewVersion=0.2.0-SNAPSHOT
```

It usually takes ten minutes or so before the artifact is searchable, longer before it appears in
search.maven.org.

## What can go wrong

| | |
| --- | --- |
| `401 Unauthorized` | The token in `settings.xml` is the GitHub password rather than the generated user token, or the server `id` is not `central` |
| The Portal rejects the bundle for a missing signature | The key is not on a keyserver yet, or `-Prelease` was left off |
| It rejects it for a missing field | `name`, `description`, `url`, `licenses`, `developers` and `scm` are all required. They are in the pom; check they survived any edit |
| The namespace is not verified | The verification repository has to be public and named exactly as the Portal spelled it |

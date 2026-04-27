# gradle-jfrog-private-test
BKLN-2092 E2E test (Gradle counterpart): Gradle project consuming a private artifact from JFrog (com.backline.test:demo-lib:1.0.0) that transitively pulls in vulnerable log4j-core. Validates that the JFrogRegistryMavenPackageRegistryHandler-written ~/.gradle/init.gradle authenticates correctly.

# ComposeToJAR

Easy to use Gradle Kotlin DSL code to transform your Desktop Compose app into a universal JAR.

I was struggling with creating native distributables for Jetpack Compose on every single OS.

Jetbrains doesn't give a guide on this nor does anybody else as far as I've seen.

**Please note that this hasn't been tested on Mac, but should work just fine.**

This isn't a complete guide on everything, but it should help you get started.

**Every single gradle task is going to be under the group ``custom jar``**

Inside of your main modules `build.gradle.kts`, you are going to create a new task to automate this process for you.

Before you start, add these imports to the top of your `build.gradle.kts`

```
import org.jetbrains.kotlin.org.apache.commons.compress.archivers.tar.TarArchiveInputStream
import java.net.HttpURLConnection
import java.net.URI
import java.util.zip.GZIPInputStream
```

First add this to your `build.gradle.kts` file outside any other scope.

```
val graalVersion = "21.0.2"
val baseUrl = "https://github.com/graalvm/graalvm-ce-builds/releases/download/jdk-$graalVersion"

enum class GraalJDK(
    val folderName: String,
    val filePattern: String,
    val extension: String
) {

    WINDOWS_AMD64(
        "windows-amd64",
        "graalvm-community-jdk-%s_windows-x64_bin.zip",
        "zip"
    ),
    LINUX_AMD64(
        "linux-amd64",
        "graalvm-community-jdk-%s_linux-x64_bin.tar.gz",
        "tar.gz"
    ),
    LINUX_ARM64(
        "linux-arm64",
        "graalvm-community-jdk-%s_linux-aarch64_bin.tar.gz",
        "tar.gz"
    ),
    MAC_AMD64(
        "mac-amd64",
        "graalvm-community-jdk-%s_macos-x64_bin.tar.gz",
        "tar.gz"
    ),
    MAC_ARM64(
        "mac-arm64",
        "graalvm-community-jdk-%s_macos-aarch64_bin.tar.gz",
        "tar.gz"
    );

    fun resolvedFileName(version: String): String {
        return filePattern.format(version)
    }
}
```

GraalVM is going to be used since it's on github and easy to retrieve. You can swap it for any other JDK if you want.

Notice the `%s` placeholder in the names. Kotlin enums require constants or literals, so we must solve this issue with string formatting which is what `resolvedFileName` is for. 

*Always remember this if you're going to use another JDK.*

Now add this gradle task:

```
tasks.register("downloadJdks") {

    group = "custom jar"
    description = "Downloads and extracts JDKs into jdks/runtime-*"

    doLast {

        val jdksDir = file("jdks")
        jdksDir.mkdirs()

        for (jdk in GraalJDK.values()) {

            val platform = jdk.folderName
            val fileName = jdk.resolvedFileName(graalVersion)
            val ext = jdk.extension

            val downloadUrl = "$baseUrl/$fileName"
            val outputFile = File(jdksDir, fileName)
            val runtimeDir = File(jdksDir, "runtime-$platform")

            if (runtimeDir.exists()) {
                println("Skipping $platform. JDK already exists.")
                continue
            }

            val connection = URI(downloadUrl).toURL().openConnection() as HttpURLConnection
            connection.requestMethod = "HEAD"
            val remoteSize = connection.contentLengthLong
            if (outputFile.exists() && outputFile.length() == remoteSize) {
                println("File $fileName already downloaded.")
            } else {
                println("Downloading $platform JDK ($fileName)")
                outputFile.outputStream().use { out ->
                    URI(downloadUrl).toURL().openStream().use { input ->
                        input.copyTo(out)
                    }
                }
            }

            println("Extracting $platform JDK")

            if (ext == "zip") {
                copy {
                    from(zipTree(outputFile))
                    into(jdksDir)
                }
            } else if (ext == "tar.gz") {
                TarArchiveInputStream(GZIPInputStream(outputFile.inputStream())).use { tarIn ->
                    var entry = tarIn.nextEntry
                    while (entry != null) {
                        val outFile = File(jdksDir, entry.name)
                        if (entry.isDirectory) {
                            outFile.mkdirs()
                        } else {
                            outFile.parentFile.mkdirs()
                            outFile.outputStream().use { out ->
                                tarIn.copyTo(out)
                            }
                        }
                        entry = tarIn.nextEntry
                    }
                }
            }

            val extractedDir = jdksDir.listFiles()?.firstOrNull {
                it.name.startsWith("graalvm-community") && it.isDirectory
            }

            val contentsHome = File(extractedDir, "Contents/Home")
            val isMac = contentsHome.exists()

            val runtimeTarget = if (isMac) contentsHome else extractedDir
            runtimeTarget?.renameTo(runtimeDir)

            if (isMac) {
                extractedDir?.deleteRecursively()
            }

            val debloatTargets = listOf(
                "lib/src.zip",
                "lib/missioncontrol",
                "lib/visualvm",
                "lib/plugin.jar",
                "jmods",
                "lib/jfr",
                "lib/oblique-fonts",
                "lib/javafx",
                "release",
                "bin/jaotc",
                "lib/installer",
                "lib/classlist",
                "lib/dt.jar"
            )

            debloatTargets.forEach { relativePath ->
                File(runtimeDir, relativePath).deleteRecursively()
            }

            if (isMac) {
                listOf(
                    File(runtimeDir.parentFile, "Contents/MacOS"),
                    File(runtimeDir.parentFile, "Contents/Info.plist")
                ).forEach { f ->
                    if (f.exists()) {
                        f.deleteRecursively()
                    }
                }
            }
        }
    }
}
```

This task is going to download each individual JDK, debloat it as much as possible and create a folder with it's respective name.

This is a one time task as it checks if the JDKs exist later on.

Next you are going to add this task to package the universal JAR:

```
tasks.register<Jar>("packageFatJar") {
    
    val projectName = "YourProjectName"
    
    group = "custom jar"
    description = "Builds a fat JAR."
    archiveBaseName.set(projectName)
    //archiveClassifier.set("all") if you want to append the jar name
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
    destinationDirectory.set(layout.buildDirectory.dir("custom-jars"))

    //use your main class
    manifest {
        attributes["Main-Class"] = "MainKt"
    }

    doFirst {
        val jarFile = destinationDirectory.get().file(
            "$projectName-${project.version}.jar"
        ).asFile
        if (jarFile.exists()) {
            jarFile.delete()
        }
    }

    //you can add extra sourceSets if your program depends on them.
    //sourceSets["main"].java.srcDir("build/generated/source/kapt/main")
    //you can also include kapt as well or any other tasks
    //dependsOn("kaptKotlin")
    //this will include all the jars and configurations
    dependsOn(configurations.runtimeClasspath)

    //change to your own sourceSets name
    from(sourceSets["main"].output)

    from({
        configurations.runtimeClasspath.get().map { file ->
            if (file.isDirectory) {
                file
            } else {
                zipTree(file).matching {
                    exclude(
                        "META-INF/*.SF",
                        "META-INF/*.DSA",
                        "META-INF/*.RSA",
                        "META-INF/*.EC"
                    )
                }
            }
        }
    })

    //include resources from your sourceset
    from(sourceSets["main"].resources)

    doLast {
        val jarFile = destinationDirectory.get().file("$projectName-${project.version}.jar").asFile
        if (jarFile.exists()) {
            println("The fatJar has been created in: ${jarFile.absolutePath}")
        } else {
            println("Failed to find the created fatJar.")
        }
    }
}
```

**Note that your `project.version` is `version = "0.0.1"` inside your `build.gradle.kts`**

Now inside `build.gradle.kts` add this function outside of any scope:

```
fun isFatJarBuild(): Boolean {
    return project.gradle.startParameter.taskNames.any {
        it.contains("packageFatJar", ignoreCase = true) ||
                it.contains("runFatJar", ignoreCase = true) ||
                it.contains("packageJARDistributables", ignoreCase = true)
    }
}
```

Then inside of your `dependecies` block, you are going to add this:

```
//... other dependencies... ie: implementation(compose.material3)
//should be the latest. im not entirely sure if it should match your current compose version.
val skikoVersion = "0.9.17"

    val allSkikoLibs = listOf(
        "org.jetbrains.skiko:skiko-awt-runtime-windows-x64:$skikoVersion",
        "org.jetbrains.skiko:skiko-awt-runtime-linux-x64:$skikoVersion",
        "org.jetbrains.skiko:skiko-awt-runtime-linux-arm64:$skikoVersion",
        "org.jetbrains.skiko:skiko-awt-runtime-macos-x64:$skikoVersion",
        "org.jetbrains.skiko:skiko-awt-runtime-macos-arm64:$skikoVersion"
    )

    if (isFatJarBuild()) {
        allSkikoLibs.forEach {
            implementation(it)
        }
    } else {
        implementation(compose.desktop.currentOs)
    }
```

Which will include all compose libraries for every single os to officially make it a universal JAR. Since you are creating separate zips for each OS, you can technically still split these, but I though this was easier.

This is going to check if it's universal or fat JAR build and include them, else it will just add them for your current OS and is crucial.

**Don't change the task names. If you do, be sure to adjust the `isFatJarBuild` function.**

Now finally you are going to add the task:

```
tasks.register("packageJARDistributables") {

    val projectName = "YourProjectName"

    group = "custom jar"
    description = "Creates a distributable folder for each OS with bundled JDK and launch scripts."

    dependsOn("downloadJdks")
    dependsOn("packageFatJar")
    //dependsOn(":bootstrap:packageBootstrapJar") any other tasks it should depend on

    doLast {

        val jarName = "$projectName-${project.version}.jar"
        val outputJar = file("build/custom-jars/$jarName")
        val jdkFolder = file("jdks")
        val distRoot = file("build/distributions")

        distRoot.listFiles()?.forEach { it.deleteRecursively() }

        for (graalJDK in GraalJDK.values()) {

            val runtimeDir = File(
                jdkFolder,
                "runtime-${graalJDK.folderName}"
            )

            if (!runtimeDir.exists()) {
                println("Skipping ${graalJDK.folderName}. JDK not found at: ${runtimeDir.absolutePath}")
                continue
            }

            val distDir = File(
                distRoot,
                "ZenDownloader-${graalJDK.folderName}"
            )
            distDir.deleteRecursively()
            distDir.mkdirs()

            val appDir = File(distDir, "app").also { it.mkdirs() }
            outputJar.copyTo(File(appDir, jarName), overwrite = true)

            File(distDir, "runtime").also {
                runtimeDir.copyRecursively(it, overwrite = true)
            }

            val isWindows = graalJDK.folderName.startsWith("windows")
            if (isWindows) {
                File(distDir, "run.bat").writeText(
                    """
                    @echo off
                    set DIR=%~dp0
                    "%DIR%runtime\bin\java.exe" -jar "%DIR%\app\$jarName"
                    pause
                    """.trimIndent()
                )
            } else {
                File(distDir, "run.sh").apply {
                    writeText(
                        """
                        #!/bin/bash
                        DIR="$(cd "$(dirname "$0")" && pwd)"
                        "${'$'}DIR/runtime/bin/java" -jar "${'$'}DIR/app/$jarName"
                        """.trimIndent()
                    )
                    setExecutable(true)
                }
            }

            val zipFile = File(distRoot, "$jarName-${graalJDK.folderName}.zip")
            ant.withGroovyBuilder {
                "zip"("destfile" to zipFile.absolutePath) {
                    "fileset"(
                        "dir" to distRoot.absolutePath,
                        "includes" to "${distDir.name}/**"
                    )
                }
            }

            println("Created distributable: ${zipFile.name}")

        }

    }
}
```

This task will package everything inside of their respective packages which will include `.bat` or `.sh` scripts for easy running.

Since this task is placing your universal JAR inside the `app` folder, the scripts are written to use it. You can change this if you know what you're doing.

Now inside your gradle tasks, look for the group labeled `custom jar`, open it and run `packageJARDistributables` which will package everything all together.

You can also only create the JAR if you please with `packageFatJar`.

If you want to build the jar and run it, you can also add this task:

```
tasks.register("runFatJar") {
    
    val projectName = "YourProjectName"
    
    group = "custom jar"
    description = "Builds and runs the fat JAR."

    dependsOn("packageFatJar")

    doLast {
        val jarName = "$projectName-${project.version}.jar"
        val jarFile = layout.buildDirectory.file("custom-jars/$jarName").get().asFile

        if (jarFile.exists()) {

            val injected = project.objects.newInstance<InjectedExecOps>()

            println("Launching: ${jarFile.absolutePath}")

            injected.execOps.javaexec {
                mainClass.set("-jar")
                args = listOf(jarFile.absolutePath)
            }

        } else {
            throw GradleException("fatJar not found at: ${jarFile.absolutePath}")
        }
    }
}
```

Everything will be created inside your projects `build` folder.

`project-folder > build > custom-jars`

`project-folder > build > distributions`


That's it! I won't provide much other details, but I will fix any issues.

I hope this has helped you. 

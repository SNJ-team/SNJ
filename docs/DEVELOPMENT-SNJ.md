# SNJ

## About SNJ
- SNJ is an experimental framework for creating Minecraft Paper plugins in Swift using JNI and swift-java.
this is how it works:
```
Minecraft Server
    ↓
Paper Plugin (Java)
    ↓ JNI
Swift dylib
    ↓
SwiftJava bridge
    ↓
Java Runtime / Bukkit API
```


## Important
**Current limitations**
- Apple Sillicon only (currently tested)
- Manual path configuration
- No automated dependency managment yet 


## Dependencies
- Java 25+
- Gradle
- Swift 6.2.x+
- Xcode


## Project Structure Example
```
SNJ/
	libs/
    ├── swift-java/								# From swift-java github
    Projects/
		SwiftPlugin/
		├── Package.swift                          	# Swift package config
		├── Sources/SwiftPlugin/
    	│   ├── PaperAPI/							# ...
		│   ├── Plugin.swift                       	# Swift Code
		│   └── SwiftPluginModule+SwiftJava.swift  	# Generated jextract Swift Java bridge
		│
		└── java/
    		├── build.gradle.kts                   	# Gradle config for java 
    		├── generated/
    		│   ├── java/com/example/swiftplugin/
    		│   │   └── SwiftPlugin.java           	# Generated jextract – Java wrap
    		│   └── swift/
    		│       └── SwiftPluginModule+SwiftJava.swift  # ...
    		└── src/main/
	        	├── java/com/example/swiftplugin/
    	    	│   ├── Main.java					# Main java class (extends JavaPlugin)
        		│   └── SwiftFunc.java          	# Java using swift functions
        		└── resources/
            		└── plugin.yml                 	# ...
```

## Getting started
1. check your Swift and Java version
```swift --version``` (must be 6.2.x+)
```java --version``` (must be 25+)
2. create your working directory as in example
```zsh
mkdir -p ~/SNJ/libs ~/SNJ/Projects
```
3. clone apple's swift-java git repo in SNJ/libs
```zsh
cd ~/SNJ/libs
git clone https://github.com/swiftlang/swift-java.git       
cd ~/SNJ/libs/swift-java
```
4. collect swift-java libraries to local maven repo ``(~/.m2)``
```zsh
cd ~/SNJ/libs/swift-java
export JAVA_HOME=$JAVA_HOME_25
./gradlew publishToMavenLocal -PskipSamples=true
```
5. now, create your first project!
```zsh
cd ~/SNJ/Projects
mkdir YourPluginName
cd YourPluginName
swift package init --type library --name YourPluginName
```
6. now open your project (we will use Xcode)
```zsh
open Package.swift
```
7. replace existing code with this, don't forget to change names

```swift
import PackageDescription

let package = Package(
    name: "YourPluginName",
    platforms: [
      .macOS(.v15),
    ],
    products: [
        .library(
            name: "YourPluginName",
            type: .dynamic,
            targets: ["YourPluginName"]
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/swiftlang/swift-java.git", branch: "main")
    ],
    targets: [
        .target(
            name: "YourPluginName",
            dependencies: [
                .product(name: "SwiftJava", package: "swift-java"),
                .product(name: "SwiftRuntimeFunctions", package: "swift-java"),
                .target(name: "PaperAPI"),
            ]
        ),
        .target(
            name: "PaperAPI",
            dependencies: [
                .product(name: "SwiftJava", package: "swift-java"),
            ],
            exclude: ["swift-java.config"]
        ),
        .testTarget(
            name: "YourPluginNameTests",
            dependencies: ["YourPluginName"]
        ),
    ]
)
```


8. create Java structure on your project
```zsh
mkdir -p ~/SNJ/projects/YourPluginName/java/src/main/java/com/yourpackage
mkdir -p ~/SNJ/projects/YourPluginName/java/src/main/resources
touch ~/SNJ/projects/YourPluginName/java/src/main/java/com/yourpackage/Main.java
touch ~/SNJ/projects/YourPluginName/java/src/main/resources/plugin.yml
touch ~/SNJ/projects/YourPluginName/java/build.gradle.kts
```
9. now we need to write ``build.gradle.kts``
- You need to change names according to your files 
- Example build.gradle.kts:

```kts
plugins {
    java
}

group = "com.yourpackage"
version = "1.0.0"

repositories {
    mavenLocal()
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    compileOnly("io.papermc.paper:paper-api:1.21.4-R0.1-SNAPSHOT")
    implementation("org.swift.swiftkit:swiftkit-core:1.0-SNAPSHOT")
    implementation("org.swift.swiftkit:swiftkit-ffm:1.0-SNAPSHOT")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
}

sourceSets {
    main {
        java {
            srcDirs("src/main/java", "generated/java")
        }
        resources {
            srcDirs("src/main/resources")
        }
    }
}

tasks.withType<ProcessResources> {
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}

> note: absolute paths are temporary. SNJ Tools will automate this in the future.
> for release build, change `debug` to `release` in all paths
tasks.jar {
    archiveFileName.set("YourPluginName.jar")
    from("/Users/YourUser/SNJ/libs/swift-java/.build/arm64-apple-macosx/debug/libSwiftRuntimeFunctions.dylib")
    from("/Users/YourUser/SNJ/projects/YourPluginName/.build/arm64-apple-macosx/debug/libYourPluginName.dylib")
    from("/Users/YourUser/SNJ/projects/YourPluginName/.build/arm64-apple-macosx/debug/libSwiftJava.dylib")
    from(configurations.runtimeClasspath.get().map { if (it.isDirectory) it else zipTree(it) })
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```
10. let's write Main.java and simple plugin.yml
- open plugin.yml using xcode or ``open ~/SNJ/projects/YourPluginName/java/src/main/resources/plugin.yml`` 
- use this example to make your plugin.yml

```yml
name: YourPluginName
version: '1'
main: com.yourpackage.Main
api-version: '1.21'
description: YourPluginName is your first SNJ plugin!
author: YourName

commands:
  YourPluginName:
    description: Main YourPluginName command
    usage: /<command> [args]
    aliases: [YPN]
```
- now Main.java ``open ~/SNJ/projects/YourPluginName/java/src/main/java/com/yourpackage/Main.java``

```Java
package com.yourpackage;

import org.bukkit.plugin.java.JavaPlugin;
import java.io.File;

public class Main extends JavaPlugin {
    
    @Override
    public void onEnable() {
        getLogger().info("=================================");
        getLogger().info("  YourPluginName" );
        getLogger().info("  enabling...");
        getLogger().info("=================================");
        // swiftlibs/ folder is created automatically by build.sh
        // it should be located at: server/plugins/swiftlibs/
        // currently it is the only method but we will improve this later  
        File swiftLibsDir = new File(getDataFolder().getParent(), "swiftlibs");
        System.load(new File(swiftLibsDir, "libYourPluginName.dylib").getAbsolutePath());
        
    }
        
    @Override
    public void onDisable() {
        getLogger().info("=================================");
        getLogger().info("  YourPluginName disabling...");
        getLogger().info("=================================");
    }
}
```

- some words about ``libYourPluginName.dylib`` - this is your plugin's swift library, plugin can't run without it. Don't worry, it will be generated automaticly using build.sh which we will make in next steps!

---

> now we need to generate Swift wraps for java classes
>
> we need to decide which classes will we need..
>
> let's create simple plugin which will show greetings for player and server stats!
>
> for our requirements, we need java.lang.Runtime to see server info, but how do we get it?

11. let's create our "Swift-Java bridge builder"
- first, we will create directory where our Swift wraps and ``swift-java.config`` will be stored
> we need swift-java.config to specify which classes we need

```zsh
mkdir ~/SNJ/projects/YourPluginName/Sources/PaperAPI
touch ~/SNJ/projects/YourPluginName/Sources/PaperAPI/swift-java.config
```

- open swift-java.config via Xcode or 
``open ~/SNJ/projects/YourPluginName/Sources/PaperAPI/swift-java.config``
- and paste this inside

```json
{
  "classes": {
    "java.lang.Runtime": "Runtime"
  }
}
```

- also we need to create wrap-java.sh
>wrap-java.sh reads swift-java.config and generates Swift wrappers for specified Java classes. These wrappers are placed in Sources/PaperAPI/ and allow you to use Java classes directly in Swift code using JavaClass<ClassName> syntax.

**IMPORTANT**
- You need to have local paper server, that's where ``wrap-java.sh`` will find Bukkit classes, so let's create one!
```zsh
mkdir ~/SNJ/server/YourServerName
```

- download paper.jar from https://papermc.io/downloads/paper and move it in SNJ/server/YourServerName folder
- let's generate simple launch script
```zsh
echo "java -Xms4G -Xmx4G -jar YourPaperVersion.jar" >> start.sh
chmod +x start.sh
```

- launch start.sh using this command:
```zsh
cd ~/SNJ/server/YourServerName
./start.sh
```

- after first start, you need to accept EULA and restart server
- go to eula.txt and change eula=true
- start your server again, let it fully load, and then you can stop it.
```zsh
cd ~/SNJ/server/YourServerName
./start.sh
```

- after all this, its time to create ``wrap-java.sh``
```zsh
cd ~/SNJ/projects/YourPluginName
touch wrap-java.sh
chmod +x wrap-java.sh
```

- open it via Xcode and use this template to create your script:

```sh
#!/bin/bash
LIBS=~/SNJ/server/YourServerName/libraries

CP_ARGS=$(find $LIBS -name "*.jar" | xargs -I {} echo "--cp {}" | tr '\n' ' ')
CP_ARGS="$CP_ARGS --cp /Users/YourUser/SNJ/projects/YourPluginName/java/build/libs/YourPluginName.jar"

swift run --package-path ~/SNJ/libs/swift-java swift-java wrap-java \
  --swift-module PaperAPI \
  $CP_ARGS \
  --output-directory ~/SNJ/projects/YourPluginName/Sources/PaperAPI
```


- now let's run our script 
```zsh
cd ~/SNJ/projects/YourPluginName
./wrap-java.sh
```

- new .swift files will be created in Sources/PaperAPI folder, in our case it's Runtime.swift

**12. now here is where Swift finally kick's in!**
- let's open our .swift file using Xcode or this command:
 
 ```zsh
 open ~/SNJ/projects/YourPluginName/Sources/YourPluginName/YourPluginName.swift
 ```
 
 - now it's time to actually write some swift logic!
 - we will create 2 examples, with and without ``JavaClass``
 
```Swift
import SwiftJava
import PaperAPI

// basic example - no JavaClass needed
public func helloFromSwift(playerName: String) -> String {
    return "Hello from Swift, \(playerName)!"
}

// advanced example - using JavaClass to access Bukkit API
public func getServerStats(playerName: String) -> String {
    do {
        let runtimeClass = try JavaClass<Runtime>()

        guard let runtime = runtimeClass.getRuntime() else {
            return "Failed to get Runtime instance"
        }

        let totalMemory = runtime.totalMemory()
        let freeMemory = runtime.freeMemory()
        let usedMemory = totalMemory - freeMemory

        return """
        Server Stats for \(playerName):
        Memory: \(usedMemory / 1_048_576)MB / \(totalMemory / 1_048_576)MB
        """
    } catch {
        return "Failed to get stats: \(error)"
    }
}
```
 

13. now we can use jextract!
- let's create our first script..
```zsh
touch jextract.sh
chmod +x jextract.sh
open -a TextEdit jextract.sh
```
- TextEdit window will appear, paste this commands there, change project names and save.
```shell
#!/bin/bash
swift run --package-path ~/SNJ/libs/swift-java swift-java jextract \
  --mode jni \
  --input-swift ~/SNJ/projects/YourPluginName/Sources/YourPluginName \
  --swift-module YourPluginName \
  --output-swift ~/SNJ/projects/YourPluginName/Sources/YourPluginName/generated \
  --output-java ~/SNJ/projects/YourPluginName/java/generated/java \
  --java-package com.snj
```
- launch this script using ./jextract.sh

```zsh
...
[swift-java] Imported Swift module 'YourPluginName': done.
```

- now, bridge between Swift and Java is done and we can use our Swift functions in Java, but first, we need to build Swift..
```zsh
swift build
```

14. now let's actually use our Swift unctions in Java
- let's create new .java file and bind swift functions to commands
```zsh
touch ~/SNJ/projects/YourPluginName/java/src/main/java/com/yourpackage/SwiftCommand.java 
```

- open SwiftCommand.java in Xcode or any other IDE and write this simple function:

```Java
package com.yourpackage;

import com.snj.YourPluginName;
import org.bukkit.entity.Player;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;

public class SwiftCommand implements CommandExecutor {
    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!(sender instanceof Player)) return false;
        Player player = (Player) sender;
        
        if (args.length == 0) {
            player.sendMessage(YourPluginName.helloFromSwift(player.getName()));
            return true;
        }
        
        if (args[0].equals("args")) {
            player.sendMessage(YourPluginName.getServerStats(player.getName()));
            return true;
        }
        
        return false;
    }
}
``` 

- and register our command in main..

```Java
package com.yourpackage;

import org.bukkit.plugin.java.JavaPlugin;
import java.io.File;

public class Main extends JavaPlugin {

    @Override
    public void onEnable() {
        getLogger().info("=================================");
        getLogger().info("  YourPluginName" );
        getLogger().info("  enabling...");
        getLogger().info("=================================");
        // swiftlibs/ folder is created automatically by build.sh
        // it should be located at: server/plugins/swiftlibs/
        // currently it is the only method but we will improve this later
        File swiftLibsDir = new File(getDataFolder().getParent(), "swiftlibs");
        System.load(new File(swiftLibsDir, "libYourPluginName.dylib").getAbsolutePath());
        getCommand("YourPluginName").setExecutor(new SwiftCommand()); //<--------
    }

    @Override
    public void onDisable() {
        getLogger().info("=================================");
        getLogger().info("  YourPluginName disabling...");
        getLogger().info("=================================");
    }
}
```

**15. FINAL STEP,** - build.sh
- let's create simple script to build and deploy our plugin to server!
```zsh
cd ~/SNJ/projects/YourPluginName
touch build.sh
chmod +x build.sh
```

- build.sh will be created, use this example to make your build.sh


```shell
#!/bin/bash

# Change to "release" for production build
BUILD_CONFIG="debug"
# BUILD_CONFIG="release"

set -e

# Fix libswiftCompatibilitySpan if needed
if [ ! -f ~/SNJ/libs/swift-java/.build/arm64-apple-macosx/debug/libswiftCompatibilitySpan.dylib ]; then
    echo "Fixing libswiftCompatibilitySpan..."
    cp "/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift-6.2/macosx/libswiftCompatibilitySpan.dylib" "~/SNJ/libs/swift-java/.build/arm64-apple-macosx/debug/"
fi


echo "Wrapping Java classes..."
cd ~/SNJ/projects/YourPluginName
./wrap-java.sh

echo "Extracting Java bindings from Swift..."
./jextract.sh

echo "Building Swift..."
swift build -c $BUILD_CONFIG --package-path ~/SNJ/projects/YourPluginName

if [ ! -f ~/SNJ/libs/swift-java/.build/arm64-apple-macosx/debug/libSwiftRuntimeFunctions.dylib ]; then
    echo "Building SwiftRuntimeFunctions..."
    swift build --package-path ~/SNJ/libs/swift-java --product SwiftRuntimeFunctions
fi

echo "Copying Swift libraries..."
mkdir -p ~/SNJ/server/YourServerName/plugins/swiftlibs
mkdir -p ~/SNJ/projects/YourPluginName/dist/swiftlibs
cp ~/SNJ/projects/YourPluginName/.build/arm64-apple-macosx/$BUILD_CONFIG/libYourPluginName.dylib ~/SNJ/projects/YourPluginName/dist/swiftlibs/
cp ~/SNJ/projects/YourPluginName/.build/arm64-apple-macosx/$BUILD_CONFIG/libSwiftJava.dylib ~/SNJ/projects/YourPluginName/dist/swiftlibs/
cp ~/SNJ/libs/swift-java/.build/arm64-apple-macosx/debug/libSwiftRuntimeFunctions.dylib ~/SNJ/projects/YourPluginName/dist/swiftlibs/



echo "Building Java jar with updated bindings..."
cd ~/SNJ/projects/YourPluginName/java
gradle build
cp ~/SNJ/projects/YourPluginName/java/build/libs/YourPluginName.jar ~/SNJ/projects/YourPluginName/dist/

echo "Deploying..."
cp build/libs/YourPluginName.jar ~/SNJ/server/YourServerName/plugins/
cp ~/SNJ/projects/YourPluginName/dist/swiftlibs/*.dylib ~/SNJ/server/YourServerName/plugins/swiftlibs/

echo "Done!"
```

- let's launch it ``./build.sh``

```zsh
...
[swift-java] Imported Swift module 'YourPluginName': done.
Building Swift...
Building for debugging...
[16/16] Linking libYourPluginName.dylib
Build complete! (1.29s)
Copying Swift libraries...
Building Java jar with updated bindings...

BUILD SUCCESSFUL in 999ms

Deploying...
Done!
```

- **CONGRATULATIONS!** you just created your firts minecraft using Swift and SNJ method
now let's test it!!

```zsh
cd ~/SNJ/server/YourServerName
./start.sh
```

let's read server console to see whether our plugin started or not
```bash
...
[YourPluginName] Enabling YourPluginName v1
[YourPluginName] =================================
[YourPluginName]   YourPluginName
[YourPluginName]   enabling...
[YourPluginName] =================================
```

- let's join server and test functionality
- /yourpluginname 
> Hello from Swift, (yourName)!

- /yourpluginname args   
> Server Stats for (yourName):
> Memory: xxxMB / xxxxMB

- if everything is working, then congrats, you actually created fully functional minecraft plugin working on Swift.



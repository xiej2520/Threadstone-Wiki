This page is about how you read Minecraft's source code.

The preferred way to read them is using the [Ornithe Project](https://ornithemc.net/).
In the past, people used [MCP](http://www.modcoderpack.com/), in combination with the [carpet mod 1.12](https://github.com/gnembon/carpetmod112/) project.

# How to use Ornithe

Use the [Ornithe Mod Template](https://github.com/OrnitheMC/ornithe-mod-template) to create a template mod.
Follow the instructions in the repository's README and change the Minecraft version to the one you want.

Ensure you have Java JDK and Gradle installed. Running
```bash
./gradlew --refresh-dependencies
./gradlew genSources`
```
or the equivalent in your IDE, will decompile the Minecraft code.
Your IDE may need to have its cached decompiled code invalidated with a restart if you changed Minecraft versions.

IDEs with Java language server support should now be able to "go to definition" for Minecraft code within the mod, such as

- Visual Studio Code with a [Java Language support extension](https://marketplace.visualstudio.com/items?itemName=redhat.java)
- IntelliJ IDEA with the [Minecraft Development extension](https://plugins.jetbrains.com/plugin/8327-minecraft-development).

See the [Fabric guide](https://wiki.fabricmc.net/tutorial:reading_mc_code) for reading Minecraft source for more details.

## Feather

The [Feather](https://github.com/OrnitheMC/feather) mappings are a community-made set of mappings for versions before Mojmap (1.15 snapshots and below).
It can be used to generate a deobfuscated Minecraft jar, or directly decompile the code.

Clone the repository and follow the README.

For example, `python feather.py decompileWithCfr 1.12.2` decompiles Minecraft source code into a folder `1.12.2-decompiledSrc`.

## GitCraft

The [GitCraft](https://github.com/OrnitheMC/gitcraft) project can be used to generate a git repository of decompiled Minecraft.
It supports several mappings, including feather.

Clone the repository and follow the README.

In particular,
```bash
./gradlew run --args="--mappings=feather --only-version=1.12.2"
```
will decompile `1.12.2` with feather mappings.

Tuinity ![Java CI](https://github.com/Spottedleaf/Tuinity/workflows/Java%20CI/badge.svg)
==

Fork of [Paper](https://github.com/PaperMC/Paper) aimed at improving server performance at high playercounts.

## Contact
[IRC](http://irc.spi.gt/iris/?channels=tuinity) | [Discord](https://discord.gg/CgDPu27)

## How To (Server Admins)
Tuinity uses the same paperclip jar system that Paper uses.

You can download the latest build of Tuinity by going [here](https://ci.codemc.io/job/Spottedleaf/job/Tuinity/).

You can also [build it yourself](https://github.com/Spottedleaf/Tuinity#building)

## How To (Plugin developers)
In order to use Tuinity as a dependency you must [build it yourself](https://github.com/Spottedleaf/Tuinity#building).
Each time you want to update your dependency you must re-build tuinity.

Tuinity-API maven dependency:
```xml
<dependency>
    <groupId>com.tuinity</groupId>
    <artifactId>tuinity-api</artifactId>
    <version>1.15.2-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
 </dependency>
 ```

Tuinity-Server maven dependency:
```xml
<dependency>
    <groupId>com.tuinity</groupId>
    <artifactId>tuinity</artifactId>
    <version>1.15.2-R0.1-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
```

There is no repository required since the artifacts should be locally installed
via building tuinity.

## Building

Requirements:
- You need `git` installed, with a configured user name and email. 
   On windows you need to run from git bash.
- You need `maven` installed
- You need `jdk` 8+ installed to compile (and `jre` 8+ to run)
- Anything else that `paper` requires to build

If all you want is a paperclip server jar, just run `./tuinity jar`

Otherwise, to setup the `Tuinity-API` and `Tuinity-Server` repo, just run the following command
in your project root `./tuinity patch` additionally, after you run `./tuinity patch` you can run `./tuinity build` to build the 
respective api and server jars.

`./tuinity patch` should initialize the repo such that you can now start modifying and creating
patches. The folder `Tuinity-API` is the api repo and the `Tuinity-Server` folder
is the server repo and will contain the source files you will modify.

#### Creating a patch
Patches are effectively just commits in either `Tuinity-API` or `Tuinity-Server`.
To create one, just add a commit to either repo and run `./tuinity rb`, and a
patch will be placed in the patches folder. Modifying commits will also modify its
corresponding patch file.

## License
The PATCHES-LICENSE file describes the license for api & server patches,
found in `./patches` and its subdirectories except when noted otherwise.

Everything else is licensed under the MIT license, except when note otherwise.
See https://github.com/starlis/empirecraft and https://github.com/electronicboy/byof
for the license of material used/modified by this project.

### Note

The fork is based off of aikar's EMC framework found [here](https://github.com/starlis/empirecraft)

### Tuinity環境
```
# import Foo
import IWorldInventory
import Clearable
import IHopper
import Material
import IInventoryHolder
import MaterialMapColor
import Blocks
```

### Some script things
```
./tuinity FASTjar fast api 此指令可以快速地build, 且會build Tuinity-API
./tuinity FASTjar fast 快速地build, 且會略過 build Tuinity-API
```

### Manual method (Edit patch)
In case you need something more complex or want more control, this step-by-step instruction does
exactly what the above slightly automated system does.

1. If you have changes you are working on type `git stash` to store them for later.
   - Later you can type `git stash pop` to get them back.
2. Type `git rebase -i upstream/upstream`
   - It should show something like [this](https://gist.github.com/zachbr/21e92993cb99f62ffd7905d7b02f3159).
3. Replace `pick` with `edit` for the commit/patch you want to modify, and "save" the changes.
   - Only do this for one commit at a time.
4. Make the changes you want to make to the patch.
5. Type `git add .` to add your changes.
6. Type `git commit --amend` to commit.
   - **MAKE SURE TO ADD `--amend`** or else a new patch will be created.
   - You can also modify the commit message here.
7. Type `git rebase --continue` to finish rebasing.
8. Type `./paper rebuild` in the main directory.
   - This will modify the appropriate patches based on your commits.
9. PR your modifications back to this project.

### Code分析
```
#WorldServer.java
1. private final TickListServer<FluidType> nextTickListFluid; 
^ List
2. this.nextTickListFluid = new com.destroystokyo.paper.server.ticklist.PaperTickList<>(this, (fluidtype) -> { // Paper - optimise TickListServer
            return fluidtype == null || fluidtype == FluidTypes.EMPTY;
        }, IRegistry.FLUID::getKey, IRegistry.FLUID::get, this::a, "Fluids"); // Paper - Timings
^ 新的PaperTickList<T>物件，傳入判斷fluid是否要加入tickList的Predicate和如何tick的Consumer
，PaperTickList<T>本身有tick()函式，PaperTickList<T> extends TickListServer<T>
3. this.nextTickListFluid.b();
^ 此行出現在WorldServer#doTick裡，已經有分離Fluid和一般Block的Tick了。b()只是拿來呼叫PaperTickList#tick()的Overload
4. ((com.destroystokyo.paper.server.ticklist.PaperTickList)this.nextTickListFluid).onChunkSetTicking(chunkX, chunkZ);
^ Paper看來已經重寫過ticklist server到PaperTickList裡了。
注意：onChunkSetTicking內有async catcher,但是經過搜尋，似乎FluidTypeFlowing裡面並沒有呼叫

#TickListServer<T>.java (Watch with caution, We should look at PaperTickList First, 
then TickListServer, PaperTickList is the rewrite version of TickListServer)
1. public void schedule(BlockPosition blockposition, T t0, int i, TickListPriority ticklistpriority) {
^ 裡面包含了Fluid的Predicate的.test()
2. private void a(NextTickListEntry<T> nextticklistentry) {
        if (!this.nextTickListHash.contains(nextticklistentry)) {
            this.nextTickListHash.add(nextticklistentry);
            this.nextTickList.add(nextticklistentry);
        }
    }
^ (1.)裡呼叫的函式，TickListServer本身是泛型Class，我們處理的T==FluidType

#FluidTypeFlowing.java
以下是tick的順序
1.
@Override
public void a(World world, BlockPosition blockposition, Fluid fluid) {
2.
protected void a(GeneratorAccess generatoraccess, BlockPosition blockposition, Fluid fluid) {
^ 裡頭包含了spread DOWN和spread to sides
```

### Code寫法
Comment掉WorldServer#this.nextTickListFluid.b();
改為一個counter，給AsyncFluid.java(自定義Class)用。
AsyncFluid內貼上PaperTickList\<T\>的Code
處理async catcher(把Event改成async event)
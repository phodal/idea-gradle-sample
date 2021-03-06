# Android Studio 构建 APK 分析

IDEA 是我日常使用最多的开发工具之一，大概仅次于 Firefox （Google 和 StackOverflow 是生产力）和 iTerm 2。因为研究编辑器和 IDE 的关系，我研究了一段时间的 IDEA 插件的相关机制。其中作为研究的主要对象是 Android Studio 对于 Android APK 的构建部分。

总体的流程来说，还是比较简单的，所以太长不读版（可以直接看代码）：

1. IDEA 逻辑部分：`plugin.xml` -> `BuildApkAction`
2. 构建 Gradle 模块部分：-> OutputBuildActionUtil -> OutputBuildAction -> BulidPath -> ProjectPaths
3. 准备 tasks 和 listener：assemble -> createRequest for Task Background -> createTask Listener for UI
4. 执行 tasks： GradleTasksExecutorImpl -> run -> invokeGradleTasks -> GradleExecutionHelper.execute

嗯，就是这么简单。在这里我采用的是和 Android Studio 插件类似的目录结构和路径，以方便于后期扩展和映射。

## IDEA 入口

就入口来说，Jetbrains 对于 IDEA 的配置还是蛮简洁的。只需要配置好对应的 `android-plugin.xml` 配置，然后编写对应的类实现即可：

```xml
<action id="Android.BuildApk" class="com.android.tools.idea.gradle.actions.BuildApkAction">
    <add-to-group group-id="BuildMenu" relative-to-action="Android.GenerateSignedApk" anchor="before"/>
</action>
```

接着，使用万能的 `Alt` + `Enter`，就能快速生成对应的类，并实现对应的方法。上述的配置中，还在菜单栏中添加了对应的构建命令。这样一来，对于用户来说，他们只需要点击一下，就可以执行对应类的 `actionPerformed` 方法。

随后，我们就可以从父类中获取 `project` 相关的信息了：

```java
class BuildApkAction : AnAction() {
    override fun actionPerformed(e: AnActionEvent) {
        val project = e.project!!
        val projectPath = project.basePath!!

        val moduleManager = ModuleManager.getInstance(project)
        val modules = moduleManager.modules
        val buildAction = OutputBuildActionUtil.create(modules);

        val invoker = GradleBuildInvoker.getInstance(project)
        invoker.assemble(projectPath, buildAction);
    }
}
```

嗯，从主逻辑上来说，简化后的逻辑就是这么简单。

## 构建 Gradle 模块

就这个过程来说，稍微不太一样：

1. 注册 `FacetType`，即 AndroidGradleFacetType
2. 启动时，生成 `Facet` 配置
3. 构建时，读取对应类似的 `Facet` 配置

为了触发对应的类型构建，还需要由对应的插件来绑定对应的 type，如 Android 里的：`apply plugin: 'com.android.application'`。

随后，我们就可以调用 `OutputBuildActionUtil.create`，来创建包含生成所有模块的路径的 `BuildAction`：

```kotlin
fun create(modules: Array<Module>): OutputBuildAction? {
    val moduleGradlePaths = getModuleGradlePaths(modules)
    return OutputBuildAction(moduleGradlePaths)
}

private fun getModuleGradlePaths(modules: Array<Module>): Set<String> {
    val gradlePaths = mutableSetOf<String>()
    modules.mapNotNullTo(gradlePaths) {
        val facet = GradleFacet.getInstance(it)
        facet?.configuration?.GRADLE_PROJECT_PATH
    }
    return gradlePaths
}
```

如上面的代码所示：在 `getModuleGradlePaths` 方法里会遍历 `GradleFacet` 中的模块信息，并取出 `GRADLE_PROJECT_PATH`

## 准备任务和监听器

接着，我们就可以执行 `GradleBuildInvoker` 里的 `assemble` 方法，这里的代码就相对简单一些，主要是会准备一个 IDEA 相关的 `listener`：

```kotlin
private fun executeTaskRequest(request: Request, buildAction: BuildAction<*>?) {
    val jvmArguments: List<String> = ArrayList()

    val buildTaskListener = createBuildTaskListener(request, "Build")
    request
            .setJvmArguments(jvmArguments)
            .setCommandLineArguments()
            .setBuildAction(buildAction)
            .setTaskListener(buildTaskListener);

    executeTasks(request)
}

private fun executeTasks(request: Request) {
    val executor: GradleTasksExecutor = GradleTasksExecutorImpl(request)
    executor.queue()
}
```

即上述代码中的 `createBuildTaskListener`，它实现了一个 IDEA 的 `ExternalSystemEventDispatcher`，用于辅助系统的通知显示。如 IDEA 中的 `Build` 的 ToolWindow，用于接收、显示 Gradle 的构建日志：

```kotlin
...
override fun onTaskOutput(id: ExternalSystemTaskId, text: String, stdOut: Boolean) {
    myBuildEventDispatcher.setStdOut(stdOut)
    myBuildEventDispatcher.append(text)
}
...
```

于是，当我们真正执行构建任务的时候，就会在 `Build` 工具窗口中显示如下的信息：

```bash
Gradle Daemon started in 892 ms
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:compileDebugAidl NO-SOURCE
...
```

## 执行任务

随后，真正开始执行 `GradleTasksExecutorImpl`，它继承自 `Backgroundable`，顾名思义就是在后台执行。执行的步骤主要是：

1. 构建 Gradle 的 `LongRunningOperation`
2. 为 Gradle 设置 JavaHome
3. 添加构建任务
4. 执行 operation

主要的准备的执行代码如下所示：

```kotlin
val operation: LongRunningOperation; = connection.action(buildAction)
val javaHome: String = SystemProperties.getJavaHome();
            operation.setJavaHome(File(javaHome))

(operation as BuildActionExecuter<*>).forTasks(*ArrayUtil.toStringArray(myRequest.getGradleTasks()))
connection.use {
    operation.run()
}
```

随后，我们将调用 Gradle 及其插件中定义的 plugins 中的任务。

## 其它 - 笔记

Gradal Build Logs:

```bash
Executing tasks: [assemble] in project /Users/fdhuang/IntelliJIDEAProjects/MyApplication

Starting Gradle Daemon...
Gradle Daemon started in 892 ms
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:compileDebugAidl NO-SOURCE
> Task :app:generateDebugBuildConfig UP-TO-DATE
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :app:mainApkListPersistenceDebug UP-TO-DATE
> Task :app:generateDebugResValues UP-TO-DATE
> Task :app:generateDebugResources UP-TO-DATE
> Task :app:mergeDebugResources UP-TO-DATE
> Task :app:createDebugCompatibleScreenManifests UP-TO-DATE
> Task :app:extractDeepLinksDebug UP-TO-DATE
> Task :app:mergeDebugShaders UP-TO-DATE
> Task :app:processDebugManifest UP-TO-DATE
> Task :app:processDebugResources UP-TO-DATE
> Task :app:compileDebugKotlin UP-TO-DATE
> Task :app:javaPreCompileDebug UP-TO-DATE
> Task :app:compileDebugJavaWithJavac UP-TO-DATE
> Task :app:compileDebugSources UP-TO-DATE
> Task :app:compileDebugShaders UP-TO-DATE
> Task :app:generateDebugAssets UP-TO-DATE
> Task :app:mergeDebugAssets UP-TO-DATE
> Task :app:processDebugJavaRes NO-SOURCE
> Task :app:mergeDebugJavaResource UP-TO-DATE
> Task :app:dexBuilderDebug UP-TO-DATE
> Task :app:checkDebugDuplicateClasses UP-TO-DATE
> Task :app:mergeDebugJniLibFolders UP-TO-DATE
> Task :app:validateSigningDebug UP-TO-DATE
> Task :app:mergeDebugNativeLibs UP-TO-DATE
> Task :app:mergeExtDexDebug UP-TO-DATE
> Task :app:mergeDexDebug UP-TO-DATE
> Task :app:stripDebugDebugSymbols UP-TO-DATE
> Task :app:packageDebug UP-TO-DATE
> Task :app:assembleDebug UP-TO-DATE
> Task :app:preReleaseBuild UP-TO-DATE
> Task :app:compileReleaseAidl NO-SOURCE
> Task :app:compileReleaseRenderscript NO-SOURCE
> Task :app:generateReleaseBuildConfig UP-TO-DATE
> Task :app:mainApkListPersistenceRelease UP-TO-DATE
> Task :app:generateReleaseResValues UP-TO-DATE
> Task :app:generateReleaseResources UP-TO-DATE
> Task :app:mergeReleaseResources UP-TO-DATE
> Task :app:createReleaseCompatibleScreenManifests UP-TO-DATE
> Task :app:extractDeepLinksRelease UP-TO-DATE
> Task :app:processReleaseManifest UP-TO-DATE
> Task :app:processReleaseResources UP-TO-DATE
> Task :app:compileReleaseKotlin UP-TO-DATE
> Task :app:javaPreCompileRelease UP-TO-DATE
> Task :app:compileReleaseJavaWithJavac UP-TO-DATE
> Task :app:compileReleaseSources UP-TO-DATE
> Task :app:prepareLintJar UP-TO-DATE
> Task :app:checkReleaseDuplicateClasses UP-TO-DATE
> Task :app:lintVitalRelease
> Task :app:dexBuilderRelease UP-TO-DATE
> Task :app:mergeReleaseShaders UP-TO-DATE
> Task :app:compileReleaseShaders UP-TO-DATE
> Task :app:generateReleaseAssets UP-TO-DATE
> Task :app:mergeReleaseAssets UP-TO-DATE
> Task :app:mergeExtDexRelease UP-TO-DATE
> Task :app:mergeDexRelease UP-TO-DATE
> Task :app:processReleaseJavaRes NO-SOURCE
> Task :app:mergeReleaseJavaResource UP-TO-DATE
> Task :app:mergeReleaseJniLibFolders UP-TO-DATE
> Task :app:mergeReleaseNativeLibs UP-TO-DATE
> Task :app:stripReleaseDebugSymbols UP-TO-DATE
> Task :app:packageRelease UP-TO-DATE
> Task :app:assembleRelease
> Task :app:assemble

BUILD SUCCESSFUL in 21s
49 actionable tasks: 1 executed, 48 up-to-date
```

Android Studio logs:

```
Executing tasks: [:app:assembleDebug] in project /Users/fdhuang/works/fework/MyApplication

Starting Gradle Daemon...
Gradle Daemon started in 868 ms
> Task :app:preBuild UP-TO-DATE
> Task :app:preDebugBuild UP-TO-DATE
> Task :app:generateDebugBuildConfig UP-TO-DATE
> Task :app:compileDebugAidl NO-SOURCE
> Task :app:compileDebugRenderscript NO-SOURCE
> Task :app:generateDebugResValues UP-TO-DATE
> Task :app:generateDebugResources UP-TO-DATE
> Task :app:createDebugCompatibleScreenManifests UP-TO-DATE
> Task :app:extractDeepLinksDebug UP-TO-DATE
> Task :app:processDebugManifest UP-TO-DATE
> Task :app:mergeDebugResources UP-TO-DATE
> Task :app:processDebugResources UP-TO-DATE
> Task :app:compileDebugKotlin UP-TO-DATE
> Task :app:javaPreCompileDebug UP-TO-DATE
> Task :app:compileDebugJavaWithJavac UP-TO-DATE
> Task :app:compileDebugSources UP-TO-DATE
> Task :app:mergeDebugShaders UP-TO-DATE
> Task :app:compileDebugShaders NO-SOURCE
> Task :app:generateDebugAssets UP-TO-DATE
> Task :app:mergeDebugAssets UP-TO-DATE
> Task :app:processDebugJavaRes NO-SOURCE
> Task :app:dexBuilderDebug UP-TO-DATE
> Task :app:mergeDebugJavaResource UP-TO-DATE
> Task :app:checkDebugDuplicateClasses UP-TO-DATE
> Task :app:mergeExtDexDebug UP-TO-DATE
> Task :app:mergeDexDebug UP-TO-DATE
> Task :app:mergeDebugJniLibFolders UP-TO-DATE
> Task :app:mergeDebugNativeLibs UP-TO-DATE
> Task :app:stripDebugDebugSymbols NO-SOURCE
> Task :app:validateSigningDebug UP-TO-DATE
> Task :app:packageDebug UP-TO-DATE
> Task :app:assembleDebug UP-TO-DATE

BUILD SUCCESSFUL in 10s
21 actionable tasks: 21 up-to-date

Build Analyzer results available
```

# Dependency-Analysis-Plugin-Repro
Reproduces an issue I'm experiencing using the Dependency Analysis Plugin where kapt plugin is reported as unused.

# Issue
The [Dependency Analysis Plugin](https://github.com/autonomousapps/dependency-analysis-android-gradle-plugin) is raising an error that the `kapt` plugin is unused even when it's applied to an [_excluded_](https://github.com/ivanalvarado/Dependency-Analysis-Plugin-Repro/blob/main/build-configuration/project-health.gradle#L14) unused dependency in a gradle project.

For example, in this project our dependency analysis plugin is configured to not fail if `'com.google.dagger:hilt-android-compiler'` is unused for annotation processors:
[`build-configuration/project-health.gradle`](https://github.com/ivanalvarado/Dependency-Analysis-Plugin-Repro/blob/main/build-configuration/project-health.gradle)
```groovy
dependencyAnalysis {
    issues {
        onUnusedDependencies {
            severity('ignore')
        }
        onUsedTransitiveDependencies {
            severity('ignore')
        }
        onIncorrectConfiguration {
            severity('ignore')
        }
        onUnusedAnnotationProcessors {
            severity('fail')
            exclude('com.google.dagger:hilt-android-compiler')
        }
        onRedundantPlugins {
            severity('fail')
        }

        ignoreKtx(true)
    }
}
```

In the [`mylibrary/build.gradle`](https://github.com/ivanalvarado/Dependency-Analysis-Plugin-Repro/blob/main/mylibrary/build.gradle) we have `'com.google.dagger:hilt-android-compiler:2.43.2'` configured with `kapt`:
```groovy
apply plugin: "com.android.library"
apply plugin: "kotlin-android"
apply plugin: "kotlin-kapt"

...

dependencies {
    kapt 'com.google.dagger:hilt-android-compiler:2.43.2'
    ...
    implementation 'com.google.dagger:hilt-android:2.43.2'
}
```

When we run dependency analysis plugin by doing `./gradlew :mylibrary:projectHealth`, the plugin correctly excludes `'com.google.dagger:hilt-android-compiler'` from being reported as an unused annotation processor, but raises that the `kapt` plugin was applied but no annotation processors were used:
```
> Task :mylibrary:projectHealth FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':mylibrary:projectHealth'.
> Unused plugins that can be removed:
    kotlin-kapt: this project has the kotlin-kapt (org.jetbrains.kotlin.kapt) plugin applied, but there are no used annotation processors.
```

According to the [Hilt Android documentation](https://developer.android.com/training/dependency-injection/hilt-android#setup), we need to configure the Hilt compiler with `kapt`:
```groovy
dependencies {
    implementation "com.google.dagger:hilt-android:2.38.1"
    kapt "com.google.dagger:hilt-compiler:2.38.1"
}
```

## Assumption
We assume that the plugin is detecting that because `'com.google.dagger:hilt-android-compiler'` is unused and configured with `kapt`, then it effectively detects the `kapt` plugin as unused as well and thus raises this as an error.

## Expected Behavior
**We expect the plugin to detect that `'com.google.dagger:hilt-android-compiler'` is excluded and because it's using the `kapt` plugin, the plugin shouldn't be reported as unused.**

## Workaround
We are able to add a workaround for now by excluding `kotlin-kapt` as a redundant plugin in the dependency analysis configuration. However, we feel this is a bit aggressive:
```groovy
onRedundantPlugins {
    severity('fail')
    exclude('kotlin-kapt')
}
```

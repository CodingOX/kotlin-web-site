---
type: doc
layout: reference
category: "Tools"
title: "Kotlin 与 OSGi"
---

# Kotlin 与 OSGi

要使用 Kotlin 的 OSGi 支持功能, 你需要使用 `kotlin-osgi-bundle`, 而不是通常的 Kotlin 库文件.
此外还建议你删除 `kotlin-runtime`, `kotlin-stdlib` 和 `kotlin-reflect` 依赖, 因为 `kotlin-osgi-bundle` 已经包含了这些库的内容. 此外还需要注意不要引用外部的 Kotlin 库文件.
大多数通常的 Kotlin 库依赖都不能用于 OSGi 环境, 因此你不应该使用它们, 要将它们从你的工程中删除.

## Maven

在 Maven 工程中引入 Kotlin OSGi bundle:

<div class="sample" markdown="1" mode="xml" auto-indent="false" theme="idea" data-highlight-only>

```xml
<dependencies>
    <dependency>
        <groupId>org.jetbrains.kotlin</groupId>
        <artifactId>kotlin-osgi-bundle</artifactId>
        <version>${kotlin.version}</version>
    </dependency>
</dependencies>
```

</div>

从外部库中删除 Kotlin 的标准库(注意, exclusion 设置中星号只在 Maven 3 中有效):

<div class="sample" markdown="1" mode="xml" auto-indent="false" theme="idea" data-highlight-only>

```xml
<dependency>
    <groupId>some.group.id</groupId>
    <artifactId>some.library</artifactId>
    <version>some.library.version</version>

    <exclusions>
        <exclusion>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>*</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

</div>

## Gradle

在 Gradle 工程中引入 `kotlin-osgi-bundle`:

<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
compile "org.jetbrains.kotlin:kotlin-osgi-bundle:$kotlinVersion"
```

</div>

通过传递依赖, 你可能会间接依赖到一些默认的 Kotlin 库, 你可以使用以下方法删除这些库:

<div class="sample" markdown="1" mode="groovy" theme="idea">

```groovy
dependencies {
 compile (
   [group: 'some.group.id', name: 'some.library', version: 'someversion'],
   .....) {
  exclude group: 'org.jetbrains.kotlin'
}
```

</div>

## FAQ

#### 为什么不直接向所有的 Kotlin 库添加需要的 manifest 设值呢?

虽然这是提供 OSGi 支持时最优先的方法, 但很不幸, 目前我们无法做到这一点, 原因是所谓的 ["包分裂(package split)" 问题](http://wiki.osgi.org/wiki/Split_Packages), 这个问题很难解决, 所以目前我们不打算进行这样巨大的变更. 另外还有一种 `Require-Bundle` 功能, 但也不是最好的选择, 而且并不推荐采用这种方案.
因此我们决定为 OSGi 创建一个独立的库文件.

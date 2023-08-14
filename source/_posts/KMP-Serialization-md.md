---
title: KMP中的序列化
date: 2023-01-02 23:00:00
tags:
- KMP
---
Kotlin在语言层级没有直接支持序列化与反序列化，但是Kotlin multiplatform提供了现成的库kotlinx.serialization支持相关功能。

### 配置

要使用这个库需要在项目根目录的build.gradle.kts文件中加入

```kotlin
classpath("org.jetbrains.kotlin:kotlin-serialization:1.6.10")
```

<img src="https://s2.loli.net/2023/01/02/bZRSaWLIAi6EhmG.png" width="400px" />

接着在shared文件夹中build.gradle.kts中的plugins添加

```kotlin
kotlin("plugin.serialization")
```
<img src="https://s2.loli.net/2023/01/02/8OzqHQp3tUBlP7f.png" width="400px" />

sync一下，库就会加入到项目中。

项目里会有4种build.gradle.kts文件：

* build.gradle.kts: 位于项目的根目录，用于配置Android app 和 shared module，包含依赖项下载的仓库。

* shared/build.gradle.kts： shared module的配置文件，包含插件、库、目标平台等。

* androidApp/build.gradle.kts： Android app的配置文件，定义了用于编译项目的参数、sdk版本、依赖和编译flags。

* desktopApp/build.gradle.kts:  桌面app的配置文件， 和Android对应的文件配置一致。

#### 支持的序列化

kotlinx.serialization支持以下序列化格式

* JSON: kotlinx-serialization-json

* Protocol buffers: kotlinx-serialization-protobuf

* CBOR: 一种二进制数据序列化格式  kotlinx-serialization-cbor

* Properties:  一种key-value文件，例如gradle.properties。kotlinx-serialization-properties

* HOCON: JSON的超集，有更好的可读性，多用于配置文件。kotlinx.serailization-hocon

除了kotilin-serialization-json，其他库都还只是在beta阶段，意味着api在后续的版本中大概率会大改。

在shared/build.gradle.kts中的commonMain里的dependencies中添加

```kotlin
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.3.2")
```
<img src="https://s2.loli.net/2023/01/02/ZyzUXBNI7JAlMcR.png" width="400px" />

再次sync一下，就ready-to-use了。

## 自定义serializer
假设有如下enum
```kotlin
enum class RECORDTYPE(val value: String) {
  RECORD("record")
  TASK("task"),
  AUTO("auto"),
  HISTORY("history")
}
```
kotlinx-serialization-json不能解析RECORD、TASK、 AUTO、 HISTORY等关键字，因为RECORDTYPE类型并不能被识别，所以就需要自定义serializer/deserializer。

首先要在自定义serializer文件中增加findByKey函数:
```kotlin
private fun findByKey(key: String, default: RECORDTYPE = RECORDTYPE.RECORD): RECORDTYPE {
  return RECORDTYPE.values().find { it.value == key } ?: default
}
```
findByKey函数主要起到mapping的作用。

```kotlin
/*serializer文件*/


@OptIn(ExperimentalSerializationApi::class)

// 将CustomSerializer与特定的class关联起来，此处为RECORDTYPE
@Serializer(forClass = RECORDTYPE::class)

// 拓展KSerializer类并定义要被序列化或反序列化对象的类型
object CustomSerializer : KSerializer<RECORDTYPE> {

// 定义参数读取的方式
  override val descriptor: SerialDescriptor =
    PrimitiveSerialDescriptor("RECORDTYPE", PrimitiveKind.STRING)

  override fun serialize(encoder: Encoder, value: RECORDTYPE) {
    encoder.encodeString(value.value)
  }

  override fun deserialize(decoder: Decoder): RECORDTYPE {
    return try {
      val key = decoder.decodeString()
      findByKey(key)
    } catch (e: IllegalArgumentException) {
      RECORDTYPE.RECORD
    }
  }
}
```

最后在RECORDTYPE的声明前加上
```kotlin
@Serializable(with = CustomSerializer::class)
```
即可
```kotlin
@Serializable(with = CustomSerializer::class)
enum class RECORDTYPE(val value: String) {
    ...
}
```

## Data to Model
假设一个接口返回的json结构是
```json
{
"recordtype": xx,
"title": xx,
"url": https://www.xx.com/xxx
}
```
那么项目中中对应的model为
```kotlin
data class Record {
    var recordType: RECORDTYPE
    var title: String
    var url: String
}
```
这时候需要在model声明前加上`@Serializable`，通过这个标记接口返回的jsonstring就会mapping转换成对应的model。

除此之外遇到undefinekey会有崩溃的可能，所以进行过滤处理。

```kotlin
private val json = Json { ignoreUnknownKeys = true }

val content: List<Record> by lazy {
  json.decodeFromString(RAW_JSONSTRING)
}
```



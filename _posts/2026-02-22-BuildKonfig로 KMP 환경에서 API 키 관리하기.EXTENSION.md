---
title: "BuildKonfig로 KMP 환경에서 API 키 관리하기"
date: 2026-02-22 18:00:00 +0900
categories: [Kotlin Multiplatform]
tags: [Kotlin Multiplatform, BuildConfig, BuildKonfig]
---
Kotlin Multiplatform으로 개발하고 있던 중 당연하게 될 것 같던 기능이 동작하지 않아,  
이를 해결하는 간단한 방법을 공유하고자 글을 쓴다.    

KMP가 아니더라도 안드로이드 개발을 해본 사람이라면,  
서버와의 통신에 필요한 URL이라던가, API 키, 그리고 토큰 등을 private하게 관리하고자 할 것이다.  
보통은 local.properties 파일에 작성해 `BuildConfig`를 이용하여 가져오거나,  
구글에서 제공하는 [Secrets Gradle 플러그인](https://developers.google.com/maps/documentation/android-sdk/secrets-gradle-plugin?hl=ko&_gl=1*1s559tr*_up*MQ..*_ga*NjY1Nzg0MTA5LjE3NzE3NjE1MjY.*_ga_SM8HXJ53K2*czE3NzE3NjE1MjYkbzEkZzAkdDE3NzE3NjE1MjYkajYwJGwwJGgw*_ga_NRWSTWS78N*czE3NzE3NjE1MjYkbzEkZzAkdDE3NzE3NjE1MjYkajYwJGwwJGgw)을 사용할 것이다.   

KMP의 경우, 안드로이드 공식문서에서 아래와 같이 소개하고 있다.   
> `BuildConfig` 기능은 여러 빌드 변형(variant)을 사용하는 환경에서 가장 유용합니다.  
> 하지만 Kotlin Multiplatform 라이브러리 플러그인은 변형(variant)에 종속되지 않는 구조로,  
> **빌드 타입이나 product flavor를 지원하지 않기 때문에 해당 기능이 구현되어 있지 않습니다.**    
> 대신 모든 타깃에 대한 메타데이터를 생성하려면 `BuildKonfig` 플러그인이나 이와 유사한 커뮤니티 솔루션을 사용하는 것을 권장합니다.   
>  
> 참고: [안드로이드 공식문서](https://developer.android.com/kotlin/multiplatform/plugin#unsupported-features-and-workarounds)       
{: .prompt-info}

즉, **KMP에서는 `BuildConfig`를 지원하지 않고, 이를 대체하는 특정 플러그인을 사용하라** 명시되어 있다.  
대체 방법으로 `BuildKonfig`를 언급하고 있는데, `BuildKonfig`가 뭘까?  

# BuildKonfig
---
`BuildKonfig`는 서드파티 라이브러리이다.  
이 라이브러리를 사용하면, Android에서 환경을 설정하는 방식과 유사하게 작업을 수행할 수 있다.  

## 사용 방법
1. libs.version.toml  
    ```toml
    buildKonfig = "0.17.1"

    [plugins]
    buildKonfig = { id = "com.codingfeline.buildkonfig", version.ref = "buildKonfig" }
    ``` 
2. 프로젝트 루트 build.gradle.kts  
    ```kotlin
    plugins {
        // ...
        alias(libs.plugins.buildKonfig) apply false
    }
    ```
3. 사용할 모듈 내 build.gradle.kts  
    ```kotlin
    plugins {
        alias(libs.plugins.kotlinMultiplatform)
        alias(libs.plugins.androidKotlinMultiplatformLibrary)
        alias(libs.plugins.androidLint)
        alias(libs.plugins.buildKonfig)
    }

    val localProperties = Properties().apply {
        val file = rootProject.file("local.properties")
        if (file.exists()) {
            file.inputStream().use(::load)
        }
    }
    val apiKey = localProperties.getProperty("API_KEY")

    buildkonfig {
        packageName = "패키지 명"

        defaultConfigs {
            buildConfigField(STRING, "API_KEY", apiKey)
        }
    }
    ```
4. Gradle 동기화  
`generateBuildConfig`를 실행한다.  
![generateBuildConfig](/assets/img/20260222_1.png)  
동기화가 완료되면 `BuildKonfig` 클래스가 생성된다.   
(build.gradle.kts 내 `buildkonfig{}`에서 `objectName`를 통해 원하는 이름으로도 설정 가능하다.)   
    ```kotlin
    internal object BuildKonfig {

        public val API_KEY: String = "..."
    }

    ```

## targetConfigs
아래와 같이 `targetConfigs`를 통해, KMP 환경에 맞는 각 OS별 `BuildKonfig`를 생성할 수도 있다.  
```kotlin
// 사용 모듈 내 build.gradle.kts
buildkonfig {
    packageName = "패키지 명"

    // default config is required
    defaultConfigs {
        buildConfigField(STRING, "name", "value")
    }

    targetConfigs {
        // names in create should be the same as target names you specified
        create("android") {
            buildConfigField(STRING, "name2", "value2")
            buildConfigField(STRING, "nullableField", "NonNull-value", nullable = true)
        }

        create("iosArm64") {
            buildConfigField(STRING, "name", "valueForNative")
        }

        // ..
    }
}
```
결과적으로 아래와 같은 코드가 생성된다.  
```kotlin
// commonMain
internal expect object BuildKonfig {
    public val name: String
}
```
```kotlin
// androidMain
internal actual object BuildKonfig {
    public actual val name: String = "value"
    public val name2: String = "value2"
    public val nullableField: String? = "NonNull-value"
}
```
```kotlin
// iosArm64Main
internal actual object BuildKonfig {
    public actual val name: String = "valueForNative"
}
```

## productFlavor
Android Gradle Plugin(AGP)에서 지원하는 productFlavor 기능에 대해서도 일부 지원하고 있는데,  
전역적으로 적용되지 않기 때문에 gradle.properties에 직접 명시해야한다.  
```
# ROOT_DIR/gradle.properties
buildkonfig.flavor=dev
```
```kotlin
buildkonfig {
    // ...
    // flavor is passed as a first argument of defaultConfigs
    defaultConfigs("dev") {
        buildConfigField(STRING, "name", "devValue")
    }

    // ...
    // flavor is passed as a first argument of targetConfigs
    targetConfigs("dev") {
        create("ios") {
            buildConfigField(STRING, "name", "devValueIos")
        }
    }
}
```
더 자세한 사용 방법은 [BuildKonfig Github](https://github.com/yshrsmz/BuildKonfig)를 참고하면 된다.  

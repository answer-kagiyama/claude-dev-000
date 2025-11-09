# Gradle チートシート

## 基本コマンド

```bash
# Gradleバージョン確認
gradle -v
gradle --version

# タスク一覧
gradle tasks
gradle tasks --all

# ビルド
gradle build
gradle build --info        # 詳細ログ
gradle build --debug       # デバッグログ
gradle build --stacktrace  # スタックトレース表示

# クリーン
gradle clean

# テスト
gradle test
gradle test --tests TestClass
gradle test --tests TestClass.testMethod

# 実行
gradle run

# 依存関係確認
gradle dependencies
gradle dependencies --configuration compileClasspath
gradle dependencies --configuration runtimeClasspath

# プロジェクト情報
gradle projects
gradle properties

# キャッシュクリア
gradle clean build --no-build-cache
gradle --stop  # デーモン停止

# オフラインモード
gradle build --offline

# 並列ビルド
gradle build --parallel

# 継続実行（エラーがあっても続行）
gradle build --continue

# プロファイリング
gradle build --profile
```

## build.gradle (Groovy DSL)

### 基本構成

```groovy
// プラグイン
plugins {
    id 'java'
    id 'application'
    id 'org.springframework.boot' version '3.2.0'
}

// グループID、バージョン
group = 'com.example'
version = '1.0.0'
sourceCompatibility = '17'

// リポジトリ
repositories {
    mavenCentral()
    mavenLocal()
    maven {
        url 'https://repo.spring.io/milestone'
    }
}

// 依存関係
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

    runtimeOnly 'org.postgresql:postgresql'

    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.9.0'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

// テスト設定
test {
    useJUnitPlatform()
    testLogging {
        events "passed", "skipped", "failed"
    }
}

// アプリケーション設定
application {
    mainClass = 'com.example.Main'
}

// カスタムタスク
task hello {
    doLast {
        println 'Hello, Gradle!'
    }
}
```

### Java プロジェクト

```groovy
plugins {
    id 'java'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.google.guava:guava:32.1.3-jre'
    testImplementation 'junit:junit:4.13.2'
}

jar {
    manifest {
        attributes 'Main-Class': 'com.example.Main'
    }
    from {
        configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) }
    }
}
```

### Spring Boot プロジェクト

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    sourceCompatibility = '17'
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'org.postgresql:postgresql'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}

bootJar {
    archiveFileName = 'app.jar'
}
```

## build.gradle.kts (Kotlin DSL)

### 基本構成

```kotlin
plugins {
    java
    application
    id("org.springframework.boot") version "3.2.0"
}

group = "com.example"
version = "1.0.0"

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(17))
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")

    runtimeOnly("org.postgresql:postgresql")

    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.junit.jupiter:junit-jupiter:5.9.0")
}

tasks.test {
    useJUnitPlatform()
}

application {
    mainClass.set("com.example.Main")
}

tasks.register("hello") {
    doLast {
        println("Hello, Gradle!")
    }
}
```

### マルチプロジェクト

```kotlin
// settings.gradle.kts
rootProject.name = "my-project"
include("app", "lib")

// build.gradle.kts (root)
plugins {
    java
}

allprojects {
    group = "com.example"
    version = "1.0.0"

    repositories {
        mavenCentral()
    }
}

subprojects {
    apply(plugin = "java")

    dependencies {
        testImplementation("org.junit.jupiter:junit-jupiter:5.9.0")
    }

    tasks.test {
        useJUnitPlatform()
    }
}

// app/build.gradle.kts
dependencies {
    implementation(project(":lib"))
    implementation("org.springframework.boot:spring-boot-starter-web:3.2.0")
}

// lib/build.gradle.kts
dependencies {
    implementation("com.google.guava:guava:32.1.3-jre")
}
```

## settings.gradle

```groovy
rootProject.name = 'my-project'

// マルチプロジェクト
include 'app', 'lib', 'common'

// プラグイン管理
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```

## タスク定義

```groovy
// 基本タスク
task hello {
    doLast {
        println 'Hello!'
    }
}

// 依存関係
task taskB {
    dependsOn 'taskA'
    doLast {
        println 'Task B'
    }
}

// 型付きタスク
task copyFiles(type: Copy) {
    from 'src'
    into 'dest'
}

// カスタムタスククラス
class GreetingTask extends DefaultTask {
    @Input
    String greeting = 'hello'

    @TaskAction
    void greet() {
        println greeting
    }
}

task greet(type: GreetingTask) {
    greeting = 'Hello, World!'
}

// Exec タスク
task runScript(type: Exec) {
    commandLine 'bash', 'script.sh'
}

// Zip タスク
task packageApp(type: Zip) {
    from 'build/libs'
    archiveFileName = 'app.zip'
    destinationDirectory = file('dist')
}
```

## 依存関係管理

```groovy
dependencies {
    // コンパイル時
    implementation 'com.google.guava:guava:32.1.3-jre'

    // コンパイル時のみ（実行時不要）
    compileOnly 'org.projectlombok:lombok:1.18.30'

    // 実行時のみ
    runtimeOnly 'org.postgresql:postgresql:42.6.0'

    // アノテーションプロセッサ
    annotationProcessor 'org.projectlombok:lombok:1.18.30'

    // テスト
    testImplementation 'org.junit.jupiter:junit-jupiter:5.9.0'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

    // バージョン範囲
    implementation 'org.apache.commons:commons-lang3:3.+'

    // 除外
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }

    // ローカルJAR
    implementation files('libs/local.jar')
    implementation fileTree(dir: 'libs', include: '*.jar')

    // プロジェクト依存
    implementation project(':lib')
}

// 依存関係解決戦略
configurations.all {
    resolutionStrategy {
        // 特定バージョンを強制
        force 'com.google.guava:guava:32.1.3-jre'

        // 動的バージョンのキャッシュ時間
        cacheDynamicVersionsFor 10, 'minutes'

        // スナップショットのキャッシュ時間
        cacheChangingModulesFor 0, 'seconds'
    }
}
```

## プラグイン

```groovy
// プラグイン適用
plugins {
    id 'java'
    id 'application'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

// 旧形式
apply plugin: 'java'

// カスタムプラグイン
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.example:custom-plugin:1.0.0'
    }
}

apply plugin: 'com.example.custom'
```

## よく使うプラグイン

### Java Plugin

```groovy
plugins {
    id 'java'
}

sourceSets {
    main {
        java {
            srcDirs = ['src/main/java']
        }
        resources {
            srcDirs = ['src/main/resources']
        }
    }
    test {
        java {
            srcDirs = ['src/test/java']
        }
    }
}

compileJava {
    options.encoding = 'UTF-8'
    options.compilerArgs << '-parameters'
}
```

### Application Plugin

```groovy
plugins {
    id 'application'
}

application {
    mainClass = 'com.example.Main'
    applicationDefaultJvmArgs = ['-Xmx1g']
}

run {
    args 'arg1', 'arg2'
    systemProperty 'key', 'value'
}
```

### Shadow Plugin (Fat JAR)

```groovy
plugins {
    id 'com.github.johnrengelman.shadow' version '8.1.1'
}

shadowJar {
    archiveBaseName = 'app'
    archiveClassifier = ''
    archiveVersion = '1.0.0'

    manifest {
        attributes 'Main-Class': 'com.example.Main'
    }
}
```

## gradle.properties

```properties
# JVMオプション
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m

# 並列実行
org.gradle.parallel=true

# デーモン
org.gradle.daemon=true

# キャッシュ
org.gradle.caching=true

# ビルドスキャン
org.gradle.build-scan=true

# カスタムプロパティ
appVersion=1.0.0
springBootVersion=3.2.0
```

## Gradle Wrapper

```bash
# Wrapper生成
gradle wrapper

# Gradleバージョン指定
gradle wrapper --gradle-version 8.5

# 配布形式指定
gradle wrapper --distribution-type all

# Wrapper経由で実行
./gradlew build        # Linux/Mac
gradlew.bat build      # Windows

# Wrapperアップグレード
./gradlew wrapper --gradle-version 8.5
```

## カスタムタスク例

```groovy
// ファイルコピー
task copyResources(type: Copy) {
    from 'src/resources'
    into 'build/resources'
    include '**/*.properties'
    exclude '**/*.tmp'
}

// Zip作成
task packageDist(type: Zip) {
    from 'build/libs'
    into('config') {
        from 'config'
    }
    archiveFileName = 'app-dist.zip'
}

// 条件付き実行
task conditionalTask {
    onlyIf {
        project.hasProperty('runTask')
    }
    doLast {
        println 'Running conditional task'
    }
}

// 増分ビルド
task processFiles {
    inputs.dir 'src'
    outputs.dir 'build/processed'

    doLast {
        // 処理
    }
}
```

## マルチプロジェクト構成

```groovy
// settings.gradle
rootProject.name = 'my-app'
include 'app', 'lib', 'common'

// build.gradle (root)
subprojects {
    apply plugin: 'java'

    repositories {
        mavenCentral()
    }

    dependencies {
        testImplementation 'junit:junit:4.13.2'
    }
}

project(':app') {
    dependencies {
        implementation project(':lib')
        implementation project(':common')
    }
}

project(':lib') {
    dependencies {
        implementation project(':common')
    }
}
```

## ビルドキャッシュ

```groovy
// build.gradle
buildCache {
    local {
        enabled = true
        directory = file("$rootDir/.gradle/build-cache")
        removeUnusedEntriesAfterDays = 30
    }

    remote(HttpBuildCache) {
        url = 'https://example.com/cache/'
        push = true
    }
}
```

## テスト設定

```groovy
test {
    useJUnitPlatform()

    // 並列実行
    maxParallelForks = Runtime.runtime.availableProcessors()

    // JVMオプション
    jvmArgs '-Xmx1g'

    // システムプロパティ
    systemProperty 'property', 'value'

    // 環境変数
    environment 'ENV_VAR', 'value'

    // ログ設定
    testLogging {
        events "passed", "skipped", "failed"
        exceptionFormat "full"
        showStandardStreams = true
    }

    // フィルタ
    include '**/*Test.class'
    exclude '**/*IntegrationTest.class'
}

// テストレポート
tasks.withType(Test) {
    reports {
        html.enabled = true
        junitXml.enabled = true
    }
}
```

## Tips

```groovy
// 環境変数から値取得
def apiKey = System.getenv('API_KEY') ?: 'default-key'

// プロパティから値取得
def version = project.findProperty('version') ?: '1.0.0'

// 条件分岐
if (project.hasProperty('production')) {
    // 本番用設定
} else {
    // 開発用設定
}

// タスク実行前後の処理
gradle.taskGraph.beforeTask { Task task ->
    println "Executing: $task"
}

// ビルド失敗時の処理
gradle.buildFinished { buildResult ->
    if (buildResult.failure) {
        println "Build failed!"
    }
}
```

## よく使うコマンド組み合わせ

```bash
# クリーンビルド
./gradlew clean build

# テストスキップでビルド
./gradlew build -x test

# 特定タスクのみ実行
./gradlew :app:build

# 依存関係の更新チェック
./gradlew dependencyUpdates

# ビルドスキャン
./gradlew build --scan

# デバッグモードで実行
./gradlew build --debug > build.log 2>&1

# キャッシュを使わずにビルド
./gradlew clean build --no-build-cache

# プロファイリング
./gradlew build --profile --offline --rerun-tasks
```

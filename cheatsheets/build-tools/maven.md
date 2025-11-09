# Maven チートシート

## 基本コマンド

```bash
# Mavenバージョン確認
mvn -v
mvn --version

# クリーン
mvn clean

# コンパイル
mvn compile

# テスト
mvn test
mvn test -Dtest=TestClass
mvn test -Dtest=TestClass#testMethod

# パッケージ（JAR/WAR作成）
mvn package

# インストール（ローカルリポジトリへ）
mvn install

# デプロイ（リモートリポジトリへ）
mvn deploy

# サイト生成
mvn site

# 依存関係確認
mvn dependency:tree
mvn dependency:list
mvn dependency:analyze

# プロジェクト情報
mvn help:effective-pom
mvn help:describe -Dplugin=compiler

# スキップオプション
mvn install -DskipTests              # テスト実行スキップ
mvn install -Dmaven.test.skip=true  # テストコンパイルもスキップ

# オフラインモード
mvn install -o
mvn install --offline

# 詳細ログ
mvn install -X        # デバッグ
mvn install -e        # エラー詳細
mvn install -q        # 簡潔表示

# 並列ビルド
mvn install -T 4      # 4スレッド
mvn install -T 1C     # CPUコア数分

# プロファイル指定
mvn install -P production
mvn install -P dev,test
```

## pom.xml

### 基本構成

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- プロジェクト情報 -->
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>My Application</name>
    <description>My Application Description</description>
    <url>https://example.com</url>

    <!-- プロパティ -->
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring.boot.version>3.2.0</spring.boot.version>
    </properties>

    <!-- 依存関係 -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>${spring.boot.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.30</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <!-- ビルド設定 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Spring Boot プロジェクト

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- Spring Boot親POM -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## 依存関係管理

### スコープ

```xml
<dependencies>
    <!-- compile（デフォルト） -->
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>32.1.3-jre</version>
    </dependency>

    <!-- provided（コンパイル時のみ、実行時は外部から提供） -->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>4.0.1</version>
        <scope>provided</scope>
    </dependency>

    <!-- runtime（実行時のみ） -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.6.0</version>
        <scope>runtime</scope>
    </dependency>

    <!-- test（テスト時のみ） -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.9.0</version>
        <scope>test</scope>
    </dependency>

    <!-- system（システムパス指定） -->
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>custom-lib</artifactId>
        <version>1.0</version>
        <scope>system</scope>
        <systemPath>${project.basedir}/lib/custom-lib.jar</systemPath>
    </dependency>
</dependencies>
```

### 依存関係の除外

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 依存関係管理

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- バージョン省略可能 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

## プラグイン

### Compiler Plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.11.0</version>
    <configuration>
        <source>17</source>
        <target>17</target>
        <encoding>UTF-8</encoding>
        <compilerArgs>
            <arg>-parameters</arg>
        </compilerArgs>
    </configuration>
</plugin>
```

### Surefire Plugin（テスト）

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0-M9</version>
    <configuration>
        <includes>
            <include>**/*Test.java</include>
            <include>**/*Tests.java</include>
        </includes>
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
        <argLine>-Xmx1024m</argLine>
    </configuration>
</plugin>
```

### JAR Plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.3.0</version>
    <configuration>
        <archive>
            <manifest>
                <mainClass>com.example.Main</mainClass>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

### Assembly Plugin（Fat JAR）

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.6.0</version>
    <configuration>
        <archive>
            <manifest>
                <mainClass>com.example.Main</mainClass>
            </manifest>
        </archive>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Shade Plugin（Uber JAR）

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.example.Main</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Exec Plugin

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <mainClass>com.example.Main</mainClass>
        <arguments>
            <argument>arg1</argument>
            <argument>arg2</argument>
        </arguments>
        <systemProperties>
            <systemProperty>
                <key>key</key>
                <value>value</value>
            </systemProperty>
        </systemProperties>
    </configuration>
</plugin>
```

## プロファイル

```xml
<profiles>
    <!-- 開発環境 -->
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <env>development</env>
            <db.url>jdbc:postgresql://localhost:5432/devdb</db.url>
        </properties>
    </profile>

    <!-- 本番環境 -->
    <profile>
        <id>prod</id>
        <properties>
            <env>production</env>
            <db.url>jdbc:postgresql://prod-db:5432/proddb</db.url>
        </properties>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <debug>false</debug>
                        <optimize>true</optimize>
                    </configuration>
                </plugin>
            </plugins>
        </build>
    </profile>

    <!-- OS別 -->
    <profile>
        <id>windows</id>
        <activation>
            <os>
                <family>windows</family>
            </os>
        </activation>
        <properties>
            <script.extension>.bat</script.extension>
        </properties>
    </profile>
</profiles>
```

## マルチモジュール

```xml
<!-- 親pom.xml -->
<project>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>app</module>
        <module>lib</module>
        <module>common</module>
    </modules>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <junit.version>5.9.0</junit.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter</artifactId>
                <version>${junit.version}</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>

<!-- app/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>parent</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>app</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>lib</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>com.example</groupId>
            <artifactId>common</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</project>
```

## settings.xml

### ローカル設定（~/.m2/settings.xml）

```xml
<settings>
    <!-- ローカルリポジトリ -->
    <localRepository>/path/to/local/repo</localRepository>

    <!-- ミラー -->
    <mirrors>
        <mirror>
            <id>central-mirror</id>
            <url>https://repo.maven.apache.org/maven2</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>

    <!-- サーバー認証 -->
    <servers>
        <server>
            <id>nexus</id>
            <username>admin</username>
            <password>password</password>
        </server>
    </servers>

    <!-- プロファイル -->
    <profiles>
        <profile>
            <id>nexus</id>
            <repositories>
                <repository>
                    <id>nexus</id>
                    <url>https://nexus.example.com/repository/maven-public/</url>
                </repository>
            </repositories>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>nexus</activeProfile>
    </activeProfiles>
</settings>
```

## リポジトリ

```xml
<repositories>
    <repository>
        <id>central</id>
        <url>https://repo.maven.apache.org/maven2</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>

    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </repository>
</repositories>

<pluginRepositories>
    <pluginRepository>
        <id>central</id>
        <url>https://repo.maven.apache.org/maven2</url>
    </pluginRepository>
</pluginRepositories>

<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>https://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>https://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

## ライフサイクル

### Default ライフサイクル

```
validate → compile → test → package → verify → install → deploy
```

### Clean ライフサイクル

```
pre-clean → clean → post-clean
```

### Site ライフサイクル

```
pre-site → site → post-site → site-deploy
```

## よく使うゴール

```bash
# クリーン
mvn clean

# コンパイル
mvn compile              # メインコード
mvn test-compile         # テストコード

# テスト
mvn test                 # ユニットテスト
mvn integration-test     # 統合テスト
mvn verify               # テスト検証

# パッケージ
mvn package              # JAR/WAR作成

# インストール
mvn install              # ローカルリポジトリへ

# デプロイ
mvn deploy               # リモートリポジトリへ

# サイト
mvn site                 # プロジェクトサイト生成

# 依存関係
mvn dependency:tree      # ツリー表示
mvn dependency:list      # リスト表示
mvn dependency:analyze   # 未使用依存関係検出
mvn dependency:resolve   # 依存関係解決
mvn dependency:copy-dependencies  # 依存関係コピー

# プラグイン
mvn help:describe -Dplugin=compiler
mvn versions:display-dependency-updates
mvn versions:display-plugin-updates

# 実行
mvn exec:java -Dexec.mainClass=com.example.Main
mvn spring-boot:run
```

## Tips

```bash
# プロジェクト作成
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false

# Spring Bootプロジェクト作成
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=demo \
  -DarchetypeGroupId=org.springframework.boot \
  -DarchetypeArtifactId=spring-boot-archetype

# 依存関係の更新
mvn versions:use-latest-versions

# クリーンインストール（よく使う）
mvn clean install

# テストスキップでパッケージ
mvn package -DskipTests

# オフラインモード
mvn -o install

# デバッグモード
mvn -X compile

# 特定モジュールのみビルド
mvn install -pl app -am

# ビルド結果の強制更新
mvn clean install -U

# 並列ビルド
mvn -T 4 clean install
```

## プロジェクト構造

```
my-app/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/
│   │   │       └── Main.java
│   │   ├── resources/
│   │   │   └── application.properties
│   │   └── webapp/        # WARの場合
│   │       └── WEB-INF/
│   │           └── web.xml
│   └── test/
│       ├── java/
│       │   └── com/example/
│       │       └── MainTest.java
│       └── resources/
└── target/                # ビルド成果物
    ├── classes/
    ├── test-classes/
    └── my-app-1.0.0.jar
```

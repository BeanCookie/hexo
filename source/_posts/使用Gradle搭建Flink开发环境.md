---
title: 使用Gradle搭建Flink开发环境
date: 2021-11-19 13:51:12
categories:
- Flink
tags: 
- Flink
- Gradle
---

### Gradle配置文件
```groovy
// build.gradle

buildscript {
    repositories {
        maven {
            url 'https://maven.aliyun.com/nexus/content/groups/public/'
        }
        maven {
            url 'https://maven.aliyun.com/nexus/content/repositories/jcenter'
        }
        mavenCentral()
    }

    dependencies {
        classpath 'com.github.jengelman.gradle.plugins:shadow:2.0.4'
    }
}

plugins {
    id 'java'
    id 'application'
    id 'com.github.johnrengelman.shadow' version '7.0.0'
}

group 'com.zeaho'
version '1.0-SNAPSHOT'

repositories {
    maven {
        url 'https://maven.aliyun.com/nexus/content/groups/public/'
    }
    maven {
        url 'https://maven.aliyun.com/nexus/content/repositories/jcenter'
    }
    mavenCentral()
}

ext {
    javaVersion = '1.8'
    flinkVersion = '1.13.2'
    scalaBinaryVersion = '2.11'
    slf4jVersion = '1.7.32'
    log4jVersion = '2.14.1'
}

sourceCompatibility = javaVersion
targetCompatibility = javaVersion
tasks.withType(JavaCompile) {
    options.encoding = 'UTF-8'
}

applicationDefaultJvmArgs = ["-Dlog4j.configurationFile=log4j2.properties"]

configurations {
    flinkShadowJar // dependencies which go into the shadowJar

    // always exclude these (also from transitive dependencies) since they are provided by Flink
    flinkShadowJar.exclude group: 'org.apache.flink', module: 'force-shading'
    flinkShadowJar.exclude group: 'com.google.code.findbugs', module: 'jsr305'
    flinkShadowJar.exclude group: 'org.slf4j'
    flinkShadowJar.exclude group: 'org.apache.logging.log4j'
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.8.1'

    implementation "org.apache.flink:flink-streaming-java_${scalaBinaryVersion}:${flinkVersion}"

    // 本地调试
    implementation "org.apache.flink:flink-clients_${scalaBinaryVersion}:${flinkVersion}"

    compileOnly 'org.projectlombok:lombok:1.18.22'
    annotationProcessor 'org.projectlombok:lombok:1.18.22'

    implementation "io.krakens:java-grok:0.1.9"

    implementation "org.apache.logging.log4j:log4j-api:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-core:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-slf4j-impl:${log4jVersion}"
    implementation "org.slf4j:slf4j-log4j12:${slf4jVersion}"
}


test {
    useJUnitPlatform()
}

sourceSets {
    main.compileClasspath += configurations.flinkShadowJar
    main.runtimeClasspath += configurations.flinkShadowJar

    test.compileClasspath += configurations.flinkShadowJar
    test.runtimeClasspath += configurations.flinkShadowJar

    javadoc.classpath += configurations.flinkShadowJar
}

run.classpath = sourceSets.main.runtimeClasspath

jar {
    manifest {
        attributes 'Built-By': System.getProperty('user.name'), 'Build-Jdk': System.getProperty('java.version')
    }
}

shadowJar {
    configurations = [project.configurations.flinkShadowJar]
}
```
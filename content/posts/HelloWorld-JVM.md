---
title: HelloWorld | JVM
abbrlink: 9610
date: 2020-08-01 00:43:58
tags:
  - JVM
  - Kotlin
  - Java
  - C
---
# HelloWorld | JVM
因為是部落格上的第一篇，所以想來測試基本語法高亮
首先是Java
```java
// 測試Java註解
public static void main(string[] args){
    System.out.println("hello world");
}
```
接下來是Kotlin
```kotlin
fun main(){
    "hello world".println()
}

fun String.println(){
    println(this)
}
```
最後基本的C語言好了
```c
int main(){
    printf("hello world");
    return 0;
}
```
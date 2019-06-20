### android studio搭建本地仓库

   首先在工程的项目 gradle.properties 中定义一些属性
   ```
   #仓库地址
   LOCAL_REPO_URL=file:///E:/repo/
   # 包信息
   PROJ_GROUP=com.xxx.testLib 
   PROJ_ARTIFACTID=lib
   PROJ_VERSION=1.0.0
   ```
   也就是 `'com.xxx.testLib:lib:1.0.0'`
   
   创建一个model: testLib ,在 module 的build.gradle中配置:
   ```
   apply plugin: 'maven'
   
   uploadArchives {
       repositories.mavenDeployer {
           repository(url: LOCAL_REPO_URL)
           pom.groupId = PROJ_GROUP
           pom.artifactId = PROJ_ARTIFACTID
           pom.version = PROJ_VERSION
       }
   }
   ```
   
   在控制台运行上传:
   
   ` gradlew -p testLib clean build uploadArchives --info  `
 
   在项目的 build.gradle 中添加这个本地仓库：
   ```
   allprojects {
       repositories {
           google()
           jcenter()
           maven{
               url 'file:///E:/repo/'
           }
       }
   }

   ```
   在 module 的 build.gradle 中添加依赖：
   
   `implementation "com.xxx.testlib:lib:1.0.0"`
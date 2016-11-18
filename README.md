// ant filter, 用于拷贝时进行参数替换
import org.apache.tools.ant.filters.ReplaceTokens
import org.apache.tools.ant.filters.FixCrLfFilter

// 定义默认构建任务，即./gradlew如果不加参数会启动该任务
defaultTasks 'bundle'

// 定义全局变量，通常项目依赖包的版本号会放在这里，另外，如果子项目依赖的版本不同，可在子项目中overwrite
ext {
    aspectjweaverVersion = '1.8.2'
    commonsCodecVersion = '1.10'
    commonsLang3Version = '3.4'
    commonsLoggingVersion = '1.1.3'
    commonsPool2Version = '2.2'
    cxfRtFrontendJaxrsVersion = '3.0.1'
    dbcpVersion = '2.0.1'
    gexinRpSdkHttpVersion = '4.0.1.0'
    gsonVersion = '2.3'
    httpclientVersion = '4.3.5'
    jacksonJaxrsVersion = '1.9.11'
    javaxWsRsApiVersion = '2.0'
    jedisVersion = '2.5.2'
    junitVersion = '3.8.1'
    mybatisVersion = '3.1.1'
    mybatisSpringVersion = '1.2.0'
    mysqlConnectorJavaVersion = '5.1.32'
    plexusUtilsVersion = '2.0.5'
    quartzVersion = '2.2.1'
    slf4jLog4j12Version = '1.7.7'
    slf4jApiVersion = '1.7.7'
    springDataRedisVersion = '1.3.2.RELEASE'
    springVersion = '4.0.6.RELEASE'
    springContextVersion = '3.2.1.RELEASE'
    wizcryptVersion = '2.2'
}

// 获取命令行参数，以指定不同的构建环境
if (!hasProperty("env")) ext.env = "dev"

// 定义所有工程共有的逻辑
allprojects {
    group = 'com.example.xxxx'
    // 版本号会被子项目的version覆盖
    version = '1.0.0-SNAPSHOT'
}

// 定义所有子工程共有的逻辑
subprojects {
    // 使用java插件
    apply plugin: 'java'
    
    // 设置最低Java兼容版本
    sourceCompatibility = 1.7
    // 生成class的Java版本
    targetCompatibility = 1.7
    
    // 设置编译时的编码格式
    tasks.withType(JavaCompile) {
        options.encoding = 'UTF-8'
    }

    // 配置仓库
    repositories {
        // maven中心仓库
        mavenCentral()

        // 其他远程仓库，通常是第三方Jar包存放的位置
        maven { url 'http://repo.spring.io/plugins-release'}
        maven { url 'http://mvn.gt.igexin.com/nexus/content/repositories/releases/'}
        maven { url 'http://repo.example.me/nexus/content/groups/public'}
    }
}

subprojects {
    // 将所有子项目的源代码打成Jar包，用于上传到Maven仓库
    task sourcesJar(type: Jar, dependsOn: classes) {
        classifier = 'sources'
        from sourceSets.main.allSource
    }
}

// upms-start项目用于将整个项目打成一个可执行包
project(':xxxx-start') {
    apply plugin: 'application'
    // 指定项目main方法所在的类
    mainClassName = "com.xxxx.xxxx.start.Start"
    // 配置程序启动参数
    applicationDefaultJvmArgs = ['-Xms500m', '-Xmx500m', '-XX:PermSize=128m', '-XX:-UseGCOverheadLimit']

    // 根据环境选择配置脚本，将其解析为key/value字典
    Properties props = new Properties()
    props.load(new FileInputStream(project(':xxxx-config').file("src/main/resources/$env"+".properties")))

    // 清理配置文件目录
    task cleanConfDir(type: Delete) {
        delete fileTree(dir: "$buildDir/conf")
    }

    // 将所有配置文件进行参数替换并复制到指定位置
    task copyConfFiles(type: Copy, dependsOn: cleanConfDir) {
        // 创建conf文件夹
        def confDir = new File("$buildDir/conf")
        confDir.mkdirs()

        from project(':upms-config').file("src/main/resources/start/")

        eachFile {
            if ((it.name.split('\\.')[-1] == "properties") && 
                (it.name.split('\\.')[0] != "mailhtml")) {
                filter(FixCrLfFilter)
                filter(ReplaceTokens, tokens: props)
                filteringCharset = 'UTF-8'
            }
        }

        into file("$buildDir/conf")
    }
    // 向发布包增加额外的文件
    distributions {
        main {
            basename = "xxxx-${env}"
            contents {
                from(copyConfFiles) {
                    into "lib/conf"
                }
            }
        }
    }
    
    // 添加额外的配置到classpath
    startScripts.classpath.add(files('$APP_HOME/conf'))
    
    /* 比较难看的写法
    distZip.dependsOn copyConfFiles
    distZip {
        // 将conf文件夹下的内容添加到发布包中
        into(project.name+"-"+project.version) {
            from file("$buildDir")
            include 'conf/*'
        }
    }
    */
}

task bundle(dependsOn: project(':upms-start').tasks['distZip'])

// 该任务将所有的源码编译生成的class存储到一个包中
task allJar(type: Jar, dependsOn: subprojects.tasks['jar']) {
    baseName = 'allJar'
    subprojects.each { subproject ->
        from subproject.configurations.archives.allArtifacts.files.collect {
            zipTree(it)
        }
    }
}

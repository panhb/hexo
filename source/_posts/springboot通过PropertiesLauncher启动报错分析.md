---
title: springboot通过PropertiesLauncher启动报错分析
date: 2018-05-13 13:32:00
tags: [springboot]
---

公司目前的项目都是用的springboot进行开发，项目里面有些功能需要用到可执行程序，使用CommandLine执行命令调用，在执行命令时需要程序的绝对路径，而springboot打包插件是将所有的jar包和资源文件都打包成一个可执行的jar包，这样导致无法获取资源文件下面的可执行程序的绝对路径，导致命令无法执行。于是同事写了一个打包插件，把springboot打包成一个zip文件，并生成相关的启动命令，直接调用脚本即可启动程序，类似于dubbo的打包方式。     
具体启动命令如下：  
```java   
java -Dloader.main=xxx.xxx.Application -Dloader.path=lib,config,resource,xxx.jar -Dspring.profiles.active=dev org.springframework.boot.loader.PropertiesLauncher
```
之前的项目都用的好好的，然后昨天我启动的时候，springboot突然报一个错误，如下图     
![](https://raw.githubusercontent.com/panhb/images/master/res/propertieslauncher/1526196764.jpg)    
然后百度了一下，也没找出什么问题，网上说是resource下面有中文目录导致，但是并不我这个问题的原因。后来因为看到有JarFile，本身也是加载的时候报的错，就找resource下面是否有jar包，找到有几个zip包，也没多想，就删除了再启动，然后就可以正常启动了。虽然可以正常启动了，但是为什么resource下面不能放zip文件我很费解，于是就开始跟源码，找原因。     
在github上面把springboot的源码clone下来，PropertiesLauncher源码在spring-boot-tools的spring-boot-loader下面，首先看PropertiesLauncher的main方法做了些什么    
```java
public static void main(String[] args) throws Exception {
    PropertiesLauncher launcher = new PropertiesLauncher();
    args = launcher.getArgs(args);
    launcher.launch(args);
}
```
先创建了一个PropertiesLauncher对象，再看构造函数里面，创建对象的时候初始化了一些什么信息    
```java
public PropertiesLauncher() {
    try {
        this.home = getHomeDirectory();
        initializeProperties();
        initializePaths();
        this.parent = createArchive();
    }
    catch (Exception ex) {
        throw new IllegalStateException(ex);
    }
}
```
注意initializePaths()方法，再跟进initializePaths方法，看到
```java
private void initializePaths() throws Exception {
    String path = getProperty(PATH);
    if (path != null) {
        this.paths = parsePathsProperty(path);
    }
    debug("Nested archive paths: " + this.paths);
}
```
其中PATH为loader.path，也就是我们启动命令中配置的-Dloader.path=lib,config,resource,xxx.jar，再回到main方法，继续跟进，最终落脚到getClassPathArchives方法     
```java
protected List<Archive> getClassPathArchives() throws Exception {
    List<Archive> lib = new ArrayList<>();
    for (String path : this.paths) {
        for (Archive archive : getClassPathArchives(path)) {
            if (archive instanceof ExplodedArchive) {
                List<Archive> nested = new ArrayList<>(
                        archive.getNestedArchives(new ArchiveEntryFilter()));
                nested.add(0, archive);
                lib.addAll(nested);
            }
            else {
                lib.add(archive);
            }
        }
    }
    addNestedEntries(lib);
    return lib;
}

private List<Archive> getClassPathArchives(String path) throws Exception {
    String root = cleanupPath(handleUrl(path));
    List<Archive> lib = new ArrayList<>();
    File file = new File(root);
    if (!"/".equals(root)) {
        if (!isAbsolutePath(root)) {
            file = new File(this.home, root);
        }
        if (file.isDirectory()) {
            debug("Adding classpath entries from " + file);
            //第二个参数为是否递归
            Archive archive = new ExplodedArchive(file, false);
            lib.add(archive);
        }
    }
    Archive archive = getArchive(file);
    if (archive != null) {
        debug("Adding classpath entries from archive " + archive.getUrl() + root);
        lib.add(archive);
    }
    List<Archive> nestedArchives = getNestedArchives(root);
    if (nestedArchives != null) {
        debug("Adding classpath entries from nested " + root);
        lib.addAll(nestedArchives);
    }
    return lib;
}

/**
 * Convenience class for finding nested archives (archive entries that can be
 * classpath entries).
 */
private static final class ArchiveEntryFilter implements EntryFilter {

    private static final String DOT_JAR = ".jar";

    private static final String DOT_ZIP = ".zip";

    @Override
    public boolean matches(Entry entry) {
        return entry.getName().endsWith(DOT_JAR) || entry.getName().endsWith(DOT_ZIP);
    }

}
```
在获取jar包时，**将文件名后缀是.zip和.jar都视为jar包加载**。然后在为zip数据保存构造字符串时，发生数组下表越界。   
那么问题来了，如果我一定要将zip文件放在resource下面，应该怎么处理呢。继续看代码，如果是目录的话，默认是不执行递归的，就是说如果当前的path是resource，那么只会遍历到resource/xx下面，不会遍历resource/xx/xx这个目录下面的文件，也就是说resource/xx/xx/xx.zip这种情况是允许的。
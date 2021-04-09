##### MultiDex

```
Android 5.0以上版本不需要设置多dex模式，因为5.0以上默认的虚拟机是ART，ART没有65536的问题

编译流程：
1、打包资源文件，生成R.java文件（使用工具AAPT）
2、处理AIDL文件，生成java代码（没有AIDL则忽略）
3、编译 java 文件，生成对应.class文件（java compiler）
4、.class 文件转换成.dex文件（dex compiler）
5、打包成没有签名的apk（使用工具apkbuilder）
6、使用签名工具给apk签名（使用工具Jarsigner）
7、对签名后的.apk文件进行对齐处理，不进行对齐处理不能发布到Google Market（使用工具zipalign）

MultiDex安装原理：
1、解压apk，遍历里面的dex文件，如class1.dex/class2.dex等，然后又压缩成class1.zip/class2.zip，然后返回zip文件列表
2、第一次加载才会有解压和压缩过程，第二次直接读取sp中保存的dex信息，直接返回file list信息
3、反射ClassLoader的pathList字段
4、找到pathList字段对应的类makeDexElements方法
5、通过MultiDex.expandFieldArray这个方法扩展dexElements数组，添加到数组的后面去
6、反射将原来的Element数组和新增的Element数组合并，然后反射赋值给dexElements变量


ClassLoader加载类的过程：
不管是 PathClassLoader还是DexClassLoader，都继承自BaseDexClassLoader，加载类的代码在 BaseDexClassLoader中
1、构造方法通过传入dex路径，创建了DexPathList
2、ClassLoader的findClass方法最终是调用DexPathList.findClass()
3、DexPathList里面定义了一个dexElements数组，findClass就是遍历dexElements数组，拿到里面的dexFile对象，通过DexFile.loadClassBinaryName()加载一个类
```

> 为什么一个应用中方法数不能超过65535？
>
> 因为在Dalvik指令集里，调用方法的`invoke-kind`指令中，method reference index只给了16bits，最多能调用65535个方法，所以在生成dex文件的过程中，当方法数超过65535就会报错。细看指令集，除了method，field和class的index也是16bits，所以也存在65535的问题。一般来说，method的数目会比field和class多，所以method数会首先遇到65535问题，你可能都没机会见到field过65535的情况。

##### Think

```
优点：
1、支持类、资源、so修复
2、兼容性处理的很好，全平台支持
3、由于不用插桩，所以性能损耗很小
4、完善的开发文档和官方技术支持
5、gradle支持，再自己定义下可以一键打补丁包
6、dexDiff算法使得补丁文件较小
7、扩展性良好，代码中处处为开发者留出开放接口，简直业界良心
8、支持多次补丁

缺点：
1、不支持及时生效，下发补丁需要重启生效，MultiDex方案决定的
2、占用ROM空间较大，这点空间在如今的手机大ROM下也不算个事

加载补丁dex
在补丁前dex顺序是这样的：oldDex1 -> oldDex2 -> oldDex3..，那么假如修改了dex1中的文件，那么补丁顺序是这样的newDex1 -> oldDex1 -> oldDex2...其中合成后的newDex1中的类是oldDex1中除了dex.loader中标明的类之外的所有类，dex.loader中的类依然在oldDex1中。
原理同MultiDex，根据差量生成新的dex然后放在数组的最前面。
并且根据不同的版本做了相应的兼容性处理。


Android N之后的加载机制
在Dalvik虚拟机中，总是在运行时通过JIT（Just-In—Time）把字节码文件编译成机器码文件再执行，这样跑起来程序就很慢，所在ART上，改为AOT（Ahead-Of—Time）提前编译，即在安装应用或OTA系统升级时提前把字节码编译成机器码，这样就可以直接执行了，提高了运行效率。但是AOT有个缺点就是每次执行的时间都太长了，并且占用的ROM空间又很大，所以在Android N上Google做了混合编译同时支持JIT和AOT。混合编译的作用简单来说，在应用运行时分析运行过的代码以及“热代码”，并将配置存储下来。在设备空闲与充电时，ART仅仅编译这份配置中的“热代码”。

简单来说，就是在应用安装和首次运行不做AOT编译，先让用户愉快的玩耍起来，然后把在运行中JIT解释执行的那部分代码收集起来，在手机空闲的时候通过dex2aot编译生成一份名为app image的base.art文件，然后在下次启动的时候一次性把app image加载进来到缓存，【预先加载代替用时查找】以提升应用的性能。

Android N（7.x）对热补丁的影响
app image中已经存在的类会被插入到ClassLoader的ClassTable，再次加载类时，直接从ClassTable中取而不会走DefineClass。假设base.art文件在补丁前已经存在，这里存在三种情况：
1、补丁修改的类都不appimage中；这种情况是最理想的，此时补丁机制依然有效；
2、补丁修改的类部分在appimage中；这种情况我们只能更新一部分的类，此时是最危险的。一部分类是新的，一部分类是旧的，app可能会出现地址错乱而出现crash。
3、补丁修改的类全部在appimage中；这种情况只是造成补丁不生效，app并不会因此造成crash。

Thinker的解决方案是，完全废弃掉PathClassloader，而采用一个新建Classloader来加载后续的所有类，即可达到将cache无用化的效果。


加载补丁资源
全量替换资源
由于App加载资源是依赖Context.getResources()方法返回的Resources对象，Resources 内部包装了 AssetManager，最终由 AssetManager 从 apk 文件中加载资源。我们要做的就是新建一个AssetManager()，hook掉其中的addAssetPath()方法，将我们的资源补丁目录传递进去，然后循环替换Resources对象中的AssetManager对象，达到资源替换的目的。
```


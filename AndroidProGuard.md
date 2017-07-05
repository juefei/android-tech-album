# 混淆的原理

Java 是一种跨平台、解释型语言，Java 源代码编译成的class文件中有大量包含语义的变量名、方法名的信息，很容易被反编译为Java 源代码。为了防止这种现象，我们可以对Java字节码进行混淆。混淆不仅能将代码中的类名、字段、方法名变为无意义的名称，保护代码，也由于移除无用的类、方法，并使用简短名称对类、字段、方法进行重命名缩小了程序的size。

ProGuard由shrink、optimize、obfuscate和preverify四个步骤组成，每个步骤都是可选的，需要哪些步骤都可以在脚本中配置。  参见ProGuard官方介绍。

压缩(Shrink): 侦测并移除代码中无用的类、字段、方法、和特性(Attribute)。
优化(Optimize): 分析和优化字节码。
混淆(Obfuscate): 使用a、b、c、d这样简短而无意义的名称，对类、字段和方法进行重命名。
上面三个步骤使代码size更小，更高效，也更难被逆向工程。

预检(Preveirfy):  在java平台上对处理后的代码进行预检。

Proguard读入input jars(or wars,zips or directories)，经过四个步骤生成处理之后的jars(or wars,ears,zips or directories),Optimization步骤可选择多次进行。

为了确定哪些代码应该被保留，哪些代码应该被移除或混淆，需要确定一个或多个Entry Point。Entry Point经常是带有main methods,applets,midlets的classes,它们在混淆过程中会被保留。我们来看一下Proguard的几个步骤如何处理Entry Points。  

1.在压缩阶段，Proguard从上述Entry Points开始遍历搜索哪些类和类成员被使用。其他没有被使用的类和类成员会移除。

2.在优化阶段，Proguard进一步设置非Entry Point的类和方法为private、static和final来进行优化，不使用的参数会被移除，某些方法会被标记被内联。

3.在混淆阶段，Proguard重命名非Entry Points的类和类成员。

4.预检阶段是唯一没有触及Entry Points的阶段。

# Android Studio 默认的混淆方案及字段解读

## 开启混淆

参见google官方文档压缩代码和资源
要通过Proguard启动代码压缩，在build.gradle文件内相应的构建类型中添加minifyEnabled true。
```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```
除了 minifyEnabled 属性外，还有用于定义 ProGuard 规则的 proguardFiles 属性：
```
proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
```     
      
## 构建输出

构建时Proguard都会输出下列文件:

(1)dump.txt —-      说明APK中所有类文件的内部结构

(2)mapping.txt  —-  提供原始与混淆过的类、方法和字段名称之间的转换

(3)seeds.txt  —-    列出未进行混淆的类和成员

(4)usage.txt  —-    列出从APK移除的代码

这些文件保存在/build/outputs/mapping/release目录下。

## 默认的混淆方案及字段解读
下面结合默认混淆文件中的内容来解释混淆的参数:  参见Proguard官方字段解读

```  
不使用大小写混写类名
-dontusemixedcaseclassnames
```  
默认情况下混淆的类名可以包含大小写字符的混合。

```  
不忽略公共类库
-dontskipnonpubliclibraryclasses
```  
指定不去忽略非public的library classes。从Proguard 4.5开始，是默认的设置。

```  
-dontoptimize
-dontpreverify
```  
默认optimize和preverify选项是关闭的，因为Android的dex并不像Java虚拟机需要optimize(优化)和previrify(预检)两个步骤。

```  
指定哪个属性不要混淆，可一次指定多个属性
-keepattributes [attribute_filter]
```  
通常Exceptions, Signature, Deprecated, SourceFile, SourceDir, LineNumberTable, LocalVariableTable, LocalVariableTypeTable, Synthetic, EnclosingMethod, RuntimeVisibleAnnotations, RuntimeInvisibleAnnotations, RuntimeVisibleParameterAnnotations, RuntimeInvisibleParameterAnnotations, and AnnotationDefault属性需要被保留，根据项目具体使用情况保留。

这里需要特别注意的一点是,gradle默认的keepattributes属性不全，只保留了Annotation,Signature,InnerClasses,EnclosingMethod,为了混淆之后定位csh代码方便，我们需要在proguard_rules.pro中手动添加抛出异常时保留代码行号,并且重命名抛出异常时的文件名称,这样能方便定位问题:

```  
抛出异常时保留代码行号
-keepattributes SourceFile,LineNumberTable
```  
```  
重命名抛出异常时的文件名称
-renamesourcefileattribute SourceFile
```  
keep选项非常重要，keep指定了哪些类，哪些方法不被混淆，从而保证了程序的正常运行。官方的keep用法有6种:


左边不带names的选项为From being removed or renames,即不会被移除或重命名，即使类或类成员未被使用。带有names的选项为From being renamed,不会被重命名，如果是无用的类或类成员，会被移除。

(1)-keep(names)选项   指定类和类成员(变量和方法)不被混淆
```  
-keep [,modifier,...] class_specification
eg.
```  
```  
指定类名不被改变
-keep public class com.google.vending.licensing.ILicensingService
```  
```  
指定使用了Keep注解的类和类成员都不被改变
-keep @android.support.annotation.Keep class * {*;}
```  
关于Keep注解的解释参见文末参考链接

(2)-keepclassmembers(names)  指定类成员不被混淆,类名会被混淆
```  
-keepclassmembers [,modifier,...] class_specification
eg.keep setters in views 使得animations仍然能够工作

-keepclassmembers public class * extends android.view.View {
    void set*(***);
    *** get*();
}
```  

(3)-keepclasseswithmembers(names) 指定类和类成员都不被混淆
```
-keepclasseswithmembers [,modifier,...] class_specification
```
eg.包含native方法的类名和native方法都不能被混淆，如果native方法未被调用，则被移除。由于native方法与对应so库中的方法名称对应，方法名被混淆会导致调用出现问题，所以native方法不能被混淆。
```
-keepclasseswithmembernames class * {
    native <methods>;
}
```
通用Options:
(1)-verbose 打印混淆详细信息

(2)-dontnote选项:指定不去输出打印该类产生的错误或遗漏
```
-dontnote com.android.vending.licensing.ILicensingService

-dontnote android.support.**
```
(3)-dontwarn选项:指定不去warn unresolved references和其他重要的problem
```
-dontwarn android.support.**
```
如上面(2)(3)所示，android.support的libraries需要保留

至此，gradle自带的proguard-android.txt文件相关字段已解析完毕。下面将介绍我们自定义的proguard-rules.pro文件需要添加什么参数。

## 自定义混淆文件

一般而言，我们会定义我们自己的proguard-rules.pro,下面列出自定义的一个proguard-rules.pro供大家参考。在看自定义的混淆文件之前，先讲解一下Filters和assumenosideeffects,以便更好地理解下面的指令。

(1)Filters
```
?    matches any single character in a name.(匹配一个字符)
*    matches any part of a name not containing the directory separator.（匹配一个名字，除了目录分隔符外的任意部分）
**    matches any part of a name, possibly containing any number of directory separators.(匹配任意名,可能包含任意路径分隔符)
!  exclude
<field>     匹配类中的所有字段
<method>    匹配类中所有的方法
<init>      匹配类中所有的构造函数
```
eg.
```
-keep class com.lily.test.** 本包和所包含子包下的类名都保持
-keep class com.lily.test.* 保持该包下的类名
-keep class com.lily.test.** {*;} 保持包和子包的类名和里面的内容均不被混淆
```
(2)-assumenosideeffects 指令:  下文会用在android log的移除上
assumeosideeffects是Optimization过程中的选项，所以为保证指令的有效，需要开启optimization。这个指令的含义是Proguard会在optimization过程中删除对这些方法的调用，需要注意:Only use this option if you know what you’re doing!

下面是自定义混淆文件的一个范例,四大组件,native方法，反射用到的类，一些引入的第三方库等都不能进行混淆:
```
# 代码混淆压缩比，在0~7之间
-optimizationpasses 5# 混合时不使用大小写混合，混合后的类名为小写
-dontusemixedcaseclassnames

# 指定不去忽略非公共库的类
-dontskipnonpubliclibraryclasses

# 不做预校验，preverify是proguard的四个步骤之一，Android不需要preverify，去掉这一步能够加快混淆速度。
-dontpreverify

-verbose

#google推荐算法
-optimizations !code/simplification/arithmetic,!code/simplification/cast,!field/*,!class/merging/*

# 避免混淆Annotation、内部类、泛型、匿名类
-keepattributes *Annotation*,InnerClasses,Signature,EnclosingMethod

# 重命名抛出异常时的文件名称
-renamesourcefileattribute SourceFile

# 抛出异常时保留代码行号
-keepattributes SourceFile,LineNumberTable

# 处理support包
-dontnote android.support.**
-dontwarn android.support.**

# 保留四大组件，自定义的Application等这些类不被混淆
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Appliction
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.preference.Preference
-keep public class com.android.vending.licensing.ILicensingService

# 保留本地native方法不被混淆
-keepclasseswithmembernames class * {
    native <methods>;
}

# 保留枚举类不被混淆
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# 保留Parcelable序列化类不被混淆
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

#第三方jar包不被混淆
-keep class com.github.test.** {*;}

#保留自定义的Test类和类成员不被混淆
-keep class com.lily.Test {*;}
#保留自定义的xlog文件夹下面的类、类成员和方法不被混淆
-keep class com.test.xlog.** {
    <fields>;
    <methods>;
}

#assume no side effects:删除android.util.Log输出的日志
-assumenosideeffects class android.util.Log {
    public static *** v(...);
    public static *** d(...);
    public static *** i(...);
    public static *** w(...);
    public static *** e(...);
}

#保留Keep注解的类名和方法
-keep,allowobfuscation @interface android.support.annotation.Keep
-keep @android.support.annotation.Keep class *
-keepclassmembers class * {
    @android.support.annotation.Keep *;
}
```

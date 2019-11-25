本文从JVM源码角度分析JVM启动流程。

- 操作系统：macOS 10.13.6
- JDK版本：OpenJDK9u

注：下文中`jvm`表示OpenJDK源码根目录

## main()

程序主入口

> jvm/jdk/src/java.base/share/native/launcher/main.c

```C
int
main(int argc, char **argv)
{
    ...
    return JLI_Launch(margc, margv,
                   sizeof(const_jargs) / sizeof(char *), const_jargs,
                   0, NULL,
                   VERSION_STRING,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   HAS_JAVA_ARGS,
                   const_cpwildcard, const_javaw, 0);
}
```

## JLI_Launch

> jvm/jdk/src/java.base/share/native/libjli/java.c

```C
/*
 * Entry point.
 */
int
JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* UNUSED dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* unused */
)
{
    ...

    // 创建运行环境，如检查系统使用的数据模型（32bit、64bit），
    // 获取使用的JRE路径，找到jvm.cfg解析已知的vm类型，
    // 设置新的LD_LIBRARY_PATH变量
    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               jvmcfg,  sizeof(jvmcfg));
    ...
    // 加载Java虚拟机
    if (!LoadJavaVM(jvmpath, &ifn)) {
        return(6);
    }
    ...
    // 解析命令行参数
    if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath))
    {
        return(ret);
    }
    ...
    // 最终会调用到JavaMain函数
    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

## LoadJavaVM & JVMInit

> jvm/jdk/src/java.base/macosx/native/libjli/java_md_macosx.c

macOS对应的是java_md_macosx.c，其他系统位于目录src/java.base对应的操作系统类型下的java_md.c或java_md_xxx.c

```C
jboolean
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    ...
    // 装载动态链接库, jre/lib/libjvm.so
    libjvm = dlopen(jvmpath, RTLD_NOW + RTLD_GLOBAL);
    // 导出动态链接库函数JNI_CreateJavaVM, 挂载到ifn
    ifn->CreateJavaVM = (CreateJavaVM_t)
        dlsym(libjvm, "JNI_CreateJavaVM");
    ...
}

// MacOSX we may continue in the same thread
int
JVMInit(InvocationFunctions* ifn, jlong threadStackSize,
                 int argc, char **argv,
                 int mode, char *what, int ret) {
    if (sameThread) {
        ...
        rslt = JavaMain(&args);
        return rslt;
    } else {
        ...
        // 其他操作系统需要在新线程中执行，原因参考：
        // Java launcher should create JVM from non-primordial thread
        // https://bugs.openjdk.java.net/browse/JDK-6316197
    }
}
```

无论何种操作系统，最后都会执行`JavaMain`函数

## JavaMain

> jvm/jdk/src/java.base/share/native/libjli/java.c

```C
int JNICALL
JavaMain(void * _args)
{
    ...
    // 初始化虚拟机，通过调用libvm.so导出的CreateJavaVM执行真正的
    // 创建虚拟机逻辑，最主要的时把env绑定到具体的jvm上，
    // 后续的操作都是通过JNIEnv实现的
    if (!InitializeJVM(&vm, &env, &ifn)) {
        ...
    }
    // 加载主类
    mainClass = LoadMainClass(env, mode, what);
    // 获取主类的main()方法id
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    // 调用main方法
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);
    ...
    // 当main方法执行完成后JVM结束，程序退出
    LEAVE();
}

static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    ...
    // 执行JNI_CreateJavaVM方法，下文将讲到
    r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
    ...
}

/*
 * Loads a class and verifies that the main class is present and it is ok to
 * call it for more details refer to the java implementation.
 */
static jclass
LoadMainClass(JNIEnv *env, int mode, char *name)
{
    ...
    // 获取sun.launcher.LauncherHelper类
    jclass cls = GetLauncherHelperClass(env);
    // 获取LauncherHelper类的checkAndLoadMain()方法id
    NULL_CHECK0(mid = (*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;"));
    // 调用LauncherHelper类的checkAndLoadMain()方法检查并加载主类
    NULL_CHECK0(result = (*env)->CallStaticObjectMethod(env, cls, mid,
                                            USE_STDERR, mode, str));
}

jclass
GetLauncherHelperClass(JNIEnv *env)
{
    if (helperClass == NULL) {
        NULL_CHECK0(helperClass = FindBootStrapClass(env,
                "sun/launcher/LauncherHelper"));
    }
    return helperClass;
}
```

## JNI_CreateJavaVM

> jvm/hotspot/src/share/vm/prims/jni.cpp

```C
_JNI_IMPORT_OR_EXPORT_ jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {
    ...
    result = JNI_CreateJavaVM_inner(vm, penv, args);
    ...
}

static jint JNI_CreateJavaVM_inner(JavaVM **vm, void **penv, void *args) {
  ...
  result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
  ...
}
```

## Threads::create_vm

> jvm/hotspot/src/share/vm/runtime/thread.cpp

```C
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  // 解析虚拟机参数，如：-XX:+PrintVMOptions
  jint parse_result = Arguments::parse(args);
  if (parse_result != JNI_OK) return parse_result;
  ...
}
```

## Arguments::parse

> jvm/hotspot/src/share/vm/runtime/arguments.cpp

```C
jint Arguments::parse(const JavaVMInitArgs* initial_cmd_args) {
  ...
  code = expand_vm_options_as_needed(initial_java_options_args.get(),
                                     &mod_java_options_args,
                                     &cur_java_options_args);
  ...
  jint result = parse_vm_init_args(cur_java_tool_options_args,
                                 cur_java_options_args,
                                 cur_cmd_args);
   ...
}

jint Arguments::expand_vm_options_as_needed(const JavaVMInitArgs* args_in,
                                            ScopedVMInitArgs* mod_args,
                                            JavaVMInitArgs** args_out) {
  jint code = match_special_option_and_act(args_in, mod_args);
  ...
}

jint Arguments::match_special_option_and_act(const JavaVMInitArgs* args,
                                             ScopedVMInitArgs* args_out) {
    ...
    if (match_option(option, "-XX:+PrintVMOptions")) {
      PrintVMOptions = true;
      continue;
    }
    if (match_option(option, "-XX:-PrintVMOptions")) {
      PrintVMOptions = false;
      continue;
    }
    ...
}

jint Arguments::parse_vm_init_args(const JavaVMInitArgs *java_tool_options_args,
                                   const JavaVMInitArgs *java_options_args,
                                   const JavaVMInitArgs *cmd_line_args) {
  ...
  result = parse_each_vm_init_arg(java_options_args, &patch_mod_javabase, Flag::ENVIRON_VAR);
  ...
}

jint Arguments::parse_each_vm_init_arg(const JavaVMInitArgs* args, bool* patch_mod_javabase, Flag::Flags origin) {
    ...
    // -Xms
    } else if (match_option(option, "-Xms", &tail)) {
        ...
    // -Xmx
    } else if (match_option(option, "-Xmx", &tail) || match_option(option, "-XX:MaxHeapSize=", &tail)) {
      ...
}
```

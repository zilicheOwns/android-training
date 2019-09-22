# gradle相关学习
最近一直在怼Android的gradle插件，写了个view防止快速点击的小插件，有groovy和kotlin版本。直戳「[FunctionPlugin](https://github.com/zilicheOwns/SingleClickPlugin)」，持续更新中。

记录分享一下最近gradle的学习过程。遇到了很多坑，发现最合理的学习方法就是看源码和官方文档，百度google可以用，但要拒绝ctrlC & ctrlV，一定要经过自己验证得出自己的结论，否则很容易被坑，事倍功半，得不偿失。

* [gradle官网 ](https://docs.gradle.org/current/userguide/getting_started.html)

* [gradle文档中文版](https://dongchuan.gitbooks.io/gradle-user-guide-/build_script_basics/projects_and_tasks.html)

* [gradle源码](https://github.com/gradle/gradle/tree/v4.1.0)

* [android plugin DSL](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.BaseExtension.html#com.android.build.gradle.BaseExtension:jacoco(org.gradle.api.Action))

* [gradle DSL](https://docs.gradle.org/current/dsl/index.html)

* [Android修炼手册](https://github.com/5A59/android-training) 对于gradle插件原理，执行过程。gradle打包apk的过程介绍的非常详细。认真学下来，有很大的收获。感谢博主分享。

* [Hunter](https://github.com/Leaking/Hunter) 比较好的开源学习项目

### Build Workflow

![avatar](https://upload-images.jianshu.io/upload_images/2516746-fafc29dfa781d1fc.png?imageMogr2/auto-orient/strip|imageView2/2/w/993/format/webp)

## A&Q

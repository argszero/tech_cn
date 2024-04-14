专注于代码和逻辑，是开发者心中的理想，但现实中，漫长的打包过程和上传到Testflight的繁琐操作极大的浪费了时间。本文将介绍如何通过fastlane，在GitHub上实现FlutterApp的自动打包上传到Testflight。

# 创建FlutterApp

假设我们要做一款面试辅助工具，我们其命名为Ace。我们首先通过flutter命令，创建项目，并确保项目可以正常打包，如下所示
```bash
flutter create --org fun.args ace
cd ace
flutter build apk
flutter build ipa
```
注意： 因为我们的org是fun.args， fun是kotlin里的关键字，不能直接作为包名称，所以打包时会遇到编译错误。我们只需要打开android/app/src/main/kotlin/fun/args/ace/MainActivity.kt,将其中的包名称
```
package fun.args.ace
```
修改为：
```
package `fun`.args.ace
```
即可

其中，对应IOS，如果遇到错误“Signing for "Runner" requires a development team.“， 我们需要`open ios/Runner.xcodeproj`, 使用xcode打开项目，点击左上角的“+”按钮，选择`Team`， 然后点击`Add Team`按钮，选择`Team ID`， 然后在`Team ID`输入`fun.args`即可

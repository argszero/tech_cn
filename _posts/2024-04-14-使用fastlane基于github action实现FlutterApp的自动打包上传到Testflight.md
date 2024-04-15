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



# 上传代码到github

在github上创建一个空的repository，执行以下命令将代码上车到github
```bash
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:argszero/ace.git
git push -u origin main
```

# 配置fastlane

配置fastlane是很容易出错的,官方文档省略了很多重要的步骤,按照官方文档一步步下来总是遇到各种问题。另外，因为配置方式不固定，官方文档会给出多种选择，让你无从下手。这里我们只使用一条固定的方式来实现。只需要按照以下步骤，一定可以实现自动打包上传到Testflight。后续如果你再有其他需求或换用其他方式，可以再自行修改。

1. 安装并初始化fastlane
```bash
xcode-select --install
cd ios
vim Gemfile
vim Podfile
ge install bundler
gem bundle install
pod install
```
其中， Gemfile的内容如下:
```
source "https://rubygems.org"

gem "fastlane"
gem "cocoapods"

plugins_path = File.join(File.dirname(__FILE__), 'fastlane', 'Pluginfile')
eval_gemfile(plugins_path) if File.exist?(plugins_path)
```
Podfile的内容如下:
```
# Uncomment this line to define a global platform for your project
# platform :ios, '12.0'

# CocoaPods analytics sends network stats synchronously affecting flutter build latency.
ENV['COCOAPODS_DISABLE_STATS'] = 'true'

project 'Runner', {
  'Debug' => :debug,
  'Profile' => :release,
  'Release' => :release,
}

def flutter_root
  generated_xcode_build_settings_path = File.expand_path(File.join('..', 'Flutter', 'Generated.xcconfig'), __FILE__)
  unless File.exist?(generated_xcode_build_settings_path)
    raise "#{generated_xcode_build_settings_path} must exist. If you're running pod install manually, make sure flutter pub get is executed first"
  end

  File.foreach(generated_xcode_build_settings_path) do |line|
    matches = line.match(/FLUTTER_ROOT\=(.*)/)
    return matches[1].strip if matches
  end
  raise "FLUTTER_ROOT not found in #{generated_xcode_build_settings_path}. Try deleting Generated.xcconfig, then run flutter pub get"
end

require File.expand_path(File.join('packages', 'flutter_tools', 'bin', 'podhelper'), flutter_root)

flutter_ios_podfile_setup

target 'Runner' do
  use_frameworks!
  use_modular_headers!

  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))
  target 'RunnerTests' do
    inherit! :search_paths
  end
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
  end
end
```

2. 配置fastlane

```bash
bundle install
fastlane init
```
可以使用`bundle exec fastlane init`命令，因为这条命令其实生成的配置文件还非常简单，还需要手动修改，所以在这里我们采用手动创建配置文件的方式：

在ios文件夹下创建fastlane文件夹,并创建以下文件
```
fastlane init
Appfile
Fastfile
Matchfile
Pluginfile
```
文件内容如下：
``` bash    
# Appfile
app_identifier("your-app-bundle-id") # The bundle identifier of your app
apple_id("your-apple-id") # Your Apple Developer Portal username

itc_team_id("your-itc-team-id") # App Store Connect Team ID
team_id("your-team-id") # Developer Portal Team ID
```

```bash
# Fastfile

Spaceship::ConnectAPI::App.const_set('ESSENTIAL_INCLUDES', 'appStoreVersions')

default_platform(:ios)

platform :ios do
  desc "Set Info.plist Version and Build Number"
  lane :set_full_version do
    version = flutter_version()
    puts "app_store_build_number----------------------"
    build_num = app_store_build_number( version:version['version_name'],live: false,api_key_path: "./app_store_connect.json",initial_build_number:1)
    puts build_num
    puts "app_store_build_number----------------------"

    increment_version_number(version_number: version['version_name'])
    increment_build_number(build_number: build_num + 1)
  end

  desc "Push a new build to TestFlight"
  lane :deploy do
    setup_ci if is_ci
    match(
      type: "appstore",
      readonly: is_ci,
    )
    set_full_version()
    build_app(workspace: "Runner.xcworkspace", scheme: "Runner")
    upload_to_testflight(api_key_path: "./app_store_connect.json", skip_waiting_for_build_processing: true)

  end
end
```

```bash
# Matchfile
git_url("your-git-url")
storage_mode("git")
type("appstore") # The default type, can be: appstore, adhoc, enterprise or development
app_identifier("fun.args.ace") # The bundle identifier of your app
username("your-apple-id") # Your Apple Developer Portal username
```

```bash
# Pluginfile
gem 'fastlane-plugin-flutter_version'
```

创建完成后，打开fastlane文件夹，执行以下命令
执行命令:
```
fastlane match appstore ; fastlane match adhoc;fastlane match development;
```

在Github上,不能执行自动签名,所以需要手动签名。
打开xcode，选择Runner - `Signing & Capabilites`, 取消选择Atomically mangage signing， Provisioning Profiles选择 match Appstore fun.args.ace

3. 在Appstore创建App

在appstore创建App, 
Platforms:iOS
Name:面试专家
Primary Language: Chinese
Bundle ID: 选择fun.args.ace
sku: fun.args.ace
User Access: 选择limit access
选择创建

4. 配置github

编辑github action文件：
```bash
# .github/workflows/ios-testflight.yaml
name: build-ios-app
on:
  push:
    branches:
      - master
      - main
jobs:
  build:
    runs-on: macos-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: stable
          cache: true
      
      - name: Set up App Store Connect
        run: |
          echo "${{ secrets.APP_STORE_CONNECT }}" | base64 --decode > ios/app_store_connect.json
      
      - name: Build and Deploy
        run: |
          flutter pub get
          cd ios
          pod install
          bundle install
          bundle exec fastlane deploy
        env:
            MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
            MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.GIT_BASIC_AUTHORIZATION }}
      - name: show error logs
        if: always()
        run: |
          cat /Users/runner/Library/Logs/gym/Runner-Runner.log
```

设置运行变量:
* MATCH_PASSWORD
* MATCH_GIT_BASIC_AUTHORIZATION
* APP_STORE_CONNECT

![humnail.png](https://github.com/horlengg/storage_repo/blob/dev/ios-home-widget.jpeg?raw=true)

<br>

# Implement an iOS Home Widget in Flutter App

<br>

*In this blog post, I’ll guide you step-by-step on how to create and integrate a native iOS Home Widget into your Flutter application. Home Widgets provide quick, glanceable information right on the iPhone’s home screen, improving user engagement and app visibility.*
<br>
<br>

*Whether you’re new to Flutter or want to enhance your app with useful widgets, this tutorial will help you deliver a seamless widget experience on iOS.*

<br>

![IOS Home Widget](https://res.cloudinary.com/decme1qrv/image/upload/v1754796524/ios-home-widget-demo_fspmcx.gif)

<br>
<br>
<br>
<br>

## Contents

1. *[Prerequisites](#prerequisites)*
1. *[Set up](#set-up)*
2. *[Creating the iOS Widget Extension](#creating-the-ios-widget-extension)*
3. *[Configuring App Groups for Data Sharing](#configuring-app-groups-for-data-sharing)*
4. *[Implementing Widget UI with SwiftUI](#implementing-widget-ui-with-swiftui)*
5. *[Conclusion](#conclusion)*

<br>
<br>
<br>
<br>


## Prerequisites
*For this implementation I'm using technologies version :*
<br>

```txt
    Flutter 3.32.8
    Dart SDK version: 3.8.1
    OS Device version : IOS 18
```

<br>
<br>

*Notic : If you using other version  please make sure it's follow requirement 
 [home_widget](https://pub.dev/packages/home_widget)*


<br>
<br>

## Set up

<br>

*Create a flutter project*

```
    flutter create your_app_name
```
<br>

*Add dependencies*

```yaml
    dependencies:
    flutter:
        sdk: flutter

    # The following adds the Cupertino Icons font to your application.
    # Use with the CupertinoIcons class for iOS style icons.
    cupertino_icons: ^1.0.8
    home_widget: ^0.8.0
```
<br>
<br>
<br>

## Creating the iOS Widget Extension

*On iOS Widgets are an App Extension. The following steps explain how to add the correct App Extension, how to do the basic configuration to read/display data you send from your App in the Widget and how to setup GroupIds to ensure the correct communication between your App and the Widget.*

<br>
<br>

### Add a Widget to your App in Xcode
<br>

*Add a widget extension by going ***File > New > Target > Widget Extension****
<br>
<br>


![widget_extension](https://res.cloudinary.com/decme1qrv/image/upload/v1754799079/horleng/Screenshot_2025-08-10_at_11.11.12_in_the_morning_mrf9ef.png)
<br>
<br>

***Fill in your desired name for the Widget***

<br>

![create-widget-extension-ios](https://res.cloudinary.com/decme1qrv/image/upload/v1754799041/horleng/Screenshot_2025-08-10_at_11.10.27_in_the_morning_cjropw.png)

<br>

*Notic : If you want implement background work, You need to add AppIntent to your widget extension!.*

<br>

*The generated Widget code includes the following classes:*

-   ****TimelineProvider*** - Provides a Timeline of entries at which the System will update the Widget automatically*
-   ****TimelineEntry*** - Represents the Data Object used to build the Widget. The date field is necessary and defines the point in time at which the Timeline would update*
-   ****View*** - The Widget itself, which is built with SwiftUI*
-   ****Widget*** - Configuration: Make note of the kind you set in the Configuration as this is what's needed to update the Widget from Flutter*

<br>
<br>

## Configuring App Groups for Data Sharing

*home_widget syncs data between your App and the Widget using App Groups.*
<br>
<br>

*Go to your Apple Developer Account and add a new group. Add this group to your Runner and the Widget Extension inside XCode: ***Signing & Capabilities > App Groups > +.****

<br>
<br>

![Group ID](https://res.cloudinary.com/decme1qrv/image/upload/v1754800391/horleng/Screenshot_2025-08-10_at_11.33.01_in_the_morning_vvhegs.png)

<br>

![Group ID](https://res.cloudinary.com/decme1qrv/image/upload/v1754800335/horleng/Screenshot_2025-08-10_at_11.32.07_in_the_morning_vcm7eg.png)

<br>

*Notic : Make sure your Group ID in Runner and Widget Extension are the same value!.*
<br> 
<br> 
<br> 

### AppIntent

*Using AppIntents you need to select both your App and your WidgetExtension in the Target Membership panel.*
<br>

- *Open ***DemoWidget/AppIntent.swift****
- *In Target Membership panel select both your app and WidgetExtension*
- *Add WidgetExtension to your PodFile*

<br>
<br>

![Membership panel](https://res.cloudinary.com/decme1qrv/image/upload/v1754807894/horleng/Screenshot_2025-08-10_at_1.37.43_in_the_afternoon.png)

<br>
<br>

```Podfile

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

  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))
  target 'RunnerTests' do
    inherit! :search_paths
  end
  target 'DemoWidgetExtension' do
    use_frameworks!
    use_modular_headers!
 
    pod 'home_widget', :path => '.symlinks/plugins/home_widget/ios'
 end
end

post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
  end
end

```
<br>
<br>

*Start install dependencies :*
<br>

```
flutter clean
flutter pub get
cd ios
pod install
```


<br>
<br>

## Implementing Widget UI with SwiftUI
*Let's create IOS Home widget with swiftUI and share data with flutter app.*
<br>
*Here is sample code for implement both interaction as background work and detect widget click.*


<br>
<br>

### Native Code
<br>
<i>DemoWidget/DemoWidget.swift</i>
<br>

```swift

//
//  DemoWidget.swift
//  DemoWidget
//
//  Created by ly.houleng on 9/8/25.
//

import WidgetKit
import SwiftUI

// Timeline entry with date and message
struct SimpleEntry: TimelineEntry {
    let date: Date
    let counter: Int
}

// Timeline provider that supplies entries
struct Provider: TimelineProvider {
    func placeholder(in context: Context) -> SimpleEntry {
        loadWidgetData()
    }

    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> Void) {
        completion(loadWidgetData())
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<SimpleEntry>) -> Void) {
        let timeline = Timeline(entries: [loadWidgetData()], policy: .never)
        completion(timeline)
    }
    func loadWidgetData() -> SimpleEntry {
        let userDefaults = UserDefaults(suiteName: "group.demo.ioshomewidget")
        let counter = userDefaults?.integer(forKey: "counter") ?? 0
        return SimpleEntry(date: Date(), counter: counter)
    }
}

// The widget view with tap URL
struct DemoWidgetEntryView : View {
    var entry: SimpleEntry

    var body: some View {
        VStack {
            
            HStack(alignment: .top) {
                
                VStack{
                    Text("Background work")
                        .font(.system(size: 14))
                        .foregroundColor(.green)
                    Spacer()
                        .frame(height: 20)
                    HStack {
                        Button(intent: BackgroundIntent(method: "decrement")) {
                            Image(systemName: "minus")
                                .font(.system(size: 16))
                                .foregroundColor(.white)
                                .frame(width: 40, height: 40)
                                .background(Circle().fill(Color.blue))
                        }
                        .buttonStyle(PlainButtonStyle())
                        Text("\(entry.counter)")
                            .font(.system(size: 30))
                        
                        Button(intent: BackgroundIntent(method: "increment")) {
                            Image(systemName: "plus")
                                .font(.system(size: 16))
                                .foregroundColor(.white)
                                .frame(width: 40, height: 40)
                                .background(Circle().fill(Color.blue))
                        }
                        .buttonStyle(PlainButtonStyle())
                    }
                }
                Rectangle()
                    .frame(width: 1)   // thickness of the vertical line
                    .foregroundColor(.gray)  // line color
                    .frame(maxHeight: .infinity)
                    .opacity(0.4)
                
                VStack(alignment: .leading) {
                    Text("Detect widget click")
                        .font(.system(size: 14))
                        .foregroundColor(.green)
                    Spacer()
                        .frame(height: 20)
                    
                    HStack {
                        
                        Link(destination: URL(string: "myhomewidgetapp://func_show_qr?homeWidget")!) {
                            Text("Show QR")
                                .font(.system(size: 14))
                                .foregroundColor(.white)
                                .padding(8)
                                .background(Color.green)
                                .cornerRadius(6)
                        }
                        Link(destination: URL(string: "myhomewidgetapp://func_scan_qr?homeWidget")!) {
                            Text("Scan QR")
                                .font(.system(size: 14))
                                .foregroundColor(.white)
                                .padding(8)
                                .background(Color.green)
                                .cornerRadius(6)
                        }
                        
                    }
                    
                    
                }
                    
                }
                
        }
        
    }
}

//
struct DemoWidget: Widget {
    let kind: String = "DemoWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            if #available(iOS 17.0, *) {
                DemoWidgetEntryView(entry: entry)
                    .containerBackground(Color.white, for: .widget)
            } else {
                DemoWidgetEntryView(entry: entry)
                    .padding()
                    .background(Color.white)
            }
        }
        .configurationDisplayName("My Custom Home Widget")
        .description("Display QR Code and Scan QR Function")
        .supportedFamilies([.systemMedium])
    }
}

#Preview(as: .systemMedium) {
    DemoWidget()
} timeline: {
   SimpleEntry(date: .now, counter: 0)
}

```

<br>
<br>

*DemoWidget/AppIntent.swift*
<br>

```swift

//
//  AppIntent.swift
//  DemoWidget
//
//  Created by ly.houleng on 9/8/25.
//

import AppIntents
import Foundation
import home_widget



@available(iOS 17, *)
@available(iOSApplicationExtension, unavailable)
extension BackgroundIntent: ForegroundContinuableIntent {}


@available(iOS 17, *)
public struct BackgroundIntent: AppIntent {
   static public var title: LocalizedStringResource = "HomeWidget Background Intent"


    @Parameter(title: "Method")
    var method: String

    public init() {
        method = "increment"
      }

      public init(method: String) {
        self.method = method
      }


   public func perform() async throws -> some IntentResult {
      await HomeWidgetBackgroundWorker.run(
        url: URL(string: "myhomewidgetapp://\(method)"),
        appGroup: "group.demo.ioshomewidget"
      )
       return .result()
   }
}

```

<br>
<br>

<i>AppDelegate.swift</i>
<br>

```swift

import UIKit
import Flutter
import home_widget

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
      
   GeneratedPluginRegistrant.register(with: self)
    
    // This is required for App Intents (iOS 17+) background updates
    if #available(iOS 17, *) {
      HomeWidgetBackgroundWorker.setPluginRegistrantCallback { registry in
        GeneratedPluginRegistrant.register(with: registry)
      }
    }
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}

```

<br>
<br>

### Dart Code

<br>
<i>lib/main.dart</i>
<br>

```dart


import 'dart:async';

import 'package:flutter/material.dart';
import 'package:home_widget/home_widget.dart';

const appGroupID = 'group.demo.ioshomewidget';
const _countKey = 'counter';

/// Gets the currently stored Value
Future<int> get _value async {
  final value = await HomeWidget.getWidgetData<int>(_countKey, defaultValue: 0);
  return value!;
}

/// Retrieves the current stored value
/// Increments it by one
/// Saves that new value
/// @returns the new saved value
Future<void> _increment() async {
  final value = await _value;
  await _sendAndUpdate(value + 1);
}

/// Clears the saved Counter Value
Future<void> _decrement() async {
  final oldValue = await _value;
  await _sendAndUpdate(oldValue - 1);
}

/// Stores [value] in the Widget Configuration
Future<void> _sendAndUpdate([int? value]) async {
  await HomeWidget.saveWidgetData(_countKey, value);
  await HomeWidget.updateWidget(
    iOSName: 'DemoWidget',
  );
}

@pragma("vm:entry-point")
FutureOr<void> backgroundCallback(Uri? uri) async {
  debugPrint('backgroundCallback() : $uri');
   if (uri?.host == 'increment') {
    _increment();
  } else if (uri?.host == 'decrement') {
    _decrement();
  }
}


final GlobalKey<NavigatorState> navigatorKey = GlobalKey<NavigatorState>();

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await HomeWidget.setAppGroupId(appGroupID);
  await HomeWidget.registerInteractivityCallback(backgroundCallback);
  runApp(const MyApp());
}



class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  @override
  void initState() {
    super.initState();
    _initHomeWidget();
  }

  void _handleLinkHomeWidgetApp(Uri? uri){
    debugPrint("_handleLinkHomeWidgetApp() : $uri");
    if(uri == null) return;
    WidgetsBinding.instance.addPostFrameCallback((_) {
      Future.delayed(Duration(milliseconds: 100), () {
        navigatorKey.currentState?.pushNamed('/${uri.host}');
      });
    });

  }

  Future<void> _initHomeWidget() async {
    final launchedFromWidget = await HomeWidget.initiallyLaunchedFromHomeWidget();
    _handleLinkHomeWidgetApp(launchedFromWidget);
    HomeWidget.widgetClicked.listen((uri) {
      _handleLinkHomeWidgetApp(uri);
    });
    debugPrint('Launched from widget: $launchedFromWidget');
  }
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      navigatorKey: navigatorKey,
      initialRoute: '/',
      routes: {
        '/': (context) => const HomePage(),
        '/func_scan_qr': (context) => const ScanQRPage(),
        '/func_show_qr': (context) => const ShowQRPage(),
      },
    );
  }
}

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Text("Home Page",style: TextStyle(fontSize: 24,color: Colors.green)),
      ),
    );
  }
}

class ScanQRPage extends StatelessWidget {
  const ScanQRPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Text("Scan QR Page",style: TextStyle(fontSize: 24,color: Colors.green)),
      ),
    );
  }
}
class ShowQRPage extends StatelessWidget {
  const ShowQRPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Text("Show QR Page",style: TextStyle(fontSize: 24,color: Colors.green)),
      ),
    );
  }
}

```

<br>
<br>
<br>

## Conclusion
<br>


*Adding an iOS Home Widget to your Flutter app can greatly improve user engagement by delivering quick, glanceable information right on the home screen. With the combination of Flutter’s flexibility and SwiftUI’s native performance, you can create beautiful, functional widgets that seamlessly integrate with your app experience.*

<br>

*By following this guide, you should now be able to:*

<br>

- *Set up and configure an iOS widget extension in Flutter*
- *Share data between your widget and app using App Groups*
- *Handle user interaction through deep links and background updates*

<br>

*Widgets are a powerful way to keep your app in front of your users daily — so take advantage of them to deliver value instantly.*

<br>

*Full implementation source code :*

*[flutter_ios_home_widget](https://github.com/horlengg/flutter_ios_home_widget)*

<br>
<br>

***Thank you for reading!***

*I hope this tutorial helps you successfully implement your own iOS Home Widget with Flutter. If you have questions, feedback, or are interested in working together on a Flutter or iOS project, feel free to reach out:*
<br>

*[Find me](https://horleng.vercel.app/)*



<br>
<br>
<br>
<br>
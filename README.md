# 前言

之前在公司內部分享Flutter時，發現大部分的人還是著重於想要知道，如何讓公司現有的專案整合flutter。單純寫的flutter app不困難，但要跟原本的app進行整合這就有需要有點步驟了。

在官方文件提到利用[platform channels](https://flutter.dev/docs/development/platform-integration/platform-channels#architecture)來跟原本host app進行溝通。也舉了一個簡單的例子([battery level](https://flutter.dev/docs/development/platform-integration/platform-channels#example))說明如何使用，但[method channel並不是一個typedsafe的溝通方式](https://flutter.dev/docs/development/platform-integration/platform-channels#pigeon)，所以在文件後面又提到如何使用[Pigeon](https://pub.dev/packages/pigeon)這個package來達到type safe的溝通。

因此，在這裡我就是要利用pigeon來進行示範，先用一張手繪圖來說明pigeon在這個整合過程中所扮演的角色是什麼。



# 目標

影片中的**喜愛商品**那頁是用flutter改寫，其他都是原生的畫面，

目標
<u>native page -> push flutter page -> present native page 當然也要可以反向回去</u>

#![1](https://github.com/htaiwan/fluuter_add_to_app_iOS/blob/main/assets/1.gif)

# 宏觀

 ![IMG_0478](https://github.com/htaiwan/fluuter_add_to_app_iOS/blob/main/assets/1.jpg)
1. 定義溝通介面
```dart
class WatchList {
...
}

class Listing {
...
}

@FlutterApi() 
abstract class Api2Flutter {
  void fetchWatchlistings(WatchList list);
}

@HostApi()
abstract class Api2Host {
  void showWatchListing(Listing listing);
  void popViewController();
}
```

2. 利用pigeon來自動生成對應不同平台的實作碼

- OC端(部分實作碼)

```objective-c
@interface HTWatchList : NSObject
...
@end

@interface HTListing : NSObject
...
@end

// 利用這個method將host app所取得得資料傳送到flutter端
@interface HTApi2Flutter : NSObject
- (void)fetchWatchlistings:(HTWatchList*)input completion:(void(^)(NSError* _Nullable))completion;
@end
// 利用protcol來實作flutter希望host app要做的事情
@protocol HTApi2Host
-(void)showWatchListing:(HTListing*)input error:(FlutterError *_Nullable *_Nonnull)error;
-(void)popViewController:(FlutterError *_Nullable *_Nonnull)error;
@end

extern void HTApi2HostSetup(id<FlutterBinaryMessenger> binaryMessenger, id<HTApi2Host> _Nullable api);
```

- Flutter端(部分實作碼)
```dart
class WatchList {
...
}

class Listing {
...
}

// 利用這個abstract class來取得host app所傳送的資料
abstract class Api2Flutter {
  void fetchWatchlistings(WatchList arg);
  ...
}

// 利用這個method通知host app執行所需要的動作
class Api2Host {
   Future<void> showWatchListing(Listing arg) async {
     ..
   }
  
  Future<void> popViewController() async {
    ...
  }
}

```

透過pigeon幫我們自動產生的實作碼，我們只需要定義介面，然後在host app跟flutter的適當地方進行對應的實作就可以達成溝通。


# 步驟

- [ ] Step1: [在現有專案中建立flutter module](https://flutter.dev/docs/development/add-to-app/ios/project-setup#create-a-flutter-module)
  - 將專案(MyApp)跟產生flutter module(my_flutter)的目錄要在同一層。

  ```
  some/path/
  ├── my_flutter/
  │   └── .ios/
  │       └── Flutter/
  │         └── podhelper.rb
  └── MyApp/
    └── Podfile
  
  ```

  ```
  my_flutter/
  ├── .ios/
  │   ├── Runner.xcworkspace
  │   └── Flutter/podhelper.rb
  ├── lib/
  │   └── main.dart
  ├── test/
  └── pubspec.yaml
  ```
  - [利用coocapod來安裝flutter SDK](https://flutter.dev/docs/development/add-to-app/ios/project-setup#option-a---embed-with-cocoapods-and-the-flutter-sdk)
    - 修改目前專案的podfile

- [ ] Step2:  [修改Local Network Privacy Permissions，用來debug](https://flutter.dev/docs/development/add-to-app/ios/project-setup#local-network-privacy-permissions)

- [ ] Step3:  [optional 如果是用apple M1，需要進行的修改](https://flutter.dev/docs/development/add-to-app/ios/project-setup#apple-silicon-arm64-macs)

- [ ] Step4: [安裝pigeon](https://pub.dev/packages/pigeon#usage)

  - 注意pigeon中宣告介面的.dart file的所在位置，不能在原本lib目錄底下

  - 宣告需要的interface(.dart)

  - 執行pigeon產生對應的實作碼(自行更換所要目錄位置跟檔案名稱)

    - 將OC產生的目錄指定到.ios/Flutter/ 下面，之後可以利用cocoapod將產生實作碼安裝到原本的專案底下![截圖 2021-08-18 下午4.38.21](https://github.com/htaiwan/fluuter_add_to_app_iOS/blob/main/assets/2.png)

    ```
    flutter pub run pigeon \
      --input pigeons/message.dart \
      --dart_out lib/pigeon.dart \
      --objc_header_out .ios/Flutter/pigeon.h \
      --objc_source_out .ios/Flutter/pigeon.m \
      --java_out ./android/app/src/main/java/dev/flutter/pigeon/Pigeon.java \
      --java_package "dev.flutter.pigeon"
      --objc_prefix HT
    ```

- [ ] Step4: 修改Flutter module中的podspec

  - 增加 s.source_file![截圖 2021-08-18 下午4.43.53](https://github.com/htaiwan/fluuter_add_to_app_iOS/blob/main/assets/3.png)

- [ ] Step5: 到原本的專案下，執行pod update動作，安裝剛剛的產生的OC實作碼

![截圖 2021-08-18 下午4.48.59](https://github.com/htaiwan/fluuter_add_to_app_iOS/blob/main/assets/4.png)

- [ ] Step6: 實作flutter端的串接部分
  - 實作abstract class，來取得host app所傳來的資料後，所需要的動作
  -  執行相關所需物件的初始化
  - 在對應所需的位置通知host app執行所需要的動作

- [ ] Step7: 實作Host端的串接部分
  - 實作protocol，來取得flutter app所傳來的資料後，所需要的動作
  -  執行相關所需物件的初始化
  - 在對應所需的位置通知flutter app執行所需要的動作


# 細項說明
- 實作flutter端的串接部分

```dart
// 實作abstract class Api2Flutter
class _WatchListingPageState extends State<WatchListingPage>
    implements Api2Flutter {
  late Api2Host _api2Host;
  late WatchList _watchList;

  @override
  void initState() {
    // 執行相關所需物件的初始化
    Api2Flutter.setup(this);
    _api2Host = Api2Host();
    super.initState();
  }
  
  ...
  ...
 
  // 實作abstract class Api2Flutter中宣告的method
  @override
    // 取得host app所傳來的資料
  void fetchWatchlistings(WatchList watchList) {
    setState(() {
      _watchList = watchList;
    });
  }
  ...
  ...
  void _pushListingPage(int index) {
    Listing listng = Listing()
      ..listingId = _watchList.listingIds![index] as String;
       // 呼叫host app, 執行對應所需要的動作
    _api2Host.showWatchListing(listng);
  }
  ...
  ...
  leading: IconButton(
            icon: Image.asset('assets/icons/Icon-Back.png',
                height: 20, width: 20, color: Colors.black),
            onPressed: () {
               // 呼叫host app, 執行對應所需要的動作
              _api2Host.popViewController();
            },
          ),
  
  
```

- 實作Host端的串接部分
```objc
// 繼承FlutterAppDelegate並實作HTApi2Host的method
@interface EAAppDelegate : FlutterAppDelegate <UIApplicationDelegate, HTApi2Host>
  ...
  @property (strong, nonatomic) FlutterEngine *flutterEngine;
	@property (strong, nonatomic) HTApi2Flutter *api2Flutter;
  
  
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  ...
    // 執行相關所需物件的初始化
     self.flutterEngine = [[FlutterEngine alloc] initWithName:@"ECSuperFlutterEngine"];
    [self.flutterEngine run];
    self.api2Flutter = [[HTApi2Flutter alloc] initWithBinaryMessenger:self.flutterEngine.binaryMessenger];
    HTApi2HostSetup(self.flutterEngine.binaryMessenger, self);
    
  ...
}


// 實作HTApi2Host
- (void)showWatchListing:(nonnull HTListing *)input error:(FlutterError * _Nullable __autoreleasing * _Nonnull)error
{
  	....
}

- (void)popViewController:(FlutterError *_Nullable *_Nonnull)error
{
 ....
}

// 在所需的viewController執行下列程式碼
{
  EAAppDelegate *delegate = ((EAAppDelegate *)UIApplication.sharedApplication.delegate);
						// 將在host取得資料，傳送到flutter中
            [delegate.api2Flutter fetchWatchlistings:watchList completion:^(NSError * _Nullable error) {
                if (error != nil) {
                    NSLog(@"%@",error.localizedDescription);
                } else {
                    FlutterViewController *flutterViewController =
                    [[FlutterViewController alloc] initWithEngine:delegate.flutterEngine nibName:nil bundle:nil];
                    self.navigationController.navigationBarHidden = YES;
                    [self.navigationController pushViewController:flutterViewController animated:YES];
                }
            }];
}

```

# 參考資料

- [Add Flutter to existing app](https://flutter.dev/docs/development/add-to-app)

- [Writing custom platform-specific code](https://flutter.dev/docs/development/platform-integration/platform-channels)

- [How big is the Flutter engine?](https://flutter.dev/docs/resources/faq#how-big-is-the-flutter-engine)

- [Pigeon 与 Flutter 混合工程](https://www.bilibili.com/video/BV11z4y1y7ph/)

  


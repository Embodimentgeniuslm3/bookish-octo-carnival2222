---
id: backward-compatibility-fabric-components
title: Fabric Components as Native Components
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import constants from '@site/core/TabsConstants';
import BetaTS from './\_markdown_beta_ts_support.mdx';

:::info
The creation of a backward compatible Fabric Component requires the knowledge of how to create a Fabric Component. To recall these concepts, have a look at this [guide](pillars-fabric-components).

Fabric Components only work when the New Architecture is properly setup. If you already have a library that you want to migrate to the New Architecture, have a look at the [migration guide](../new-architecture-intro) as well.
:::

Creating a backward compatible Fabric Component lets your users continue leverage your library, independently from the architecture they use. The creation of such a module requires a few steps:

1. Configure the library so that dependencies are not installed in the old architecture.
1. Update the codebase so that the New Architecture types are not compiled when not available.
1. Uniform the JavaScript API so that your user code won't need changes.

While the last step is the same for all the platforms, the first two steps are different for iOS and Android.

## Configure the Fabric Component Dependencies

### <a name="dependencies-ios" />iOS

The Apple platform installs Fabric Components using [Cocoapods](https://cocoapods.org) as dependency manager.

Every Fabric Component defines a `podspec` that looks like this:

```ruby
require "json"

package = JSON.parse(File.read(File.join(__dir__, "package.json")))

folly_version = '2021.06.28.00-v2'
folly_compiler_flags = '-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1 -Wno-comma -Wno-shorten-64-to-32'

Pod::Spec.new do |s|
  # Default fields for a valid podspec
  s.name            = "<FC Name>"
  s.version         = package["version"]
  s.summary         = package["description"]
  s.description     = package["description"]
  s.homepage        = package["homepage"]
  s.license         = package["license"]
  s.platforms       = { :ios => "11.0" }
  s.author          = package["author"]
  s.source          = { :git => package["repository"], :tag => "#{s.version}" }

  s.source_files    = "ios/**/*.{h,m,mm,swift}"
  # React Native Core dependency
  s.dependency "React-Core"

  # The following lines are required by the New Architecture.
  s.compiler_flags = folly_compiler_flags + " -DRCT_NEW_ARCH_ENABLED=1"
  s.pod_target_xcconfig    = {
      "HEADER_SEARCH_PATHS" => "\"$(PODS_ROOT)/boost\"",
      "OTHER_CPLUSPLUSFLAGS" => "-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1",
      "CLANG_CXX_LANGUAGE_STANDARD" => "c++17"
  }

  s.dependency "React-RCTFabric"
  s.dependency "React-Codegen"
  s.dependency "RCT-Folly", folly_version
  s.dependency "RCTRequired"
  s.dependency "RCTTypeSafety"
  s.dependency "ReactCommon/turbomodule/core"
end
```

The **goal** is to avoid installing the dependencies when the app is prepared for the Old Architecture.

When we want to install the dependencies, we use the following commands depending on the architecture:

```sh
# For the Old Architecture, we use:
pod install

# For the New Architecture, we use:
RCT_NEW_ARCH_ENABLED=1 pod install
```

Therefore, we can leverage this environment variable in the `podspec` to exclude the settings and the dependencies that are related to the New Architecture:

```diff
+ if ENV['RCT_NEW_ARCH_ENABLED'] == '1' then
    # The following lines are required by the New Architecture.
    s.compiler_flags = folly_compiler_flags + " -DRCT_NEW_ARCH_ENABLED=1"
    # ... other dependencies ...
    s.dependency "ReactCommon/turbomodule/core"
+ end
end
```

This `if` guard prevents the dependencies from being installed when the environment variable is not set.

### Android

:::warning
Add paragraph on Android
:::

## Update the codebase

### iOS

The second step is to instruct Xcode to avoid compiling all the lines using the New Architecture types and files when we are building an app with the Old Architecture.

A Fabric Component requires an header file and an implementation file to add the actual `View` to the module.

For example, the `RNYourComponentView.h` header file could look like this:

```objective-c
#import <React/RCTViewComponentView.h>
#import <UIKit/UIKit.h>

#ifndef NativeComponentExampleComponentView_h
#define NativeComponentExampleComponentView_h

NS_ASSUME_NONNULL_BEGIN

@interface RNYourComponentView : RCTViewComponentView
@end

NS_ASSUME_NONNULL_END

#endif /* NativeComponentExampleComponentView_h */
```

The implementation `RNYourComponentView.mm` file, instead, could look like this:

```objective-c
#import "RNYourComponentView.h"

// <react/renderer imports>

#import "RCTFabricComponentsPlugins.h"

using namespace facebook::react;

@interface RNYourComponentView () <RCTColoredViewViewProtocol>

@end

@implementation RNYourComponentView {
    UIView * _view;
}

+ (ComponentDescriptorProvider)componentDescriptorProvider
{
    // ... return the descriptor ...
}

- (instancetype)initWithFrame:(CGRect)frame
{
  // ... initialize the object ...
}

- (void)updateProps:(Props::Shared const &)props oldProps:(Props::Shared const &)oldProps
{
  // ... set up the props ...

  [super updateProps:props oldProps:oldProps];
}

Class<RCTComponentViewProtocol> YourComponentViewCls(void)
{
  return RNYourComponentView.class;
}

@end
```

To make sure that Xcode skips these files, we can wrap **both** of them in some `#ifdef RCT_NEW_ARCH_ENABLED` compilation pragma. For example, the header file could change as follows:

```diff
+ #ifdef RCT_NEW_ARCH_ENABLED
#import <React/RCTViewComponentView.h>
#import <UIKit/UIKit.h>

// ... rest of the header file ...

#endif /* NativeComponentExampleComponentView_h */
+ #endif
```

The same two lines should be added in the implementation file, as first and last lines.

The above snippet uses the same `RCT_NEW_ARCH_ENABLED` flag used in the previous [section](#dependencies-ios). When this flag is not set, Xcode skips the lines within the `#ifdef` during compilation and it does not include them into the compiled binary. The compiled binary will have a the `RNYourComponentView.o` object but it will be an empty object.

### Android

:::warning
Add paragraph on Android
:::

## Unify the JavaScript specs

The last step makes sure that the JavaScript behaves transparently to chosen architecture.

For a Fabric Component, the source of truth is the `<YourModule>NativeComponent.js` (or `.ts`) spec file. The app accesses the spec file like this:

```ts
import YourComponent from 'your-component/src/index';
```

The **goal** is to conditionally `export` from the `index` file the proper object, given the architecture chosen by the user. We can achieve this with a code that looks like this:

<Tabs groupId="fabric-component-backward-compatibility"
      defaultValue={constants.defaultFabricComponentSpecLanguage}
      values={constants.fabricComponentSpecLanguages}>
<TabItem value="Flow">

```ts
// @flow
import { requireNativeComponent } from 'react-native';

const isFabricEnabled = global.nativeFabricUIManager != null;

const yourComponent = isFabricEnabled
  ? require('./YourComponentNativeComponent').default
  : requireNativeComponent('YourComponent');

export default yourComponent;
```

</TabItem>
<TabItem value="TypeScript">

```ts
import requireNativeComponent from 'react-native/Libraries/ReactNative/requireNativeComponent';

const isFabricEnabled = global.nativeFabricUIManager != null;

const yourComponent = isFabricEnabled
  ? require('./YourComponentNativeComponent').default
  : requireNativeComponent('YourComponent');

export default yourComponent;
```

</TabItem>
</Tabs>

Whether you are using Flow or TypeScript for your specs, we understand which architecture is running by checking if the `global.nativeFabricUIManager` object has been set or not.

:::caution
The `global.nativeFabricUIManager` API may change in the future for a function that encapsulate this check.
:::

- If that object is `null`, the app has not enabled the Fabric feature. It's running on the Old Architecture, and the fallback is to use the default [`Native Components` implementation](../native-components-intro).
- If that object is set, the app is running with Fabric enabled and it should use the `<YourComponent>NativeComponent` spec to access the Fabric Component.

<BetaTS />

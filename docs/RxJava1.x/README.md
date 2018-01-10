# ReactiveNetwork

[![Android Arsenal](https://img.shields.io/badge/Android%20Arsenal-ReactiveNetwork-brightgreen.svg?style=flat)](https://android-arsenal.com/details/1/2290)

ReactiveNetwork is an Android library listening **network connection state** and **Internet connectivity** with RxJava Observables. It's a successor of [Network Events](https://github.com/pwittchen/NetworkEvents) library rewritten with Reactive Programming approach. Library supports both new and legacy network monitoring strategies. Min sdk version = 9.

| Current Branch | Branch  | Artifact Id | Build Status  | Coverage | Maven Central |
|:--------------:|:-------:|:-----------:|:-------------:|:--------:|:-------------:|
| :ballot_box_with_check: | [`RxJava1.x`](https://github.com/pwittchen/ReactiveNetwork/tree/RxJava1.x) | `reactivenetwork` | [![Build Status for RxJava1.x](https://travis-ci.org/pwittchen/ReactiveNetwork.svg?branch=RxJava1.x)](https://travis-ci.org/pwittchen/ReactiveNetwork) | [![codecov](https://codecov.io/gh/pwittchen/ReactiveNetwork/branch/RxJava1.x/graph/badge.svg)](https://codecov.io/gh/pwittchen/ReactiveNetwork/branch/RxJava1.x) | ![Maven Central](https://img.shields.io/maven-central/v/com.github.pwittchen/reactivenetwork.svg?style=flat) |
| | [`RxJava2.x`](https://github.com/pwittchen/ReactiveNetwork/tree/RxJava2.x) | `reactivenetwork-rx2` | [![Build Status for RxJava2.x](https://travis-ci.org/pwittchen/ReactiveNetwork.svg?branch=RxJava2.x)](https://travis-ci.org/pwittchen/ReactiveNetwork) | [![codecov](https://codecov.io/gh/pwittchen/ReactiveNetwork/branch/RxJava2.x/graph/badge.svg)](https://codecov.io/gh/pwittchen/ReactiveNetwork/branch/RxJava2.x) | ![Maven Central](https://img.shields.io/maven-central/v/com.github.pwittchen/reactivenetwork-rx2.svg?style=flat) |

Contents
--------

- [Usage](#usage)
  - [Observing network connectivity](#observing-network-connectivity)
    - [Connectivity class](#connectivity-class)
    - [Network Observing Strategies](#network-observing-strategies)
  - [Observing Internet connectivity](#observing-internet-connectivity)
    - [Customization of observing Internet connectivity](#customization-of-observing-internet-connectivity)
    - [Internet Observing Strategies](#internet-observing-strategies)
    - [Custom host](#custom-host)
  - [ProGuard configuration](#proguard-configuration)
- [Examples](#examples)
- [Download](#download)
- [Tests](#tests)
- [Code style](#code-style)
- [Static code analysis](#static-code-analysis)
- [Who is using this library?](#who-is-using-this-library)
- [Getting help](#getting-help)
  - [Tutorials](#tutorials)
- [Caveats](#caveats)
- [Changelog](#changelog)
- [JavaDoc](#javadoc)
- [Releasing](#releasing)
- [Contributors](#contributors)
- [References](#references)
  - [Mentions](#mentions)
- [License](#license)

Usage
-----

**Please note**: Due to memory leak in `WifiManager` reported
in [issue 43945](https://code.google.com/p/android/issues/detail?id=43945) in Android issue tracker
it's recommended to use Application Context instead of Activity Context.

### Observing network connectivity

We can observe `Connectivity` with `observeNetworkConnectivity(context)` method in the following way:

```java
ReactiveNetwork.observeNetworkConnectivity(context)
    .subscribeOn(Schedulers.io())
    ... // anything else what you can do with RxJava
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Connectivity>() {
      @Override public void call(Connectivity connectivity) {
        // do something with connectivity
        // you can call connectivity.getState();
        // connectivity.getType(); or connectivity.toString();
      }
    });
```

When `Connectivity` changes, subscriber will be notified. `Connectivity` can change its state or type.

We can react on a concrete state, states, type or types changes with the `filter(...)` method from RxJava, `hasState(NetworkInfo.State... states)` and `hasType(int... types)` methods located in `ConnectivityPredicate` class.

```java
ReactiveNetwork.observeNetworkConnectivity(context)
    .subscribeOn(Schedulers.io())
    .filter(ConnectivityPredicate.hasState(NetworkInfo.State.CONNECTED))
    .filter(ConnectivityPredicate.hasType(ConnectivityManager.TYPE_WIFI))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Connectivity>() {
      @Override public void call(Connectivity connectivity) {
        // do something
      }
    });
```

`observeNetworkConnectivity(context)` checks only connectivity with the network (not Internet) as it's based on `BroadcastReceiver` for API 20 and lower and uses `NetworkCallback` for API 21 and higher.
 Concrete WiFi or mobile network may be connected to the Internet (and usually is), but it doesn't have to.

You can also use method:

```java
Observable<Connectivity> observeNetworkConnectivity(Context context, NetworkObservingStrategy strategy)
```

This method allows you to apply your own network observing strategy and is used by the library under the hood to determine appropriate strategy depending on the version of Android system.

#### Connectivity class

`Connectivity` class is used by `observeNetworkConnectivity(context)` and `observeNetworkConnectivity(context, networkObservingStrategy)` methods. It has the following API:

```java
Connectivity create()
Connectivity create(Context context)

NetworkInfo.State getState()
NetworkInfo.DetailedState getDetailedState()
int getType()
int getSubType()
boolean isAvailable()
boolean isFailover()
boolean isRoaming()
String getTypeName()
String getSubTypeName()
String getReason()
String getExtraInfo()

class Builder
```

#### Network Observing Strategies

Right now, we have the following strategies for different Android versions:

- `LollipopNetworkObservingStrategy`
- `MarshmallowNetworkObservingStrategy`
- `PreLollipopNetworkObservingStrategy`

All of them implements `NetworkObservingStrategy` interface. Concrete strategy is chosen automatically
depending on the Android version installed on the device.
With `observeNetworkConnectivity(context, strategy)` method we can use one of these strategies explicitly.

### Observing Internet connectivity

We can observe connectivity with the Internet in the following way:

```java
ReactiveNetwork.observeInternetConnectivity()
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<Boolean>() {
          @Override public void call(Boolean isConnectedToInternet) {
            // do something with isConnectedToInternet value
          }
        });
```

An `Observable` will return `true` to the subscription if device is connected to the Internet and `false` if not.

Internet connectivity will be checked _as soon as possible_.

**Please note**: This method is less efficient than `observeNetworkConnectivity(context)` method, because it opens socket connection with remote host (default is www.google.com) every two seconds with two seconds of timeout and consumes data transfer. Use this method if you really need it. Optionally, you can unsubscribe subcription right after you get notification that Internet is available and do the work you want in order to decrease network calls.

#### Customization of observing Internet connectivity

Methods in this section should be used if they are really needed due to specific use cases.

If you want to adjust checking Internet connectivity to your needs (custom ping host, port, ping interval, timeout, strategy, etc.),
you can use one of the following methods:

```java
Observable<Boolean> observeInternetConnectivity(int interval, String host, int port, int timeout)
Observable<Boolean> observeInternetConnectivity(int initialIntervalInMs, int intervalInMs, String host, int port, int timeout)
Observable<Boolean> observeInternetConnectivity(final int initialIntervalInMs, final int intervalInMs, final String host, final int port, final int timeoutInMs, final ErrorHandler errorHandler)
Observable<Boolean> observeInternetConnectivity(final InternetObservingStrategy strategy, final int initialIntervalInMs, final int intervalInMs, final String host, final int port, final int timeoutInMs, final ErrorHandler errorHandler)
Observable<Boolean> observeInternetConnectivity(final InternetObservingStrategy strategy)
Observable<Boolean> observeInternetConnectivity(final InternetObservingStrategy strategy, final String host)
```

These methods are created to allow the users to fully customize the library and give them more control.

For more details check JavaDoc at: http://pwittchen.github.io/ReactiveNetwork/javadoc/RxJava1.x

#### Internet Observing Strategies

Right now, we have the following strategies for observing Internet connectivity:

- `SocketInternetObservingStrategy` - monitors Internet connectivity via opening socket connection with the remote host
- `WalledGardenInternetObservingStrategy` - opens connection with a remote host and respects countries in the Walled Garden (e.g. China)

All of these strategies implements `NetworkObservingStrategy` interface.
Default strategy used right now is `WalledGardenInternetObservingStrategy`,
but with `observeInternetConnectivity(strategy)` method we can use one of these strategies explicitly.

#### Custom host

If you want to ping custom host during checking Internet connectivity, it's recommended to use `SocketInternetObservingStrategy`.
You can do it as follows:

```java
ReactiveNetwork.observeInternetConnectivity(new SocketInternetObservingStrategy(), "www.yourhost.com")
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Boolean>() {
        @Override public void call(Boolean isConnectedToHost) {
            // do something with isConnectedToHost value
        }
     });
```

### ProGuard configuration

```
-dontwarn com.github.pwittchen.reactivenetwork.library.ReactiveNetwork
-dontwarn io.reactivex.functions.Function
-dontwarn rx.internal.util.**
-dontwarn sun.misc.Unsafe
```

Examples
--------

Exemplary application is located in `app` directory of this repository.

If you want to know, how to use this library with Kotlin, check `app-kotlin` directory.

Download
--------

You can depend on the library through Maven:

```xml
<dependency>
    <groupId>com.github.pwittchen</groupId>
    <artifactId>reactivenetwork</artifactId>
    <version>0.12.2</version>
</dependency>
```

or through Gradle:

```groovy
dependencies {
  compile 'com.github.pwittchen:reactivenetwork:0.12.2'
}
```

Tests
-----

Tests are available in `library/src/test/java/` directory and can be executed on JVM without any emulator or Android device from Android Studio or CLI with the following command:

```
./gradlew test
```

To generate test coverage report, run the following command:

```
./gradlew test jacocoTestReport
```

Code style
----------

Code style used in the project is called `SquareAndroid` from Java Code Styles repository by Square available at: https://github.com/square/java-code-styles.

Static code analysis
--------------------

Static code analysis runs Checkstyle, FindBugs, PMD and Lint. It can be executed with command:

 ```
 ./gradlew check
 ```

Reports from analysis are generated in `library/build/reports/` directory.

Who is using this library?
--------------------------
- [PAT Track - realtime Tracker for the public transit in Pittsburgh, PA](https://play.google.com/store/apps/details?id=rectangledbmi.com.pittsburghrealtimetracker)
- [Eero - Home WiFi System](https://play.google.com/store/apps/details?id=com.eero.android)
- [ACN Android Framework](https://github.com/ugurcany/ACN-Android-Framework)
- [Spatial Connect Android SDK](https://github.com/boundlessgeo/spatialconnect-android-sdk)
- [Qiscus SDK for Android](https://github.com/qiscus/qiscus-sdk-android)
- [Internet Radio](https://play.google.com/store/apps/details?id=com.stc.radio.player)
- [Tachiyomi](https://github.com/inorichi/tachiyomi)
- [Actinium - V2Ray client for Android](https://github.com/V2Ray-Android/Actinium)
- [Project Bass - Android app](http://projectbass.org/)
- and more...

Are you using this library in your app and want to be listed here? Send me a Pull Request or an e-mail to piotr@wittchen.biz.pl

Getting help
------------

Do you need help related to using or configuring this library? 

You can do the following things:
- [Ask the question on StackOverflow](http://stackoverflow.com/questions/ask?tags=reactivenetwork)
- [Create new GitHub issue](https://github.com/pwittchen/ReactiveNetwork/issues/new)

Don't worry. Someone should help you with solving your problems.

### Tutorials

If you speak Spanish (Español), check out this tutorial: [ReactiveNetwork - Como funciona y como se integra en una app](https://www.youtube.com/watch?v=H7xGmQaKPsI) made by [Video Tutorials Android](https://www.youtube.com/channel/UC2q5P9JVoA6N8mE622gRP7w).

Caveats
-------

Since version **0.4.0**, functionality releated to **observing WiFi Access Points** and **WiFi signal strength (level)** is removed in favor of [ReactiveWiFi](https://github.com/pwittchen/ReactiveWiFi) library.
If you want to use this functionality, check [**ReactiveWiFi**](https://github.com/pwittchen/ReactiveWiFi) project.

Changelog
---------

See [CHANGELOG.md](https://github.com/pwittchen/ReactiveNetwork/blob/RxJava1.x/CHANGELOG.md) file.

JavaDoc
-------

JavaDoc is available at: http://pwittchen.github.io/ReactiveNetwork/javadoc/RxJava1.x

Releasing
---------

See [RELEASING.md](https://github.com/pwittchen/ReactiveNetwork/blob/RxJava1.x/RELEASING.md) file.

Contributors
------------

- [Piotr Wittchen](https://github.com/pwittchen) - project lead
- [Tushar Acharya](https://github.com/tushar-acharya)
- [Timothy Kist](https://github.com/Kisty)
- [@dilongl](https://github.com/dilongl)
- [@llp](https://github.com/llp)
- [Adam Gabryś](https://github.com/agabrys)
- [@lion4ik](https://github.com/lion4ik)
- [@futtetennista](https://github.com/futtetennista)
- [Manu Sridharan](https://github.com/msridhar)
- [Alexander Perfilyev](https://github.com/aperfilyev)

References
----------
- [Android Documentation - Detect network changes, then change app behavior](https://developer.android.com/develop/quality-guidelines/building-for-billions-connectivity.html#network-behavior)
- [Android Documentation - Provide onboarding experiences for users' network choices](https://developer.android.com/develop/quality-guidelines/building-for-billions-data-cost.html#configurablenetwork-onboarding)
- [Android Documentation - Managing Network Usage](https://developer.android.com/training/basics/network-ops/managing.html)
- [RxJava](https://github.com/ReactiveX/RxJava)
- [DroidCon Poland 2017 presentation slides - Is your app really connected?](https://speakerdeck.com/pwittchen/is-your-app-really-connected)

### Mentions
- [Android Weekly #166](http://androidweekly.net/issues/issue-166)
- [Android Weekly #289](http://androidweekly.net/issues/issue-289)
- [Android Weekly China #44](http://www.androidweekly.cn/android-dev-weekly-issue44/)

License
-------

    Copyright 2016 Piotr Wittchen

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
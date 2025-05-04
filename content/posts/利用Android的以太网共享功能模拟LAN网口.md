---
tags:
  - android/framework
date: 2025-05-04
title: 利用Android的以太网共享功能模拟LAN网口
---

本文的目标是在一个安装了 Android 系统的设备上模拟软路由。这个设备拥有三个网口，最终的目标是希望模拟出一个 WAN 网口和两个 LAN 网口。目标系统为 Android 14。

Android 14 提供了以太网共享网络的功能，借助这个功能，我们可以将一个网口模拟成 LAN 口，但可惜的是，这个功能只对一个网口生效。我们先简要分析一下其实现原理，然后再给出在有三个网口的机器上模拟出两个 LAN 接口的技术解决方案。

## 代码分析

### Settings

我们从系统已有的功能出发。网络共享的功能可以在原生 Settings 中找到相应开关，因此也可以找到相应的开启以太网共享的 SDK 接口：

```java
// packages/apps/Settings/src/com/android/settings/network/tether/TetherSettings.java

@Override
public boolean onPreferenceTreeClick(Preference preference) {
    if (preference == mUsbTether) {
    } else if (preference == mBluetoothTether) {
    } else if (preference == mEthernetTether) {
        if (mEthernetTether.isChecked()) {
            startTethering(TETHERING_ETHERNET);
        } else {
            mCm.stopTethering(TETHERING_ETHERNET);
        }
    }

    return super.onPreferenceTreeClick(preference);
}

private void startTethering(int choice) {
    if (choice == TETHERING_BLUETOOTH) {
        // Turn on Bluetooth first.
        BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
        if (adapter.getState() == BluetoothAdapter.STATE_OFF) {
            mBluetoothEnableForTether = true;
            adapter.enable();
            mBluetoothTether.setEnabled(false);
            return;
        }
    }

    mCm.startTethering(choice, true, mStartTetheringCallback, mHandler);
}
```

当我们点击界面上的 Switch 时，`onPreferenceTreeClick()`方法被回调，Settings 根据点击的按钮不同，将会使用不同的 Choice 来调用`startTethering`方法。

`TetherSettings#startTethering()`方法调用了`ConnectivityManager#startTethering()`来启用某一种网络共享，网络共享的具体通道（蓝牙、USB 、以太网）则由参数 choice 来决定。

### ConnectivityManager

```java
// packages/modules/Connectivity/framework/src/android/net/ConnectivityManager.java

@SystemApi
@Deprecated
@RequiresPermission(android.Manifest.permission.TETHER_PRIVILEGED)
public void startTethering(int type, boolean showProvisioningUi,
                           final OnStartTetheringCallback callback, Handler handler) {
    final Executor executor = new Executor() {
        @Override
        public void execute(Runnable command) {
        }
    };

    final StartTetheringCallback tetheringCallback = new StartTetheringCallback() {
        @Override
        public void onTetheringStarted() {
            callback.onTetheringStarted();
        }

        @Override
        public void onTetheringFailed(final int error) {
            callback.onTetheringFailed();
        }
    };

    final TetheringRequest request = new TetheringRequest.Builder(type)
        .setShouldShowEntitlementUi(showProvisioningUi).build();

    getTetheringManager().startTethering(request, executor, tetheringCallback);
}

private TetheringManager getTetheringManager() {
    synchronized (mTetheringEventCallbacks) {
        if (mTetheringManager == null) {
            mTetheringManager = mContext.getSystemService(TetheringManager.class);
        }
        return mTetheringManager;
    }
}
```

`ConnectivityManager#startTethering()`这个 API 已经被标记为废弃了。它的实现的大意为，将回调和网络共享`type`的信息进一步封装起来，并传递给`TetheringManager`进行处理。`getTetheringManager()`的实现告诉我们 TetheringManager 实际上也对应一个 System Server，为了节约篇幅，我们跳过 Manager 直接关注其背后实现 Server 的代码：

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/TetheringService.java

@Override
public void startTethering(TetheringRequestParcel request, String callerPkg,
                           String callingAttributionTag, IIntResultListener listener) {
    if (checkAndNotifyCommonError(callerPkg, callingAttributionTag,
        request.exemptFromEntitlementCheck /* onlyAllowPrivileged */, listener)) {
        return;
    }

    mTethering.startTethering(request, callerPkg, listener);
}
```

`TetheringService` 在完成参数校验后，委托给`Tethering`类来处理共享的具体逻辑。

### Tethering

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/Tethering.java

void startTethering(final TetheringRequestParcel request, final String callerPkg,
                    final IIntResultListener listener) {
    mHandler.post(() -> {
        enableTetheringInternal(request.tetheringType, true /* enabled */, listener);
    });
}

private void enableTetheringInternal(int type, boolean enable,
                                     final IIntResultListener listener) {
    int result = TETHER_ERROR_NO_ERROR;
    switch (type) {
        case TETHERING_WIFI:
            result = setWifiTethering(enable);
            break;
        case TETHERING_USB:
            result = setUsbTethering(enable);
            break;
        case TETHERING_BLUETOOTH:
            setBluetoothTethering(enable, listener);
            break;
        case TETHERING_NCM:
            result = setNcmTethering(enable);
            break;
        case TETHERING_ETHERNET:
            result = setEthernetTethering(enable);
            break;
        default:
            Log.w(TAG, "Invalid tether type.");
            result = TETHER_ERROR_UNKNOWN_TYPE;
    }

    // The result of Bluetooth tethering will be sent by #setBluetoothTethering.
    if (type != TETHERING_BLUETOOTH) {
        sendTetherResult(listener, result, type);
    }
}

private int setEthernetTethering(final boolean enable) {
    final EthernetManager em = (EthernetManager) mContext.getSystemService(
        Context.ETHERNET_SERVICE);
    if (enable) {
        if (mEthernetCallback != null) {
            Log.d(TAG, "Ethernet tethering already started");
            return TETHER_ERROR_NO_ERROR;
        }

        mEthernetCallback = new EthernetCallback();
        mEthernetIfaceRequest = em.requestTetheredInterface(mExecutor, mEthernetCallback);
    } else {
        stopEthernetTethering();
    }
    return TETHER_ERROR_NO_ERROR;
}
```

`Tethering`类可以处理多种不同通道的网络共享，我们重点关注的以太网共享功能，其实现关键位于`setEthernetTethering()`这个方法。

这个方法大意为，首先获取`EthernetManager`，然后调用`requestTetheredInterface`来尝试获取一个可用的以太网接口，`Tethering`将会在这个接口上开启一系列关键的服务，如 DHCP 等。`requestTetheredInterface`的结果将通过`EthernetCallback`这个回调接口来回传。我们先观察一下`EthernetCallback`的实现：

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/Tethering.java

private class EthernetCallback implements EthernetManager.TetheredInterfaceCallback {
    @Override
    public void onAvailable(String iface) {
        if (this != mEthernetCallback) {
            // Ethernet callback arrived after Ethernet tethering stopped. Ignore.
            return;
        }
        enableIpServing(TETHERING_ETHERNET, iface, getRequestedState(TETHERING_ETHERNET));
        mConfiguredEthernetIface = iface;
    }

    @Override
    public void onUnavailable() {
        if (this != mEthernetCallback) {
            // onAvailable called after stopping Ethernet tethering.
            return;
        }
        stopEthernetTethering();
    }
}
```

`Tethering`获得这个接口的信息后，用这个接口的名称来调用`enableIpServing`。`enableIpServing`将会开启绑定在这个接口上的`IpServer`，这个类提供了 DHCP 等关键服务，对实现网络共享至关重要。

### EthernetManager

我们跳过`EthernetManager#requestTetheredInterface()`，直接找到真正的实现`EthernetServiceImpl#requestTetheredInterface()`

```java
// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetServiceImpl.java

@Override
public void requestTetheredInterface(ITetheredInterfaceCallback callback) {
    Objects.requireNonNull(callback, "callback must not be null");
    PermissionUtils.enforceNetworkStackPermissionOr(
        mContext,
        android.Manifest.permission.NETWORK_SETTINGS
    );
    mTracker.requestTetheredInterface(callback);
}

// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java

public void requestTetheredInterface(ITetheredInterfaceCallback callback) {
    mHandler.post(() -> {
        if (!mTetheredInterfaceRequests.register(callback)) {
            // Remote process has already died
            return;
        }
        if (mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
            if (mTetheredInterfaceWasAvailable) {
                notifyTetheredInterfaceAvailable(callback, mTetheringInterface);
            }
            return;
        }

        setTetheringInterfaceMode(INTERFACE_MODE_SERVER);
    });
}
```

`EthernetServiceImpl`又将具体的实现委托给了`EthernetTracker`来执行。而我们会注意到，`EthernetTracker`将直接尝试返回类变量`mTetheringInterface`，这个变量将作为以太网的共享接口。

`EthernetTracker`对 mTetheringInterface 这个特殊的共享接口进行了许多特殊操作，另外两个与之相关的类变量是`mTetheredInterfaceWasAvailable`和`mTetheringInterfaceMode`

```java
// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java

// The first interface discovered is set as the mTetheringInterface. It is the interface that is
// returned when a tethered interface is requested; until then, it remains in client mode. Its
// current mode is reflected in mTetheringInterfaceMode.
private String mTetheringInterface;
private int mTetheringInterfaceMode = INTERFACE_MODE_CLIENT;
// Tracks whether clients were notified that the tethered interface is available
private boolean mTetheredInterfaceWasAvailable = false;
```

mTetheringInterface 将在`maybeTrackInterface`这个方法内被选举出来：

```java
// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java

private void maybeTrackInterface(String iface) {
    if (!isValidEthernetInterface(iface)) {
        return;
    }

    // If we don't already track this interface, and if this interface matches
    // our regex, start tracking it.
    if (mFactory.hasInterface(iface) || iface.equals(mTetheringInterface)) {
        if (DBG) Log.w(TAG, "Ignoring already-tracked interface " + iface);
        return;
    }
    if (DBG) Log.i(TAG, "maybeTrackInterface: " + iface);

    // Do not use an interface for tethering if it has configured NetworkCapabilities.
    if (mTetheringInterface == null &&
        !mNetworkCapabilities.containsKey(iface)) {
        mTetheringInterface = iface;
    }

    addInterface(iface);

    broadcastInterfaceStateChange(iface);
}
```

简单来说，第一个没有配置`NetworkCapabilities`的以太网接口将被作为共享接口。

最后，我们关注一下`isValidEthernetInterface`的实现，这个方法是`EthernetTracker`判断一个接口是不是以太网接口的通用方式。

```java
// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java

private final String mIfaceMatch;

EthernetTracker() {
    final ConnectivityResources resources = new ConnectivityResources(context);

    mIfaceMatch = resources.get().getString(
        com.android.connectivity.resources.R.string.config_ethernet_iface_regex);
}

private boolean isValidEthernetInterface(String iface) {
    return iface.matches(mIfaceMatch) || isValidTestInterface(iface);
}
```

据此我们可以得出结论，确认一个接口是否是以太网接口的方式是用正则表达式对接口的名字做匹配。相关的字符串资源文件位于下方：

```xml
// packages/modules/Connectivity/service/ServiceConnectivityResources/res/values/config.xml

<string translatable="false" name="config_ethernet_iface_regex">eth\\d</string>
```

显然，默认情况下，只有以 eth 开头，并且后面接一位数字的接口才会被认为是以太网接口。

**如何获得`EthernetTracker`的信息？**

通过 `dumpsys ethernet`。

信息的含义请参考`EthernetTracker`的`dump`方法。

### 随机子网

无论我们是使用 Wifi 热点还是以太网网络共享，设备虚拟出来的子网是随机的，生成这些随机子网的逻辑藏在`IpServer`内：

```java
// packages/modules/Connectivity/Tethering/src/android/net/ip/IpServer.java

private LinkAddress requestIpv4Address(final int scope, final boolean useLastAddress) {
    if (mStaticIpv4ServerAddr != null) return mStaticIpv4ServerAddr;

    if (shouldNotConfigureBluetoothInterface()) return new LinkAddress(BLUETOOTH_IFACE_ADDR);

    return mPrivateAddressCoordinator.requestDownstreamAddress(this, scope, useLastAddress);
}
```

`IpServer`调用了`PrivateAddressCoordinator`的相关方法来生成随机的地址。

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/PrivateAddressCoordinator.java

public LinkAddress requestDownstreamAddress(final IpServer ipServer, final int scope,
                                            boolean useLastAddress) {
    if (mConfig.shouldEnableWifiP2pDedicatedIp()
        && ipServer.interfaceType() == TETHERING_WIFI_P2P) {
        return new LinkAddress(LEGACY_WIFI_P2P_IFACE_ADDRESS);
    }

    final AddressKey addrKey = new AddressKey(ipServer.interfaceType(), scope);
    // This ensures that tethering isn't started on 2 different interfaces with the same type.
    // Once tethering could support multiple interface with the same type,
    // TetheringSoftApCallback would need to handle it among others.
    final LinkAddress cachedAddress = mCachedAddresses.get(addrKey);
    if (useLastAddress && cachedAddress != null
        && !isConflictWithUpstream(asIpPrefix(cachedAddress))) {
        mDownstreams.add(ipServer);
        return cachedAddress;
    }

    for (IpPrefix prefixRange : mTetheringPrefixes) {
        final LinkAddress newAddress = chooseDownstreamAddress(prefixRange);
        if (newAddress != null) {
            mDownstreams.add(ipServer);
            mCachedAddresses.put(addrKey, newAddress);
            return newAddress;
        }
    }

    // No available address.
    return null;
}

private static class AddressKey {
    private final int mTetheringType;
    private final int mScope;

    private AddressKey(int type, int scope) {
        mTetheringType = type;
        mScope = scope;
    }

    @Override
    public boolean equals(@Nullable Object obj) {
        if (!(obj instanceof AddressKey)) return false;
        final AddressKey other = (AddressKey) obj;

        return mTetheringType == other.mTetheringType && mScope == other.mScope;
    }
}
```

`mCachedAddresses`是`AddressKey`到`LinkAddress`之间的映射，从第44行`AddressKey`的`equals`方法实现来看，同一个类型的网络共享一般会命中同一个 cache，因此，一般来说同一个类型的网络分享在关闭又重新开启后，使用的依然是之前计算好的随机子网。

### 网络分享内的设备为什么无法互通

阻碍来自 iptables 的转发规则，网络共享的转发规则会专门存放在 tetherctrl_FORWARD 表中（这个表被FORWARD表进一步引用），它的默认规则直接 DROP 掉了局域网内的流量，这直接导致通过网络共享功能连接到 Android 系统的各个设备，相互之间无法成功通信。

我们可以通过以下命令来查看 iptables 规则：

```
iptables -nvL
```

如果拥有 su 权限，我们可以使用下述命令来删除 tetherctrl_FORWARD 表中的规则：

```
iptables -D tetherctrl_FORWARD -j DROP
```

但是这是一种治标不治本的方式，每次网络变动（比如插拔网线）， tetherctrl_FORWARD 表会被重置。因此我们必须找到相关的代码来修改其行为。

system/netd/server/TetherController.cpp 中包含了与以太网网络共享相关的 iptables 修改逻辑，我们这里指出其中两个关键的方法：

```cpp
// system/netd/server/TetherController.cpp
int TetherController::setDefaults() {
    std::string v4Cmd = StringPrintf(
        "*filter\n"
        ":%s -\n"
        "-A %s -j DROP\n"
        "COMMIT\n"
        "*nat\n"
        ":%s -\n"
        "COMMIT\n", LOCAL_FORWARD, LOCAL_FORWARD, LOCAL_NAT_POSTROUTING);


    int res = iptablesRestoreFunction(V4, v4Cmd, nullptr);
    if (res < 0) {
        return res;
    }

    return 0;
}
```

根据实际效果而言，`setDefaults()`方法中填写的规则将会在 WAN 口尚未接入网线但开启了网络共享功能时生效。

```cpp
int TetherController::setForwardRules(bool add, const char *intIface, const char *extIface) {
    // ...
    if (add) {
        v4.push_back(StringPrintf("-D %s -j DROP", LOCAL_FORWARD));
        v4.push_back(StringPrintf("-A %s -j DROP", LOCAL_FORWARD));
    }
    // ...
}
```

`setForwardRules()`方法将会在开启了网络共享功能后， WAN 口接入网线时生效。我们可以看到实现重新添加了`DROP`规则。

修改 iptables 规则需要小心，一旦上述规则有任何的执行错误，网络共享功能就会被停止。比如说，我们强行在命令行中影响了 iptables 规则，就有可能导致网络共享功能在网线接入或移除时被停止。

我们举一个例子，比如我们手动删除了 DROP 规则，根据上述代码，我们知道，网线接入时`TetherController`也会尝试删除这个规则，由于我们提前删除了，所以上述代码的第 4 行规则在执行时会出错，这个错误就会导致网络共享功能被停止。总之，网线接入/移除时网络共享功能莫名停止，多半是 iptables 规则不对。

## 解决方案

根据上面的代码调研，我们发现， SDK 在多处实现中都假设以太网网络共享只会分享一个接口，原生系统无法达到获得两个类似于路由器上的 LAN 口的效果，我们必须加以改造。

改造的思路大致有两种：

- 创建一个以太网网桥，并将 eth1 和 eth2 连接到网桥上，网络共享则统一使用这个网桥。
- 修改 SDK 实现，让 eth1 和 eth2 同时开启`IpServer`，每个接口将分配到不同的子网上。

### 网桥

我们这里省略如何编译 Android 系统上可用的网桥控制命令 brctl ，重点关注如何利用它创建网桥。我们假设 brctl 将安装到 /system/bin 下。首先，我们在 /system/bin/ 下编写一个脚本 init.brctl.sh

```shell
#!/bin/sh

/system/bin/brctl addbr br0
/system/bin/brctl addif br0 eth2
/system/bin/brctl addif br0 eth1
/system/bin/brctl stp br0 off
ifconfig br0 up
ifconfig eth1 up
ifconfig eth2 up
```

然后，我们使用 initrc 机制让这个脚本在开机时得以执行：

```
service brctl_init /system/bin/init.brctl.sh
    user root
    group root system
    disabled
    oneshot

on early-init
    start brctl_init
```

完成这一步后，我们就可以在开机后得到一个新的接口`br0`，它是我们虚拟出来的以太网接口。接下来，我们需要稍微修改 framework 代码，使得这个接口可以被识别为以太网接口，并保证它一定被选择作为以太网共享接口。

首先，我们修改 xml 资源文件：

```xml
// packages/modules/Connectivity/service/ServiceConnectivityResources/res/values/config.xml

<string translatable="false" name="config_ethernet_iface_regex">eth\\d|br\\d</string>
```

然后，我们修改`EthernetTracker`，让它固定选中 br0 接口用来以太网共享。

```java
// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java

private void maybeTrackInterface(String iface) {
    // ...
    // Do not use an interface for tethering if it has configured NetworkCapabilities.
    if (mTetheringInterface == null && "br0".equals(iface)) {
        mTetheringInterface = iface;
    }
    // ...
}
```

为了让 Settings 也能够把网桥识别为以太网共享，从而可以正常通过开关开启和关闭共享，需要修改`TetherSettings`内的正则表达式，如下：

```java
// packages/apps/Settings/src/com/android/settings/network/tether/TetherSettings.java
public class TetherSettings extends RestrictedSettingsFragment
        implements DataSaverBackend.Listener {
    
    private String mEthernetRegex;
    
    @Override
    public void onCreate(Bundle icicle) {
        //mEthernetRegex = "eth\\d";
        mEthernetRegex = "eth\\d|br\\d";
    }
}
```

### 同时开启两个网口的共享

与第一种思路相比，这个思路的实现相对来说复杂一些，但涉及到的文件很少，只需要修改`Tethering`和`EthernetTracker`即可。

对于`Tethering`的修改集中在将变量`mConfiguredEthernetIface`替换为集合`mConfiguredEthernetIfaceSet`，并让`Tethering`开启两个`IpServer`

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/Tethering.java

private Set<String> mConfiguredEthernetIfaceSet = new HashSet<>();

private void stopEthernetTethering() {
    // old
    //if (mConfiguredEthernetIface != null) {
    //    ensureIpServerStopped(mConfiguredEthernetIface);
    //    mConfiguredEthernetIface = null;
    //}
    // new
    for (String iface: mConfiguredEthernetIfaceSet) {
        ensureIpServerStopped(iface);
    }
    // end
    mConfiguredEthernetIfaceSet.clear();
    if (mEthernetCallback != null) {
        mEthernetIfaceRequest.release();
        mEthernetCallback = null;
        mEthernetIfaceRequest = null;
    }
}

private class EthernetCallback implements EthernetManager.TetheredInterfaceCallback {
    @Override
    public void onAvailable(String iface) {
        if (this != mEthernetCallback) {
            // Ethernet callback arrived after Ethernet tethering stopped. Ignore.
            return;
        }
        enableIpServing(TETHERING_ETHERNET, iface, getRequestedState(TETHERING_ETHERNET));
        // old
        //mConfiguredEthernetIface = iface;
        // new
        mConfiguredEthernetIfaceSet.add(iface);
        // end
    }
    // ...
}
```

对`EthernetTracker`的修改会比较多，但思路仍然是用集合代替单一的接口变量。我们声明一个新的内部类用于同时保存接口名、状态和可用性，并打算用 HashMap 来平替之前的变量。

```java
// packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java

private final Map<String, TetheringInterfaceState> mTetheringInterfaceStates = new HashMap<>();

private static class TetheringInterfaceState {
    final String mTetheringInterface;
    int mTetheringInterfaceMode = INTERFACE_MODE_CLIENT;
    boolean mTetheredInterfaceWasAvailable = false;

    public TetheringInterfaceState(String tetheringInterface) {
        mTetheringInterface = tetheringInterface;
    }
}
```

接下来就是非常琐碎的变量平替修改，我们这里仅记录 diff

```
diff --git a/packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java b/packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java
index 9707adf7248..dfe66d98f98 100644
--- a/packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java
+++ b/packages/modules/Connectivity/service-t/src/com/android/server/ethernet/EthernetTracker.java
@@ -61,8 +61,10 @@ import com.android.server.connectivity.ConnectivityResources;
 import java.io.FileDescriptor;
 import java.net.InetAddress;
 import java.util.ArrayList;
+import java.util.HashMap;
 import java.util.Iterator;
 import java.util.List;
+import java.util.Map;
 import java.util.Objects;
 import java.util.concurrent.ConcurrentHashMap;
 
@@ -134,6 +136,8 @@ public class EthernetTracker {
     // Tracks whether clients were notified that the tethered interface is available
     private boolean mTetheredInterfaceWasAvailable = false;
 
+    private final Map<String, TetheringInterfaceState> mTetheringInterfaceStates = new HashMap<>();
+
     private int mEthernetState = ETHERNET_STATE_ENABLED;
 
     private class TetheredInterfaceRequestList extends
@@ -165,7 +169,8 @@ public class EthernetTracker {
         }
 
         private void onNewLink(String ifname, boolean linkUp) {
-            if (!mFactory.hasInterface(ifname) && !ifname.equals(mTetheringInterface)) {
+            //if (!mFactory.hasInterface(ifname) && !ifname.equals(mTetheringInterface)) {
+            if (!mFactory.hasInterface(ifname) && !mTetheringInterfaceStates.containsKey(ifname)) {
                 Log.i(TAG, "onInterfaceAdded, iface: " + ifname);
                 maybeTrackInterface(ifname);
             }
@@ -447,8 +452,14 @@ public class EthernetTracker {
             for (String iface : getClientModeInterfaces(canUseRestrictedNetworks)) {
                 unicastInterfaceStateChange(listener, iface);
             }
-            if (mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
-                unicastInterfaceStateChange(listener, mTetheringInterface);
+            //if (mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
+            //    unicastInterfaceStateChange(listener, mTetheringInterface);
+            //}
+            for (TetheringInterfaceState state: mTetheringInterfaceStates.values()) {
+                if (state.mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
+                    unicastInterfaceStateChange(listener, state.mTetheringInterface);
+                    return;
+                }
             }
 
             unicastEthernetStateChange(listener, mEthernetState);
@@ -495,11 +506,19 @@ public class EthernetTracker {
                 // Remote process has already died
                 return;
             }
-            if (mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
-                if (mTetheredInterfaceWasAvailable) {
-                    notifyTetheredInterfaceAvailable(callback, mTetheringInterface);
+            //if (mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
+            //    if (mTetheredInterfaceWasAvailable) {
+            //        notifyTetheredInterfaceAvailable(callback, mTetheringInterface);
+            //    }
+            //    return;
+            //}
+            for (TetheringInterfaceState state: mTetheringInterfaceStates.values()) {
+                if (state.mTetheringInterfaceMode == INTERFACE_MODE_SERVER) {
+                    if (state.mTetheredInterfaceWasAvailable) {
+                        notifyTetheredInterfaceAvailable(callback, state.mTetheringInterface);
+                    }
+                    return;
                 }
-                return;
             }
 
             setTetheringInterfaceMode(INTERFACE_MODE_SERVER);
@@ -531,19 +550,39 @@ public class EthernetTracker {
 
     private void maybeUntetherInterface() {
         if (mTetheredInterfaceRequests.getRegisteredCallbackCount() > 0) return;
-        if (mTetheringInterfaceMode == INTERFACE_MODE_CLIENT) return;
-        setTetheringInterfaceMode(INTERFACE_MODE_CLIENT);
+        //if (mTetheringInterfaceMode == INTERFACE_MODE_CLIENT) return;
+        //setTetheringInterfaceMode(INTERFACE_MODE_CLIENT);
+        for (TetheringInterfaceState state: mTetheringInterfaceStates.values()) {
+            if (state.mTetheringInterfaceMode != INTERFACE_MODE_CLIENT) {
+                setTetheringInterfaceMode(state, INTERFACE_MODE_CLIENT);
+            }
+        }
     }
 
     private void setTetheringInterfaceMode(int mode) {
         Log.d(TAG, "Setting tethering interface mode to " + mode);
-        mTetheringInterfaceMode = mode;
-        if (mTetheringInterface != null) {
-            removeInterface(mTetheringInterface);
-            addInterface(mTetheringInterface);
+        //mTetheringInterfaceMode = mode;
+        //if (mTetheringInterface != null) {
+        //    removeInterface(mTetheringInterface);
+        //    addInterface(mTetheringInterface);
+        //    // when this broadcast is sent, any calls to notifyTetheredInterfaceAvailable or
+        //    // notifyTetheredInterfaceUnavailable have already happened
+        //    broadcastInterfaceStateChange(mTetheringInterface);
+        //}
+        for (TetheringInterfaceState state: mTetheringInterfaceStates.values()) {
+            setTetheringInterfaceMode(state, mode);
+        }
+    }
+
+    private void setTetheringInterfaceMode(TetheringInterfaceState state, int mode) {
+        Log.d(TAG, "Setting tethering interface mode to " + mode);
+        state.mTetheringInterfaceMode = mode;
+        if (state.mTetheringInterface != null) {
+            removeInterface(state.mTetheringInterface);
+            addInterface(state.mTetheringInterface);
             // when this broadcast is sent, any calls to notifyTetheredInterfaceAvailable or
             // notifyTetheredInterfaceUnavailable have already happened
-            broadcastInterfaceStateChange(mTetheringInterface);
+            broadcastInterfaceStateChange(state.mTetheringInterface);
         }
     }
 
@@ -572,8 +611,11 @@ public class EthernetTracker {
     }
 
     private int getInterfaceMode(final String iface) {
-        if (iface.equals(mTetheringInterface)) {
-            return mTetheringInterfaceMode;
+        //if (iface.equals(mTetheringInterface)) {
+        //    return mTetheringInterfaceMode;
+        //}
+        if (mTetheringInterfaceStates.containsKey(iface)) {
+            return mTetheringInterfaceStates.get(iface).mTetheringInterfaceMode;
         }
         return INTERFACE_MODE_CLIENT;
     }
@@ -585,9 +627,10 @@ public class EthernetTracker {
 
     private void stopTrackingInterface(String iface) {
         removeInterface(iface);
-        if (iface.equals(mTetheringInterface)) {
-            mTetheringInterface = null;
-        }
+        //if (iface.equals(mTetheringInterface)) {
+        //    mTetheringInterface = null;
+        //}
+        mTetheringInterfaceStates.remove(iface);
         broadcastInterfaceStateChange(iface);
     }
 
@@ -669,7 +712,9 @@ public class EthernetTracker {
     }
 
     private void maybeUpdateServerModeInterfaceState(String iface, boolean available) {
-        if (available == mTetheredInterfaceWasAvailable || !iface.equals(mTetheringInterface)) {
+        //if (available == mTetheredInterfaceWasAvailable || !iface.equals(mTetheringInterface)) {
+        TetheringInterfaceState state = mTetheringInterfaceStates.get(iface);
+        if (state == null || state.mTetheredInterfaceWasAvailable == available) {
             return;
         }
 
@@ -686,7 +731,8 @@ public class EthernetTracker {
             }
         }
         mTetheredInterfaceRequests.finishBroadcast();
-        mTetheredInterfaceWasAvailable = available;
+        //mTetheredInterfaceWasAvailable = available;
+        state.mTetheredInterfaceWasAvailable = available;
     }
 
     private void maybeTrackInterface(String iface) {
@@ -696,7 +742,8 @@ public class EthernetTracker {
 
         // If we don't already track this interface, and if this interface matches
         // our regex, start tracking it.
-        if (mFactory.hasInterface(iface) || iface.equals(mTetheringInterface)) {
+        //if (mFactory.hasInterface(iface) || iface.equals(mTetheringInterface)) {
+        if (mFactory.hasInterface(iface) || mTetheringInterfaceStates.containsKey(iface)) {
             if (DBG) Log.w(TAG, "Ignoring already-tracked interface " + iface);
             return;
         }
@@ -704,8 +751,10 @@ public class EthernetTracker {
 
         // Do not use an interface for tethering if it has configured NetworkCapabilities.
+         //if (mTetheringInterface == null && !mNetworkCapabilities.containsKey(iface)) {
+        //    mTetheringInterface = iface;
+        //}
+        if ("eth1".equals(iface) || "eth2".equals(iface)) {
+            mTetheringInterfaceStates.put(iface, new TetheringInterfaceState(iface));
         }
 
         addInterface(iface);
@@ -998,8 +1047,12 @@ public class EthernetTracker {
             pw.println("Ethernet State: "
                     + (mEthernetState == ETHERNET_STATE_ENABLED ? "enabled" : "disabled"));
             pw.println("Ethernet interface name filter: " + mIfaceMatch);
-            pw.println("Interface used for tethering: " + mTetheringInterface);
-            pw.println("Tethering interface mode: " + mTetheringInterfaceMode);
+            //pw.println("Interface used for tethering: " + mTetheringInterface);
+            //pw.println("Tethering interface mode: " + mTetheringInterfaceMode);
+            for (TetheringInterfaceState state: mTetheringInterfaceStates.values()) {
+                pw.println("\tTethering interface: " + state.mTetheringInterface);
+                pw.println("\tTethering mode: " + state.mTetheringInterfaceMode);
+            }
             pw.println("Tethered interface requests: "
                     + mTetheredInterfaceRequests.getRegisteredCallbackCount());
             pw.println("Listeners: " + mListeners.getRegisteredCallbackCount());
@@ -1038,4 +1091,15 @@ public class EthernetTracker {
             mTransport = tokens.length > 3 ? tokens[3] : null;
         }
     }
+
+    private static class TetheringInterfaceState {
+        final String mTetheringInterface;
+        int mTetheringInterfaceMode = INTERFACE_MODE_CLIENT;
+        boolean mTetheredInterfaceWasAvailable = false;
+
+        public TetheringInterfaceState(String tetheringInterface) {
+            mTetheringInterface = tetheringInterface;
+        }
+    }
+
 }

```

### 开机后自动开启网络共享

重启后，Settings 并不会保存网络共享的状态，因此我们需要保证下述代码在开机后会执行一遍。

```java
ConnectivityManager connectivityManager = getSystemService(ConnectivityManager.class);

connectivityManager.startTethering(
    TETHERING_ETHERNET,
    true,
    new ConnectivityManager.OnStartTetheringCallback() {},
    new Handler(Looper.getMainLooper())
);
```

### iptables

对于默认规则，我们强行让其变为`ACCEPT`

```cpp
// system/netd/server/TetherController.cpp
int TetherController::setDefaults() {
    std::string v4Cmd = StringPrintf(
        "*filter\n"
        ":%s -\n"
        "-A %s -j ACCEPT\n"
        "COMMIT\n"
        "*nat\n"
        ":%s -\n"
        "COMMIT\n", LOCAL_FORWARD, LOCAL_FORWARD, LOCAL_NAT_POSTROUTING);

    int res = iptablesRestoreFunction(V4, v4Cmd, nullptr);
    if (res < 0) {
        return res;
    }

    return 0;
}
```

接下来，我们不允许其重置 DROP 规则，包括移除和添加。（在没有 DROP 规则的情况下尝试移除也会报错）

```cpp
// system/netd/server/TetherController.cpp
int TetherController::setForwardRules(bool add, const char *intIface, const char *extIface) {
    if (add) {
        //v4.push_back(StringPrintf("-D %s -j DROP", LOCAL_FORWARD));
        //v4.push_back(StringPrintf("-A %s -j DROP", LOCAL_FORWARD));
    }
}
```

### 固定共享子网地址

修改`PrivateAddressCoordinator`这个类即可。我们可以通过`IpServer`获得当前共享的网络接口的名称，根据这个名称我们提前拦截随机分配的逻辑，下面展示双网口共享的修改方式：

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/PrivateAddressCoordinator.java
public LinkAddress requestDownstreamAddress(final IpServer ipServer, final int scope,
                                            boolean useLastAddress) {
    if (mConfig.shouldEnableWifiP2pDedicatedIp()
        && ipServer.interfaceType() == TETHERING_WIFI_P2P) {
        return new LinkAddress(LEGACY_WIFI_P2P_IFACE_ADDRESS);
    }
    
    String iface = ipServer.interfaceName();
    if ("eth1".equals(iface)) {
        mDownstreams.add(ipServer);
        return getLinkAddress(new IpPrefix("192.168.51.0/24"), 2);
    } else if ("eth2".equals(iface)) {
        mDownstreams.add(ipServer);
        return getLinkAddress(new IpPrefix("192.168.52.0/24"), 2);
    }
    // ...
}
```

单网桥的修改方式类似

```java
// packages/modules/Connectivity/Tethering/src/com/android/networkstack/tethering/PrivateAddressCoordinator.java
public LinkAddress requestDownstreamAddress(final IpServer ipServer, final int scope,
                                            boolean useLastAddress) {
    if (mConfig.shouldEnableWifiP2pDedicatedIp()
        && ipServer.interfaceType() == TETHERING_WIFI_P2P) {
        return new LinkAddress(LEGACY_WIFI_P2P_IFACE_ADDRESS);
    }     
    String iface = ipServer.interfaceName();
    if ("br0".equals(iface)) {
        mDownstreams.add(ipServer);
        return getLinkAddress(new IpPrefix("192.168.51.0/24"), 2);
    }
    // ...
}
```


> 参考资料：
>
> [Android14 以太网共享功能](https://blog.csdn.net/wenzhi20102321/article/details/141533109)
>
> [Android下打通RNDIS(USB网络共享)与WIFI热点](https://www.liyanfeng.com/post/139.html)
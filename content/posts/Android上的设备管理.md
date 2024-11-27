---
tags:
  - android
title: Android上的设备管理
date: 2024-06-18
---


> https://source.android.com/docs/devices/admin
>
> https://developer.android.com/work/guide
>
> https://developers.google.com/android/work/play/emm-api/prov-devices

本文提到的“设备管理”并不面向终端用户，而是一种支持企业管理、监控和保护所有用于工作的移动设备的实际需要而产生的一系列框架。

**关键术语**

- Device Policy Controller：设备政策控制器，简称 DPC
- Device Owner：设备所有者，是 Android 设备管理框架运行的一种模式，有时简称为 DO
- Profile Owner：工作配置所有者，是 Android 设备管理框架运行的一种模式，有时简称为 PO

## 设备管理方案

一套完整的设备管理方案通常包含了两个方面的支持。其一是安装在设备端的设备政策控制器（Device Policy Controller）应用，其二是云端的控制台。企业客户在设备上安装 DPC 应用并与云端关联注册后，可以通过云端将设备管理政策下发至 DPC 应用，DPC 应用最终执行之。

## DPC 应用的运行模式

Android 提供了以下三种设备管理模式，其中一种已在 9.0 后废弃，无法再使用。

- Device Owner（设备所有者）

  DPC 应用成为设备所有者，用于管理整个设备。这是**权限最高的模式**，一旦开启，除非 DPC 应用主动注销自己，否则只能通过恢复出厂设置来重置。

- Profile Owner（工作配置所有者）

  DPC 应用被设置为 Profile Owner。启用这个模式后，相当于系统开启一个额外的工作分区，工作分区和个人分区的内容互不影响，DPC 应用只有管理工作分区的权限。该设备依然可以用于个人用途。此类设备管理主要用于个人和组织同时持有的设备。权限比 Device Owner 低。

- Device Admin，此模式在 9.0 中被废弃并在 10.0 中完全移除相应功能。

## 成为Device Owner

前提条件：

需要成为 Device Owner 的应用必须包含继承`android.app.admin.DeviceAdminReceiver`的组件。假设自定义的组件名为`TestReceiver`。

- 通过二维码，必须在开机向导结束前设置。

  > Google 没有具体提供采用这种方式激活成为 Device Owner 的详细步骤，据测试，使用下述 intent 可以启动`ManagedProvisioning`框架
  > ```shell
  > am start -a android.app.action.PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE --ecn android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME "packageName/.TestReceiver"
  > ```

- 通过 adb shell

  使用如下命令：

  ```
  adb shell dpm set-device-owner packageName/.TestReceiver
  ```

- 通过 NFC

## 通过二维码注册为 Device Owner

根据[Google Play EMM API](https://developers.google.com/android/work/play/emm-api/prov-devices)，我们可以总结出通过二维码注册 Device Owner 的主要流程。

### 准备二维码

二维码的内容应当是一个采用 UTF-8 编码的 JSON 字符串，内部包含一些参数。这些参数包括：

**必须**

`android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME`

**如果未安装 DPC，则必须提供下载路径和校验码**

`android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM`

`android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION`

更多参数请参考上述资料。

一个可用的二维码内容的示例：

```json
{
"android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME":
"com.emm.android/com.emm.android.DeviceAdminReceiver",

"android.app.extra.PROVISIONING_DEVICE_ADMIN_SIGNATURE_CHECKSUM":
"gJD2YwtOiWJHkSMkkIfLRlj-quNqG1fb6v100QmzM9w=",

"android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION":
"https://path.to/dpc.apk",
    "android.app.extra.PROVISIONING_SKIP_ENCRYPTION": false,
    "android.app.extra.PROVISIONING_WIFI_SSID": "GuestNetwork",
    "android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE": {
        "dpc_company_name": "Acme Inc.",
        "emm_server_url": "https://server.emm.biz:8787",
        "another_custom_dpc_key": "dpc_custom_value"
    }
}
```

### 扫码

扫码模块必须是开机向导的一部分，开机向导完成后就无法通过二维码的方式设置 Device Owner。

扫码模块在解析完毕 JSON 内所有参数后，发送一个 Intent。Intent 的 action 应当是`DevicePolicyManager#ACTION_PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE`，extra则携带解析 JSON 获得的参数。

### ManagedProvisioning

相应的 Intent 将会启动`ManagedProvisioning`应用。如果 DPC 应用尚未安装到设备上，那么`ManagedProvisioning`会负责下载并安装 DPC 应用，前提是提供了下载路径和校验码这两个关键参数。

当应用已安装到设备上，`ManagedProvisioning`接下来的行为与 Android 版本相关。我们只讨论 Android 10 及以上的平台版本。

在 Android 10 及以上的平台版本，`ManagedProvisioning`将会交由 DPC 应用来决定（实际上是要求用户决定）成为 Device Owner 还是 Profile Owner。`ManagedProvisioning`将会进行一次`startActivityForResult`，ACTION 为`DevicePolicyManager#ACTION_GET_PROVISIONING_MODE`。

### DPC应用

一般来说收到 Intent 后，DPC 应用应该展示一个界面，让用户选择 DPC 应用成为 Device Owner 或 Profile Owner ，结果回传的 key 可以参阅`DevicePolicyManager#ACTION_GET_PROVISIONING_MODE`的文档。

### 最后的步骤

`ManagedProvisioning`将会完成使能 Device Owner 或 Profile Owner 的最后步骤。

设备配置后，DPC 会收到以下这些广播：

- DeviceAdminReceiver#ACTION_READY_FOR_USER_INITIALIZATION
- DeviceAdminReceiver#ACTION_PROFILE_PROVISIONING_COMPLETE

并且 DPC 的 `DevicePolicyManager#ACTION_ADMIN_POLICY_COMPLIANCE` 处理程序会被启动。

-----------------------

**低平台版本的行为**

在 Android 9 及以下的平台版本，`ManagedProvisioning`将会自动判断到底将 DPC 应用设置为 Device Owner 还是 Profile Owner。判断的依据是`ProvisioningParams#provisioningAction`

```java
// packages/apps/ManagedProvisioning/src/com/android/managedprovisioning/preprovisioning/PreProvisioningController.java
// Android 9
public boolean isProfileOwnerProvisioning() {
    // mParams类型为ProvisioningParams
    return mUtils.isProfileOwnerAction(mParams.provisioningAction);
}
// packages/apps/ManagedProvisioning/src/com/android/managedprovisioning/common/Utils.java
// Android 9
public final boolean isProfileOwnerAction(String action) {
    return ACTION_PROVISION_MANAGED_PROFILE.equals(action) || ACTION_PROVISION_MANAGED_USER.equals(action);
}

public final boolean isDeviceOwnerAction(String action) {
    return ACTION_PROVISION_MANAGED_DEVICE.equals(action) || ACTION_PROVISION_MANAGED_SHAREABLE_DEVICE.equals(action);
}
```

这个 ACTION 一般来说和启动它的 Intent 的 ACTION 一致，但有几个特殊的 ACTION 会被转换，`ACTION_PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE`就是其中一例：

```java
// packages/apps/ManagedProvisioning/src/com/android/managedprovisioning/common/Utils.java
// Android 9
public String mapIntentToDpmAction(Intent intent) throws IllegalProvisioningArgumentException {
    // ...
    switch (intent.getAction()) {
        case ACTION_PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE:
            dpmProvisioningAction = ACTION_PROVISION_MANAGED_DEVICE;
            break;
    }
    // ...
}
```

因此，启动 DPC 应用这一步将被省略，并且也不会发送ACTION 为`DevicePolicyManager#ACTION_ADMIN_POLICY_COMPLIANCE` 的 Intent。

## 流程梳理

我们的主要分析目标是`ManagedProvisioning`这个应用，它是 AOSP 的一部分，与设备管理功能强关联。

### 开始

流程的第一步是发送一个 Intent 以启动`ManagedProvisioning`，ACTION 有四种选择，这些ACTION 均可在`DevicePolicyManager`中被找到：

- android.app.action.PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE
- android.app.action.PROVISION_FINANCED_DEVICE
- android.app.action.PROVISION_MANAGED_PROFILE
- android.app.action.PROVISION_MANAGED_DEVICE

这四个 ACTION 到底有什么区别呢？主要的区别在于，前两个 ACTION 是给系统应用使用的；后两个 ACTION 是给 DPC 应用使用的，但在 API 31 （Android 12）及以后，`PROVISION_MANAGED_DEVICE`被弃用了。DPC 应用现在只能主动成为 Profile Owner，`ManagedProvisioning`虽然还会响应`PROVISION_MANAGED_DEVICE` ，但是添加了一个条件分支阻止进一步使能为 DO。

携带的 extra 参数的要求可以参考相应 ACTION 的文档。一般来说都需要携带`android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME`这个参数来指明组件。

测试脚本：

```shell
am start -a android.app.action.PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE --ecn android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME "com.afwsamples.testdpc/.DeviceAdminReceiver"
```

### PreProvisioningActivity

从清单中很容易能够发现，`PreProvisioningActivity`响应了`ACTION_PROVISION_MANAGED_PROFILE`和`ACTION_PROVISION_MANAGED_DEVICE`：

```xml
<activity
    android:name=".preprovisioning.PreProvisioningActivity"
    android:excludeFromRecents="true"
    android:immersive="true"
    android:launchMode="singleTop"
    android:exported="true"
    android:theme="@style/SudThemeGlifV3.DayNight">
    <intent-filter android:priority="10">
        <action android:name="android.app.action.PROVISION_MANAGED_PROFILE" />
        <action android:name="android.app.action.PROVISION_MANAGED_DEVICE" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

但其实不太容易发现这个 Activity 同时还响应了前面所提到的其他 ACTION，清单中定义了一个不太常见的别名结构，根据注释来看，这个别名添加了一个 system 应用才能申请的权限，主要是为了限制这两个 ACTION 只给特权应用使用

```xml
<!--
    Trusted app entry for device owner provisioning, protected by a permission so only
    privileged app can trigger this.
-->
<activity-alias
    android:name=".PreProvisioningActivityViaTrustedApp"
    android:targetActivity=".preprovisioning.PreProvisioningActivity"
    android:permission="android.permission.DISPATCH_PROVISIONING_MESSAGE"
    android:exported="true">
    <intent-filter android:priority="10">
        <action android:name="android.app.action.PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE"/>
        <action android:name="android.app.action.PROVISION_FINANCED_DEVICE" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity-alias>

```

`PreProvisioningActivity`对 Intent 参数处理的初始化逻辑大部分位于`PreProvisioningActivityController#initiateProvisioning`

```java
public void initiateProvisioning(Intent intent, String callingPackage) {
    // 尝试将Intent中携带的参数解析为ProvisioningParams
    // 解析通过ViewModel转接到Parser, 因此下文会从ViewModel中获取解析好的对象
    if (!tryParseParameters(intent)) {
        return;
    }

    ProvisioningParams params = mViewModel.getParams();
    // 检查是否存在不允许恢复出厂设置的保护
    if (!checkFactoryResetProtection(params, callingPackage)) {
        return;
    }

    // 检查CallerPackage是否合法，这里阻止了非DPC应用发送PROVISION_MANAGED_DEVICE和PROVISION_MANAGED_PROFILE这两个ACTION
    if (!verifyActionAndCaller(intent, callingPackage)) {
        return;
    }

    // Check whether provisioning is allowed for the current action. This check needs to happen
    // before any actions that might affect the state of the device.
    // Note that checkDevicePolicyPreconditions takes care of calling
    // showProvisioningErrorAndClose. So we only need to show the factory reset dialog (if
    // applicable) and return.
    // 重要函数, 实际上调用了DevicePolicyManager#checkProvisioningPreCondition
    // 检查是否真的可以启用设备管理功能
    if (!checkDevicePolicyPreconditions()) {
        return;
    }

    // Android 12以后额外添加的分支，彻底阻止PROVISION_MANAGED_DEVICE这个ACTION
    if (!isIntentActionValid(intent.getAction())) {
        ProvisionLogger.loge(
                ACTION_PROVISION_MANAGED_DEVICE + " is no longer a supported intent action.");
        mUi.abortProvisioning();
        return;
    }
    // ...

    if (mUtils.checkAdminIntegratedFlowPreconditions(params)) {
        // 进入这个分支意味着不是NFC方式，也不是PROVISION_FINANCED_DEVICE这个ACTION
        // ACTION为PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE时会进入这个分支
        if (mUtils.shouldShowOwnershipDisclaimerScreen(params)) {
            // 跳转到免责声明屏幕
            mUi.showOwnershipDisclaimerScreen(params);
        } else {
            // 这个方法有两个跳转方向, 如果DPC应用还没被下载安装,那么跳转到等待屏幕
            // 如果DPC应用已经安装, 那么会直接向其发送Intent以确定成为DO还是PO
            mUi.prepareAdminIntegratedFlow(params);
        }
        mViewModel.onAdminIntegratedFlowInitiated();
    } else if (mUtils.isFinancedDeviceAction(params.provisioningAction)) {
        // ACTION为PROVISION_FINANCED_DEVICE时会进入这个分支
        mUi.prepareFinancedDeviceFlow(params);
    } else if (params.isNfc) {
        // ...
    } else if (isProfileOwnerProvisioning()) {
        startManagedProfileFlow();
    } else if (isDpcTriggeredManagedDeviceProvisioning(intent)) {
        // TODO(b/175678720): Fail provisioning if flow started by PROVISION_MANAGED_DEVICE
        startManagedDeviceFlow();
    }
}
```

### 如何使能为 Owner

由于使能为 DO 或 PO 的路径之间存在不少可以共享的代码节点（比如下载和安装 DPC 应用），因此，`ManagedProvisioning`将每一个独立的节点抽象为 Task，而一个逻辑链路上需要执行的 Task 以及其先后顺序由一个抽象的 Controller 来控制，这些 Controller 又由一个静态工厂来根据`ProvisioningParams`控制初始化。

```java
// ProvisioningControllerFactory.java
public class ProvisioningControllerFactory {
    public AbstractProvisioningController createProvisioningController(
            Context context,
            ProvisioningParams params,
            ProvisioningControllerCallback callback) {
        // ACTION为PROVISION_MANAGED_DEVICE_FROM_TRUSTED_SOURCE时会进入这个分支
        if (mUtils.isDeviceOwnerAction(params.provisioningAction)) {
            int deviceOwner = mUtils.isHeadlessSystemUserMode()
                        ? UserHandle.USER_SYSTEM : UserHandle.myUserId();
            ProvisionLogger.logi("Setting device owner on user: " + deviceOwner);
            return DeviceOwnerProvisioningController.createInstance(
                    context,
                    params,
                    deviceOwner,
                    callback,
                    mUtils);
        // ACTION为PROVISION_FINANCED_DEVICE时会进入这个分支
        } else if (mUtils.isFinancedDeviceAction(params.provisioningAction)) {
            return FinancedDeviceProvisioningController.createInstance(
                    context,
                    params,
                    UserHandle.myUserId(),
                    callback);
        } else {
            return ProfileOwnerProvisioningController.createInstance(
                    context,
                    params,
                    UserHandle.myUserId(),
                    callback);
        }
    }
}
```

对于`DeviceOwnerProvisioningController`来说，负责使能为 DO 的 Task 为`ProvisionFullyManagedDeviceTask`：

```java
public class ProvisionFullyManagedDeviceTask extends AbstractProvisioningTask {
    // ...
        @Override
    public void run(@UserIdInt int userId) {
        startTaskTimer();
        FullyManagedDeviceProvisioningParams params;

        try {
            params = buildManagedDeviceProvisioningParams(userId);
        } catch (IllegalProvisioningArgumentException e) {
            ProvisionLogger.loge("Failure provisioning managed device, failed to "
                    + "infer the device admin component name", e);
            error(/* resultCode= */ 0);
            return;
        }

        try {
            // mDpm是DevicePolicyManager
            mDpm.provisionFullyManagedDevice(params);
        } catch (Exception e) {
            // Catching all Exceptions to allow Managed Provisioning to handle any failure
            // during provisioning properly and perform any necessary cleanup.
            ProvisionLogger.loge("Failure provisioning device owner", e);
            error(/* resultCode= */ 0);
            return;
        }

        // ...
    }
    
    
    private FullyManagedDeviceProvisioningParams buildManagedDeviceProvisioningParams(
            @UserIdInt int userId)
            throws IllegalProvisioningArgumentException {
        ComponentName adminComponent = mProvisioningParams.inferDeviceAdminComponentName(
                mUtils, mContext, userId);
        return new FullyManagedDeviceProvisioningParams.Builder(
                adminComponent,
                mContext.getResources().getString(
                        R.string.default_owned_device_username))
                .setLeaveAllSystemAppsEnabled(
                        mProvisioningParams.leaveAllSystemAppsEnabled)
                .setTimeZone(mProvisioningParams.timeZone)
                .setLocalTime(mProvisioningParams.localTime)
                .setLocale(mProvisioningParams.locale)
                // The device owner can grant sensors permissions if it has not opted
                // out of controlling them.
                .setDeviceOwnerCanGrantSensorsPermissions(
                        !mProvisioningParams.deviceOwnerPermissionGrantOptOut)
                .build();
    }
    // ...

}
```

对于`FinancedDeviceProvisioningController`，负责使能为 DO 的 Task 为`SetDeviceOwnerPolicyTask`：

```java
public class SetDeviceOwnerPolicyTask extends AbstractProvisioningTask {
    // ...
    
    @Override
    public void run(int userId) {
        boolean success;
        try {
            // 取出需要成为DO的组件名，一般来自我们Intent传入的参数，
            // 如果只传了包名但不明确是哪个组件, 那么这个函数会尝试找一个合适的组件
            ComponentName adminComponent =
                    mProvisioningParams.inferDeviceAdminComponentName(mUtils, mContext, userId);
            
            String adminPackage = adminComponent.getPackageName();
            
            // 确保DPC应用已经被启用
            enableDevicePolicyApp(adminPackage);
            // 先将相应组件注册为ActiveAdmin
            setActiveAdmin(adminComponent, userId);
            
            // 将DPC注册为DO
            success = setDeviceOwner(adminComponent,
                    mContext.getResources().getString(R.string.default_owned_device_username),
                    userId);

            // 如果ACTION的类型和分期付款相关，会修改DeviceOwner的类型
            if (success && mUtils.isFinancedDeviceAction(mProvisioningParams.provisioningAction)) {
                mDevicePolicyManager.setDeviceOwnerType(adminComponent, DEVICE_OWNER_TYPE_FINANCED);
            }
        } catch (Exception e) {
            ProvisionLogger.loge("Failure setting device owner", e);
            error(0);
            return;
        }

        // ...
    }
    
    private boolean setDeviceOwner(ComponentName component, String owner, int userId) {
        ProvisionLogger.logd("Setting " + component + " as device owner of user "
                             + userId);
        if (!component.equals(mUtils.getCurrentDeviceOwnerComponentName(
            mDevicePolicyManager))) {
            // 关键函数
            return mDevicePolicyManager.setDeviceOwner(component, owner, userId);
        }
        return true;
    }
    
    // ...

}
```
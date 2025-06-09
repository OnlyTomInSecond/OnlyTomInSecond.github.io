---
title: android 充电电池灯亮度调节修复
---

手持 `Redmi K30S Ultra`，该设备带一个呼吸灯，充电时或者通知来时可亮起。刷了 `Lineageos` 后，呼吸灯亮度不能在设置里调节，于是着手想着修复一下这个问题。

首先，灯能亮，能控制灯呼吸的频率，只是仅仅不能调节亮度，先验证一下内核层面有没有问题。
```sh
echo 100 > /sys/class/led/white/brightness
```
执行后发现灯的亮度能发生变化，说明内核层面没有问题。

按照android的设计，内核层上是HAL，实现对内核的接口转接。于是开始debug HAL。查看设备树的代码知道，`device/xiaomi/apollon` 里包含了 `device/xiaomi/sm8250-common`，而 `sm8250` 全系使用的 `Light HAL` 是 `hardware/xiaomi` 里的
```sh
grep -irn "light" device/xiaomi/sm8250-common
...
./kona.mk:290:# Lights
./kona.mk:292:    android.hardware.light-service.xiaomi
```

进 `hardware/xiaomi` 里继续搜 `android.hardware.light-service.xiaomi`，发现被定义在 `hardware/xiaomi/aidl/light/Android.bp` 里，同时也知道了是 `aidl hal` ，而且这个版本的 `light hal` 是2.0版本。`hidl` 文件夹里也有一个 `light hal` ，版本是1.0 。2.0版本的hal里有这些文件：
```sh
tree hardware/xiaomi/aidl/light
hardware/xiaomi/aidl/light
├── Android.bp
├── android.hardware.light-service.xiaomi.rc
├── android.hardware.light-service.xiaomi.xml
├── Backlight.cpp
├── Backlight.h
├── LED.cpp
├── LED.h
├── Lights.cpp
├── Lights.h
├── service.cpp
├── Utils.cpp
└── Utils.h

1 directory, 12 files
```

看名字就能猜出来 `Backlight.cpp` 指lcd的背光相关， `LED.cpp` 大概率和呼吸灯有关。但是，在看这些代码之前，先要知道这个 `light hal`  “转接” 哪些东西（也就是实现了什么功能）。如果不实现弄清除这些，就无法知道各个函数的真正作用，也浪费时间（比如笨笨的我浪费了好多时间）

根据[aosp的文档](https://source.android.com/docs/core/architecture/aidl/aidl-hals?hl=zh-cn)， 系统的hal的接口在以下这些文件夹中：
- `hardware/interfaces`
- `frameworks/hardware/interfaces`
- `system/hardware/interfaces`

进 `hardware/interfaces` ，找相关接口的定义：
```sh
tree hardware/interfaces/light/aidl/android/hardware/light/
---
hardware/interfaces/light/aidl/android/hardware/light/
├── BrightnessMode.aidl
├── FlashMode.aidl
├── HwLight.aidl
├── HwLightState.aidl
├── ILights.aidl
└── LightType.aidl

1 directory, 6 files
```

和呼吸灯相关的aidl：
```java
// ILights.aidl
/*
 * Copyright (C) 2020 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package android.hardware.light;

import android.hardware.light.HwLightState;
import android.hardware.light.HwLight;

/**
 * Allows controlling logical lights/indicators, mapped to LEDs in a
 * hardware-specific manner by the HAL implementation.
 */
@VintfStability
interface ILights {
    /**
     * Set light identified by id to the provided state.
     *
     * If control over an invalid light is requested, this method exists with
     * EX_UNSUPPORTED_OPERATION. Control over supported lights is done on a
     * device-specific best-effort basis and unsupported sub-features will not
     * be reported.
     *
     * @param id ID of logical light to set as returned by getLights()
     * @param state describes what the light should look like.
     */
    void setLightState(in int id, in HwLightState state);

    /**
     * Discover what lights are supported by the HAL implementation.
     *
     * @return List of available lights
     */
    HwLight[] getLights();
}
```

主要是两个函数，分别负责获取呼吸灯的状态和设置呼吸灯的状态。由于只是呼吸灯亮度不能调控，因此理所当然在 `setLightState()` 这个函数里调试。

回到hal的实现， 查看 `setLightState()` 函数：
```cpp
// Lights.cpp
ndk::ScopedAStatus Lights::setLightState(int32_t id, const HwLightState& state) {
    rgb color(state.color);

    LightType type = static_cast<LightType>(id);
    switch (type) {
        case LightType::BACKLIGHT:
            if (mBacklightDevice) mBacklightDevice->setBacklight(color.toBrightness());
            break;
        case LightType::BUTTONS:
            for (auto& buttons : mButtonsPaths) writeToFile(buttons, color.isLit());
            break;
        case LightType::BATTERY:
        case LightType::NOTIFICATIONS:
        case LightType::ATTENTION:
            mLEDMutex.lock();

            if (type == LightType::BATTERY)
                mLastBatteryState = state;
            else if (type == LightType::NOTIFICATIONS)
                mLastNotificationState = state;
            else if (type == LightType::ATTENTION)
                mLastAttentionState = state;

            setLED();

            mLEDMutex.unlock();

            break;
        default:
            return ndk::ScopedAStatus::fromExceptionCode(EX_UNSUPPORTED_OPERATION);
            break;
    }

    return ndk::ScopedAStatus::ok();
}
```

通过各种判断，执行 `setLED()` 来设置灯的状态。先继续跟踪这个函数
```cpp
// Lights.cpp
void Lights::setLED() {
    bool rc = true;

    bool isBatteryLit = rgb(mLastBatteryState.color).isLit();
    bool isAttentionLit = rgb(mLastAttentionState.color).isLit();

    const HwLightState state = isBatteryLit     ? mLastBatteryState
                               : isAttentionLit ? mLastAttentionState
                                                : mLastNotificationState;

    rgb color(state.color);
    uint8_t blink = (state.flashOnMs != 0 && state.flashOffMs != 0);

    switch (state.flashMode) {
        case FlashMode::HARDWARE:
        case FlashMode::TIMED:
            if (mWhiteLED) {
                rc = kLEDs[WHITE].setBreath(blink);
            } else {
                rc = kLEDs[RED].setBreath(blink && color.red);
                rc &= kLEDs[GREEN].setBreath(blink && color.green);
                rc &= kLEDs[BLUE].setBreath(blink && color.blue);
            }
            if (rc) break;
            FALLTHROUGH_INTENDED;
        default:
            if (mWhiteLED) {
                rc = kLEDs[WHITE].setBrightness(color.toBrightness());
            } else {
                rc = kLEDs[RED].setBrightness(color.red);
                rc &= kLEDs[GREEN].setBrightness(color.green);
                rc &= kLEDs[BLUE].setBrightness(color.blue);
            }
            break;
    }

    return;
}
```

执行  `setBrightness()` 来设置亮度

```cpp
bool LED::setBrightness(uint8_t value) {
    return writeToFile(mBasePath + "brightness", value * mMaxBrightness / 0xFF);
}
```

到这里可以知道这个hal干的事情也是直接写 `/sys/class/led/white/brightness` 来实现亮度调节。现在问题就是，亮度没法调，于是想了一下几种可能：
- 1. 权限不够。仔细一想发现不对，权限不对会直接没法调灯的状态，而且日志里也没有avc日志
-  2. 亮度值有问题，可以得知有 `color.toBrightness()` 的操作，可能资格过程有问题。

为了排除亮度问题，先得知道亮度是的值是多少。又看了一会源码，发现 `color` 值是一个无符号int，32位，分4个8位，分别代表alpha、R、G、B，当led支持彩色显示时，可以直接使用这个值（直接将对应的颜色值分别写入不同的rgb控制）；若不支持彩色显示，则对这个值进行亮度比例的缩放，再写入brightness。

于是在 `setLightState()` 、`setLED()` 和 `setBrightness()` 设置log打印 `color` 的值。编译后刷入手机测试，发现这三处的 `color` 变量的值都不会随着设置里的亮度设置变化，同时三者也完全相等，说明整个过程的判断和转化都没有问题。

这样看来hal层并没有问题，hal成功完成了框架层交给它的任务。所以，是框架层的问题吗？很可能是。

众所周知，纯 aosp 是不带led亮度设置功能的，因此这个功能大概率是 LOS 自己做的，于是翻 gerrit，找到了相关的[提交](https://review.lineageos.org/q/topic:%2213-lights%22)。从提交中看出修改了如下文件：
```
services/core/java/com/android/server/BatteryService.java
services/core/java/com/android/server/notification/NotificationManagerService.java
services/core/java/com/android/server/notification/NotificationRecord.java
```



感兴趣的就是第一个文件，看提交内容发现引入了 `org.lineageos.internal.notification.LedValues` 这个包。能直接推断这个包里的类就是实现led相关的控制。这个topic里的提交在framework里不是设备特有的，而在别的设备上灯的亮度和呼吸频率的控制都是没有问题的，说明问题并不在这几个文件里，而是在这个包的实现里。而且看包名很容易知道这是los自己的包，于是首先猜测在 `lineage-sdk` 里。用grep搜索一番，便直接能找到这个类。

进入 `lineage-sdk/sdk/src/java/org/lineageos/internal/notification/` 目录，发现 `LedValues` 这个类只是led类的属性类，里面有各种 `setXXX` 和 `getXXX`，而调用这些函数的操作肯定定义在别处。此文件夹下还有这些文件
```sh
tree lineage-sdk/sdk/src/java/org/lineageos/internal/notification/
---
lineage-sdk/sdk/src/java/org/lineageos/internal/notification/
├── LedValues.java
├── LightsCapabilities.java
├── LineageBatteryLights.java
├── LineageNotification.java
└── LineageNotificationLights.java

1 directory, 5 files

```

凭直觉直接去 `LineageBatteryLights.java` 里找。先找有无方便调试的标志，找到debug 的tag，设置 `DEBUG = true` 重新编译并刷入手机，设置里调整呼吸灯亮度的同时记录日志。

日志抓了，但被我删了(T_T)

源码上可以看出，发出日志的函数主要是 `calcLights()` ，该函数代码如下：
```java
public void calcLights(LedValues ledValues, int level, int status, boolean low) {
        if (DEBUG) {
            Slog.i(TAG, "calcLights input:"
                    + " ledValues={ " + ledValues + " }"
                    + " level=" + level
                    + " status=" + status
                    + " low=" + low
                    + " mCanAdjustBrightness=" + mCanAdjustBrightness
                    + " mHALAdjustableBrightness=" + mHALAdjustableBrightness
                    + " mMultiColorBatteryLed=" + mMultiColorBatteryLed
                    + " mUseSegmentedBatteryLed=" + mUseSegmentedBatteryLed
                    + " mBatteryBrightnessLevel=" + mBatteryBrightnessLevel
                    + " mBatteryBrightnessZenLevel=" + mBatteryBrightnessZenLevel
                    + " mZenMode=" + mZenMode
                    + " mLightFullChargeDisabled=" + mLightFullChargeDisabled
            );
        }

        final boolean isChargingOrFull = status == BatteryManager.BATTERY_STATUS_CHARGING ||
                status == BatteryManager.BATTERY_STATUS_FULL;
        final boolean isCharged = status == BatteryManager.BATTERY_STATUS_FULL || level >= 100;

        // The only meaningful ledValues values received by frameworks BatteryService
        // are the pulse times (for low battery). Explicitly set enabled state and
        // color to ensure that we arrive at a deterministic outcome.
        ledValues.setEnabled(false);
        ledValues.setColor(0);

        if (!mLightEnabled || (mLightFullChargeDisabled && isCharged)) {
            return;
        }

        final int brightness;
        if (mUseSegmentedBatteryLed) {
            brightness = level;
        } else if (!mCanAdjustBrightness) {
            // No brightness support available
            brightness = LedValues.LIGHT_BRIGHTNESS_MAXIMUM;
        } else if (mZenMode == Global.ZEN_MODE_OFF) {
            brightness = mBatteryBrightnessLevel;
        } else {
            brightness = mBatteryBrightnessZenLevel;
        }
        ledValues.setBrightness(brightness);

        if (low) {
            if (status == BatteryManager.BATTERY_STATUS_CHARGING) {
                // Battery is charging and low.
                ledValues.setColor(mBatteryLowARGB);
                ledValues.setSolid();
            } else if (mLedPulseEnabled) {
                // Battery is low, not charging and pulse is enabled
                // (pulsing values are set by frameworks BatteryService).
                ledValues.setColor(mBatteryLowARGB);
            }
        } else if (isChargingOrFull) {
            if (isCharged || level >= 90) {
                // Battery is full or charging and nearly full.
                ledValues.setColor(mBatteryFullARGB);
                ledValues.setSolid();
            } else {
                // Battery is charging and not nearly full.
                ledValues.setColor(mBatteryMediumARGB);
                ledValues.setSolid();
            }
        }

        // If a color was set, enable light.
        if (ledValues.getColor() != 0) {
            ledValues.setEnabled(true);
        }
        if (mCanAdjustBrightness) {
            if (!mHALAdjustableBrightness) {
                // If lights HAL does not support adjustable brightness then
                // scale color value here instead.
                ledValues.applyAlphaToBrightness();
                ledValues.applyBrightnessToColor();
            } else if (mUseSegmentedBatteryLed && !mMultiColorBatteryLed) {
                // For non-RGB segmented LEDs, we must set the brightness as the
                // color, since the alpha channel contains the battery level
                int segmentLevel = mZenMode == Global.ZEN_MODE_OFF
                        ? mBatteryBrightnessLevel : mBatteryBrightnessZenLevel;
                ledValues.setColor(segmentLevel | (segmentLevel << 8) | (segmentLevel << 16));
            }
        }
        // If LED is segmented, reset brightness field to battery level
        // (applyBrightnessToColor() changes it to 255)
        if (mUseSegmentedBatteryLed) {
            ledValues.setBrightness(brightness);
        }

        if (DEBUG) {
            Slog.i(TAG, "calcLights output: ledValues={ " + ledValues + " }");
        }
    }
```

同时，函数的日志会输出计算前和计算后的 `ledValues`， 在日子里对比发现，在调整设置的时候，这个对象的里面的值并没有发生变化，而找遍该目录其他文件，并没有修改这个值的函数调用。因此可以判定，这个函数在计算亮度值的时候出了问题。

看这个函数的后半段，理解了这个亮度值是怎么计算和设置的：
```java
if (mCanAdjustBrightness) {
            if (!mHALAdjustableBrightness) {
                // If lights HAL does not support adjustable brightness then
                // scale color value here instead.
                ledValues.applyAlphaToBrightness();
                ledValues.applyBrightnessToColor();
            } else if (mUseSegmentedBatteryLed && !mMultiColorBatteryLed) {
                // For non-RGB segmented LEDs, we must set the brightness as the
                // color, since the alpha channel contains the battery level
                int segmentLevel = mZenMode == Global.ZEN_MODE_OFF
                        ? mBatteryBrightnessLevel : mBatteryBrightnessZenLevel;
                ledValues.setColor(segmentLevel | (segmentLevel << 8) | (segmentLevel << 16));
            }
        }
        // If LED is segmented, reset brightness field to battery level
        // (applyBrightnessToColor() changes it to 255)
        if (mUseSegmentedBatteryLed) {
            ledValues.setBrightness(brightness);
        }
```
- 1.判断 `mCanAdjustBrightness` 和 `mHALAdjustableBrightness`，若后者为假这直接写alpha和颜色的值，看日志可以得出两者都为true。
- 2. 继续判断 `mUseSegmentedBatteryLed` 和 `mMultiColorBatteryLed`，若分别为一真一假，则进行下一步判断灯在勿扰模式和正常模式下的亮度选择，再调用 `setColor` 来设置 `color` （ `ledValue` 类成员）的值。从日志上看，刚好相反，因此此段代码并不会执行。
- 3.判断 `mUseSegmentedBatteryLed`，为真时将设置亮度值。而亮度值在前面的代码中已经有判断。

会不会是没设置 `mUseSegmentedBatteryLed` 呢？于是在设备树的overlay里设置后，编译刷入，发现亮度调节能直接工作。这个bug貌似就这样修复了。


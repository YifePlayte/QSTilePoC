# QSTile PoC

This is an PoC which can make most Hyper OS system tiles disappear in edit view.

这是一个能使大多数 Hyper OS 的系统磁贴在编辑界面消失的 PoC (Proof of Concept)。

This bug has been fixed / 此Bug已被官方修复
---

## 使用方式

- 编译此项目，并将应用安装在任意一台运行 Hyper OS (MIUI?) 系统的手机上
- 编辑控制中心中的磁贴，添加“QSTile PoC”磁贴
- 退出编辑界面，重新打开编辑界面
- 观察“系统快捷开关”中的磁贴，你将观察到有很多未添加的系统磁贴缺失

| Before                                                   | After                                                  |
|----------------------------------------------------------|--------------------------------------------------------|
| <img alt="before" height="300px" src="/pic/before.jpg"/> | <img alt="after" height="300px" src="/pic/after.jpg"/> |

## 原理

Android 系统将已添加的快捷开关（或者说磁贴）存放在`content://settings/secure`
中，名称为 `sysui_qs_tiles`，其以`,`
分割，系统磁贴一般为单个单词，应用声明的磁贴为`custom(<应用包名>/<磁贴类名>)`，每项代表一个已放置的磁贴。

而 Hyper OS 在渲染编辑界面时，会将未添加的磁贴临时创建并显示。而此时，为了防止已添加的磁贴被重复创建，需要判断将要临时创建的磁贴是否在已添加磁贴中存在，如果存在就跳过。

按照正常的逻辑，这里应该：

```java
package com.android.systemui.qs.customize;

public final class MiuiQSCustomizerController extends ViewController {
    public void show(int i, int i2, boolean z) {
        // ...
        // 系统磁贴列表
        String stockTilesString = mContext.getString(R.string.miui_quick_settings_tiles_stock).split(",");
        // 已添加磁贴列表
        String addedQuickSettingsListString = Settings.Secure.getString(
                mInstrumentation.getContext().getContentResolver(), "sysui_qs_tiles");

        // 要显示的所有磁贴
        ArrayList<String> quickSettingsList = new ArrayList<>();

        // 把已添加磁贴加入列表
        if (addedQuickSettingsListString != null) {
            quickSettingsList.addAll(Arrays.asList(addedQuickSettingsListString.split(",")));
        } else {
            addedQuickSettingsListString = "";
        }

        // 把未添加的系统磁贴放入列表
        for (String tile : stockTilesString.split(",")) {
            // 若已存在此磁贴，就不添加
            if (!quickSettingsList.contains(tile)) {
                quickSettingsList.add(tile);
            }
        }
        // ...
    }
}
```

但是不知道是出于什么目的，此处 Hyper OS 的判断重复逻辑是：

```java
package com.android.systemui.qs.customize;

public final class MiuiQSCustomizerController extends ViewController {
    public void show(int i, int i2, boolean z) {
        // ...
        // 把未添加的系统磁贴放入列表
        for (String tile : stockTilesString.split(",")) {
            // !!! 若 addedQuickSettingsListString 字符串中已包含此磁贴，就不添加
            if (!addedQuickSettingsListString.contains(tile)) {
                quickSettingsList.add(tile);
            }
        }
        // ...
    }
}
```

这就导致，如果“碰巧”第三方应用磁贴的包名中包含了未被添加的系统磁贴名，且此第三方磁贴被添加了，那么在打开编辑界面时，这个系统磁贴就会被判断“已存在”而导致此系统磁贴消失。

此错误逻辑在“系统界面”与解耦后的“系统界面组件”中都有存在。

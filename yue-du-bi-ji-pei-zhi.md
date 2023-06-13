# 阅读笔记-配置

### 如何切换中/英键盘？

* 问题简述：

在 `3.2.13` 前，可以通过 `*.trime.yaml` 文件的配置，切换中/英文键盘。 切换过程中，中/英文输入模式也跟着切换。在新版中如何实现？

* 解决方式：

在中文模式和英文模式的键盘中，都加入 `reset_ascii_mode: true` 配置项 例子：

```yaml
#键盘布局
preset_keyboards:
  default:
    author: "Shitlime"
    name: 26键默认布局
    width: 10
    lock: true
    ascii_mode: 0
    # 在这里加入新的配置项：
    reset_ascii_mode: true
    keys:
      - {click: q, long_click: 1, swipe_up: '!', key_back_color: bh1, key_text_color: wk}
      - {click: w, long_click: 2, swipe_up: '@', key_back_color: bh1, key_text_color: wk}
      - {click: e, long_click: 3, swipe_up: '#', key_back_color: bh1, key_text_color: wk}
      - {click: r, long_click: 4, swipe_up: '$', key_back_color: bh1, key_text_color: wk}
      - {click: t, long_click: 5, swipe_up: '%', key_back_color: bh1, key_text_color: wk}
      - {click: y, long_click: 6, swipe_up: '^', key_back_color: bh1, key_text_color: wk}
      - {click: u, long_click: 7, swipe_up: '&', key_back_color: bh1, key_text_color: wk}
      - {click: i, long_click: 8, swipe_up: '*', key_back_color: bh1, key_text_color: wk}
      # ……
```

* 对应源码片段：

`TextInputManager.kt` 中，存在这段代码：

```kt
KeyEvent.KEYCODE_EISU -> { // Switch keyboard
                KeyboardSwitcher.switchKeyboard(event.select)
                /** Set ascii mode according to keyboard's settings, can not place into [Rime.handleRimeNotification] */
                if (shouldResetAsciiMode && KeyboardSwitcher.currentKeyboard.isResetAsciiMode) {
                    Rime.setOption("ascii_mode", KeyboardSwitcher.currentKeyboard.asciiMode)
                }
                trime.bindKeyboardToInputView()
                trime.updateComposing()
            }
```

* 其他相关信息参考：
  * [issue#913](https://github.com/osfans/trime/issues/913)
  * [pr#908](https://github.com/osfans/trime/pull/908)

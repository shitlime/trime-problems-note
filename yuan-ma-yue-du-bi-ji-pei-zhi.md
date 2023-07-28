# 源码阅读笔记-配置

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

*   其他相关信息参考：

    * [issue#913](https://github.com/osfans/trime/issues/913)
    * [pr#908](https://github.com/osfans/trime/pull/908)





### 悬浮窗消失问题



* 问题描述：

使用同文风主题可以复现：

1. 先打开键盘，输入任意字符可以看到编码悬浮窗；
2. 打开液体键盘（liquid keyboard）；
3. 返回到主键盘
4. 输入任意字符，发现编码悬浮窗消失，并且之后都消失了；

但是默认的 `trime.yaml` 主题却表现正常。



* 解决方式：

在以后的版本中将解决这个bug，如果使用旧版，可以按如下方式解决这个问题：



给有这个bug的主题配置文件（ `*.trime.yaml` ）加上 `liquid\_keyboard\_window`

```yaml
  liquid_keyboard_window: #液态键盘模式下显示的悬浮窗口組件
    - {start: "", click: "space", label: " 空格 "}
    - {start: "", click: "BackSpace", label: " 删除 "}
    - {start: "", click: "Return", label: " 回车 "}
    - {start: "", click: "liquid_keyboard_exit", label: " 返回 "}
```

* bug对应源码片段：

`trime/app/src/main/java/com/osfans/trime/ime/text/Composition.java`

```java
  /** 设置悬浮窗, 用于liquidKeyboard的悬浮窗工具栏 */
  public void setWindow() {
    if (getVisibility() != View.VISIBLE) return;
    if (liquid_keyboard_window_comp.isEmpty()) {
      this.setVisibility(GONE);
      return;
    }

    ss = new SpannableStringBuilder();

    for (Map<String, Object> m : liquid_keyboard_window_comp) {
      if (m.containsKey("composition")) appendComposition(m);
      else if (m.containsKey("click")) appendButton(m);
    }
    setSingleLine(!ss.toString().contains("\n"));

    setText(ss);
    setMovementMethod(LinkMovementMethod.getInstance());
  }
```

在上面这个方法中使用了 `this.setVisibility(GONE);` 导致编码悬浮窗不可见，
然后在下面的方法中执行 `mComposition.getRootView().setVisibility(View.VISIBLE);`
但是 `.getRootView()` 获取的是View，并不是 `Composition` 本身，所以 `setVisibility(View.VISIBLE)` 实际并没有产生效果。

`trime/app/src/main/java/com/osfans/trime/ime/core/Trime.java`

```java
 /** 更新Rime的中西文狀態、編輯區文本 */
  public int updateComposing() {
    final @Nullable InputConnection ic = getCurrentInputConnection();
    activeEditorInstance.updateComposingText();
    if (ic != null && !isWinFixed()) isCursorUpdated = ic.requestCursorUpdates(1);
    int startNum = 0;
    if (mCandidateRoot != null) {
      if (isPopupWindowEnabled) {
        Timber.d("updateComposing() SymbolKeyboardType=%s", symbolKeyboardType.toString());
        if (symbolKeyboardType != SymbolKeyboardType.NO_KEY
            && symbolKeyboardType != SymbolKeyboardType.CANDIDATE) {
          mComposition.setWindow();
          showCompositionView(false);
        } else {
          mComposition.getRootView().setVisibility(View.VISIBLE);
          startNum = mComposition.setWindow(minPopupSize, minPopupCheckSize, Integer.MAX_VALUE);
          mCandidate.setText(startNum);
          // if isCursorUpdated, showCompositionView will be called in onUpdateCursorAnchorInfo
          // otherwise we need to call it here
          if (!isCursorUpdated) showCompositionView(true);
        }
      } else {
        mCandidate.setText(0);
      }
```

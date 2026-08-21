---
name: wpf-list-virtualization-debug
description: Diagnose and fix WPF ListView/ListBox/DataGrid UI freezing caused by broken UI virtualization. Use when a WPF list with hundreds or thousands of rows freezes the UI on load/scroll, when the user suspects "large list rendering blocks the UI thread", or mentions ListView lag, virtualization not working, VirtualizingStackPanel, or CanContentScroll.
---

# WPF 列表虚拟化诊断与修复

WPF 列表（`ListView`/`ListBox`/`DataGrid`）数据量大时卡死/冻结 UI，**绝大多数情况不是数据量本身或 WPF 固有缺陷，而是 UI 虚拟化被破坏**，导致所有行被一次性实体化（创建容器 + 建立绑定 + 测量布局），瞬间占满 UI 线程。

本 skill 给出一套**先确诊再修复**的流程，避免盲目优化。

## 核心原理（必须先理解）

UI 虚拟化生效需要一条**完整的属性透传链**，缺任何一环都会全量实体化：

```
ListView (VirtualizingPanel.IsVirtualizing="True" + ScrollViewer.CanContentScroll="True")
  └─ 模板内 PART_ScrollViewer        ← 需要 CanContentScroll="{TemplateBinding ScrollViewer.CanContentScroll}"
       └─ ScrollContentPresenter     ← 真正的开关：CanContentScroll="{TemplateBinding CanContentScroll}"
            └─ ItemsPresenter
                 └─ VirtualizingStackPanel  ← 只有视口高度有限时才虚拟化
```

**真正的开关是 `ScrollContentPresenter.CanContentScroll`。**
- `True` → 把有限的视口高度喂给 `VirtualizingStackPanel`，只实体化可见行（通常 10~40 个）。
- `False`（默认）→ 喂无限高度，`VirtualizingStackPanel` 认为所有行都可见 → **全部实体化**。

**缺一不可**：链路上每一层都必须显式透传。实战中常见的坑是只补了外层 `PART_ScrollViewer` 的 `CanContentScroll`，却漏掉最内层 `ScrollContentPresenter` —— 此时虚拟化**仍然失效，容器数不下降**。必须把透传一路打通到 `ScrollContentPresenter` 这个最终开关，虚拟化才会生效。

**最常见的真凶**：项目里有**自定义的全局 `ScrollViewer` / `ListView` `ControlTemplate`**（常见于"微信风格""圆角滚动条"等美化样式），重写模板时**漏掉了 `CanContentScroll` 的 `TemplateBinding`**。此时无论在页面上怎么设 `CanContentScroll="True"`，到了模板内部都被丢回默认 `False`。

## 诊断流程

复制此清单并逐步执行：

```
诊断进度：
- [ ] 步骤 1：插入诊断探针，统计实际生成的容器数量
- [ ] 步骤 2：运行并读取诊断结果，判断虚拟化是否生效
- [ ] 步骤 3：若被破坏，定位破坏点（自定义模板优先）
- [ ] 步骤 4：修复属性透传链
- [ ] 步骤 5：重新运行验证容器数下降
- [ ] 步骤 6：清理诊断探针
```

### 步骤 1-2：确诊虚拟化是否生效

不要靠猜。插入诊断探针统计**实际生成的容器数量**，这是分水岭：

- 容器数 ≈ 可见行数（如 14、30） → 虚拟化**正常**，卡顿另有原因（见"虚拟化正常仍卡"）。
- 容器数 ≈ 总数据行数（如 407 行生成 407 个） → 虚拟化**被破坏**，这就是真凶。

完整探针代码见 [diagnostic-probe.md](diagnostic-probe.md)，直接粘贴到列表所在 `UserControl`/`Window` 的 code-behind，在数据加载完成后调用一次。它会把
`[虚拟化诊断] 数据行=N，实际生成 ListViewItem 容器=M，ItemsPanel=...` 输出到 Debug 窗口和界面。

### 步骤 3：定位破坏点

按以下顺序排查（命中率从高到低）：

1. **自定义全局 `ScrollViewer` 模板**：搜索 `TargetType="ScrollViewer"` 的 `ControlTemplate`，检查 `ScrollContentPresenter` 是否有 `CanContentScroll="{TemplateBinding CanContentScroll}"`。**漏写是头号原因。**
2. **自定义 `ListView`/`ListBox` 模板**：搜索对应 `ControlTemplate`，检查内部 `ScrollViewer` 是否透传 `CanContentScroll`。
3. **页面未启用**：列表上缺少 `ScrollViewer.CanContentScroll="True"`（DataGrid 默认 True，ListView 默认也是 True，但被外层覆盖时需显式设）。
4. **外层容器给了无限高度**：列表被放进 `StackPanel`、`Grid` 的 `Auto` 行、或外层再套一个 `ScrollViewer` —— 这些会给列表无限测量高度，绕过虚拟化。

搜索命令（用 Grep/ripgrep）：

```
CanContentScroll|ScrollContentPresenter|TargetType=.ScrollViewer|TargetType=.ListView
```

### 步骤 4：修复

针对最常见的"自定义模板漏透传"，补上 `TemplateBinding` 即可。修复 `ScrollViewer` 模板（关键）：

```xml
<!-- 错误：CanContentScroll 恒为默认 false，虚拟化失效 -->
<ScrollContentPresenter />

<!-- 正确：透传开关 -->
<ScrollContentPresenter CanContentScroll="{TemplateBinding CanContentScroll}" />
```

若 `ListView` 模板也自定义了，同样给内部 `ScrollViewer` 补透传：

```xml
<ScrollViewer x:Name="PART_ScrollViewer"
              CanContentScroll="{TemplateBinding ScrollViewer.CanContentScroll}"
              HorizontalScrollBarVisibility="{TemplateBinding ScrollViewer.HorizontalScrollBarVisibility}"
              VerticalScrollBarVisibility="{TemplateBinding ScrollViewer.VerticalScrollBarVisibility}">
    <ItemsPresenter />
</ScrollViewer>
```

**安全性**：用 `TemplateBinding` 透传而非硬编码 `True`。`CanContentScroll` 默认 `False`，所以未显式开启的滚动区域行为不变（仍平滑像素滚动），只有显式设 `True` 的列表受益，不会破坏其他页面手感。

页面侧确保列表已开启虚拟化：

```xml
<ListView ScrollViewer.CanContentScroll="True"
          VirtualizingPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling"
          VirtualizingPanel.ScrollUnit="Pixel" />
```

### 步骤 5-6：验证并清理

重新运行，确认容器数从"总行数"降到"可见行数"（如 407 → 14）。冻结应当场消失。然后**删除诊断探针代码及为它临时添加的 `using`**，恢复干净代码。

## 虚拟化正常仍卡（次要原因）

若步骤 2 显示容器数已经很少但仍卡，依次排查：

- **逐条 `ObservableCollection.Add` 在 UI 线程循环** → 后台构造 `List<T>`，UI 线程一次性赋值 `ItemsSource`。
- **选中计数等 O(N²) 操作**：全选/反选时每行赋值都触发全量 `Count(...)` → 改为增量维护计数字段。
- **行模板过重**：每行几十个元素 + 大量跨元素 `ElementName` 绑定 → 精简模板。
- **数据解析在 UI 线程** → 移到 `Task.Run`。

## ItemsControl 默认不虚拟化 + 列表摊开

**`ItemsControl` 默认完全不虚拟化**（其默认 `ItemsPanel` 是普通 `StackPanel`，且自身不带 `ScrollViewer`）。当它绑定大量数据时，所有项一次性实体化，和虚拟化被破坏的后果相同。

典型症状（可从截图直接判断，无需探针）：**列表项一直延伸、超出父控件/窗口底部，没有自己的滚动条，只能靠整个页面的大滚动条查看**。根因通常是：`ItemsControl` 外层是 `StackPanel` 或被整页的 `<ScrollViewer><StackPanel>` 包裹 → 拿到无限高度 → 全部摊开。

修复：换成 `ListBox`/`ListView`（自带 `ScrollViewer` + 虚拟化），并用 **`MaxHeight`** 约束高度，让其超出时**内部滚动**而非撑开整页。

```xml
<ListBox ItemsSource="{Binding Items}"
         MaxHeight="{Binding SomeAvailableHeight, RelativeSource={RelativeSource AncestorType=UserControl}}"
         Background="Transparent" BorderThickness="0"
         HorizontalContentAlignment="Stretch"
         ScrollViewer.CanContentScroll="True"
         VirtualizingPanel.IsVirtualizing="True"
         VirtualizingPanel.VirtualizationMode="Recycling"
         VirtualizingPanel.ScrollUnit="Pixel">
    <ListBox.ItemContainerStyle>
        <!-- 纯展示列表：去掉选中高亮/内边距，Focusable=False -->
        <Style TargetType="ListBoxItem" BasedOn="{StaticResource {x:Type ListBoxItem}}">
            <Setter Property="Padding" Value="0" />
            <Setter Property="HorizontalContentAlignment" Value="Stretch" />
            <Setter Property="Background" Value="Transparent" />
            <Setter Property="BorderThickness" Value="0" />
            <Setter Property="Focusable" Value="False" />
        </Style>
    </ListBox.ItemContainerStyle>
    <!-- 原 ItemTemplate 原样保留 -->
</ListBox>
```

`MaxHeight` 的值用依赖属性 + `SizeChanged` 动态计算（= 窗口可用高 − 顶部已用 − 边距），实现随窗口缩放自适应。**注意**：在 `StackPanel` 内用 `Height="*"` 约束无效（`StackPanel` 不分配 `*`），所以用 `MaxHeight` 而非 `*`。

## 关键检查清单

- [ ] 自定义 `ScrollViewer` 模板的 `ScrollContentPresenter` 透传了 `CanContentScroll`
- [ ] 自定义 `ListView` 模板的内部 `ScrollViewer` 透传了 `CanContentScroll`
- [ ] 列表未被无限高度容器（`StackPanel`/`Auto` 行/外层 `ScrollViewer`）包裹
- [ ] 大数据列表没有用 `ItemsControl`（默认不虚拟化）；若列表项溢出父控件无内部滚动，改用 `ListBox`/`ListView` + `MaxHeight`
- [ ] 用诊断探针实测容器数，而非猜测
- [ ] 修复后容器数 ≈ 可见行数
- [ ] 清理了诊断探针

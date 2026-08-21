# 虚拟化诊断探针（C#）

把以下代码粘贴到列表所在 `UserControl`/`Window` 的 code-behind，在**数据加载完成后**调用 `DiagnoseVirtualization()` 一次。它递归遍历可视树，统计实际生成的容器数量，输出到 Debug 窗口和界面（Snackbar）。

## 前置 using

```csharp
using System.Windows.Media;   // VisualTreeHelper（诊断用，清理时删除）
```

## 探针代码

将下面三处标识符按实际情况替换：
- `TradeList` → 你的列表控件名（`ListView`/`ListBox`/`DataGrid`）
- `ListViewItem` → 对应的容器类型（`ListBoxItem` / `DataGridRow`）
- `_rows.Count` → 你的数据源行数；`Snackbar.MessageQueue?.Enqueue(...)` → 改成任意可见的提示方式（或仅保留 Debug 输出）

```csharp
/// <summary>诊断探针：统计实际生成的容器数量，判断 UI 虚拟化是否生效。用完即删。</summary>
private void DiagnoseVirtualization()
{
    // 让出一帧，确保容器实体化完成后再统计
    Dispatcher.BeginInvoke(() =>
    {
        var containers = CountVisualChildren<ListViewItem>(TradeList);
        var panelType = FindItemsHostType(TradeList);
        var msg = $"[虚拟化诊断] 数据行={_rows.Count}，实际生成 ListViewItem 容器={containers}，ItemsPanel={panelType}";
        System.Diagnostics.Debug.WriteLine(msg);
        Snackbar.MessageQueue?.Enqueue(msg); // 没有 Snackbar 时删除此行
    }, System.Windows.Threading.DispatcherPriority.Background);
}

private static int CountVisualChildren<T>(DependencyObject root) where T : DependencyObject
{
    var count = 0;
    var n = VisualTreeHelper.GetChildrenCount(root);
    for (var i = 0; i < n; i++)
    {
        var child = VisualTreeHelper.GetChild(root, i);
        if (child is T) count++;
        count += CountVisualChildren<T>(child);
    }
    return count;
}

private static string FindItemsHostType(DependencyObject root)
{
    var n = VisualTreeHelper.GetChildrenCount(root);
    for (var i = 0; i < n; i++)
    {
        var child = VisualTreeHelper.GetChild(root, i);
        if (child is System.Windows.Controls.Panel p && p.IsItemsHost)
            return child.GetType().Name;
        var nested = FindItemsHostType(child);
        if (nested != "未找到") return nested;
    }
    return "未找到";
}
```

## 判读结果

| 输出 | 含义 |
|---|---|
| 容器数 ≈ 可见行数（如 14、30），ItemsPanel=`VirtualizingStackPanel` | 虚拟化**正常**，卡顿另有原因 |
| 容器数 ≈ 总行数（如 407 行生成 407 个） | 虚拟化**被破坏** → 检查 `ScrollContentPresenter.CanContentScroll` 透传 |
| ItemsPanel 不是 `VirtualizingStackPanel` | `ItemsPanel` 被换成了非虚拟化面板（如 `StackPanel`/`WrapPanel`） |

## 清理

确认修复后，删除 `DiagnoseVirtualization` / `CountVisualChildren` / `FindItemsHostType` 三个方法、调用点，以及为诊断临时添加的 `using System.Windows.Media;`（若该文件别处未用到）。

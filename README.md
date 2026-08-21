# AceCursorSkill

个人 WPF 相关 Cursor Agent Skills。放到 `~/.cursor/skills/` 后，Agent 会按各 skill 的 description 选用；DPI 脚手架检查需你明确说要跑静态检查。

## Skills

### [wpf-project-init](wpf-project-init/)

初始化 WPF 项目基础设施：Generic Host DI、Serilog、CommunityToolkit.Mvvm、MaterialDesignThemes、LottieSharp、MailKit，以及 Per-Monitor DPI 基础配置。

### [wpf-embedded-font-setup](wpf-embedded-font-setup/)

把自定义 TTF 嵌入 WPF 程序集，通过 `FontFamily` 资源使用。自带 Inter / Noto Sans SC / JetBrains Mono（Regular–Bold），无需系统安装字体。

### [wpf-dpi-static-check](wpf-dpi-static-check/)

手动检查 DPI 脚手架，以及 **Window 级检查盖不住的遮罩对话框**（DialogHost / Popup、阴影包住文字、Viewbox）。只 Grep；你开口要检查时才跑。

### [wpf-multi-dpi-adaptation](wpf-multi-dpi-adaptation/)

多分辨率、多显示器 DPI 自动适配（含 100% ↔ 150% 跨屏拖动）。覆盖 PerMonitorV2、弹性布局、响应式断点，以及 PerMonitorV2 之后仍发虚的长尾问题。窗口漏设脚手架请先用 `wpf-dpi-static-check`。

### [wpf-list-virtualization-debug](wpf-list-virtualization-debug/)

诊断并修复 ListView / ListBox / DataGrid 因 UI 虚拟化被破坏导致的卡顿冻结。先用探针确认容器数，再定位自定义模板等破坏点。

---
id: ENG-20261102-001
date: 2026-11-02
updated: 2026-11-02
status: resolved_verified
domain: development
category: tooling
systems:
  - Python Tkinter / ttk.Treeview (Tk 8.6)
  - Windows 微信 wxapkg 离线缓存探针 V0.1 → V0.2
tags:
  - tkinter
  - treeview
  - pack-in
  - gui-layout
  - empty-table
---

# Tkinter 扫到了文件，Treeview 却完全空白：控件父容器与布局容器不一致

## 遇到了什么？

自己做的微信小游戏缓存探针能扫到 **171 个 `.wxapkg` 候选文件、3 个候选目录**，状态栏正常显示，但表格**连列标题都没有**。点击导出只得到“请先在表格中选中一个包”，因为根本没有可见的行。

用户使用 Windows 微信 4.1.13.65；探针 V0.1 用 Python Tkinter / ttk。文件数量和空白界面来自用户实际反馈，不能推断这些文件属于哪个具体小游戏。

## 我是怎么查的？

先读了 V0.1 的界面代码：`Treeview` 和竖向滚动条是以 `main` 为父控件创建的，却用 `pack(in_=content)` 布局进 `main` 的另一个子 Frame `content`：

```python
content = ttk.Frame(main)
content.pack(fill="both", expand=True)
tree = ttk.Treeview(main, columns=cols, show="headings")
tree.pack(in_=content, side="left", fill="both", expand=True)
```

这并没有在测试环境中抛异常，却导致同一位置的 `content` Frame 盖住表格。可以通过 Tk 控件树发现：`tree.winfo_parent()` 返回的是 `main`，而不是 `content`。

然后在 Linux + Tk 8.6 的 Xvfb 虚拟显示下运行 V0.1，截图复现了“标题、按钮、状态栏都在，但表格区域空白”的现象。以为先前的“GUI 能启动”测试通过就代表表格可用，这是测试覆盖不足。

## 关键修复

让 Treeview、Scrollbar **直接以用于布局的 content 为父容器**，不使用跨容器 `pack(in_=...)`：

```python
content = ttk.Frame(main)
content.pack(fill="both", expand=True)
tree = ttk.Treeview(content, columns=cols, show="headings")
bar = ttk.Scrollbar(content, orient="vertical", command=tree.yview)
tree.configure(yscrollcommand=bar.set)
tree.pack(side="left", fill="both", expand=True)
bar.pack(side="right", fill="y")
```

也修复了一个容易误判的使用问题：**基线对比为 0 个变化不代表没有缓存**。目标游戏可能早已下载；新 UI 默认显示所有候选、按修改时间倒序，还提供 AppID、文件名和状态筛选。

## 实际验证

- `python -m py_compile wx_cache_probe.py`：PASS，静态编译。
- `python wx_cache_probe.py --self-test`：PASS，合成包头、哈希、路径脱敏。
- `xvfb-run -a python tests/test_gui_smoke.py`：PASS，通过真实 Tk 按钮模拟扫描 3 个合成文件，确认列标题/行可见、筛选、基线、重扫。
- 对 V0.2 的 Xvfb 截图复核：PASS，列标题和三行合成记录均可见。
- `run_probe.cmd`：ASCII + CRLF 静态检查 PASS。

**未验证**：V0.2 尚未在真实 Windows 微信 4.1.13.65 环境运行，不能据 Linux GUI 测试宣称目标平台已验收。也未定位《猛鬼宿舍》的实际包或 AppID。

状态 `resolved_verified` 仅指跨容器布局导致表格不可见的根因在复现环境中已修复。

## 下次遇到，先查什么？

当“扫描计数正常但表格没有标题和行”时，不要先怀疑扫描失败。先看控件是否确实被渲染、是否被其他 Frame 遮盖，以及 `winfo_parent()` 与布局目标是否匹配。GUI 冒烟测试不仅要调用 `mainloop()`，至少要模拟填充数据，确认标题、第一行与用户点击选择确实可用。

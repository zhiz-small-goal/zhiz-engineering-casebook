---
id: ENG-20261112-001
date: 2026-11-12
updated: 2026-11-12
status: resolved_verified
domain: applications
category: plugin-integration
systems:
  - Godot 4.6.1 Stable (GDScript, standard build)
  - Terrain3D v1.0.2
  - RenderingServer / headless automated tests
tags:
  - godot
  - terrain3d
  - resource-lifecycle
  - editor-runtime-boundary
  - signal-side-effects
  - rid
---

# Terrain3D 森林重复生成时触发编辑器缩略图，RenderingServer 报告空 RID

## 遇到了什么？

森林原型已经可以生成草丛、蕨类、野花、石块、枯木、阔叶树、针叶树和灌木。自动化测试为了验证同 Seed 重复生成不会叠加实例，会在同一个 Terrain3D 场景里再次调用生成器。

第二轮测试中，行为断言全部通过，但终端报告 7 条基础设施错误：

- 4 条 `Parameter "instance" is null`，分别发生在 `instance_set_base`、材质 override/overlay 和 `instance_set_transform`。
- 2 条 `Parameter "viewport" is null`，发生在 `viewport_set_size` 与 `viewport_set_update_mode`。
- 1 条 `Parameter "t" is null`，发生在 dummy renderer 的 `texture_2d_get()`。

这不是七个独立的业务失败，也不能因为行为用例通过就把它们当作无害日志。

## 环境

- Godot 4.6.1 Stable，标准版，Windows。
- Terrain3D v1.0.2。
- 测试使用 headless Godot 进程；项目中的森林生成器会在同一 Terrain3D 场景里执行第二次同 Seed 生成。
- 诊断以 Terrain3D v1.0.2 上游源码为依据，不把结论外推到其他版本。

## 我是怎么查的？

先按错误里的 RenderingServer 方法名定位调用点，再回到 Terrain3D v1.0.2 源码核对这些 RID 从哪里创建、在哪个生命周期释放或初始化。

关键源码：

- [terrain_3d_assets.cpp（v1.0.2-stable）](https://github.com/TokisanGames/Terrain3D/blob/v1.0.2-stable/src/terrain_3d_assets.cpp)
- [terrain_3d_mesh_asset.cpp（v1.0.2-stable）](https://github.com/TokisanGames/Terrain3D/blob/v1.0.2-stable/src/terrain_3d_mesh_asset.cpp)

源码显示：

1. `Terrain3DAssets::initialize()` 只在编辑器模式初始化 MeshAsset 缩略图预览环境，创建预览用 Scenario、Viewport、Camera、Light 和 MeshInstance RID。
2. `update_mesh_list()` 会把 MeshAsset 的 `file_changed`、`setting_changed` 信号连接到 `_update_thumbnail()`。
3. `Terrain3DMeshAsset::set_scene_file()` 会发出 `file_changed`；修改 `height_offset` 会发出 `setting_changed`。
4. 缩略图回调进入 `create_mesh_thumbnails()`，使用预览 RID 设置 Mesh、材质、变换、Viewport，并读取预览纹理。
5. 预览环境只在编辑器模式初始化，但这些资源变更信号并没有同等的运行模式保护。

项目旧实现的重复生成路径，正好再次给已经注册的 MeshAsset 设置 `scene_file` 和 `height_offset`。信号因此进入编辑器专用缩略图路径，预览 RID 无效，产生与日志位置对应的空参数错误。

## 现在能确定什么？

- **事实（上游源码）：** v1.0.2 的缩略图预览 RID 在编辑器模式初始化；MeshAsset 变更信号会连接到缩略图更新回调；相关 setter 会发出对应信号。
- **事实（项目代码）：** 同 Seed 重复生成会重新设置已注册 MeshAsset 的 `scene_file` 与 `height_offset`；旧流程也会重复提交资产列表。
- **高可信推断：** 这条重复资源赋值链触发了未初始化的编辑器预览 RID，是七条 RenderingServer 错误的共同根因。错误调用点与上游缩略图实现吻合。
- **未知：** 没有对 Godot / Terrain3D 二进制做引擎级断点跟踪；其他 Terrain3D 版本是否仍有相同信号连接方式，必须重新核对源码。

## 怎么处理，验证了吗？

修复把资源注册与实例重建分开：

1. 新 MeshAsset 在进入 Terrain3DAssets 注册表前一次性初始化；`scene_file` 只在首次创建时设置。
2. `scene_file` setter 已将 `height_offset` 归零，因此不再额外调用会发出 `setting_changed` 的 setter。
3. 已注册且 Mesh/LOD 有效的资产直接复用；运行时发现资源无效则明确报错并停止，不借道编辑器缩略图路径重建。
4. 只有 MeshAsset 确实新增时才重新提交 `assets.mesh_list`。
5. 同 Seed 回归测试在第二次生成前监听八类资产的 `file_changed` / `setting_changed`；重复生成后要求信号数量为零，同时继续核对实例变换集合与数量不变。

修复后，用户在 Godot 4.6.1 本机反馈全量测试通过。此案例状态为 `resolved_verified`，但不代表视觉和性能专项验收通过。

## 下次遇到，先查什么？

当重复生成、刷新或重载后出现 RenderingServer 空 RID，先区分业务断言与基础设施错误，再检查：

- 重复操作是否重新注册了已注册资源；
- setter 是否发出 Signal，Signal 接收者是否调用编辑器工具、预览或 RenderingServer；
- 对应 RID 是否只在 EditorPlugin / editor hint 生命周期创建；
- 重复操作应该重建的是实例数据，还是资源本身。

不要先屏蔽引擎 ERROR，也不要用重复赋值来“确保资源状态正确”。先确认资源所有权和生命周期，再让重复操作只修改它真正拥有的状态。

## 参考

- [Terrain3D v1.0.2-stable 源码](https://github.com/TokisanGames/Terrain3D/tree/v1.0.2-stable)
- [Terrain3D Assets 源码](https://github.com/TokisanGames/Terrain3D/blob/v1.0.2-stable/src/terrain_3d_assets.cpp)
- [Terrain3D MeshAsset 源码](https://github.com/TokisanGames/Terrain3D/blob/v1.0.2-stable/src/terrain_3d_mesh_asset.cpp)

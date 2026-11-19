# 构件信息查看器（Trimble Connect 3D 查看器侧栏扩展）

在 Trimble Connect for Browser 的 **3D 查看器侧栏面板**中，实时显示当前选中构件的信息：名称、对象类型、运行时 ID、GUID / 对象 ID、所属模型、颜色、位置、包围盒尺寸以及属性集（Property Sets）。

本扩展**不做独立弹窗**，所有内容都渲染在扩展自身的 iframe 面板内。

---

## 一、功能说明

| 功能 | 说明 |
| --- | --- |
| 实时跟随选中 | 监听 `viewer.onSelectionChanged` 事件，在查看器中点选构件后自动刷新面板 |
| 多选支持 | 一次选中多个构件 / 多个模型时逐项展示，最多显示 20 个对象卡片（完整数据见原始响应） |
| 手动读取 | 「读取当前选中」按钮：探测并调用 `viewer.getSelectedObjects` 等候选接口；不可用时回退到最近一次事件缓存 |
| 清空显示 | 「清空显示」按钮清空面板与原始响应 |
| 原始响应 | 可折叠的 JSON 区，展示事件负载、归一化选中、四个 ViewerAPI 的原始返回与错误信息，便于排查 API 结构 |
| 状态栏 | 连接状态、当前项目、最近事件、`accessToken` 状态（默认不申请 token） |
| 健壮性 | 所有 API 调用均 `try/catch` + 友好提示；standalone 直接打开页面不会报错或白屏 |
| 日志 | 统一前缀 `[TCSelect]`，F12 控制台过滤即可查看完整过程 |

---

## 二、⚠️ 部署前必须替换的占位项

`manifest.json` 中的 **`YOUR_USERNAME` 全部要换成你自己的 GitHub 用户名（或改成你自己的 HTTPS 托管域名）**，共 3 处：

```json
"url":     "https://YOUR_USERNAME.github.io/tc-selection-info/index.html",
"icon":    "https://YOUR_USERNAME.github.io/tc-selection-info/icon.svg",
"infoUrl": "https://github.com/YOUR_USERNAME/tc-selection-info"
```

同时把下面的地址替换成你自己的仓库地址：

- `index.html` 中 standalone 降级说明里提到的 manifest 路径（若你改了仓库名）
- 本文中的示例仓库地址

未替换会导致 TC 无法下载 manifest 或图标。

---

## 三、部署步骤（GitHub Pages）

1. 新建 GitHub 仓库 `tc-selection-info`，把本目录的 4 个文件上传到仓库根目录。
2. 仓库 → **Settings → Pages** → Build and deployment：
   - Source 选 **Deploy from a branch**
   - Branch 选 `main`，目录选 `/ (root)` → Save。
3. 等待部署完成，确认以下地址可访问（把 `YOUR_USERNAME` 换成你的用户名）：
   - `https://YOUR_USERNAME.github.io/tc-selection-info/index.html`
   - `https://YOUR_USERNAME.github.io/tc-selection-info/manifest.json`
   - `https://YOUR_USERNAME.github.io/tc-selection-info/icon.svg`
4. **CORS**：GitHub Pages 默认会返回 `Access-Control-Allow-Origin: *`，满足 TC 要求。若改用自建服务器 / IIS / Nginx，请显式配置：

   ```http
   Access-Control-Allow-Origin: *
   Access-Control-Allow-Methods: GET, OPTIONS
   Access-Control-Allow-Headers: Content-Type
   ```

   并用浏览器开发者工具 Network 面板确认 `manifest.json` 的响应头中包含上述字段。
5. 必须是 **HTTPS**。localhost / http:// 无法被 TC 添加（TC 会拒绝非 HTTPS manifest）。

---

## 四、在 Trimble Connect 中安装

1. 登录 Trimble Connect for Browser，打开任意项目并进入 **3D 查看器**。
2. 左侧栏 → **Settings（项目设置）→ Apps & Capabilities**。
3. 点击 **+ Add Custom**。
4. 在 **Capability Manifest URL** 中粘贴你的 manifest URL：

   ```
   https://YOUR_USERNAME.github.io/tc-selection-info/manifest.json
   ```

5. 点击 **Add**，确认扩展状态为 **Enabled**。
6. 回到 3D 查看器，在侧栏中找到「构件信息查看器」面板并打开。
7. 在 3D 视图中点击任意构件，面板应自动显示构件信息。

> `manifest.json` 的 `extensionType` 为 `["3dviewer"]`，这是出现在**查看器侧栏**的关键；参考 demo 用的 `["project"]` 只会出现在项目页左侧导航。

---

## 五、验证方法

1. 按 **F12** 打开开发者工具，切换到 **Console**。
2. 在过滤框输入 `[TCSelect]`。
3. 正常流程应看到如下日志：

   ```
   [TCSelect] Workspace API 脚本已加载，检测到宿主 iframe，开始连接…
   [TCSelect] 连接成功。API 命名空间检查：project = 有 | viewer = 有 | …
   [TCSelect] 项目信息： {…}
   [TCSelect] 收到事件： viewer.onSelectionChanged 负载： {…}
   [TCSelect] viewer.getObjectProperties 返回： {…}
   [TCSelect] 原始响应已更新…
   ```

4. 若没有收到选中事件，点击面板上的「读取当前选中」，观察控制台中探测到了哪些候选方法。
5. 展开「原始响应 JSON」可查看事件负载的真实结构，用于确认你的 TC 版本字段是否一致。

---

## 六、关键 API 查证结论

### 6.1 选中事件（已确认）

- **事件名：`viewer.onSelectionChanged`**
- 依据（官方文档 `modules.html`）：

  > `SelectionCallback: EventCallback<"viewer.onSelectionChanged", SelectionChangedEventArgument>`
  > —— The signature of the callback in the event of changed selection. The first parameter is the event name and the second parameter contains the selection data.

  参考链接：<https://components.connect.trimble.com/trimble-connect-workspace-api/modules.html#SelectionCallback>

- 事件参数 `SelectionChangedEventArgument` 继承 `EventArgument<Selection>`：

  | 字段 | 类型 | 说明 |
  | --- | --- | --- |
  | `data` | `Selection` | 选中内容，必填 |
  | `action` | `string?` | 可选动作 |
  | `origin` | `{ isSelf?: boolean }?` | 可选，是否由本客户端 API 调用触发 |

  参考链接：<https://components.connect.trimble.com/trimble-connect-workspace-api/interfaces/SelectionChangedEventArgument.html>

- `Selection = ModelObjectIds[]`，而 `ModelObjectIds` 为：

  | 字段 | 类型 | 说明 |
  | --- | --- | --- |
  | `modelId` | `string` | 模型标识（等于文件版本 `File.versionId`） |
  | `objectRuntimeIds` | `number[]?` | 对象运行时 ID 数组 |
  | `recursive` | `boolean?` | 是否递归 |

  参考链接：<https://components.connect.trimble.com/trimble-connect-workspace-api/interfaces/ModelObjectIds.html>

> 注意：官方首页的 `switch(event)` 示例只列了 `extension.command` / `extension.accessToken` / `extension.userSettingsChanged`，**容易造成"没有选中事件"的误解**；完整事件名是在 `modules.html` 的 `SelectionCallback` 类型字面量中找到的。代码已对 `viewer.onSelectionChanged` 及若干别名做兼容，并开启事件名模糊匹配（含 `selection` / `selected` 即识别）。

### 6.2 已验证可用的接口（官方文档明确列出）

来源：<https://components.connect.trimble.com/trimble-connect-workspace-api/interfaces/ViewerAPI.html>

| 方法 | 签名 | 本扩展用途 |
| --- | --- | --- |
| `getObjectProperties` | `(modelId: string, objectRuntimeIds: number[]): Promise<ObjectProperties[]>` | 名称、类型、颜色、位置、属性集 |
| `getObjectBoundingBoxes` | `(modelId: string, objectRuntimeIds: number[]): Promise<ObjectBoundingBox[]>` | 包围盒与尺寸 |
| `getObjectPositions` | `(modelId: string, objectRuntimeIds: number[]): Promise<ObjectPosition[]>` | 位置兜底 |
| `convertToObjectIds` | `(modelId: string, runtimeIds: number[]): Promise<string[]>` | GUID / 对象 ID |
| `convertToObjectRuntimeIds` | `(modelId: string, objectIds: string[]): Promise<number[]>` | 已确认存在，本扩展未调用 |
| `getLayers` | `(modelId: string): Promise<Layer[]>` | 模型图层数量 |
| `getHierarchyChildren` | `(modelId, entityIds, hierarchyType?, recursive?): Promise<HierarchyEntity[]>` | 已确认存在，本扩展未调用 |
| `getColoredObjects` | `(): Promise<ModelObjects[]>` | 已确认存在，本扩展未调用 |
| `project.getProject` | `(): Promise<ConnectProject>` | 状态栏当前项目 |

`ObjectProperties`、`ObjectBoundingBox`、`ObjectPosition`、`ModelObjectIds`、`PropertySet`、`Property`、`Product`、`Box3`、`Vector3` 等字段均按官方 interface 文档解析。

### 6.3 待验证 / 不存在的接口

- **官方文档中不存在**按需读取当前选中的方法（无 `getSelection` / `getSelectedObjects`）。因此「读取当前选中」按钮会：
  1. 依次探测 `viewer.getSelectedObjects` / `getSelection` / `getSelectedObjectIds` / `getSelectedObjectRuntimeIds` / `getSelectedEntities` / `getSelected`（部分 TC 版本可能私有提供）；
  2. 若都不可用，尝试 `propertyPanel.getPropertyPanelData()`（官方文档确认存在，返回 `entities: string[]`，但为 FRN 编码，无法直接映射为运行时 ID，仅展示提示）；
  3. 最后回退到最近一次 `viewer.onSelectionChanged` 的事件缓存，并给出明确中文提示。
- 轮询兜底默认关闭（`CONFIG.ENABLE_POLLING = false`）。因为官方没有读取接口，开启轮询也没有实际数据源；如你的 TC 版本提供了私有读取方法，可把它加入 `READ_METHOD_NAMES` 并将该开关设为 `true`，代码会自动在探测不到接口时停止轮询。

---

## 七、已知限制

1. **事件是唯一可靠数据源**：如果侧栏面板是在选中之后才打开的，`viewer.onSelectionChanged` 已经错过，面板会显示"无选中"；此时点选任意构件刷新即可，或点击「读取当前选中」尝试私有接口。
2. **FRN 实体**：`propertyPanel` 返回的是 FRN 编码字符串（如 `frn:entity:3CqVfw%24t15ihB2vPgB1wri`），本扩展不把它当作运行时 ID 查询，避免误报。
3. **GUID 顺序假设**：`convertToObjectIds` 返回 `string[]`，按请求的 `runtimeIds` 顺序一一对应；若个别版本返回顺序不同，原始响应 JSON 中可核对。
4. **对象数量上限**：单次最多渲染 20 个对象卡片，避免大选中导致卡顿；完整数据始终在「原始响应 JSON」中。
5. **token 不展示**：默认不申请 `accessToken`（`CONFIG.REQUEST_ACCESS_TOKEN = false`），本扩展只读取选中信息，不需要 token。如需增强，改为 `true` 即可，事件回调只记录长度，绝不把 token 渲染到页面。
6. **TC 版本差异**：不同 TC 版本对 `3dviewer` 侧栏的字段支持可能不同；任何接口缺失或调用失败都会在面板显示友好提示，并在控制台留下 `[TCSelect]` 警告日志，不会白屏。
7. **脚本依赖**：`index.html` 通过 CDN 引入 `index.js`，首次打开需要能访问 `components.connect.trimble.com`。

---

## 八、文件清单

| 文件 | 说明 |
| --- | --- |
| `index.html` | 扩展主页面：连接、事件监听、信息卡片、手动读取、原始 JSON、standalone 降级 |
| `manifest.json` | 扩展清单，`extensionType: ["3dviewer"]`，含占位 HTTPS URL（**必须替换**） |
| `icon.svg` | 构件 + 信息图标 |
| `README.md` | 本文档 |

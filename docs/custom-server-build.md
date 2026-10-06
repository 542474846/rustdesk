# 自建服务器地址编译指南

本 fork 的私有补丁:把客户端内置的默认服务器地址替换为自建服务器。地址与公钥通过
**编译期环境变量**注入,只存在于 GitHub Secrets 中,仓库(包括本文档)不含任何隐私信息。
本文用于同步上游主线后,快速重做全部修改。

## 一、机制与变量

RustDesk 客户端的内置默认服务器定义在 `libs/hbb_common/src/config.rs`:
`RENDEZVOUS_SERVERS`(ID 服务器)与 `RS_PUB_KEY`(公钥)两个常量。中继与 API 服务器
在运行时按回退链解析。本 fork 把这些改为 `option_env!` 编译期读取:构建时检测到环境
变量就用它,否则回退官方默认值。GitHub Actions 中这 4 个环境变量来自仓库 Secrets。

取值优先级:**用户在客户端设置页手填的 > 编译期烧入的 > 原有回退逻辑**。
不设变量时行为与上游完全一致。

| 设置项 | 变量 / Secret | 是否必填 | 说明 |
|---|---|---|---|
| ID 服务器 | `RENDEZVOUS_SERVER` | 必填 | hbbs 地址,不带端口(自动补 21116) |
| Key | `RS_PUB_KEY` | 必填 | 服务器上 `id_ed25519.pub` 文件的内容 |
| API 服务器 | `API_SERVER` | 建议 | 如 `http://域名:21114`;不设则指向官方(原因见第六节) |
| 中继服务器 | `RELAY_SERVER` | 可选 | hbbr 地址;与 hbbs 同机同端口时可不设 |

## 二、补丁清单(共 6 处)

> 行号会随上游演进而漂移,用"锚点"搜索定位。

### 1. 子模块 `libs/hbb_common` → `src/config.rs`

锚点:搜索 `pub const RENDEZVOUS_SERVERS`。原来是两行常量定义,整体替换为:

```rust
// Compile-time override for private deployments: set RENDEZVOUS_SERVER, RELAY_SERVER,
// API_SERVER and RS_PUB_KEY in the build environment to bake in a self-hosted server
// without committing the addresses. Empty or unset values keep the upstream defaults.
// String literal patterns are not allowed in const matches (E0015), so empty values
// are resolved at the use sites / in the workflow instead.
pub const RENDEZVOUS_SERVERS: &[&str] = &[match option_env!("RENDEZVOUS_SERVER") {
    Some(s) => s,
    None => "rs-ny.rustdesk.com",
}];
pub const RS_PUB_KEY: &str = match option_env!("RS_PUB_KEY") {
    Some(s) => s,
    None => "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=",
};
pub const RELAY_SERVER: &str = match option_env!("RELAY_SERVER") {
    Some(s) => s,
    None => "",
};
pub const API_SERVER: &str = match option_env!("API_SERVER") {
    Some(s) => s,
    None => "",
};
```

注意:const 匹配里**只能写变量绑定**(`Some(s) => s`),不能写字符串字面量模式
(`Some("")`),否则触发 E0015 编译错误。空值在消费点按"不覆盖"处理。

### 2. 子模块 `libs/hbb_common` → `build.rs`

锚点:`fn main() {` 的开头。插入:

```rust
    println!("cargo:rerun-if-env-changed=RENDEZVOUS_SERVER");
    println!("cargo:rerun-if-env-changed=RS_PUB_KEY");
    println!("cargo:rerun-if-env-changed=RELAY_SERVER");
    println!("cargo:rerun-if-env-changed=API_SERVER");
    println!("cargo:rerun-if-changed=protos/rendezvous.proto");
```

说明:一旦声明任何 `rerun-if` 指令,cargo 就不再默认监视整个包目录,
所以必须补上 proto 文件监视,否则 proto 变更不会重新生成代码。

### 3. 主仓库 → `.gitmodules`

把 hbb_common 的 url 指向自己的 fork:

```ini
[submodule "libs/hbb_common"]
	path = libs/hbb_common
	url = https://github.com/<你的用户名>/hbb_common.git
```

### 4. 主仓库 → `src/rendezvous_mediator.rs`(中继服务器)

锚点:`fn get_relay_server`(搜索 `provided_by_rendezvous_server`)。
在"用户 option"与"hbbs 下发"之间插一段:

```rust
    fn get_relay_server(&self, provided_by_rendezvous_server: String) -> String {
        let mut relay_server = Config::get_option("relay-server");
        if relay_server.is_empty() {
            relay_server = config::RELAY_SERVER.to_owned();
        }
        if relay_server.is_empty() {
            relay_server = provided_by_rendezvous_server;
        }
        if relay_server.is_empty() {
            relay_server = crate::increase_port(&self.host, 1);
        }
        relay_server
    }
```

(相对上游只多了第一个 `if` 块;常量为空时自然落入后续回退,行为不变。)

### 5. 主仓库 → `src/common.rs`(API 服务器)

锚点:`fn get_api_server_`(搜索 `get_custom_rendezvous_server(custom)`)。
在 `api` 选项判断之后插入:

```rust
    if !api.is_empty() {
        return api.to_owned();
    }
    if !config::API_SERVER.is_empty() {
        return config::API_SERVER.to_owned();
    }
    let s0 = get_custom_rendezvous_server(custom);
```

(相对上游只多了第二个 `if` 块。)

### 6. 主仓库 → `.github/workflows/flutter-build.yml`

锚点:顶层 `env:` 块末尾(搜索 `SIGN_BASE_URL`)。追加:

```yaml
  # Baked into the client at compile time by libs/hbb_common/src/config.rs; set as repo secrets.
  # The || fallbacks keep the values non-empty when secrets are unset (empty would break the client).
  RENDEZVOUS_SERVER: "${{ secrets.RENDEZVOUS_SERVER || 'rs-ny.rustdesk.com' }}"
  RS_PUB_KEY: "${{ secrets.RS_PUB_KEY || 'OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=' }}"
  RELAY_SERVER: "${{ secrets.RELAY_SERVER }}"
  API_SERVER: "${{ secrets.API_SERVER }}"
```

说明:`RENDEZVOUS_SERVER`/`RS_PUB_KEY` 是"替换默认值"型,Secret 未配置时 GitHub 会传
空字符串,`||` 表达式保证非空;`RELAY_SERVER`/`API_SERVER` 是"可选覆盖"型,空值在
Rust 侧本来就表示不覆盖,无需兜底。

## 三、一次性设置(重新 fork 后需重做)

1. Fork `rustdesk/rustdesk` → `<你的用户名>/rustdesk`
2. Fork `rustdesk/hbb_common` → `<你的用户名>/hbb_common`(保默认名)
3. **Workflow 权限**:仓库 Settings → Actions → General → Workflow permissions →
   选 **Read and write permissions** → Save。新仓库默认只读,不改这里整个 workflow
   会 startup_failure(报 nested job requesting 'contents: write')
4. Settings → Secrets and variables → Actions → 添加第一节表格中的 Secrets
5. 本地克隆并初始化子模块:
   ```bash
   git clone --recurse-submodules https://github.com/<你的用户名>/rustdesk.git
   cd rustdesk && git submodule sync
   ```

## 四、例行同步上游流程

前提:远程 `origin` = 自己的 fork,`upstream` = 官方仓库。
此流程会重放 fork 上的**所有**私有提交(含本文档之外的功能提交)。

```bash
# 0. 首次添加官方远程
git remote add upstream https://github.com/rustdesk/rustdesk.git

# 1. 取上游最新
git fetch upstream
git checkout master

# 2. 先变基子模块补丁分支
git ls-tree upstream/master libs/hbb_common   # 输出 = 新版上游钉住的 hbb_common 提交 → 记为 <新基>
git ls-tree master libs/hbb_common            # 输出 = 当前钉住的提交 → 记为 <旧基>
cd libs/hbb_common
git checkout custom-server
git fetch https://github.com/rustdesk/hbb_common main   # 新基可能不在 fork 里,须从官方取
git rebase --onto <新基> <旧基>
#    若 config.rs 冲突:按第二节第 1 条的目标状态解决(option_env! 常量 + 保留上游新改动)
git push --force-with-lease origin custom-server
cd ../..

# 3. 再变基主仓库
git rebase upstream/master
#    冲突处理:
#    - libs/hbb_common(指针):让子模块指到新的 custom-server:
cd libs/hbb_common && git checkout custom-server && cd ../..
git add libs/hbb_common
#    - .gitmodules:保留 fork URL
#    - flutter-build.yml:保留上游改动 + 我们的 4 行 env 块
git rebase --continue

# 4. 推送(rebase 改写了历史,需强制)
git push --force-with-lease origin master
```

推送后 push 事件会自动触发 `flutter-ci.yml` 做编译检查;通过后再到 Actions 页手动
**Run workflow** 触发 `Flutter Nightly Build` 取产物(Release → `nightly` 标签)。
注意:不要在旧 run 页面点 Re-run——re-run 固定在原来的提交上,不包含新代码。

## 五、验证

1. 构建完成后安装客户端,主界面**不填**任何服务器地址
2. 客户端应能自动注册到你的 hbbs(看服务器日志)
3. 设置页 ID/中继/API 四个字段显示为空是正常的——烧入值不在 UI 中展示
4. 改服务器地址/公钥时:只改 Secrets,重新触发构建即可,代码无需动

## 六、故障排查

| 症状 | 原因与处理 |
|---|---|
| run 显示 startup_failure、0 个 job | Workflow permissions 未设为读写(第三节第 3 条)。特征报错:nested job requesting 'contents: write' |
| 所有平台都在 Build rustdesk lib 步骤失败,exit 101 | hbb_common 编译错误。典型:E0015 cannot match on \`str\` in constants——const 匹配里写了字符串字面量模式,只能写变量绑定(第二节第 1 条) |
| checkout 阶段拉取 submodule 失败 | hbb_common fork 缺 `custom-server` 分支,或主仓库指针指向 fork 里不存在的提交。必须先推子模块、再推主仓库 |
| 构建成功但客户端连的是官方服务器 | Secrets 未配置或名字拼错。缺值不会报错,workflow 的 `\|\|` 兜底会让构建静默使用官方默认值 |
| 登录/地址簿不可用,但 ID 服务器正常 | 未设 `API_SERVER`。上游的 API 推导只读运行时选项,不读烧入的 ID 常量,不设会落到官方 |
| vcpkg / Cache service 警告 | Azure 缓存侧临时故障,依赖会自动改从源码编译,可忽略 |

## 七、隐私边界

- 仓库内容(含本文档)无任何私有地址;地址只存在 GitHub Secrets,日志中自动打码,
  也不会暴露给他人的 fork PR
- 编译产物必然内嵌地址与公钥,拿到安装包的人可用 `strings` 等工具提取
- 端口约定:21114 API、21115/21116 ID(hbbs)、21117 中继(hbbr)、21118/21119 WebSocket

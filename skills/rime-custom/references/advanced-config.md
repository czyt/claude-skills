# Rime 配置深度参考

本文档包含 Rime 输入法的高级配置细节。

---

## Schema 文件结构

完整的 schema 文件包含以下主要节点：

```yaml
# 方案名称（必需）
schema:
  name: 方案名
  schema_id: 方案ID
  version: 版本号

# 开关列表
switches:
  - name: ascii_mode      # 中英文开关
    reset: 0
    states: ["中文", "西文"]
  - name: full_shape      # 全半角
    reset: 0
    states: ["半角", "全角"]
  - name: zh_simp         # 繁简转换
    reset: 1
    states: ["漢字", "汉字"]
  - name: ascii_punct     # 标点符号
    reset: 0
    states: ["。，", "．，"]

# 引擎结构
engine:
  processors:    # 处理按键输入
    - ascii_composer
    - recognizer
    - key_binder
    - speller
    - punctuator
    - selector
    - navigator
    - express_editor

  segmentors:    # 切分输入码
    - ascii_segmentor
    - matcher
    - abc_segmentor
    - punct_segmentor
    - script_segmentor

  translators:   # 翻译输入码为候选
    - echo_translator
    - punct_translator
    - script_translator
    - table_translator@custom_table
    - reverse_lookup_translator

  filters:       # 过滤候选词
    - simplifier
    - uniquifier
    - charset_filter
    - lua_filter@my_filter

# 拼音拼写器
speller:
  alphabet: zyxwvutsrqponmlkjihgfedcba  # 有效字母
  delimiter: " '"                        # 分隔符
  algebra:                                # 拼写运算规则
    - erase/^xx$/
    - abbrev/^([a-z]).+$/$1/

# 翻译器配置
translator:
  dictionary: rime_ice        # 词库
  prism: rime_ice             # 棱镜（拼音映射）
  preedit_format:             # 预编辑格式
    - xlit/abc/def/
  comment_format:             # 注释格式
    - xform/([nl])v/$1ü/

# 反查翻译器
reverse_lookup:
  dictionary: stroke
  prefix: "v"
  suffix: "'"
  tips: "〔笔画〕"

# 标点符号
punctuator:
  import_preset: default
  full_shape:
    "," : {commit: "，"}
    "." : {commit: "。"}
  half_shape:
    "," : {commit: "，"}
    "." : {commit: "."}
  symbols:
    "/star": ["★", "☆"]

# 按键绑定
key_binder:
  import_preset: default
  bindings:
    - {accept: "Tab", send: "Page_Down", when: has_menu}

# 识别器
recognizer:
  import_preset: default
  patterns:
    punct: "^/[0-9a-z]*$"
    reverse_lookup: "^v[a-z]*$"

# 外观
style:
  color_scheme: native
  horizontal: true
  font_face: "Microsoft YaHei"
  font_point: 14
```

---

## 引擎组件详解

### Processors（按键处理器）

| 处理器 | 功能 | 常用配置 |
|--------|------|---------|
| `ascii_composer` | 中英文状态切换 | `good_old_caps_lock: true` |
| `recognizer` | 识别特殊输入模式 | patterns 正则 |
| `key_binder` | 按键重映射 | bindings 列表 |
| `speller` | 拼写处理 | alphabet, delimiter |
| `punctuator` | 标点符号 | symbols 映射 |
| `selector` | 候选选择 | select_keys |
| `navigator` | 光标移动 | - |
| `express_editor` | 编辑器 | - |

### Segmentors（切分器）

| 切分器 | 功能 |
|--------|------|
| `ascii_segmentor` | 切分 ASCII 文本 |
| `matcher` | 匹配正则模式 |
| `abc_segmentor` | 切分字母文本 |
| `punct_segmentor` | 切分标点 |
| `script_segmentor` | 切分脚本 |

### Translators（翻译器）

| 翻译器 | 功能 | 常用参数 |
|--------|------|---------|
| `echo_translator` | 回显输入码 | - |
| `punct_translator` | 标点翻译 | - |
| `script_translator` | 拼音翻译 | dictionary, prism |
| `table_translator` | 码表翻译 | dictionary |
| `reverse_lookup_translator` | 反查翻译 | dictionary, prefix |
| `lua_translator@name` | Lua 翻译器 | 在 rime.lua 定义 |
| `grammar_translator` | 语法翻译 | language_model |

### Filters（过滤器）

| 过滤器 | 功能 | 常用参数 |
|--------|------|---------|
| `simplifier` | 繁简转换 | opencc_config |
| `uniquifier` | 唯一化候选 | - |
| `charset_filter` | 字符集过滤 | - |
| `cjk_min_filter` | CJK 最小化 | - |
| `lua_filter@name` | Lua 过滤器 | 在 rime.lua 定义 |
| `emoji_suggestion` | Emoji 建议 | - |

---

## 雾凇拼音(rime-ice) 特色配置

雾凇拼音是目前最流行的简体拼音方案，由 iDvel 开发维护。

### 特色功能

- **丰富的词库**：基础词库 + 扩展词库 + 腾讯词向量，共 100万+ 词条
- **内置 Emoji 支持**：纯手搓 Emoji 映射
- **智能纠错**：拼写容错、模糊音
- **英文输入优化**：20k 常见单词 + 扩展词库，中英混输
- **拆字反查**：`uU` + 拼音，拆字辅码 `拼音 + `` + 拆字辅码`
- **以词定字**：左右中括号 `[`、`]`
- **Unicode 输入**：`U` + Unicode 码位
- **数字大写**：`R` + 数字
- **日期时间**：内置 date_translator
- **农历**：`N` + 八位数字；全拼 `nl`，双拼 `lunar`
- **简易计算器**：`cC` + 算式
- **UUID**：`uuid`
- **特殊符号**：全拼 `v` + 首字母缩写；双拼 `V` + 首字母缩写

### 版本要求

- librime ≥ 1.8.5
- 含有 librime-lua 依赖
- 小狼毫 ≥ 0.15.0
- 鼠须管 ≥ 1.0.0

### 词库结构

| 词库 | 内容 | 说明 |
|------|------|------|
| `cn_dicts/base` | 基础词库 | 两字词 + 调频 |
| `cn_dicts/ext` | 扩展词库 | 多音字注音 |
| `cn_dicts/tencent` | 腾讯词向量 | 大词库，自动注音 |
| `en_dicts/en` | 英文词库 | 20k 常见单词 |
| `en_dicts/en_ext` | 英文扩展 | 缩写 + 互联网相关 |

### 常用覆写配置

```yaml
# rime_ice.custom.yaml
patch:
  # 候选词数量
  "menu/page_size": 9

  # 外观
  "style/horizontal": true
  "style/color_scheme": native

  # 模糊拼音（追加规则）
  'speller/algebra/@before 0':
    - derive/^([zcs])h/$1/  # zh/z 模糊
    - derive/([aei])n$/$1ng/  # en/eng 模糊

  # 启用 Lua 扩展
  'engine/translators/@next': lua_translator@time_translator

  # 简繁开关
  switches/@next:
    name: zh_simp
    reset: 1
    states: ["漢字", "汉字"]
```

### 安装方式

```bash
# Git 安装（推荐）
git clone https://github.com/iDvel/rime-ice.git Rime --depth 1

# 东风破安装
bash rime-install iDvel/rime-ice:others/recipes/full

# 双拼补丁（小鹤双拼）
bash rime-install iDvel/rime-ice:others/recipes/config:schema=flypy
```

---

## 白霜拼音(rime-frost) 特色配置

白霜拼音由 gaboolic 开发，基于雾凇拼音修改，专注于词频优化。

### 特色功能

- **词频优化**：基于 745396750 字高质量语料重新统计
- **精简词库**：删除冷僻词、废词、不健康词汇
- **辅助码支持**：` 开启墨奇辅助码
- **带调韵母**：`/a` `/e` `/u` 等快速输入
- **符号输入**：`/fh` 更多符号

### 触发指令

| 指令 | 功能 |
|------|------|
| `/fh` | 符号 |
| `/a` `/e` `/u` | 带调韵母 |
| `rq` `sj` `xq` `dt` `ts` | 日期时间 |
| `` ` `` | 开启辅助码 |
| `uU` | 部件拆字反查 |
| `U` | Unicode |
| `R` | 数字金额大写 |
| `N` | 农历 |
| `V` | 计算器 |

### 双拼支持

- 自然码双拼
- 小鹤双拼
- 微软双拼
- 搜狗双拼

### 安装方式

```bash
# Git 安装
git clone --depth 1 https://github.com/gaboolic/rime-frost Rime

# 东风破安装
bash rime-install gaboolic/rime-frost:others/recipes/full
```

---

## 万象拼音(rime_wanxiang) 特色配置

万象拼音由 amzxyz 开发，支持繁简混输、多语言、语法模型。

### 版本对比

| 特性 | 标准版 | 增强版 Pro |
|------|--------|-----------|
| 方案文件 | `wanxiang.schema.yaml` | `wanxiang_pro.schema.yaml` |
| 支持类型 | 全拼、任意双拼 | 仅双拼 |
| 自动调频 | 开启 | 关闭 |
| 用户词记录 | 自动积累 | 手动造词 ` `` ` 引导 |
| 辅助码 | 仅声调 | 7 种辅助码可选 |

### 核心功能

- **全词库带调拼音**：多种带调方案、声调注释
- **深度反查**：两分、多分、笔画，支持 Unicode 17
- **Lua 扩展**：符号包裹、Tips 显示、手动排序、输入统计
- **辅码兼容**：墨奇码、鹤形、自然码、虎码等 8 种
- **语法模型**：支持 kenlm 模型，整句预测

### 切换指令

```
/flypy    → 小鹤双拼
/mspy     → 微软双拼
/zrm      → 自然码
/sogou    → 搜狗双拼
/znabc    → 智能ABC
/ziguang  → 紫光双拼
/pinyin   → 全拼
/wxsp     → 万象双拼
```

### 语法模型安装

下载语法模型文件，放置于 Rime 用户文件夹根目录即可。

---

## 薄荷输入法 特色配置

薄荷输入法专注于新手友好体验，由 Mintimate 开发。

### 特色功能

- 简单的安装流程
- 清晰的配置说明
- 默认配置即可使用
- 提供 MCP 知识库查询服务

---

## 参考资料

- 官方 Wiki：https://github.com/rime/home/wiki
- 配置详解：https://github.com/LEOYoon-Tsaw/Rime_collections/blob/master/Rime_description.md
- 雾凇拼音：https://dvel.me/posts/rime-ice/
- 万象拼音：https://github.com/amzxyz/rime_wanxiang
- 白霜拼音：https://github.com/gaboolic/rime-frost
- 薄荷输入法：https://www.mintimate.cc
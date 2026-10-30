# 猛哥的空间站 · 交接说明

写给接手的人。看完这一份就能改、能验、能发版。

---

## 1. 这是什么

个人静态站，纯前端，零框架零构建。改完刷新就生效。

- 线上：https://www.wmv587.asia
- 仓库：`wangm9277-a11y/site`
- Pages 发布源：`gh-pages` 分支；`main` 只是镜像。**两个分支必须保持同步**，否则仓库和线上会对不上。

## 2. 目录

```
index.html          首页（约 57KB，样式脚本全内联）
ktv.html            歌单，localStorage 存储，编辑密码 987459889（仅前端校验）
translate.html      五语互译（中英日维蒙），在线走 pollinations
calc.html           计算器，支持全键盘
tools/qianqian.html 乾乾学易（338KB 首屏轻量壳 + 异步详注包，见 §8）
tools/qianqian-gua.json  乾乾学易 02–64 卦详注数据包（约 745KB）
tools/qianqian-orig.json 乾乾学易 原书注释隐藏包（约 828KB，按需加载）
assets/*.webp       图：sky 背景、thumb-* 作品缩略图（640×400）
tools/*.mjs         开发工具：截图 / 量尺寸 / 冒烟测试
tools/api-commit3.py 发版脚本（见 §5）
```

## 3. 本地预览

```bash
cd 项目目录
python -m http.server 8899
# http://127.0.0.1:8899
```

## 4. 改完怎么验（必须验，别只看代码就说完成）

截图脚本依赖 Edge 路径，**环境变量默认没有，每次都要设**：

```bash
export EDGE_PATH="C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"

# 桌面首屏
node tools/shot.mjs 1440 900 out.png "http://127.0.0.1:8899/index.html?nointro"

# 手机（CDP 精确视口，不是缩放模拟）
node tools/shot.mjs 390 844 out.png "http://127.0.0.1:8899/index.html?nointro" mobile

# 想抓动画中间帧：SHOT_WAIT=毫秒（默认 3400）
SHOT_WAIT=1200 node tools/shot.mjs 1440 900 mid.png "http://127.0.0.1:8899/index.html"
```

其他两个工具：

```bash
# 量任意元素的盒模型：url 宽 高 [mobile] [选择器,选择器]
node tools/measure.mjs "http://127.0.0.1:8899/index.html?nointro" 1440 900 desktop ".scene,.island,.bot"

# 四页冒烟：抓 JS 异常、控制台报错、坏图、主题色
node tools/smoke.mjs "http://127.0.0.1:8899" 1440 900
```

`smoke.mjs` 输出 `4/4 通过` 才算没打破东西。

## 5. 发版（重要：不要用 git push）

这台机器访问 github.com 的 git 协议经常被重置，**一律走 GitHub API**：

```bash
python tools/api-commit3.py <token> gh-pages "提交说明"
python tools/api-commit3.py <token> main "提交说明"
```

脚本做的事：遍历固定文件清单 → 建 blob → 建 tree → 提交 → 移动分支指针。推送清单在脚本第 20 行附近：

```python
for p in ['index.html', 'ktv.html', 'translate.html', 'calc.html', 'tools/qianqian.html', 'tools/qianqian-gua.json', 'tools/qianqian-orig.json']:
# 之后追加 assets/*.webp 全部
```

**新增页面/资源必须手动加进这个清单，否则不会发布。**（乾乾学易曾经因为不在清单里，线上的文件被截断成半个都没人发现。）

Pages 一般 1–2 分钟生效。验证：

```bash
curl -sI https://www.wmv587.asia/index.html | grep -i content-length
```

## 6. 设计系统

底色近黑，主强调色**跟着选中的作品卡切换**，不是跟着滚动：

| 卡片 | 色值 |
|---|---|
| 乾乾学易（默认） | `#b9a4ff` |
| KTV | `#c8f830` |
| 在线翻译 | `#7fe3ff` |
| 计算器 | `#ffc46b` |
| 离线翻译 | `#5ee0a8` |

机制：`<a class="card" data-bg data-ac data-tint>`，JS 里 `pick()` 把 `--lime`、`--tint`、`#theme` 的 background 一起改。改一个 `data-*` 就换主题。

卡片选中态：`translateY(-9px) scale(1.035)` + 柔光；未选 `opacity .62`。

大标题全大写 `MENGGE / KONGJIANZHAN`，字重 900，紧字距。

## 7. 首屏机器人

**是内联 SVG，不是图片。不要再换回 PNG。** 之前用生成图，结果只能整体挤压，不能眨眼不能追指针。

结构分三层，因为两个动画同时改 `transform` 会打架：

```
.bot            入场动画 + 点击/键盘事件
  .bot-float    7s 循环浮动
    svg > g.rig 点击挤压回弹
```

交互：入场淡入 → 常驻浮动 → 随机眨眼（2.6–6s）→ 瞳孔跟指针 → 点击挤压 + 天线脉冲。`prefers-reduced-motion` 下全静。

## 8. 乾乾学易（重点，有坑）

1.9MB 单文件独立应用，内容在 `tools/qianqian.html`。**源码只在这里**，别去 `D:\dsh\site` 或 `site-repo` 找，那是历史副本。

已知问题：本机实测对该域名的下载速度只有 20–30 KB/s，1.9MB 未压缩要 40–90 秒，gzip 后 783KB 约 25 秒。原本选择框的点击事件绑在 `DOMContentLoaded` 上，整页没解析完就没有反应 → 用户以为「页面坏了」。

已打补丁：在选择框 markup 之后立刻绑一个轻量 handler，点击马上亮起「· 载入中」，等整页解析完再 `applyLaunchPreset()` 并移除遮罩。补丁位置：

- 早绑定脚本：搜索 `早绑定：整页 1.9MB`
- 补丁生效处：`function initApp()` 里的 `if (window.__pendingOrient)`

**根治方案**（未做）：把 1.2MB 卦象文本拆成分片 JSON 按需加载，或至少做一次压缩。这是重构，动之前先确认。

## 9. 外链开关

首页里乾乾学易的入口预留了外链切换：

```js
var EXT = {
  qianqian: ''   // 填 'https://xxx' 就自动变外链、新窗口打开
};
```

留空 = 走站内 `/tools/qianqian.html`。卡带和「精选作品」两处都会跟着变。

## 10. 待办

1. 移动端「空岛 ↔ 卡带」之间的空档：实测 CSS 间距只有 7.7px，视觉上的空白来自 SVG 底部 44 单位透明留白 + 24px padding，合计约 53px。要收紧改 `.deck-wrap` 和 `.deck` 的 padding-top。
2. 作品卡带 5 张，考虑加「更多作品」或分组。
3. Pages 发布源可以改成 `main` 简化流程，改完记得同步另一分支。
4. 乾乾学易拆包（见 §8）。

## 11. 别踩这些

- **`EDGE_PATH` 没设**，跑 shot/eval 会直接报找不到 Edge。
- **`D:\dsh\site-repo` 是陈旧克隆**（只有 2 次提交、只有 main、没有 gh-pages），别当源。
- **`D:\dsh\site` 是早期副本**，只有 `tools/qianqian.html` 有价值（已复制进主工作副本）。
- **工作副本根目录堆了几十张调试截图**（c1–c5、d5–d11、e1…），不进提交也不进包，但会碍眼。
- **Node 是 22**，不是 24。Python 3.13。
- 本机删除文件会被 safe-delete 中间层改名成 `*.moved-<时间戳>` 而不是真删，看到这种目录不要惊讶。

## 12. 凭据

发版用的 token 是 fine-grained PAT，**只对 `wangm9277-a11y/site` 一个仓库有 Contents 读写**。不在仓库里，也不在这份文档里，由站点主人单独提供。

接手后建议：让站点主人新建一个 token 给你，交接完成后把旧的撤销。

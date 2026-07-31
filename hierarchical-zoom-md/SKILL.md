Skill: hierarchical-zoom-md（Markdown 层级 + Zoom 单页导航）
用途

将任意内容整理为标准 Markdown 标题层级（#、##、###…），并生成一个自包含的 zoom.html。

最终目录：

{output-dir}/
└── {topic-slug}/
    └── zoom.html

双击 zoom.html 即可浏览，无需服务器、无需外部文件。

内容规范
Markdown
使用标准 Markdown 标题表示层级（#~######）
每个标题下紧跟 1~3 句摘要
父节点摘要应概括所有子节点
同层保持互斥、平级
每层控制 3~7 个子节点
递归拆分直到叶子节点无需继续拆解
输出要求

生成：

zoom.html

要求：

Markdown 内嵌在 HTML 中
不得读取外部 .md 文件
不使用 fetch、iframe、XMLHttpRequest
浏览器直接 file:// 双击即可打开
HTML 模板要求

Markdown 放在 HTML 最上面或最下面，方便修改，例如：

<script>
const MARKDOWN_CONTENT = `
# ...
`;
</script>

其余 HTML 负责：

解析 Markdown 标题层级
构建树结构
面包屑导航
子节点卡片
点击进入子节点
返回父节点

所有逻辑均内嵌于单文件，不依赖任何外部资源。

执行流程
阅读输入内容
构建符合规范的 Markdown 层级
将 Markdown 写入 MARKDOWN_CONTENT
使用固定 HTML 模板生成 zoom.html
保存至
{output-dir}/{topic-slug}/zoom.html
HTML 模板

使用固定模板生成页面。

要求：

Markdown 放在整个 HTML 最上方（推荐）或最下方
HTML 为单文件
不修改页面交互逻辑
仅替换 MARKDOWN_CONTENT
本地运行要求

必须满足：

双击 zoom.html
file://
无服务器
无网络
无外部 Markdown
无任何第三方依赖

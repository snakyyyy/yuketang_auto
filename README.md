# yuketang_auto
长江雨课堂 自动视频播放

一个运行在 Tampermonkey 中的 UserScript，用于长江雨课堂 AI Workspace 页面：读取“完成度/已完成”文本，结合 video 元素状态判断视频是否播完，播完后自动切换下一节；支持跳过答题/考试等非视频页；支持无进度文本且暂停视频的防卡死跳过。

声明：仅供 Tampermonkey、DOM 轮询和前端自动化学习交流。请遵守平台用户协议和课程要求，勿用于刷课、作弊或规避考核。


一、功能介绍
========

自动切换：查找包含“完成度”的 span.text 或 span.el-tooltip.text，解析百分比；达到 100% 或文本含“已完成”时，点击 .unit-arrow.arrow-reverse 切换下一节。

判断条件：读取不到进度文本时，若 video.ended 为真，或 video.currentTime >= video.duration - 1，判定视频基本播完。

非视频页跳过：页面既无进度文本也无 video 时，视为答题/考试等页面；存在“下一个”按钮则点击并等待。

防卡死跳过：无进度文本但有视频时，若视频暂停，连续 3 次检测，约 15 秒后强制尝试下一个；若视频正在播放且 readyState > 2，继续等待，不打扰。

播放兜底：优先调用 video.play()；失败或被拦截时，依次尝试点击页面播放按钮选择器。

日志面板：右下角悬浮显示最新状态，完整日志输出到控制台；面板设置 pointer-events: none，不挡页面点击。

防重入：使用 switching 标志避免切换过程中重复点击。


二、需求依赖
========

浏览器安装 Tampermonkey。

匹配地址：*://changjiang.yuketang.cn/ai-workspace/*

权限：@grant none；运行时机：@run-at document-idle。

依赖页面现有 DOM 结构和类名，平台前端改版后选择器可能失效。

浏览器自动播放策略可能拦截 video.play()，需要页面已有用户交互。


三、使用步骤
========

安装 Tampermonkey 扩展。

在 Tampermonkey 中新建脚本，粘贴完整代码并保存。

确认脚本已启用。

打开 https://changjiang.yuketang.cn/ai-workspace/*

页面右下角出现悬浮面板即表示运行；详细日志可在浏览器控制台查看。


四、核心判断逻辑
========

脚本采用“DOM 进度文本优先，视频底层状态兜底”的策略。

无进度文本时，用视频状态兜底判断播完：

if (!progress && video && video.duration > 0) {
if (video.ended || video.currentTime >= video.duration - 1) {
progress = { done: true, raw: '视频已播完（底层检测）' };
}
}

无进度文本但有视频时，判断是否还在正常推进：

if (!video.paused && !video.ended && video.readyState > 2) {
noProgressCount = 0;
return;
}
noProgressCount++;
if (noProgressCount >= CONFIG.MAX_NO_PROGRESS_COUNT) {
progress = { done: true, raw: '强制跳过（防卡死）' };
}





五、已知限制
========

强依赖页面 DOM 结构和类名，平台改版后可能失效。

只处理当前页面中的 video，未处理 iframe、Shadow DOM、多视频实例。

duration > 0 不能排除 Infinity；更严谨可写 Number.isFinite(video.duration) && video.duration > 0。

currentTime >= duration - 1 是启发式判断，可能提前约 1 秒切换。

paused 为 false 且 readyState > 2 不能证明 currentTime 在增长，缓冲时可能一直等待。

防卡死逻辑按“暂停次数”计数，可能把正常暂停或缓冲误判为卡死。


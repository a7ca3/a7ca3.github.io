---
<%*
// 设置文章标题
let title = tp.file.title;
if (title.startsWith("未命名")) {
 title = await tp.system.prompt("请输入文章标题");
 if(!title) return;
}
if (title == "") {
title = "未命名";
} else {
await tp.file.rename(title);
}
await tp.file.move("/post/" + title)
-%>
title: "<% title %>"
date: 2025-03-25 11:11
categories:
tags:
---

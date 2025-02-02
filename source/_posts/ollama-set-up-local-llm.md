---
title: ollama搭载本地大语言模型
date: 2025-02-03 04:23:21
cover: cover.jpg
tags:
  - llm
  - ollama
---

## 安装ollama
到[ollama下载地址](https://ollama.com/download)选择对应的操作系统版本进行安装
![ollama下载](ollama-download.png)

终端检查是否加载完成，如果有版本号输出，则表示安装成功
```
ollama -v
```

## 安装本地大语言模型
我这里使用的是deepseek-r1 7b的模型
```
ollama run deepseek-r1
```

## 可视化webui
1. 下载docker桌面版本
2. 打开docker应用
3. 进入[open-webui](https://docs.openwebui.com/)
![open-webui](open-webui-doc.png)
4. 选择 **If Ollama is on your computer, use this command** 运行
```
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```
5. 点击docker上对应open-webui的容器，点击**Ports**链接，即可访问webui。
![webui](docker-ui.png)
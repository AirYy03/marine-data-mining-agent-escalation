# 本课程仓库的发布范围与后续维护

## 固定发布目标

- **课程：海洋数据挖掘。**
- **唯一仓库：** [https://github.com/AirYy03/marine-data-mining-agent-escalation](https://github.com/AirYy03/marine-data-mining-agent-escalation)。
- **可见性：** Public；用于展示本次课程研究文档。
- **聊天分享：** [查看本次课程讨论](https://chatgpt.com/share/6af1d7d9-e370-83ee-8f11-7b318c0ef676)。

本次用户已创建独立公开仓库并提供分享链接。后续维护只针对这一仓库，不需要调整其他项目的可见性，不应导入其他项目或代码。

## 文档上传白名单

```text
README.md
report.md
SUBMISSION_CHECKLIST.md
UPLOAD_GUIDE.md
SHA256SUMS.txt
.gitignore
docs/research_process.md
docs/conversation.md
docs/references.md
```

本仓库没有可运行的机械臂控制代码或实测数据。报告中的伪代码用于说明拟研究流程，不应直接用于真实机械臂控制。

## 后续更新原则

仅在上述课程文件内更新文档；修改实验状态必须有真实日志依据，不能把示例参数或讨论预期写成实测结论。后续新增完整聊天导出、实验结果或图像时，应逐项确认其属于本课程公开范围，再明确加入清单。

不要从原有机器人工作空间整体上传，不使用 fork、mirror、仓库导入或子模块暴露其他工程。不要上传凭证、个人身份信息、模型权重、传感器原始数据及未授权代码。`.gitignore` 是辅助防线，不能代替检查实际提交列表。

## 文件完整性

`SHA256SUMS.txt` 记录本次文档版本的 SHA-256 散列（不包含其自身）。修改文件后应重新生成清单。在包含该清单的目录运行 `sha256sum -c SHA256SUMS.txt` 可核对本地文件副本是否一致。

课程实测结果和收益预测模型仍未提供；本次发布只完成文档及分享入口的整理。

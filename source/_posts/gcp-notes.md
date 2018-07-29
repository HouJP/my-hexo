---
title: Google Cloud Platform Notes
date: 2018-07-29 09:42:50
tags: [google-cloud-platform]
---

### 防火墙配置

```Shell
# 检查现有规则
gcloud compute firewall-rules list
# 开放ssh端口
gcloud compute firewall-rules create default-allow-ssh --allow tcp:22
```

<!-- more -->
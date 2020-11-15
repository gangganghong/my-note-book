---
description: 在mac上验证过的命令
---

# 常用linux命令

### 用grep找出并杀死进程

```text
# grep -v 'grep' 排除包含grep自身的进程
ps -ef | grep qemu  | grep -v 'grep' | awk '{print $2}' | xargs kill -9
```


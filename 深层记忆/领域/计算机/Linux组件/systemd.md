父级：[[Linux]]
# 详细吃透 Linux systemd（通俗+原理+核心功能+常用命令）
## 一、什么是 systemd
### 1. 定义
`systemd` 是 **Linux 系统的系统和服务管理器**，是目前绝大多数 Linux 发行版（Ubuntu、Debian、CentOS 7+、Arch、Fedora）**默认的初始化系统**。

它替代了古老的 **SysVinit**（传统 `/etc/init.d` 那种启动方式），**负责系统开机启动、进程管理、服务管理、日志管理、挂载、网络、定时任务** 等几乎所有系统底层工作。

### 2. 核心地位
- Linux 内核启动完成后，**第一个用户态进程就是 systemd**，PID = 1；
- 所有后续进程、服务，理论上都由它派生或被它管理；
- 是整个系统的**总管家**。

---

## 二、为什么要搞出 systemd？（对比老式 SysVinit）
### 老式 SysVinit 缺点
1. **串行启动**：一个服务启动完，再启动下一个，开机很慢；
2. 全靠 Shell 脚本，没有统一标准，管理混乱；
3. 不能自动重启崩溃服务、没有依赖管理；
4. 不统一管理日志、挂载、网络、定时任务。

### systemd 优势
1. **并行启动服务**：能同时启动无依赖的服务，**开机速度大幅变快**；
2. **统一管理一切**：服务、挂载、磁盘、网络、日志、定时、设备全都管；
3. **依赖关系管理**：比如必须先联网再启动 Nginx，自动按顺序来；
4. **自动守护**：服务崩溃可以**自动重启**；
5. **标准化单元文件**：不用写复杂 Shell 脚本，配置简单；
6. **统一日志系统**：自带 `journald`，集中存所有系统和服务日志。

---

## 三、systemd 核心概念：Unit（单元）
systemd 不只是管「服务」，它把系统所有资源都抽象成 **Unit 单元**，每种单元后缀不同。

常见 Unit 类型：

| 后缀         | 类型     | 作用                          |
| ---------- | ------ | --------------------------- |
| `.service` | 服务单元   | 我们常说的系统服务（Nginx、SSH、Docker） |
| `.mount`   | 挂载单元   | 磁盘、目录自动挂载                   |
| `.target`  | 目标单元   | 相当于「运行级别」，分组批量启动服务          |
| `.socket`  | 套接字单元  | 端口监听，有请求再启动服务               |
| `.timer`   | 定时单元   | 替代 crontab，定时执行任务           |
| `.device`  | 设备单元   | 管理硬件设备                      |
| `.swap`    | 交换分区单元 | 管理 swap 分区                  |

简单理解：**systemd 眼里，系统所有东西都是一个个 Unit**。

---

## 四、systemd 服务文件存放路径
1. **系统自带服务**（软件安装自带）
```
/usr/lib/systemd/system/
```
2. **管理员自定义服务**（自己写的 .service）
```
/etc/systemd/system/
```
优先级：**/etc 高于 /usr/lib**，同名会覆盖。

---

## 五、核心运行机制
1. 内核启动后，执行 `/sbin/init` 链接到 `systemd`，PID=1；
2. systemd 读取所有 Unit 配置；
3. 根据 `.target` 目标（如 graphical.target 图形桌面、multi-user.target 命令行）**并行启动所有依赖服务**；
4. 全程监控服务状态：崩溃重启、资源限制、依赖校验；
5. 所有日志统一交给 `journald` 记录。

---

## 六、systemd 最常用核心命令
### 1. 服务管理
```bash
# 查看服务状态
systemctl status sshd

# 启动/停止/重启
systemctl start nginx
systemctl stop nginx
systemctl restart nginx

# 设置开机自启 / 取消开机自启
systemctl enable nginx
systemctl disable nginx

# 重新加载配置（改了 .service 文件后必须执行）
systemctl daemon-reload
```

### 2. 查看所有单元/服务
```bash
# 列出所有已启动服务
systemctl list-units --type=service

# 列出所有已安装服务（包括没启动的）
systemctl list-unit-files --type=service
```

### 3. 系统开关机
```bash
systemctl poweroff   # 关机
systemctl reboot    # 重启
systemctl suspend   # 睡眠
systemctl hibernate # 休眠
```

### 4. 日志查看（journald）
```bash
# 看系统全部日志
journalctl

# 看某个服务日志
journalctl -u nginx

# 实时滚动查看
journalctl -u nginx -f
```

---

## 七、运行级别 → systemd target 对应
老式 Linux 有 0~6 运行级别，systemd 用 target 替代：

| 老式运行级别 | systemd target    | 含义      |
| ------ | ----------------- | ------- |
| 0      | poweroff.target   | 关机      |
| 3      | multi-user.target | 纯命令行多用户 |
| 5      | graphical.target  | 图形桌面    |
| 6      | reboot.target     | 重启      |

切换命令：
```bash
# 切到命令行
systemctl isolate multi-user.target

# 切到图形桌面
systemctl isolate graphical.target
```

---

## 八、systemd 的优缺点总结
### 优点
- 开机并行启动，速度快；
- 统一管理服务、日志、挂载、定时、设备；
- 配置标准化，不用写繁琐启动脚本；
- 自带守护、自动重启、依赖管理；
- 命令统一，所有发行版操作一致。

### 缺点
- **体量庞大**，包揽太多功能，被很多极简派诟病臃肿；
- 设计中心化，违背 Linux 「一个工具只做一件事」传统哲学；
- 日志、配置都集中，和老式传统流程差异大，老用户需要适应。

---

## 九、一句话极简总结
**systemd 是 Linux 系统的一号进程、系统总管家，替代了老式开机脚本，统一管理开机流程、系统服务、日志、挂载、定时、硬件设备，实现并行开机、服务自启自愈、标准化配置，是现在绝大多数 Linux 桌面/服务器的底层核心。**
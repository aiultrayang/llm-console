# dsh-console — vLLM / SGLang 推理服务管理控制台

单文件 Node.js 控制台（无 npm 依赖），用于在裸金属 GPU 服务器上统一管理 vLLM / SGLang 推理服务：
启动/停止模型、按端口看实时日志、GPU/显存/功耗/磁盘监控、投机解码（DFlash2 / NEXTN / MTP）参数预设。

## 文件结构

```
server.js     后端（单文件，无依赖，Node >= 18）
index.html    前端（单页，Tailwind 走本地 static/）
static/       tailwind.js + 字体（部署时不可漏，否则页面无样式）
```

## 部署（3 步）

1. 上传三件套到目标机 `~/dsh-console/`（server.js + index.html + static/，**缺一不可**）
2. systemd user 服务：

```ini
[Service]
ExecStart=<node路径> <home>/dsh-console/server.js
WorkingDirectory=<home>/dsh-console
Environment=DSH_MODELS_DIR=<模型目录>
Environment=DSH_SGLANG_VENV=<sglang venv>
Environment=DSH_VLLM_ENV=<vllm venv>
```

3. `systemctl --user daemon-reload && systemctl --user enable --now dsh-console`

监听 `0.0.0.0:8889`，如需外网访问用 frp 加一条 http 子域代理指向 8889 即可。

## 环境变量

| 变量 | 说明 | 默认 |
|---|---|---|
| `DSH_MODELS_DIR` | 模型根目录 | 自动探测 `~/models`、`/mnt/ssd/models` |
| `DSH_SGLANG_VENV` | sglang venv 路径 | 自动探测 `~/sglang-latest-venv`、`/mnt/ssd/sglang-latest-venv` |
| `DSH_VLLM_ENV` | vllm venv 路径 | 自动探测 `~/qwen38-venv`、`/mnt/ssd/qwen38-venv` |
| `VLLM_PORT` | 强制指定 backend 端口（跳过自动探测） | 自动探测 |

## 已内置的坑修复

- **后端端口探测只信宿主 LISTEN 端口**：容器/chroot 内的 vllm 进程对宿主 `/proc` 可见但端口不可达（netns 隔离），不过滤会把 backend 指到假端口导致监控全空
- **backend 优先选开了 `--enable-metrics` 的实例**：sglang 不开该参数时 `/metrics` 是 404
- **兼容 sglang 新旧 CLI**：`python -m sglang.launch_server`（老）与 `bin/sglang serve`（新）都能识别
- **GPU 进程按 uuid 归卡**：TP>1 时各 rank 正确归到真实卡号
- **sudo 免密探测 + 降级**：没配 NOPASSWD 的机器 smartctl/dmidecode/RAPL 自动静默降级，不刷日志
- **日志按端口分 Tab**：`vllm-<port>.log` / `sglang-<port>.log`，启动失败也能看到完整报错
- **存活检查 20s 观察窗**：避免慢启动被误判失败，失败按 PID 精确释放 GPU 注册表

## 注意

- sglang 服务启动时加 `--enable-metrics --enable-cache-report`，否则控制台 stats 面板推理指标为空（GPU/显存监控不受影响）
- 升级只替换 `server.js`/`index.html`/`static/` 后 `systemctl --user restart dsh-console` 即可，不影响任何在跑的推理业务

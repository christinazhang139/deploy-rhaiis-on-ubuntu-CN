# Red Hat AI Inference Server 在 Ubuntu + RTX 4090 上的完整部署指南

启动一个本地AI推理服务器，支持GPU加速的模型服务

## 系统环境

- **操作系统**: Ubuntu
- **GPU**: RTX 4090 (24GB VRAM)
- **容器运行时**: Docker 或者 Podman

## 详细指南

### 1. 验证GPU状态

```bash
# 检查GPU是否正常工作
nvidia-smi
```

### 2. 验证Docker GPU支持

```bash
# 测试Docker GPU访问
docker run --rm --gpus all nvidia/cuda:11.8-base-ubuntu20.04 nvidia-smi
```

如果上述命令正常显示GPU信息，说明环境已就绪。

## 第一步：登录并获取镜像

### 登录Red Hat容器注册表

```bash
# 使用Red Hat账户登录
docker login registry.redhat.io
# 输入用户名和密码
```

### 拉取Red Hat AI Inference Server镜像

```bash
# 拉取最新版本镜像 (3.2.0 build 1752784628) - 这个会花费一些时间
# https://catalog.redhat.com/en 最新的版本去这里找 输入关键字 rhaiis 搜索
docker pull registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.0-1752784628
```

## 第二步：准备工作目录和认证

```bash
# 创建缓存目录（用于存储模型）
mkdir -p ~/rhaiis-cache

# 创建配置目录
mkdir -p ~/rhaiis-config

# 设置适当权限（777确保容器内用户可以写入）
chmod 777 ~/rhaiis-cache ~/rhaiis-config

# 设置Hugging Face Token（访问Red Hat模型需要）
export HF_TOKEN="xxxxxxx"
```

## 第三步：超轻量级入门

**推荐用于**: 快速验证环境、首次测试

```bash
export HF_TOKEN="xxxxxx"

docker run -d \
    --gpus all \
    -p 8000:8000 \
    -v ~/rhaiis-cache:/opt/app-root/src/.cache \
    --shm-size=4g \
    --name rhaiis-server \
    --restart unless-stopped \
    -e HUGGING_FACE_HUB_TOKEN=$HF_TOKEN \
    -e CUDA_VISIBLE_DEVICES=0 \
    registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.0-1752784628 \
    --model RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8 \
    --host 0.0.0.0 \
    --port 8000 \
    --max-model-len 4096 \
    --max-num-seqs 32 \
    --tensor-parallel-size 1 \
    --enforce-eager \
    --disable-custom-all-reduce
```

## 验证部署

### 检查服务状态

```bash
# 查看容器运行状态
docker ps

# 查看服务日志
docker logs -f rhaiis-server

# 检查服务健康状态
curl http://localhost:8000/health
```

### 测试API功能

```bash
# 查看可用模型
curl http://localhost:8000/v1/models

# 测试文本生成
curl http://localhost:8000/v1/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "RedHatAI/Llama-3.2-1B-Instruct-quantized.w8a8",
        "prompt": "Hello, I am",
        "max_tokens": 20
    }'
```

## 成功运行的标志

看到以下日志表示部署成功：
```
INFO [api_server.py] vLLM API server version 0.9.2
INFO [config.py] Using max model len 4096
INFO [api_server.py] Started engine process with PID XX
INFO [api_server.py] Uvicorn running on http://0.0.0.0:8000
```

## 管理服务

```bash
# 停止服务
docker stop rhaiis-server

# 重启服务
docker restart rhaiis-server

# 删除服务
docker rm -f rhaiis-server
```

## 故障排除

如果遇到问题：

1. **检查GPU驱动**：确保 `nvidia-smi` 正常工作
2. **检查Docker GPU支持**：确保 Docker 可以访问 GPU
3. **检查容器日志**：使用 `docker logs rhaiis-server` 查看错误信息
4. **检查端口占用**：确保 8000 端口未被占用

---

**注意事项**：
- 首次运行会下载模型文件，需要一些时间
- 确保有足够的磁盘空间存储模型
- RTX 4090 的 24GB VRAM 足够运行大多数量化模型

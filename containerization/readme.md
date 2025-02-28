# ComfyUI 容器化部署说明文档

本文档描述了如何将ComfyUI打包成为Docker Image便于部署.

## 镜像构建过程

1. 首先创建基础依赖镜像的 Dockerfile (可以命名为 `Dockerfile.base`):

```Dockerfile:Dockerfile.base
# 使用官方 Python 3.12 作为基础镜像
FROM python:3.12-slim

# 设置工作目录
WORKDIR /app

# 只复制依赖文件
COPY ./src/requirements.txt .

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 这个基础镜像不需要暴露端口和启动命令
```

2. 然后创建应用镜像的 Dockerfile:

```Dockerfile:Dockerfile

# 使用comfyui-base:latest作为基础镜像, 加快构建速度
FROM devserver:5000/comfyui-base:latest

# 设置工作目录
WORKDIR /app

# 复制当前目录的内容到容器的 /app 目录
COPY ./src/. .

# 安装依赖, 防止有新增的依赖包
RUN pip install -r requirements.txt

EXPOSE 8000
# 运行命令
CMD ["python", "main.py","--port","8000","--listen","0.0.0.0"]
```

使用方法：

1. 首先构建基础镜像：
```bash
docker build -f Dockerfile.base -t devserver:5000/comfyui-base:latest .
docker push devserver:5000/comfyui-base:latest
```

2. 然后构建应用镜像：
```bash
docker build -t devserver:5000/comfyui:1.0.1 .
docker push devserver:5000/comfyui:latest
```

这样做有以下优势：
1. 依赖层被缓存在基础镜像中，只要 requirements.txt 不变，就不需要重新安装依赖
2. 开发过程中，如果只改动了应用代码，构建会更快
3. 多个相关项目可以共用同一个基础镜像
4. 基础镜像可以单独更新和维护

注意事项：
- docker build 之前请clone这个项目到一个 src文件夹, 然后将Dockerfile文件复制到跟src平级的文件夹
- cd ./src/custom_nodes, 随后逐个clone或者pull自定义的node代码
- rm ./src/models 文件夹
- 最后执行 docker build

## 镜像使用说明

### docker host上的模型

在docker run之前, 请确保所有大模型在docker host上指定的目录存在, 本例为 /d/comfyui_models, 其结构应该如下:
```sh
(comfyui) xdf@devserver:/d/comfyui_models$ tree
.
├── checkpoints
│   ├── ccm-diffusion.pth
│   ├── CRM.pth
│   ├── gokaygokay-Flux-Prompt-Enhance
│   │   ├── config.json
│   │   ├── generation_config.json
│   │   ├── model.safetensors
│   │   ├── special_tokens_map.json
│   │   ├── spiece.model
│   │   ├── tokenizer_config.json
│   │   └── tokenizer.json
│   ├── pixel-diffusion.pth
│   ├── v1-5-pruned-emaonly.ckpt
│   └── v1-5-pruned-emaonly-fp16.safetensors
├── clip
├── clip_vision
├── configs
│   ├── anything_v3.yaml
│   ├── v1-inference_clip_skip_2_fp16.yaml
│   ├── v1-inference_clip_skip_2.yaml
│   ├── v1-inference_fp16.yaml
│   ├── v1-inference.yaml
│   ├── v1-inpainting-inference.yaml
│   ├── v2-inference_fp32.yaml
│   ├── v2-inference-v_fp32.yaml
│   ├── v2-inference-v.yaml
│   ├── v2-inference.yaml
│   └── v2-inpainting-inference.yaml
├── controlnet
├── diffusers
├── diffusion_models
├── embeddings
├── gligen
├── hypernetworks
├── loras
├── photomaker
├── style_models
├── text_encoders
├── unet
├── upscale_models
├── vae
└── vae_approx
```

### 关于output

由于容器在重启以后, 容器内部产生的输出文件会消失, 实际我们使用时会有前置的反向代理, 会及时将output中的图片转移到OSS服务中. 根据需要也可以在docker host上保留output文件.

### 容器启动命令参考

```sh
docker run -d \
--gpus all \
--restart unless-stopped \
--name comfyui_9181 \
-p 9181:8000 \
-v /d/comfyui_models:/app/models \
-v /etc/apps/comfyui-docker/output_9181:/app/output \
devserver:5000/comfyui:latest

docker run -d \
--gpus all \
--restart unless-stopped \
--name comfyui_9182 \
-p 9182:8000 \
-v /d/comfyui_models:/app/models \
-v /etc/apps/comfyui-docker/output_9182:/app/output \
devserver:5000/comfyui:latest

docker run -d \
--gpus all \
--restart unless-stopped \
--name comfyui_9183 \
-p 9183:8000 \
-v /d/comfyui_models:/app/models \
-v /etc/apps/comfyui-docker/output_9183:/app/output \
devserver:5000/comfyui:latest

```
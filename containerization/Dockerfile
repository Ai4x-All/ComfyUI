
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

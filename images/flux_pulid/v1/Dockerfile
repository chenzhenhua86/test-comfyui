FROM python:3.10.9-slim AS models

RUN apt update && apt install -y aria2

# # https://huggingface.co/city96/FLUX.1-dev-gguf/blob/main/flux1-dev-Q8_0.gguf 12.7G
# RUN aria2c -x 16 --dir "/models/unet" --out "flux1-dev-Q8_0.gguf" "https://huggingface.co/city96/FLUX.1-dev-gguf/resolve/main/flux1-dev-Q8_0.gguf?download=true"

# https://huggingface.co/guozinan/PuLID/tree/main 1.14G
RUN aria2c -x 16 --dir "/models/pulid" --out "pulid_flux_v0.9.0.safetensors" "https://huggingface.co/guozinan/PuLID/resolve/main/pulid_flux_v0.9.0.safetensors?download=true"

# https://huggingface.co/city96/t5-v1_1-xxl-encoder-gguf/tree/main 5.06G
RUN aria2c -x 16 --dir "/models/clip" --out "t5-v1_1-xxl-encoder-Q8_0.gguf" "https://huggingface.co/city96/t5-v1_1-xxl-encoder-gguf/resolve/main/t5-v1_1-xxl-encoder-Q8_0.gguf?download=true"

# https://huggingface.co/comfyanonymous/flux_text_encoders/tree/main 246M
RUN aria2c -x 16 --dir "/models/clip" --out "clip_l.safetensors" "https://huggingface.co/comfyanonymous/flux_text_encoders/resolve/main/clip_l.safetensors?download=true"

# https://huggingface.co/black-forest-labs/FLUX.1-schnell/tree/main/vae 168M
RUN aria2c -x 16 --dir "/models/vae" --out "flux-ae.sft" "https://huggingface.co/black-forest-labs/FLUX.1-schnell/resolve/main/vae/diffusion_pytorch_model.safetensors?download=true"

# 343K
RUN apt update && \
    apt install unzip && \
    aria2c -x 16 --dir "/models/insightface/models" --out "antelopev2.zip" "https://github.com/deepinsight/insightface/releases/download/v0.7/antelopev2.zip" && \
    cd /models/insightface/models && \
    unzip antelopev2.zip && \
    rm antelopev2.zip

# 856M
RUN pip install -U "huggingface_hub[cli]" && huggingface-cli download QuanSun/EVA-CLIP EVA02_CLIP_L_336_psz14_s6B.pt

# 104M
RUN aria2c -x 16 --dir "/venv/lib/python3.10/site-packages/facexlib/weights" --out "detection_Resnet50_Final.pth" "https://github.com/xinntao/facexlib/releases/download/v0.1.0/detection_Resnet50_Final.pth"
# 81M
RUN aria2c -x 16 --dir "/venv/lib/python3.10/site-packages/facexlib/weights " --out "parsing_parsenet.pth" "https://github.com/xinntao/facexlib/releases/download/v0.2.2/parsing_parsenet.pth"
# 50M
RUN aria2c -x 16 --dir "/venv/lib/python3.10/site-packages/facexlib/weights" --out "parsing_bisenet.pth" "https://github.com/xinntao/facexlib/releases/download/v0.2.0/parsing_bisenet.pth"






FROM alpine/git:2.36.2 AS custom_nodes

WORKDIR /custom_nodes

RUN git clone https://github.com/ltdrdata/ComfyUI-Manager.git && \
    cd ComfyUI-Manager && \
    git checkout "584c5002477e5cc88b56fe02853a4e7c43660bbf"

RUN --mount=type=bind,source=/base/v5/built-in,target=/docker \
    cp -R /docker/custom_nodes/* /custom_nodes/





FROM registry.cn-hangzhou.aliyuncs.com/ohyee/fc-demo:fc-comfyui-lite-v5

RUN grep -r "/venv" /venv/bin/* | awk -F: "{print \$1}" | xargs -I {} sed "s@${NAS_DIR}/venv@/venv@g" -i {}

RUN apt update && apt install g++ -y
RUN pip install insightface onnxruntime
RUN cd ${BUILTIN}/custom_nodes && \
    git clone https://github.com/balazik/ComfyUI-PuLID-Flux && \
    git clone https://github.com/city96/ComfyUI-GGUF && \
    git clone https://github.com/AlekPet/ComfyUI_Custom_Nodes_AlekPet.git

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r ${COMFYUI}/requirements.txt --no-cache-dir && \ 
    find ${COMFYUI} -type f -name requirements.txt | xargs -I {} pip install -r {} && \
    find ${BUILTIN}/custom_nodes -type f -name requirements.txt | xargs -I {} pip install -r {}

RUN apt update  && apt install -y ffmpeg libsm6 libxext6

# 内置模型
RUN --mount=type=bind,from=models,source=/models,target=/models \
    --mount=type=bind,from=models,source=/venv,target=/mnt-venv \
    --mount=type=bind,from=models,source=/root,target=/mnt-root \
    mkdir -p ${BUILTIN}/models/checkpoints ${BUILTIN}/models/loras && \
    rsync -a --ignore-existing /models/* ${BUILTIN}/models/ && \
    rsync -a --ignore-existing /mnt-venv/* /venv/ && \
    mkdir -p ${BUILTIN}/root && rsync -a --ignore-existing -L /mnt-root/ ${BUILTIN}/root/

RUN argospm install translate-zh_en && \
    mkdir -p ${BUILTIN}/root/.local/share/argos-translate && \
    rsync -a --ignore-existing /root/.local/share/argos-translate/ ${BUILTIN}/root/.local/share/argos-translate/

RUN find "${BUILTIN}/root" ! -type d -printf "%P\n" | xargs -I {} bash -c "mkdir -p \$(dirname '${VIRTUAL_NAS}/root/{}') && ln -sT '${BUILTIN}/root/{}' '${VIRTUAL_NAS}/root/{}'"

# 内置自定义节点
RUN --mount=type=bind,from=custom_nodes,source=/custom_nodes,target=/custom_nodes \
    --mount=type=bind,source=./,target=/docker \
    mkdir -p ${BUILTIN}/custom_nodes && \
    rsync -av /custom_nodes/ ${BUILTIN}/custom_nodes

# 更新依赖至最新
RUN --mount=type=cache,target=/root/.cache/pip \
    find ${BUILTIN}/custom_nodes -type f -name requirements.txt | xargs -I {} pip install -r {}

ARG IMAGE_TAG=${IMAGE_TAG}
ENV IMAGE_TAG=${IMAGE_TAG}
RUN --mount=type=bind,source=/base/v5,target=/docker \
    mkdir -p ${COMFYUI}/models/configs && \
    bash /docker/init.bash
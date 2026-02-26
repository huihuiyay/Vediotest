# 📊 **metrics/README.md — 视频生成/编辑质量评测指南**

整合 5 套常用评测工具：  
1) **VBench**（🎯 单视频质量维度）  
2) **VE-Bench QA**（🧠 文本-视频一致性，输入→输出）  
3) **ArcFace**（🧑‍🤝‍🧑 生成前后人脸相似度）  
4) **OpenFace 自然度**（🎭 基于 AU 的表情自然度）  
5) **SyncNet / Wav2Lip**（👄 口型一致性：LSE-D / LSE-C）

[[_TOC_]]

---

## 1️⃣ **环境一（适用评测 1~3）** ⚙️

```bash
cd metrics/Vbench
conda env create -f vbench312.yaml
conda activate vbench312
```


### 🎯 1. VBench：单视频质量维度评测

支持维度：
subject_consistency, background_consistency, motion_smoothness, dynamic_degree, aesthetic_quality, imaging_quality

```bash
CUDA_VISIBLE_DEVICES=0 \
python evaluate.py \
  --dimension $DIMENSION \
  --videos_path /path/to/folder_or_video \
  --mode custom_input \
  --output_path ~/vbench_out
```


💡 --videos_path 可指向单个视频或包含多个视频的文件夹；结果输出在 --output_path。


### 🧠 2. VE-Bench QA：输入→生成视频质量评测（可加提示词）


```bash
python - <<'PY'
from vebench import VEBenchModel
e = VEBenchModel()
print(e.evaluate(
  "A girl stands upright, confident gaze, slight smile, hands on hips, poised and assertive.",
  "/media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/input.mp4",
  "/media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/output_00.mp4"
))
PY
```


### 🧑‍🤝‍🧑 3. ArcFace：生成前后视频面部相似度评估
```bash
python vid2vid_arcface_compare.py \
  --src /media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/input.mp4 \
  --gen /media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/output_01.mp4 \
  --num-frames 64 \
  --save-csv ./idsim_results.csv
```

## **2️⃣ 环境二（适用评测 4） 🧪**

```bash
conda env create -f faceeval.yaml
conda activate faceeval
```


### 🎭 4. 面部表情自然度评估（OpenFace AU → 评分脚本）

需先用 OpenFace 对源视频与生成视频分别提取 AU，再调用评估脚本。
如遇依赖冲突，可先设置运行库路径：export LD_LIBRARY_PATH="$HOME/.local/lib:${LD_LIBRARY_PATH}"

**4.1 提取 AU**
    
- 源视频（SRC）
```bash
"metrics/Vbench/OpenFace/build/bin/FeatureExtraction" \
  -f "/media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/input.mp4" \
  -aus -2Dfp \
  -out_dir "/media/yuzihui/17A26BDB4251F93F/project/multimodalinteract/metrics/Vbench/OpenFace/openface_out/src"
```
- 生成视频（GEN）
```bash
"metrics/Vbench/OpenFace/build/bin/FeatureExtraction" \
  -f "/media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/output_00.mp4" \
  -aus -2Dfp \
  -out_dir "/media/yuzihui/17A26BDB4251F93F/project/multimodalinteract/metrics/Vbench/OpenFace/openface_out/gen"
```

**4.2 自然度评测**

```bash
python eval_openface_naturalness.py \
  --src_csv  /media/yuzihui/17A26BDB4251F93F/project/multimodalinteract/metrics/Vbench/OpenFace/openface_out/src/input.csv \
  --gen_csv  /media/yuzihui/17A26BDB4251F93F/project/multimodalinteract/metrics/Vbench/OpenFace/openface_out/gen/output_00.csv \
  --src_video /media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/input.mp4 \
  --gen_video /media/yuzihui/17A26BDB4251F93F/project/VBench/testvideo/output_00.mp4 \
  --prompt "A girl stands upright, confident gaze, slight smile, hands on hips, poised and assertive." \
  --num_frames 256 \
  --device auto \
  --save_dir ./openface_eval_out
```


📂 输出（图表/CSV）将保存至 --save_dir。
❗ 路径大小写敏感：请确保 metrics/Vbench 等目录与仓库一致。
 
## **3️⃣ 环境三（适用评测 5） 🎵**

```bash
cd metrics/syncnet_python
conda env create -f lse_eval.yaml
conda activate lse_eval
```


### 👄 5. 口型一致性评估（Wav2Lip / SyncNet：LSE-D & LSE-C）

```bash 
sh calculate_scores_real_videos.sh /path/to/video/data/root
```

📁 传入的是数据根目录（包含待评测视频的文件夹），而不是单个视频文件。
💾 LSE 评估结果（日志/CSV）按脚本约定路径输出。


📁 建议的目录结构
```
project/
  VBench/testvideo/
    input.mp4
    output_00.mp4
    output_01.mp4
  vbench_out/                   # VBench 输出
  openface_eval_out/            # OpenFace 评测输出
  metrics/
    Vbench/
      OpenFace/
        build/bin/FeatureExtraction
        openface_out/
          src/input.csv
          gen/output_00.csv
    syncnet_python/
      lse_eval.yaml
```

### 💡 Tips

GPU 指定：CUDA_VISIBLE_DEVICES=0（或逗号分隔多个）

Conda 激活：以各 YAML 的 name 为准，比如 conda activate vbench312

路径规范：建议统一使用绝对路径，或在项目内保持稳定的相对目录

### 📝 许可与致谢

上述评测脚本与方法源于各开源项目（VBench、VE-Bench、ArcFace、OpenFace、Wav2Lip/SyncNet）。

请遵循各自 License 使用与引用。

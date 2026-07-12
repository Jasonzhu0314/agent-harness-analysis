# MinerU Pipeline 表格分类与结构识别流程

本文记录 MinerU `pipeline` 模式下，表格方向分类、表格类型分类以及后续结构识别的执行逻辑。

## 1. 整体位置

表格处理发生在 `pipeline` 后端的 `BatchAnalyze` 中：

- 入口：`mineru/backend/pipeline/batch_analyze.py`
- 核心阶段从 `# 表格识别 table recognition` 开始

大体顺序：

1. Layout 模型先检测页面中的 `table` 区域。
2. 对每个表格区域裁图，得到 `table_img` 和 `wired_table_img`。
3. 表格方向分类，必要时旋转表格图。
4. 表格类型分类，判断有线表或无线表。
5. 表格 OCR det/rec，得到单元格文本候选。
6. 先对所有表格跑无线表结构识别。
7. 对疑似有线表或无线置信度不足的表格，再跑有线表结构识别。
8. 比较 wired/wireless 结果，决定最终 HTML。

## 2. 表格方向分类

实现类：

```text
mineru/model/table/cls/mineru_table_ori_cls.py
MineruTableOrientationClsModel
```

方向分类不是独立的 ONNX 分类模型，而是复用 OCR 引擎做“候选筛选 + 多角度 OCR 打分”。

### 2.1 默认角度

`batch_predict` 一开始会把所有表格角度设为 `"0"`：

```python
rotate_labels = ["0"] * len(imgs)
```

也就是说，只有被识别为疑似旋转的表格才会进入后续评分流程。

### 2.2 OCR det 召回疑似旋转表格

对表格图做 OCR det，得到文字检测框。然后统计高窄框数量。

判断规则：

- 单个框满足 `width / height < 0.8`，认为是高窄文字框。
- 高窄框数量至少占全部检测框的 `28%`。
- 高窄框数量至少为 `3`。

对应常量：

```python
ROTATED_TEXT_ASPECT_RATIO_THRESHOLD = 0.8
ROTATED_TEXT_RATIO_THRESHOLD = 0.28
ROTATED_TEXT_MIN_BOXES = 3
```

满足这些条件的表格才进入多角度评分。

### 2.3 多角度 OCR rec 打分

候选角度：

```python
ORIENTATION_SCORE_LABELS = ("0", "90", "270")
```

对每个候选角度：

1. 旋转表格图。
2. 再做 OCR det。
3. 最多抽样 18 个检测框。
4. 裁出文本小图。
5. 调 OCR rec。
6. 计算有效文本的平均置信度。

相关参数：

```python
ORIENTATION_SCORE_MAX_SAMPLE_BOXES = 18
ORIENTATION_SCORE_MIN_VALID_RESULTS = 5
```

如果有效 OCR rec 结果少于 5 个，该角度得分为 0。

### 2.4 角度选择策略

方向选择比较保守，优先保持 0 度：

- 如果 0 度平均置信度 >= `0.9`，直接选 0 度。
- 如果最佳非 0 度比 0 度高不到 `0.08`，仍然选 0 度。
- 否则选择平均 OCR rec 置信度最高的角度。

相关常量：

```python
ORIENTATION_ZERO_SCORE_PRIORITY_THRESHOLD = 0.9
ORIENTATION_SCORE_TIE_THRESHOLD = 0.08
```

### 2.5 方向结果如何生效

方向分类只返回标签，不直接改图。真正旋转发生在：

```text
mineru/backend/pipeline/batch_analyze.py
BatchAnalyze._apply_table_rotate_label
```

逻辑：

- `"270"` -> `cv2.ROTATE_90_CLOCKWISE`
- `"90"` -> `cv2.ROTATE_90_COUNTERCLOCKWISE`
- `"0"` -> 不旋转

旋转会同时作用于：

- `table_img`
- `wired_table_img`

## 3. 表格类型分类

实现类：

```text
mineru/model/table/cls/paddle_table_cls.py
PaddleTableClsModel
```

这是一个 ONNX 图像分类模型，用来判断表格是有线表还是无线表。

模型路径：

```text
models/TabCls/paddle_table_cls/PP-LCNet_x1_0_table_cls.onnx
```

标签：

```python
self.labels = [AtomicModel.WiredTable, AtomicModel.WirelessTable]
```

即：

- `wired_table`
- `wireless_table`

### 3.1 输入

批量预测时使用的是：

```python
imgs = [item["wired_table_img"] for item in img_info_list]
```

也就是表格裁图中的 `wired_table_img`。

### 3.2 预处理

预处理流程：

1. 将表格图短边缩放到 256。
2. 中心裁剪为 `224x224`。
3. 按 ImageNet mean/std 归一化。
4. 转为 CHW。
5. 组 batch，输入 ONNXRuntime。

归一化参数：

```python
mean = [0.485, 0.456, 0.406]
std = [0.229, 0.224, 0.225]
scale = 1 / 255
```

### 3.3 输出

模型输出后取 `argmax`：

```python
idx = np.argmax(img_res)
conf = float(np.max(img_res))
```

写回：

```python
img_info["table_res"]["cls_label"] = label
img_info["table_res"]["cls_score"] = round(conf, 3)
```

## 4. 类型分类如何影响结构识别

类型分类不是直接二选一决定最终结构识别结果。

实际策略是：

1. 所有表格先跑无线表结构识别 `WirelessTable`。
2. 如果满足以下条件之一，再跑有线表结构识别 `WiredTable`：
   - 分类结果是 `wired_table`
   - 分类结果是 `wireless_table`，但置信度 `< 0.9`

对应逻辑：

```python
if (
    (table_res_dict["table_res"]["cls_label"] == AtomicModel.WirelessTable
     and table_res_dict["table_res"]["cls_score"] < 0.9)
    or table_res_dict["table_res"]["cls_label"] == AtomicModel.WiredTable
):
    wired_table_res_list.append(table_res_dict)
```

因此：

- 高置信无线表：只使用无线结构识别结果。
- 有线表：补跑有线结构识别。
- 低置信无线表：也补跑有线结构识别，防止分类不准。

## 5. 无线表结构识别

实现：

```text
mineru/model/table/rec/slanet_plus/main.py
PaddleTableModel
```

底层模型：

```text
models/TabRec/SlanetPlus/slanet-plus.onnx
```

流程：

1. 使用 SlanetPlus 预测表格结构 token 和 cell bbox。
2. 将 OCR det/rec 结果转换成文本框和文本。
3. 用 `TableMatch` 将结构、cell bbox、OCR 文本匹配起来。
4. 输出 HTML。

核心调用：

```python
pred_structures, cell_bboxes, _ = self.table_structure.process(...)
pred_html = self.table_matcher(pred_structures, cell_bboxes, dt_boxes, rec_res)
```

## 6. 有线表结构识别

实现：

```text
mineru/model/table/rec/unet_table/main.py
UnetTableModel
```

底层模型：

```text
models/TabRec/UnetStructure/unet.onnx
```

流程：

1. `TSRUnet` 预测表格线/单元格 polygon。
2. `TableRecover` 根据 polygon 恢复行列逻辑结构。
3. 将 OCR 结果匹配进单元格。
4. 生成 wired HTML。
5. 和前面 wireless HTML 比较。
6. 如果 wired 明显更差，则回退到 wireless HTML。

比较维度包括：

- 单元格数量
- OCR 文本命中数量
- 空单元格数量
- 非空单元格数量

因此，即使进入了有线表识别，最终也不一定采用 wired 结果。

## 7. 总结

表格方向分类：

- 使用 OCR det 判断是否疑似旋转。
- 对 `0/90/270` 三个角度做 OCR rec 置信度评分。
- 选择置信度最高且通过保守阈值的角度。
- 最终旋转表格裁图，供 OCR 和结构识别使用。

表格类型分类：

- 使用 ONNX 图像分类模型判断 `wired_table` / `wireless_table`。
- 输入是 `wired_table_img`。
- 输出 `cls_label` 和 `cls_score`。
- 分类结果用于决定是否补跑有线表结构识别。

结构识别策略：

- 先全部跑无线表结构识别。
- 有线或低置信无线再跑有线结构识别。
- 有线结果会和无线结果比较，必要时回退无线结果。

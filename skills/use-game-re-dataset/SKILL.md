---
name: use-game-re-dataset
description: Use when consuming the game-re world-model dataset (OSS, schema v8) from the algorithm side — pinning catalog snapshots, selecting sessions, decoding modalities including instance segmentation — or when reading the project's Feishu docs via lark-cli. Distributed via the game-re-skills repo (npx skills add neeetman/game-re-skills). Access parameters (bucket, endpoints, credentials) come from the Feishu docs, never from this file.
---

# use-game-re-dataset

世界模型训练数据集在阿里云 OSS（schema v8）。本 skill 给算法侧：文档在哪、怎么访问数据、
格式上会咬人的点。**具体访问参数（bucket、endpoint、凭证获取方式、开发机地址）一律从下方
飞书文档获取，本文只放公开无害的常驻事实**——先读文档拿参数，再访问数据。

## 文档树（权威入口）

| 文档 | 内容 |
|---|---|
| [数据使用与技术参考](https://a9ihi0un9c.feishu.cn/wiki/DWCBwd8U7iyhArkvBVac5krBnPd) | 父页：存储布局、目录与统计、快照钉住流程、使用示例 |
| [字段字典（数据说明）](https://a9ihi0un9c.feishu.cn/wiki/F2U3woeGni957TkbrKMcxp3SnDc) | 逐文件逐字段定义 + 发布校验规则（VAL）附录 |
| [坐标系统与投影](https://a9ihi0un9c.feishu.cn/wiki/HC0Fw8Ql3i7zkAkGS7ZcE66SnM9) | 坐标约定、相机模型与反投影公式、OBB、地形还原 |
| [资产库与网格引用](https://a9ihi0un9c.feishu.cn/wiki/IccBwqm9aiCfk5kgYM9cdWEqnvh) | 库组织、glb_file 解析、missing-glb 台账 |
| [实例级语义分割交付契约](https://a9ihi0un9c.feishu.cn/wiki/Z4kxw2n4pi3qmik4k9tcVF6QnJc) | 像素约定、实例表、词表与粗类体系 |
| [发布校验保证](https://a9ihi0un9c.feishu.cn/wiki/QZ9Dw9OA7i4plLkcaBOcerJ8nLe) | 已发布数据恒成立的性质（可省略的防御检查） |
| [数据消费约束](https://a9ihi0un9c.feishu.cn/wiki/Q3jiwM1Hzitx04kOkfHcL5ECn7f) | 不得隐含假设的数据形态 |
| [项目主页](https://a9ihi0un9c.feishu.cn/wiki/UYxbw0hM1iVMhKkPjA5cGvpAnwb) | 交付范围、进度、验收口径 |

## lark-cli（程序化读飞书文档）

浏览器能看就不必装。安装与完整用法以官方仓库为准：https://github.com/larksuite/cli

```bash
npx @larksuite/cli@latest install    # 官方推荐装法
npx skills add larksuite/cli -y -g   # 一键安装官方 26 个 lark-* AI skills(含 lark-doc/lark-wiki)
lark-cli config init                 # 首次:输出授权链接,打开完成应用配置
lark-cli auth login --recommend      # 用户身份授权;验证:lark-cli auth status
```

装了官方 skills 后读文档直接交给 lark-doc skill；手动读时：

```bash
lark-cli docs +fetch --doc "https://a9ihi0un9c.feishu.cn/wiki/DWCBwd8U7iyhArkvBVac5krBnPd" --doc-format markdown
# 大文档先看目录、再按节精读,别全文拉取
lark-cli docs +fetch --doc <token> --scope outline --max-depth 3
lark-cli docs +fetch --doc <token> --scope section --start-block-id <标题id>
```

判定成功看 JSON 的 `ok == true`（不是 `code`）。文档操作一律 `--as user`。

## 数据访问

- **访问参数**：bucket 名、公网/VPC 内 endpoint、开发机地址，见
  [数据使用与技术参考](https://a9ihi0un9c.feishu.cn/wiki/DWCBwd8U7iyhArkvBVac5krBnPd)与
  [阿里云推理集群使用指南](https://a9ihi0un9c.feishu.cn/wiki/NweSw3ecQisRVzkUCaXcdhgEntd)（需飞书权限）。
- **凭证**：阿里云 RAM 用户 AccessKey，登录控制台自助创建（RAM 用户登录入口，不是 aliyunidaas
  门户；详见集群使用指南）。凭证放环境变量，不进代码与命令行。
- **boto3（S3 兼容）必须两项配置**，缺一必错：

```python
from botocore.config import Config
config = Config(s3={"addressing_style": "virtual"},          # 缺省 path-style 会被 OSS 拒
                request_checksum_calculation="when_required", # 新版 boto3 的流式尾部校验和
                response_checksum_validation="when_required") #   OSS 不支持,报 NotImplemented
```

- **DuckDB 直查 catalog**（谓词下推，不下载全表，已实测）：

```sql
INSTALL httpfs; LOAD httpfs;
CREATE SECRET oss (TYPE S3, KEY_ID '…', SECRET '…',
    ENDPOINT '<endpoint 主机名>', URL_STYLE 'vhost', REGION '<地域>');
SELECT session_key, prefix
FROM read_parquet('s3://<bucket>/<数据根>/catalog/snapshots/<stamp>/sessions.parquet')
WHERE viewpoint = 'tpv' AND list_contains(modalities, 'instance_id');
```

- 开发机上批量搬运用 ossutil（已装已配，机器信息见集群使用指南）：`ossutil sync oss://… /mnt/cpfs/… --update`。

## 读数据的固定流程

1. 实验开始时读一次 `<数据根>/catalog/latest.json`，把快照戳记进实验配置；此后**只读**
   `<数据根>/catalog/snapshots/<stamp>/`（不可变 → 实验可复现）。不要引用 `catalog/` 根下的便捷副本。
2. `summary.json` 的 `min_reader_version` 高于加载器支持版本 → 直接报错退出。
3. 从快照的 `sessions.jsonl`/`sessions.parquet` 选会话：按 `facets`（camera/viewpoint/locomotion）、
   `modalities`、`schema_version` 过滤。**不要解析 `dataset_id` 字符串推断属性**。
4. 按行内 `prefix` 读会话根 `dataset.json`（存在即已发布；没有根 marker 的目录是中间态，勿读），
   分段枚举以 `segments[]` 为准（段号有洞是正常的，见 `excluded_segment_ids`）。
5. 逐样本跨模态 join 只走 `video/meta.parquet`（`sample_id` 主键）；视频帧号只能从这张表拿，
   **不可按位置对齐**。

## 格式要点（会咬人的）

- **depth.mkv**：gray16le 载荷是 **float16 位模式**（按位重解释，不是整数毫米）；单位米；
  天空 +inf，进损失前屏蔽非有限值。
- **instance_id.mkv**：同为 gray16le 但是 **raw uint16**——与 depth 解码方式不同，勿混用；
  0 = ignore（未标注，不是背景类，不得进损失）；空间重采样仅可最近邻；非零 id 段内恒定，
  必在 `instance_table.json` 有条目。
- **语义类别图不交付**：用实例帧 × `instance_table` 的 id→coarse LUT 派生，天空按 depth 非有限
  填 sky 类；粗类表在 `<数据根>/catalog/semantic_taxonomy.json`（255 恒为 ignore）。训练 pin 词表版本
  （catalog 行 `vocab` 字段 / 实例表 `vocab_ref`）。
- **坐标**：一切世界坐标 glTF 右手系、Y-up、米；四元数恒 `[x,y,z,w]`；节点 `rotation`
  已预复合几何基校正，摆放 GLB 不要再做轴变换。反投影公式见坐标子文档。
- **相机**：`far_mode="infinite"`、`far_m=null` 是常态；depth/normal 分辨率是 RGB 一半，
  反投影时内参等比缩放。
- **骨骼**：`bone_transforms.bin.zst`（zstd 分组帧，解压见 meta 的 `compression`）；解压后每骨
  7×float32，相对父骨；`pose_pairing.policy` 分支——`verbatim_indexed_buffer`（新采集，姿势与
  画面同刻）/ `render_state_blend`（旧采集，按 α=0.4 与下一条记录混合）。
- **资产**：按分段 marker 的 `asset_library` 在自选库根下解析（`asset-libraries/<id>/glb/<glb_file>`）；
  引用悬空 = 尚未提取，跳过网格即可（变换/OBB/语义仍有效），**不是错误**。
- **屏蔽语义**：`*_valid=false` = 该样本该模态不可用（不是损坏）；`opaque_mode` 三态以列为权威。

## 不要做

- 不遍历 bucket 选数据（只走 catalog）；不读数据根之外的前缀（属其他团队）。
- 不在训练里引用"线上最新"（集合漂移）；固定实验集 = 钉快照。
- 不假设分段连续、不假设引用必有文件、不从 `dataset_id` 字符串猜采集条件。

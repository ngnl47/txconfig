# txconfig 策划配置源

本仓库维护 Excel、表与枚举登记及 Luban 导出规则，供 txgame、后续的 txgameslg 使用。生成工具和双端产物不放在本仓库。

目录结构：

```text
txconfig/
├── README.md
├── luban.conf
└── excel/
    ├── __tables__.xlsx
    ├── __enums__.xlsx
    ├── login.xlsx       # 登录领域：server、note（说明页）
    ├── item.xlsx        # 道具领域：base
    └── battle.xlsx      # 战斗领域：rule
```

将 txconfig、txgame、txclient 放在同级目录，txgame 的 `gen/config.yml` 配置：

```yaml
inputDir: ../../txconfig
serverOutput: ../server/gen/config
clientOutput: ../../txclient/src/gen/config
editions:
  utc: {timeZone: UTC}
  pacific: {timeZone: America/Los_Angeles}
  beijing: {timeZone: Asia/Shanghai}
```

路径相对 `config.yml`，也支持绝对路径。`luban.conf` 的 `dataDir: "excel"` 指向 Excel 数据目录；server/client target 控制双端字段筛选，不决定产物在业务工程中的位置。

修改 Excel 并保存公式结果后，在 txgame/gen 执行 `./run.sh config`。需要 .NET 8（PATH 中的 dotnet 或 `TXGEN_DOTNET`）以及 gen/config.yml 指向的 txgen 内固定 Luban 构建。导出器校验通过后发布双端产物，服务启动时加载；当前不做热更新。

应用验收在 txgame/gen 执行：

```sh
TXGEN_DOTNET=/path/to/dotnet go test ./main/txgen -run TestLubanExportAndGeneratedGo -count=1
```

验收从配置的输入目录复制示例到临时目录，包含失败场景，不修改本仓库。txgameslg 后续接入时维护自己的生成配置和产物目录，不覆盖 txgame 或 txclient 的产物。

- `excel/__tables__.xlsx`：Luban 原生表登记，填写 `input`（`sheet@文件.xlsx`）、`mode`、`index` 等；`full_name`、`value_type` 由表内公式计算。`read_schema_from_file=true` 表示类型直接取自数据 Sheet。
- `excel/__enums__.xlsx`：原生枚举登记；`*items` 表头合并单元格覆盖 name/alias/value/comment/tags。枚举只生成代码。
- 业务工作簿按模块组织：`login.xlsx` 中的 `server`、`item.xlsx` 中的 `base`、`battle.xlsx` 中的 `rule`。`login.xlsx` 保留原有 `note` 说明页，不登记、不导出。无需 `$` 前缀或自定义扫描。
- 常规表使用 `##var`、`##type`、`##group`、`##`；`s` 仅服务端、`c` 仅客户端，空分组两端共用。
- 常量使用 `mode=one`，竖表首列 `##var#column`，仍生成可修改的数据 JSON。值列首格可以写“值”或 `value`，不属于字段定义；若在值列后追加纯说明列，首格使用 `##备注` 等 `##` 前缀，避免被当作第二条记录。此规则针对竖表，横表的说明列在 `##var` 行使用 `#备注`。

样式约定：常规横表参考 `login.xlsx` 的 `server`，深蓝白字为字段名、浅蓝为类型、浅绿为双端分组、黄色黑字为中文说明。竖表与表/枚举登记保持各自布局；竖表淡黄色区域为配置值。颜色不参与导出解析。表头统一 10pt 字号、15pt 行高，数据行统一 11pt 字号、20pt 行高，均垂直居中；普通表冻结前 4 行，表登记冻结前 2 行，枚举登记冻结前 3 行，竖表冻结第 1 行。列宽保持策划设置。

## 命名与组织规则

- **工作簿按具体业务模块聚合**，文件名使用小写 `snake_case`，例如 `login.xlsx`、`item.xlsx`、`battle.xlsx`。同模块新增相关配置时，优先添加到已有工作簿的不同 Sheet，不按每张逻辑表新建文件，也不把所有模块重新汇入一个 `game.xlsx`。
- **Sheet 使用小写 `snake_case`，不重复模块前缀**，例如 `base`、`rule`、`quality_rule`；以后增加战斗技能、奖励配置时，在 `battle.xlsx` 中增加 `skill`、`reward`。不同工作簿可以有同名 Sheet，通过完整 `sheet@文件.xlsx` 区分。这里只说明命名示例，不预建空表。
- **每个独立业务 Sheet 在 `__tables__.xlsx` 单独登记**，`input` 明确写成 `sheet@文件.xlsx`。不启用按文件自动导入，避免把同一工作簿的不同结构当作同一张表读取。说明和辅助计算 Sheet 不登记，不输出为业务数据。
- **以工作簿名作为命名空间，以 Sheet 名作为类型名来源**：去掉 `.xlsx` 后的文件名是命名空间，将 Sheet 名转为 PascalCase 后作为记录类型名，整表类型再加 `Tb` 前缀。例如 `base@item.xlsx` 计算为 `item.TbBase` / `item.Base`，`quality_rule@item.xlsx` 计算为 `item.TbQualityRule` / `item.QualityRule`。`login.Item` 和 `bag.Item` 可以共存，但完整逻辑名称必须唯一。命名转换由登记表公式完成，生成器读取保存的结果。
- **文件名与 Sheet 名会影响生成接口**：修改 `input` 会联动 `full_name`、`value_type`；重命名工作簿、重命名 Sheet 或将 Sheet 移到其他工作簿，需要同时检查业务类型引用及默认 JSON 文件名的变化。物理 Sheet 由完整 `sheet@文件` 定位，跨工作簿可重名，不能用重名逻辑表覆盖已有定义。
- **枚举集中登记**于 `__enums__.xlsx`，按命名空间区分所属模块，不输出独立 JSON；常量仍放在所属模块工作簿的竖向 Sheet 中，以 `mode=one` 登记。
- **输出代码集中、数据按表拆分**：服务端统一生成 `config.go`，客户端统一生成 `config.ts`，包含各自需要的类型、解析器与快照入口；不按工作簿或 Table 增加代码文件。JSON 按逻辑表及配置组分别输出，三个时区组共用同一份代码。显式设置 `output` 时使用小写 `snake_case` 且不带 `.json`；留空时由 Luban 根据完整表名生成，例如 `item.TbBase` 输出 `item_tbbase.json`。同一导出端的文件名不能冲突。

服务器列表登记为 `login.TbServer`（整表）/ `login.Server`（一条记录），读取 `server@login.xlsx`，`output=server_list` 显式指定 JSON 文件名。`login` 是用于组织类型与避免重名的逻辑命名空间，不绑定 Login 子服务或限制其他服务读取；导出端由 `group` 决定。当前 txgen 将带点名称映射为同一 config 包中的 `Login_TbServer`、`Login_Server`，不创建独立的 login 代码包。

### 登记表公式维护

业务工作簿直接放在 `excel/` 下，`input` 使用 `sheet@文件.xlsx`，不带子目录。B 列 `full_name`、C 列 `value_type` 为公式列，E 列 `input` 为填写入口；不要将公式列覆盖为手填文本。

为已有输入补齐公式时，只选中 `B3:C3` 并向下填充到目标行，不要下拉 E 列，也不要覆盖整行。E 列保存手动填写的路径，下拉 `E3` 会把 `server@login.xlsx` 复制到后续行，覆盖原有的 `base@item.xlsx`、`rule@battle.xlsx`。公式中的 `E3` 是相对引用（没有 `$`），从 C3 向下填充后会自动变成 E4、E5。

新增空白登记行时，可以复制已有行，再修改新行的 `input`、索引、模式、导出分组等配置，并保留 B/C 两列公式；不要用复制整行的方式为已有配置补公式。新增行不会由 txgen 自动补公式。

C3（记录类型）：

```excel
=IF(E3="","",SUBSTITUTE(MID(E3,FIND("@",E3)+1,LEN(E3)),".xlsx","")&"."&SUBSTITUTE(PROPER(SUBSTITUTE(LEFT(E3,FIND("@",E3)-1),"_"," "))," ",""))
```

B3（整表类型）：

```excel
=IF(C3="","",SUBSTITUTE(C3,".",".Tb",1))
```

修改后在 Excel 中完成计算并保存，再执行导出。Luban 使用文件中保存的公式结果，txgen 不负责重新计算，也不能判断公式缓存是否已过期。公式结果为空或报错时，先修正登记内容，不以手填结果替代公式。

## 类型与日期

第一版支持 bool、int32、int64、float、double、string、datetime、枚举，以及上述类型的一维 list；表支持单索引 map、无索引 list、one。暂不支持复杂 bean、继承、多态、可空类型或复合索引，生成时报错。

空数据单元格使用类型零值。长整数在 Excel 录入前设置为文本；数据表的 int32/int64 类型不要求单元格必须是数值格式。超过 Excel 精度的数字一旦按数值录入，导出器无法恢复原始值。

配置日期使用 datetime 类型，文本按 `2026-01-01 00:00:00` 填写，或使用 Excel 原生日期。Luban 在导出时按组时区转换；txgen 统一输出 Unix 毫秒字符串，业务加载为 Go int64 / TS bigint。示例组为 utc/pacific/beijing；同一份 Excel 导出三组 JSON，代码结构共用。`server@login.xlsx` 的 openTime 与 `rule@battle.xlsx` 的 winterStart/summerStart 展示绝对日期与冬夏时差。

datetime 请填写明确日期；原生 Luban 不将空日期当 Unix 0，不能套用普通数字单元格的空值约定。若需要可选的绝对时间或全球同一时刻，使用 int64 Unix 毫秒（空值为 0），不要将地区日期重复当 UTC 使用。导出器不重算公式，策划须用 Excel 保存准确的公式缓存结果。

## 当前道具示例

`base@item.xlsx` 只维护 `id`、`name`、`quality`、`enabled`。前三列双端导出，enabled 仅服务端导出，用于决定随机领奖候选；第三个道具为关闭示例，不参与发放。客户端从本地配置补齐名称与品质，玩家背包数据只保存 ID 和数量。

极值整数、空值、列表和导出字段筛选的生成器边界夹具位于 txgen 的 `tests/input/config`，不再要求策划业务表保留测试字段。

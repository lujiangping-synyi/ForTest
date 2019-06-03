> taskDef 代表引擎能识别并执行的最小任务定义

## 1. taskDef 示例

由于 parser 生成的 dsl 实际内容过大，所以以下代码示例中不直接给出。如 `content: astOf("def var NAME type text scope patient_master_info <=  sql(fmop.patient.patient_master_info.patient_name);")`

### globalScopeName

指示传入的 ID 列表维度，例如 patient_master_info，传入的 ID 列表为 patient_id。

### outputVar

指示表最小行的数据维度，详见 Concepts Scope 一节。

### columnSet

指示数据库表生成列，其他生成列还包括 colFromCompositeScope，colFromSequence，defaultColSet，都可以由开发者手动配置。

除此以外，还需要注意引擎将自动生成追踪列，追踪列命名名称以 `tracking_` 作为前缀，剩余名称为   globalScopeName 对应的表主键名称。如设置 globalScopeName 为 visit_record 时，且 visit_record 维度对应表主键为 visit_id，则不可设置目标表主键名为 tracking_visit_id。请各位开发者在定义目标表信息时注意。这也是唯一一列由引擎自动生成的数据。

### colFromCompositeScope

用于指示行数据来源，常用于同个表中存在多个 scope 数据并且要标记来源的情况。

### colFromSequence

用于配置针对 globalScopeName 对应的 id 列数据数目的序列统计。简而言之，如果选择了开启选项，且设置列名为 colFromSequence_id 传入的 id 列存在 id1 和 id2 两个元素。假设落库后， id1 对应的行数有 2 条， id2 对应的行数有 3 条，则最终结果将如下所示。

| primary_key |id| colFromSequence_id |
| :------: | :------  | :------ |
| 1 | id1 | 1 |
| 2 | id1 | 2 |
| 3 | id2 | 1 |
| 3 | id2 | 2 |
| 3 | id2 | 3 |

### defaultColSet

用于指定默认列配置。上文提到 DSL 引擎将自动生成追踪列信息，但这是引擎的内部行为。而对于taskDef 编写者而言，可以配置该项达到注册默认列和值的目的。类型和值约束见下文注释。

### colsFromColumnAgg

用于聚合 columnSet 数据为 jsonb 数据显示及落库。需要注意的是，被聚合的数列将从最终的输出中被删除，体现在如下的例子中，即 NAME 和 VISITID 列将不输出，而输出名为 AGG 的聚合列。同时，不要将聚合列作为 OutputVar 的字段输出。

可指定多个聚合列。

```
 {
            taskId: "task_id",
            taskVersion: "task_v1"
            variableSet: {
                defs: [
                    {
                        content: astOf("def var NAME type text scope patient_master_info <=  sql(fmop.patient.patient_master_info.patient_name);"),
                    },
                    {
                        content: astOf("def var VISITID type integer scope visit_record <=  sql(fmop.visit.visit_record.visit_id);"),
                    },
                    {
                        content: astOf("def var UNIONID type integer <=  sql(fmop.visit.inpat_record.inpat_id) union sql(fmop.visit.outpatient_record.reg_id);"),
                    },
                    {
                        content: astOf("def var OUTDEPTNAME type text scope outpatient_record <=  sql(fmop.visit.outpatient_record.visit_dept_name);"),
                    },
                    {
                        content: astOf("def var INDEPTNAME type text scope inpat_record <=  sql(fmop.visit.inpat_record.in_dept_name);"),
                    },
                ],
            },
            globalScopeName: "patient_master_info",
            scopeSet: {
                defs: [
                    {
                        parent: null,
                        name: "patient_master_info",
                    },
                    {
                        parent: "patient_master_info",
                        name: "visit_record",
                    },
                    {
                        parent: "visit_record",
                        name: "inpat_record",
                    },
                    {
                        parent: "visit_record",
                        name: "outpatient_record",
                    },
                ],
            },
            dataSource: {
                defs: [
                    {
                        sourceName: "fmop",
                        sourceConnString: "postgres://username:password@host:port/database",
                        sourceTableSet: {
                            defs: [
                                {
                                    name: "patient.patient_master_info",
                                    scope: "patient_master_info",
                                    pkey: "patient_id",
                                },
                                {
                                    name: "visit.visit_record",
                                    scope: "visit_record",
                                    pkey: "visit_id",
                                },
                                {
                                    name: "visit.inpat_record",
                                    scope: "inpat_record",
                                    pkey: "inpat_id",
                                },
                                {
                                    name: "visit.outpatient_record",
                                    scope: "outpatient_record",
                                    pkey: "reg_id",
                                },
                            ],
                        },
                        sourceTableLinkSet: {
                            defs: [
                                {
                                    scope1: "patient_master_info",
                                    scope2: "visit_record",
                                    table: "visit.visit_record",
                                    key1: "patient_id",
                                    key2: "visit_id",
                                },
                                {
                                    scope1: "visit_record",
                                    scope2: "inpat_record",
                                    table: "visit.inpat_record",
                                    key1: "inpat_id",
                                    key2: "inpat_id",
                                },
                                {
                                    scope1: "visit_record",
                                    scope2: "outpatient_record",
                                    table: "visit.outpatient_record",
                                    key1: "reg_id",
                                    key2: "reg_id",
                                },
                            ],
                        },
                    },
                ],
            },
            dataTarget: {
                defs: [
                    {
                        targetConnString: "postgres://username:password@host:port/database",
                        targetTableSet: {
                            defs: [
                                {
                                    updateOrInsert: 1
                                    schema: "public",
                                    name: "fmop_composite",
                                    pkey: "fmop_composite_id",
                                    colFromCompositeScope: {
                                        colName: "source_name",
                                        compositeScopeValues: [
                                            {
                                                scopeName: "inpat_record",
                                                colValue: "住院",
                                            },
                                            {
                                                scopeName: "outpatient_record",
                                                colValue: "门诊",
                                            },
                                        ],
                                    },
                                    colFromSequence: {
                                        colName: "fmop_composite_seq",
                                        enable: false,
                                    },
                                    defaultColSet: {
                                        defs: [
                                            {
                                                colName: "perform_id1",
                                                type: 1,
                                                value: 123,
                                            },
                                            {
                                                colName: "perform_id2",
                                                type: 2,
                                                value: 123.0,
                                            },
                                            {
                                                colName: "perform_id3",
                                                type: 3,
                                                value: "123",
                                            },
                                            {
                                                colName: "perform_id4",
                                                type: 4,
                                                value: {key: 1, value: "1"},
                                            },
                                        ],
                                    },
                                    outputVar: "UNIONID",
                                    columnSet: {
                                        defs: [
                                            {
                                                name: "NAME",
                                                variable: "NAME",
                                            },
                                            {
                                                name: "VISITID",
                                                variable: "VISITID",
                                            },
                                            {
                                                name: "UNIONID",
                                                variable: "UNIONID",
                                            },
                                            {
                                                name: "OUTDEPTNAME",
                                                variable: "OUTDEPTNAME",
                                            },
                                            {
                                                name: "INDEPTNAME",
                                                variable: "INDEPTNAME",
                                            },
                                        ],
                                    },
                                    colsFromColumnAgg: [
                                        {
                                            colName: "AGG1",
                                            columns: ["NAME", "VISITID"]
                                        },
                                        {
                                            colName: "AGG2",
                                            columns: ["NAME", "UNIONID"]
                                        },
                                    ]
                                },
                            ],
                        },
                    },
                ],
            },
        },
```

## taskDef 结构

```
export interface TaskDef {
    // taskId
    taskId: string;

    taskVersion?: string;

    // 变量集定义
    variableSet: VariableDef;

    // 全局 scope 名称
    // 通常与传入的 ids 代表维度匹配
    globalScopeName: string;

    // 完整 scope 集定义
    // 每个 scope 可认为是表别称
    scopeSet: ScopeDef;

    // 源数据源定义
    dataSource: DataSourceDef;

    // 目标数据源定义
    dataTarget: DataTargetDef;
}

export interface VariableDef {
    defs: VariableDefItem[];
}

interface VariableDefItem {
    content: any;
}

export interface ScopeDef {
    defs: ScopeDefItem[];
}

export interface ScopeDefItem {
    // 当前 scope 的关联父级 scope
    // 如 visit_record 的 parent 应为 patient_master_info
    parent: string;

    // 当前 scope 名称
    // 如 visit_record，visit，patient_master_info，patient
    name: string;
}

export interface DataSourceDef {
    defs: DataSourceDefItem[];
}

export interface DataSourceDefItem {
    // 数据源名称
    sourceName: string;

    // 数据源连接字符串
    sourceConnString: string;

    // 数据源表集合定义
    sourceTableSet: SourceTableDef;

    // 数据源表关系集定义
    // 用于确定取变量数据时的表连接
    sourceTableLinkSet: SourceTableLinkDef;
}

interface SourceTableDef {
    defs: SourceTableDefItem[];
}

export interface SourceTableDefItem {
    // 表名称
    name: string;

    // 表指定 scope
    scope: string;

    // 表主键名称
    pkey: string;
}

interface SourceTableLinkDef {
    defs: SourceTableLinkDefItem[];
}

export interface SourceTableLinkDefItem {
    // 父级表
    scope1: string;
    // 子表
    scope2: string;

    // 子表名称
    table: string;

    // 父级表与子表的连接键
    key1: string;

    // 子表连接键
    // 可认为是主键
    key2: string;
}

export interface DataTargetDef {
    defs: DataTargetDefItem[];
}

export interface DataTargetDefItem {
    // 输出类型
    // db 代表输出至数据库表中
    // json 代表以处理结果 json 形式返回
    type?: string;

    // 目标数据源连接字符串
    targetConnString: string;

    // 目标数据源表集合定义
    targetTableSet: DataTargetTableDef;
}

interface DataTargetTableDef {
    defs: DataTargetTableDefItem[];
}

export interface DataTargetTableDefItem {
    // 落库类型，1 代表原地更新，2 代表新增
    // update -> 1
    // insert -> 2
    updateOrInsert?: 1 | 2;

    // schema 名称
    schema?: string;

    // 表名称
    name: string;

    // 表指定 scope
    // 计划由 outputVar 对应的 scope 代替
    scope?: string;

    // 表主键名称
    pkey: string;

    // 计划代替作为表指定 scope 输出
    outputVar: string;

    // 匹配指定的 OutputVar ，生成固定字段和值，只适用于 Union 的情况。
    colFromCompositeScope?: ColFromCompositeScope;

    // seq 顺序配置，可配合 Union 以及 OrderBy 使用
    colFromSequence?: ColFromSequence;

    // 默认列定义
    defaultColSet?: DefaultColDef;

    // 列集合定义
    columnSet: SourceColumnDef;

    // jsonb 聚合列，只能选择 columnSet 项进行配置
    // 此项不为空时，优先以此项信息为准生成数据库表和列信息，且将 columnSet 涉及到的列排除不作为最终列输出
    // 输出形式为原始列名和值的键值对
    colsFromColumnAgg?: ColFromColumnAgg[];
}

interface ColFromColumnAgg {
    // 聚合列列名
    colName: string;

    // 待聚合的列集合，只支持从 columnSet 中指定
    columns: string[];
}

interface DefaultColDef {
    defs: DefaultCol[];
}

interface DefaultCol {
    // 默认列名
    colName: string;

    // 默认类型
    // int -> 1
    // number -> 2
    // text -> 3
    // json -> 4
    type: 1 | 2 | 3 | 4;

    // 默认值
    value: any;
}

interface ColFromSequence {
    // 顺序列名，如 dm_seq
    colName: string;

    // 是否启用
    enable: boolean;
}

interface ColFromCompositeScope {
    // 列名称
    colName: string;

    // 匹配情况集合
    compositeScopeValues: CompositeScopeValue[];
}

interface CompositeScopeValue {
    // scope 名称
    scopeName: string;

    // 匹配 scope 名称时，填充的列值
    colValue: string;
}

interface SourceColumnDef {
    defs: SourceColumnDefItem[];
}

export interface SourceColumnDefItem {
    // 列名称
    name: string;

    // 列对应的变量名称
    variable: string;
}
```

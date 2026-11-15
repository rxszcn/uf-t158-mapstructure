# T158 交付说明：接口字段标量类型锁死的根因与契约修复

## 1. 问题陈述

多轮增量装载（同一个目标对象反复 `Decode`）的分层配置场景中：

- 某接口字段（如 `any`）第一轮装入字符串 `"hello"`，第二轮装入整数 `42` 时，
  直接返回 `UnconvertibleTypeError`（"expected type 'string', got unconvertible
  type 'int'"），且**字段保持旧值不变**。
- 同一份载荷换成全新目标对象解码则一切正常，得到 `42`。
- 容器持有（`map`/`slice`/`struct`）时的增量合并是既有设计内行为，并有测试钉住。
- 两条路径（标量锁死报错 vs 容器增量合并）的契约互相矛盾，且该失败形态此前未文档化。

## 2. 根因链

根因集中在 `mapstructure.go` 的 `(*Decoder).decodeBasic`（接口 Kind 的统一入口，
由 `decode` 的 `reflect.Interface` 分支调用）。

1. 接口字段为空（nil interface）时，`val.Elem().IsValid()` 为假，走函数底部
   `val.Set(dataVal)` 分支——直接存入输入并采用其具体类型。因此**全新目标接受任意类型**。
2. 接口字段已持有具体值时，`val.Elem()` 有效，进入函数顶部分支：
   - 取 `elem := val.Elem()`（接口当前持有的具体值）。
   - 结构体字段地址可寻址，`elem.CanAddr()` 为真，`copied` 保持 `false`。
   - 递归 `d.decode(name, data, elem)`，即把新数据**按接口里上一轮的旧具体类型**解码。
3. 第二轮输入 `42`（int）被强制按旧类型 `string` 解码，进入 `decodeString`，
   非弱类型下 int 不可转 string，返回 `UnconvertibleTypeError`；解码失败，
   旧值从未被覆盖，于是**字段保持旧值**。
4. 这就是"接口字段类型被第一轮值锁死"的机制：可寻址的接口持有者被当作具体类型
   原地解码，而不是按接口"可持有任意值"的契约整体替换。

### 为什么容器路径不报错（两条路径的分叉点）

`map[string]any` 的元素值在反射里**不可寻址**。`decodeBasic` 顶部判断
`!elem.CanAddr()` 为真，于是 `copied = true`：复制出一个可写副本解码，
最后 `val.Set(elem.Elem())` **整体替换**接口值。因此容器内的接口元素每轮都按
新值整体替换，类型可变；而可寻址的结构体字段走原地解码，类型被锁死。
同一份 `decodeBasic` 因地址性差异产生了两套互相矛盾的契约。

## 3. 修复方向与取舍

### 选定方向：标量不匹配时整体替换（与全新目标 / 空接口契约对齐）

在 `decodeBasic` 顶部、进入原地递归之前增加守卫：当接口已持有具体值，且
**输入是标量类型**、其具体类型**不可赋值**给当前持有类型、且接口可设置时，
直接 `val.Set(dataVal)` 整体替换并返回，不再递归进旧具体类型。

新增辅助函数 `isScalarKind`，守卫严格限定为标量 Kind
（bool/string/int 族/uint 族/float 族/complex 族）。

### 为什么选"替换"而不是"明确失败"

- **契约一致性**：空接口和全新目标本就接受任意标量类型；同一字段在增量场景下
  没有理由被历史值锁死。替换后增量解码结果与"同载荷进新目标"完全一致，
  消除了两条路径的矛盾。
- **配置分层的实际语义**：分层/增量配置的诉求就是"后装层覆盖前装层"，
  标量整体被新值覆盖符合直觉；锁死报错且静默保留旧值反而是最意外的形态。
- **失败语义仍对具体类型保留**：非接口的具体字段（如 `string` 字段收 int）
  依旧返回 `UnconvertibleTypeError` 且保持旧值，由对照测试钉住，行为零回退。

### 为什么守卫只放行标量、显式排除复合类型与指针

- 复合类型（struct/map/slice）必须保留既有递归增量合并语义（题目硬性要求
  "逐字不回退"），因此不进入替换分支。
- 指针输入保留原递归路径，既有的指针自动解引用契约
  （如 `*any` / `*string → string`，见 `TestDecodeFrom` 用例）不受影响。
- 这样嵌套接口测试（接口持有 `*Basic`，从 map 递归填充）路径完全不变。

### 弱类型（WeaklyTypedInput）下的行为变化

旧行为：第二轮 int 被按旧 string 类型弱转换，得到 `"42"`（字符串）。
新行为：标量类型不匹配直接整体替换，得到 `42`（int），与全新目标一致。
这是方向选择的必然结果，且更符合"接口按新值采用具体类型"的统一契约，
已由 `TestBasic_InterfaceScalarHolder_WeaklyTyped` 钉住。

## 4. 钩子触发次数约定（不回退）

实测并由 `TestBasic_InterfaceScalarHolder_HookCalls` 钉住：

- 空接口首轮：`string -> interface{}` 等调用签名与次数不变。
- 持有标量且**同类型**：仍走原地递归，`string -> *string`、`string -> string`
  等既有调用全部保留。
- 持有标量且**类型不匹配**（新增替换路径）：仅在接口边界触发一次
  `int -> interface{}`，不再针对已失效的旧 string 类型递归触发；
  这是新路径，不改变任何既有路径的次数。

## 5. 测试与验证

新增测试（`mapstructure_test.go`，与既有 `TestBasic_interfaceStruct*` 相邻）：

- `TestBasic_InterfaceScalarHolder_SameType`：匹配形态，同类型原地覆盖。
- `TestBasic_InterfaceScalarHolder_TypeMismatch`：不匹配形态的多组对照表，
  逐组断言增量结果 == 全新目标结果 == 第二轮原始值。
- `TestBasic_InterfaceScalarHolder_MismatchContrastsNonHolder`：具体字段仍报
  `UnconvertibleTypeError` 且保留旧值的失败语义对照。
- `TestBasic_InterfaceScalarHolder_WeaklyTyped`：弱类型下整体替换。
- `TestBasic_InterfaceScalarHolder_NestedStructUntouched`：接口持有 `*Basic`
  仍递归填充，不被 map 整体替换。
- `TestBasic_InterfaceScalarHolder_HookCalls`：钩子调用约定。

既有保证：

- 容器增量合并 `TestMapMerge`、`TestBasic_Merge` 逐字通过。
- 全部既有接口嵌套测试（`TestNestedTypeInterface`、`TestBasic_interfaceStruct*`、
  `TestDecode_*Interface*` 等）通过。
- `go test ./... -count=1` 全量通过，`go vet ./...` 干净。

## 6. 改动文件

- `mapstructure.go`：`decodeBasic` 增加标量替换守卫；新增 `isScalarKind`；
  包文档新增 "Decoding into Interfaces" 契约章节。
- `mapstructure_test.go`：新增 6 个对照测试。
- `README.md`：新增接口解码契约简述。

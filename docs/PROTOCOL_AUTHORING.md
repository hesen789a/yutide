# Writing your own protocol definitions

> **In one sentence.** A protocol definition is a `.json` file. Drop it into the app's
> **protocols folder**, press **Refresh list** on the protocol parser page, and it becomes
> selectable — no restart, no recompile, no code.
>
> 中文一句话：协议定义就是一个 `.json` 文件。放进「协议目录」（协议解析页有「打开协议目录」
> 按钮），点「刷新列表」即可选用 —— 不用重启、不用改代码。

Three entry points on the protocol parser page:

| Button | What it does |
|---|---|
| **Open protocols folder** | Opens the user protocols folder in your file manager. |
| **Import protocol files** | Picks one or more `.json` files, **validates them first**, and copies the valid ones into the folder (never silently overwriting a file with the same name — see §7). |
| **Import a folder** | Picks a folder and imports every `*.json` inside it (same validation, same no-overwrite rule). |

---

## 1. Where definitions live, and what the "protocol name" is

| Layer | Folder | Notes |
|---|---|---|
| Built-in | compiled into the binary (`include_str!`), written to the user folder on first run if missing | Shipped formats; you are not expected to edit these (edits are preserved, they are only written when missing). |
| **Yours** | `<app data dir>/super-engine/protocols/` (Windows: `%APPDATA%\super-engine\protocols\`; Linux: `~/.local/share/super-engine/protocols/`) | Free to add / edit / delete. This is what **Open protocols folder** opens and what **Import** writes into. |

**The protocol name shown in the dropdown is the FILE NAME without `.json`.** `my-sensor.json`
appears as `my-sensor`. The `meta.name` field is metadata used for family grouping and display —
it does **not** have to match the file name, but keeping them equal avoids confusion.

> 中文要点：下拉里的协议名 = **文件名去掉 `.json`**，不是 `meta.name`。内置定义首次运行会补写到
> 用户目录（已存在的不覆盖），所以「用户可自行增删」这件事对内置定义同样成立。

---

## 2. Top-level shape

```json
{
  "meta":   { "name": "...", "description": "..." },
  "fields": [ /* ordered list of fields */ ],
  "checksum": { /* optional */ },
  "framing":  { /* optional, but required for variable-length frames */ }
}
```

Unknown keys are **rejected** on purpose (`deny_unknown_fields`): a typo such as `frameing`,
`length_offsets` or `"terminator": "0x7E"` fails loudly at import/load time instead of being
silently ignored ("I configured framing but nothing changed" was a real support case).

### 2.1 `meta`

| Key | Type | Required | Meaning |
|---|---|---|---|
| `name` | string | ✅ | Protocol identifier (metadata; the dropdown uses the file name). |
| `description` | string | ✅ | One line describing the protocol. |
| `version` | string | | Free text. |
| `author` | string | | Free text. |
| `tags` | string[] | | Free text. |
| `family` | string | | Family id. Protocols sharing a family id can be selected **once** as a family and dispatched by `meta.discriminator` / best-effort matching. Defaults to the text before the first `-` in `meta.name`. |
| `title` | string | | Human-readable family name shown in the family dropdown. |
| `discriminator` | object | | `{ "field": "cmd", "equals": 1 }` — "this field's parsed value identifies which message in the family this is". Makes family dispatch deterministic instead of score-based. `field` must exist in `fields`. |

### 2.2 `fields[]` (ordered)

Every field needs these five keys:

| Key | Type | Meaning |
|---|---|---|
| `name` | string | Field name (unique inside the protocol). Referenced by `length_field`, `target_field`, `covered_fields`, `discriminator.field`. |
| `field_type` | see below | How the bytes are interpreted. |
| `length` | `{"Fixed": n}` \| `{"Dynamic": {"length_field": "x"}}` \| `"Remaining"` \| `"VarInt7"` | How many bytes this field occupies. |
| `endianness` | `"BigEndian"` \| `"LittleEndian"` | Byte order **of this field only** — per field, not per protocol. |
| `description` | string | Free text (shown to nobody yet, but keep it — it is your field documentation). |

`field_type` values:

| Value | Meaning |
|---|---|
| `"UnsignedInt"` / `"SignedInt"` | Integer, width from `length`. |
| `"Float32"` / `"Float64"` | IEEE float, `length` must be `{"Fixed": 4}` / `{"Fixed": 8}`. |
| `"RawBytes"` | Opaque bytes. |
| `"Utf8String"` | UTF-8 text. |
| `{"FixedValue": [170, 85]}` | Magic/constant: must equal these bytes, otherwise parsing fails with a mismatch (a good early "this is not my protocol" detector). |
| `{"LengthIndicator": {"target_field": "payload"}}` | This integer's value is a byte count for `target_field`. |
| `{"Bits": [{"name": "alarm", "bit_offset": 7, "bit_width": 1, "data_type": "Bool"}]}` | Bit fields inside one fixed-width field. `data_type` is `"Bool"` / `"UInt"` / `"Int"`; offsets are counted from the LSB. `bit_offset + bit_width` must fit the field width. |
| `{"Conditional": {"branches": {"1": "sub-a"}, "default": "sub-b"}}` | Parse a different sub-protocol depending on this field's integer value. `branches` or `default` must be present. |
| `{"Repeat": {"item": "record", "count_field": "n"}}` | Repeat a sub-protocol. Give **exactly one** of `count_field` (fixed-width records) or `length_field` (byte-sized region, optional `length_delta`). |

### 2.3 `checksum` (optional)

```json
"checksum": {
  "algorithm": "CRC16Modbus",
  "covered_fields": ["header", "seq", "payload"],
  "result_field": "crc",
  "endianness": "LittleEndian"
}
```

`algorithm` is one of `CRC8`, `CRC16`, `CRC32`, `CRC16Modbus`, `CRC16CCITT`, `XOR`, `Sum8`, `Sum16`.
Coverage is exactly the listed fields, **in the listed order**, starting from the first covered
field. `result_field` must be one of your fields, and its declared width must match the
algorithm's output width (CRC16 → 2 bytes). `endianness` here is the byte order of the
**checksum result**, independent of the covered fields' own byte order.

---

## 3. Framing: how one frame is cut out of the byte stream

`framing` answers "where does a frame start and end" — it is orthogonal to `fields`. Without a
`framing` block, a frame is exactly the sum of its fields, which only works for fixed-length
frames.

```json
"framing": { "mode": "LengthField", "length_field": "len", "length_field_offset": 2 }
```

| Key | Applies to | Meaning |
|---|---|---|
| `mode` | required | `"Fixed"` \| `"LengthField"` \| `"Terminator"`. |
| `length_field` | `LengthField` | Name of the integer field that carries the length. Must exist and be `UnsignedInt`/`LengthIndicator` with a `Fixed` length. |
| `length_field_offset` | `LengthField` | Byte offset **of the length field inside the frame** (skip magic/header bytes before it). Default `0`. |
| `length_encoding` | `LengthField` | `"Fixed"` (default) or `"VarInt7"` (MQTT-style remaining length, 1–4 bytes, 7 bits per byte, MSB = "more follows"). With `VarInt7` do **not** set `length_field` — the value is not a protocol field; position comes from `length_field_offset`. |
| `includes` | `LengthField` | What the length value already counts — see §4. |
| `terminator` | `Terminator` | Trailer byte sequence, e.g. `[13, 10]` for CRLF or `[126]` for `0x7E`. Must be non-empty. |
| `search_from` | `Terminator` | Offset where scanning for the terminator starts (default `0`). Set it past your fixed header so a frame cannot be mistaken for an empty one. |
| `max_frame` | all | Safety valve: upper bound for one frame. `0`/omitted = built-in default (**512** bytes). If a garbage length value claims more than this, parsing errors out instead of silently swallowing the whole connection. |

> 中文要点：`framing` 与 `fields` 是两件事 —— `fields` 说"一帧内部怎么解释"，`framing` 说
> "一帧从哪开始、到哪结束"。全定长可以不写 `framing`；变长（长度前缀 / 结束符）**必须写**，
> 否则该协议在界面里会被标灰（"不能自动分帧"，只能手动粘贴解析）。

---

## 4. `includes`: the length value's meaning (read this twice)

The frame length is computed as:

```text
total = value + includes.delta
if !includes.self:   total += width_of_length_field
if !includes.header: total += offset_of_length_field        // = header bytes before it
```

Both switches mean "the length value **already counts** this part", so a part that is *not*
counted gets **added**:

| `self` | `header` | Length value means | Typical wording in a spec |
|---|---|---|---|
| `false` (default) | `false` (default) | payload bytes only | "data length" |
| `false` | `true` | header + payload | "length of the whole message except the length field itself" |
| `true` | `true` | header + length field + payload | "total frame length" |
| `true` | `false` | length field + payload | rare |

`delta` is an extra signed correction (e.g. a spec whose length excludes a 2-byte CRC → `delta: 2`).
There is deliberately no `trailer: true` — trailer size is not knowable in general, so use `delta`.

**Consistency rule of thumb:** if a `Dynamic` field says "this value is exactly the size of this
region", that only matches the **default** (`self=false`, `header=false`). If your length value is
a total frame length, do not also declare variable-length fields from it — declare their widths
explicitly (or leave them `"Remaining"` when they run to the end of the frame).

> 中文要点：`includes.self` / `includes.header` 的语义是「长度值**已经**把这块算进去了」，
> 所以没算进去的会**加回来**。默认（都不写）= 长度值只表示"长度字段之后的内容"。
> 最常见的错法：长度值其实是整帧总长，却按默认解释 ⇒ 每帧都多算一个长度字段的宽度。

---

## 5. Three minimal framing examples

### 5.1 Fixed length (no `framing` needed)

```json
{
  "meta": { "name": "my-fixed", "description": "Fixed 6-byte frame: AA | seq | value | crc" },
  "fields": [
    { "name": "header", "field_type": { "FixedValue": [170] }, "length": { "Fixed": 1 }, "endianness": "BigEndian" },
    { "name": "seq",    "field_type": "UnsignedInt", "length": { "Fixed": 1 }, "endianness": "BigEndian" },
    { "name": "value",  "field_type": "UnsignedInt", "length": { "Fixed": 2 }, "endianness": "LittleEndian" },
    { "name": "crc",    "field_type": "UnsignedInt", "length": { "Fixed": 2 }, "endianness": "LittleEndian" }
  ],
  "checksum": {
    "algorithm": "CRC16Modbus",
    "covered_fields": ["header", "seq", "value"],
    "result_field": "crc",
    "endianness": "LittleEndian"
  }
}
```

Every field is `Fixed` and there is no `Conditional`/`Repeat`, so the frame length is 6 bytes and
the protocol is framing-capable by itself. Writing `"framing": { "mode": "Fixed" }` is allowed and
equivalent — it only makes the intent explicit.

### 5.2 Length prefix (variable payload)

```json
{
  "meta": { "name": "my-length", "description": "AA 55 | len (2B BE) | payload (len bytes)" },
  "framing": { "mode": "LengthField", "length_field": "len", "length_field_offset": 2 },
  "fields": [
    { "name": "magic",   "field_type": { "FixedValue": [170, 85] }, "length": { "Fixed": 2 }, "endianness": "BigEndian" },
    { "name": "len",     "field_type": "UnsignedInt", "length": { "Fixed": 2 }, "endianness": "BigEndian" },
    { "name": "payload", "field_type": "RawBytes", "length": { "Dynamic": { "length_field": "len" } }, "endianness": "BigEndian" }
  ]
}
```

`includes` is omitted on purpose: the default meaning ("length = bytes after the length field")
is exactly what the `Dynamic` payload expects. A frame with `len = 3` is
`AA 55 00 03 xx xx xx` — 7 bytes.

### 5.3 Terminator

```json
{
  "meta": { "name": "my-term", "description": "AA | text | CRLF" },
  "framing": { "mode": "Terminator", "terminator": [13, 10], "search_from": 1 },
  "fields": [
    { "name": "header", "field_type": { "FixedValue": [170] }, "length": { "Fixed": 1 }, "endianness": "BigEndian" },
    { "name": "text",   "field_type": "Utf8String", "length": "Remaining", "endianness": "BigEndian" }
  ]
}
```

`search_from: 1` starts scanning after the 1-byte header, so a frame can never be mistaken for an
empty one. The terminator is consumed by the framer and is **not** part of `text`.
`frame = AA 4F 4B 0D 0A` → `text = "OK"`.

---

## 6. A minimal protocol you can paste, and what parsing it looks like

Save this as `my-greenhouse.json` in the protocols folder (or use **Import protocol files**):

```json
{
  "meta": { "name": "my-greenhouse", "description": "Minimal example: AA | temp (2B LE) | humidity (1B)" },
  "fields": [
    { "name": "header",   "field_type": { "FixedValue": [170] }, "length": { "Fixed": 1 }, "endianness": "BigEndian" },
    { "name": "temp_c",   "field_type": "UnsignedInt", "length": { "Fixed": 2 }, "endianness": "LittleEndian" },
    { "name": "humidity", "field_type": "UnsignedInt", "length": { "Fixed": 1 }, "endianness": "BigEndian" }
  ]
}
```

Select `my-greenhouse` on the protocol parser page and paste this example message:

```text
AA1900 3C      →  AA 19 00 3C   (hex "AA19003C")
```

Parse result (4 bytes in, one complete 4-byte frame, no leftover):

| Field | Offset | Raw bytes | Value | Why |
|---|---|---|---|---|
| `header` | 0 | `AA` | `—` | `FixedValue` matched `0xAA`; a mismatch would fail with "expected aa, actual ..". |
| `temp_c` | 1–2 | `19 00` | `25` | `LittleEndian` ⇒ `0x0019` = 25. |
| `humidity` | 3 | `3C` | `60` | `0x3C` = 60. |

The page also reports `frame 4 bytes / input 4 bytes` and marks the frame as complete. Feed a
longer buffer (e.g. two frames back to back) and the extra bytes are shown as *sticky packets*
with a **Parse remaining** button, so nothing is silently dropped.

> 中文要点：上面这份就是"能直接粘贴用"的最小协议。`header` 用 `FixedValue` 当魔数，
> 不匹配会立刻报错；小端字段要显式写 `LittleEndian`（每个字段自己带字节序）。

---

## 7. Import behaviour (what happens to your files)

Import validates **everything before writing anything**:

1. Every selected `.json` (or every `.json` inside a selected folder) is read and validated with
   the same parser and validator the runtime uses. A file that is not valid JSON, or whose
   definition is inconsistent (duplicate field names, a `length_field` pointing at a
   non-existent field, a `FixedValue` whose length contradicts `length`, …) is reported with a
   readable reason and **nothing is copied at all** — no "half imported" state.
2. If the file name already exists, the file is **never overwritten**: it is saved as
   `my-protocol-user-1.json` (then `-user-2`, …) and the page tells you exactly which existing
   file was in the way, so you can delete it yourself in the protocols folder if you want the
   original name back.
3. A protocol that parses but cannot auto-frame is still imported; the page repeats the same
   reason the dropdown shows for greyed-out protocols (e.g. variable-length fields without a
   `framing` block). It is usable for pasted-hex parsing, not for stream framing.
4. After a successful import the list refreshes automatically.

> 中文要点：导入**先全部校验、再落盘**，有一份不合格就一份都不写；同名**绝不覆盖**，
> 改成 `-user-1.json` 并在界面上说清是谁挡住了；能解析但不能分帧的也会导入，但会给出原因。

---

## 8. Common errors and what they look like

| Symptom | Likely cause | Fix |
|---|---|---|
| "length_field 'x' does not exist" at import | `framing.length_field` / `Dynamic.length_field` names a field that is not in `fields` (typos, or the length field was renamed). | Use the exact `name` string. |
| Valid frames but the second frame is garbage, or every frame is reported as half | `length_field_offset` is wrong (it is the offset **inside the frame**; forgetting a 2-byte magic shifts it). | Count bytes from the start of the frame to the first byte of the length field. |
| Every frame's length is off by exactly the length field's width (or the header width) | `includes.self` / `includes.header` do not match the spec's wording for the length value. | See §4 — the switches mean "already counted", un-counted parts are added back. |
| "checksum mismatch" on correct-looking data | Wrong coverage or order in `covered_fields`, wrong `algorithm`, wrong checksum `endianness`, or a `result_field` whose width does not match the algorithm. | List covered fields in wire order; CRC16 variants are `CRC16Modbus` / `CRC16CCITT`; CRC result byte order is separate from the data's. |
| Field values are wildly wrong (e.g. 256 becomes 1) | Field `endianness` does not match the device. Only the length field's byte order is taken from that field. | Set `"endianness": "LittleEndian"` per field for little-endian devices. |
| Import fails with a JSON error although the JSON parses elsewhere | Unknown/misspelled key (unknown keys are rejected), a hex **string** where a byte array is expected (`"FixedValue"` needs `[170]`, not `"AA"`), or a missing `meta.name` / `meta.description`. | Remove the extra key, use numbers for bytes, keep the required `meta` keys. |
| The protocol was imported but is not in the dropdown | The file name is not `*.json` (case matters to the loader — `.JSON` is normalised to `.json` on import), the name is the reserved `fixtures.json`, or the file is not in the protocols folder. | Rename to `name.json` inside the protocols folder, then **Refresh list**. |
| "FrameTooLarge" / connection appears dead | The length field is being read from the wrong place, so garbage (e.g. `0xFFFFFFFF`) is interpreted as a length. | Fix `length_field_offset`/`includes`; keep `max_frame` at a realistic value to fail fast instead of buffering. |

> 中文要点：最常见的四类错误 —— ① 长度字段**偏移**算错（忘了帧头/魔数）② `includes` 把
> "长度值是什么"理解反了 ③ CRC 的**覆盖范围/位置/字节序**与文档不一致 ④ **字节序**按错
> （每个字段各自带 `endianness`）。另外：未知键会被**拒绝**而不是忽略，写错键名会直接报错。

---

## 9. Responsibility

* **Your own definitions are yours.** A protocol definition you write (or import) describes a wire
  format; you are responsible for its correctness and for holding the rights to any specification,
  document, or device behaviour you derived it from.
* **Importing is copying your own JSON text.** The app does not ship, fetch, update, or recommend
  third-party protocol packages; it never decrypts or bypasses anything.
* **The built-in library** contains publicly documented formats plus the app's own synthetic
  examples. Definitions derived from a customer/project specification are **not** described,
  itemised, or attributed in any user-visible material, including this document.
* Deleting or editing files in the protocols folder is always available to you (see
  **Open protocols folder**); the app only ever writes a built-in definition when that file is
  missing.

> 中文要点：自写/导入的协议定义由**用户自行负责**（包括其依据的规格书/设备行为的权利）；
> 导入只是搬运你自己的 JSON 文本；内置协议库只含公开标准与应用自造示例，**不在用户可见处**
> 列举或署名任何客户/项目派生定义。

---

## 10. Related implementation files

| File | Role |
|---|---|
| `src-tauri/src/commands/protocol_import.rs` | Import command + user protocols folder command (validate-then-write, no silent overwrite). |
| `src-tauri/src/commands/protocol_builtin.rs` | Which built-ins are compiled in, and how missing ones are written to the user folder. |
| `src-tauri/src/commands/protocol.rs` | `list_protocols` / `list_protocol_families` / `parse_hex` (framing capability + human-readable reason). |
| `super-engine-codec/core-codec/src/definition.rs` | The schema itself: field types, `framing`, `includes`, and all validation rules. |
| `super-engine-ui/test-protocols/*.json` | Working examples for every framing mode (e.g. `demo-length-total.json`, `demo-terminator.json`). |

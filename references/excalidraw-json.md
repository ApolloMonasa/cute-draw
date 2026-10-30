# Excalidraw JSON 模板与校验（cute-draw）

## 文件整体结构

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [ ... ],
  "appState": { "gridSize": null, "viewBackgroundColor": "#ffffff" },
  "files": {}
}
```

## 元素通用字段（每个元素都必须齐全，缺一个 Excalidraw 就渲染异常）

```json
{
  "id": "唯一id",
  "type": "rectangle|ellipse|text|arrow|line|diamond",
  "x": 0, "y": 0, "width": 100, "height": 40, "angle": 0,
  "strokeColor": "#1971c2",
  "backgroundColor": "transparent 或十六进制色",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid 或 dashed",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "roundness": { "type": 3 },          // rectangle 用；text/ellipse/arrow 用 null
  "seed": 随机整数,
  "version": 1,
  "isDeleted": false,
  "boundElements": null,
  "updated": 1,
  "link": null,
  "locked": false
}
```

### text 元素追加

```json
"text": "内容",
"fontSize": 16, "fontFamily": 5,       // 5 = 手写体
"textAlign": "center 或 left", "verticalAlign": "top",
"containerId": null, "originalText": "与 text 相同",
"autoResize": true, "lineHeight": 1.25
```

### arrow 元素追加

```json
"points": [[0,0],[100,0]],
"lastCommittedPoint": null, "startBinding": null, "endBinding": null,
"startArrowhead": null, "endArrowhead": "arrow"
```

斜向箭头用负 y 的 points，如 `[[0,0],[62,-60]]`。

## 历史踩坑（真实翻过的车，写时逐条自查）

| 错误形态 | 后果 | 正确写法 |
|---|---|---|
| `"opacity": 1` | 元素几乎不可见 | `100` |
| `lineSize` / `lineWidth` | 字段被忽略 | `lineHeight: 1.25` |
| text 元素漏 `text` 或 `originalText` | 文字不显示 | 两者都写且一致 |
| arrow 漏 `points` | 箭头退化成点 | 必带 points 与 endArrowhead |
| 重复 `id`（复制元素忘改） | 后者覆盖前者 | id 唯一，seed 也别全同 |
| `strokeColor2`、`GroupsIds`、`fontFamily5` | 整个字段失效 | 键名拼写精确 |
| 非法色值 `#e03101` | 描边渲染错误 | 六位十六进制，常用值见 SKILL.md |
| 键值对少右引号 `"width:190` | JSON 解析失败 | 写完必跑校验 |

## 校验脚本（写完文件立即执行，不过不交付）

```bash
python3 - <<'EOF'
import json
p='<文件路径>'
d=json.load(open(p))
els=d['elements']
ids=[e['id'] for e in els]
assert len(ids)==len(set(ids)), 'duplicate ids'
bad_op=[e['id'] for e in els if e.get('opacity')!=100]
bad_lh=[e['id'] for e in els if e['type']=='text' and 'lineHeight' not in e]
bad_tx=[e['id'] for e in els if e['type']=='text' and not e.get('text')]
bad_pts=[e['id'] for e in els if e['type']=='arrow' and 'points' not in e]
req=['id','type','x','y','width','height','strokeColor','backgroundColor','fillStyle',
     'strokeWidth','strokeStyle','roughness','opacity','groupIds','seed','version',
     'isDeleted','boundElements','updated','link','locked']
missing=[(e['id'],k) for e in els for k in req if k not in e]
maxx=max(e['x']+e.get('width',0) for e in els)
maxy=max(e['y']+e.get('height',0) for e in els)
print('elements:',len(els),'| dup:',len(ids)!=len(set(ids)),'| opacity:',bad_op,
      '| lineHeight:',bad_lh,'| text:',bad_tx,'| points:',bad_pts,
      '| missing:',missing,'| max:',maxx,maxy)
EOF
```

判定标准：无 duplicate、四个 bad 列表全空、missing 为空、`max x ≤ 1420` 且 `max y ≤ 1010`。
若有小错误，最稳的修法是 `json.load → 改 dict → json.dump`，不要用 sed 盲修。

## 布局速查

- 主标题 `y=26`（28-32px），副标题 `y=64`（17-18px）
- 问题卡 `(80,104,1260,88)`：标签 `(104,128,72,40)`、题目 `(210,122)`、层面 `(210,154)`
- 场景 `(60,220,1300,590)`；分隔文字 `(614,836)`；知识卡 `y=866`
- 文本放框内留 16px 内边距；正文字号 15-16px 时每行 ≤14 字

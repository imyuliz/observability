# Action 分析和示例

参考文档:
-  www.baidu.com

### Source Code


```
// Action is the action to be performed on relabeling.
type Action string

const (
	// Replace performs a regex replacement.
	Replace Action = "replace"
	// Keep drops targets for which the input does not match the regex.
	Keep Action = "keep"
	// Drop drops targets for which the input does match the regex.
	Drop Action = "drop"
	// HashMod sets a label to the modulus of a hash of labels.
	HashMod Action = "hashmod"
	// LabelMap copies labels to other labelnames based on a regex.
	LabelMap Action = "labelmap"
	// LabelDrop drops any label matching the regex.
	LabelDrop Action = "labeldrop"
	// LabelKeep drops any label not matching the regex.
	LabelKeep Action = "labelkeep"
)

```

### 
---
title: "Spicedb_lex"
date: 2022-05-27T11:14:52+08:00
draft: true
---

```golang
package input

type BytePosition int // 代码块中的位置

type Position struct {  // 源码中的位置
	// 行号
	LineNumber int

	// 列号
	ColumnPosition int
}
// 源文件
type Source string

// 源文件中的范围
type SourceRange interface {
  Source() Source // 源文件
  Start() SourcePosition // 范围开始的位置
  End() SourcePosition // 范围结束的位置，包含当前位置
  ContainsPosition(position SourcePosition) (bool, error) // 判断当前位置是否在范围中
  AtStartPosition() SourceRange // 为范围开始位置时才返回范围
  String() string // 返回可读的范围
}
// 源文件中的单个位置
type SourcePosition interface {
  Source() Source // 源文件
  RunePosition() (int, error) // 源文件中 rune 位置
  LineAndColumn() (int, int, error)  // 行号和列号
  LineText() (string, error) // 当前位置所在行的文本
  String() string // 返回可读的位置
}
// 实现 SourceRange 接口
type sourceRange struct {
	source Source
	start  SourcePosition
	end    SourcePosition
}
func (sr sourceRange) Source() Source {
	return sr.source
}

func (sr sourceRange) Start() SourcePosition {
	return sr.start
}

func (sr sourceRange) End() SourcePosition {
	return sr.end
}
// 返回当前 sourceRange？
func (sr sourceRange) AtStartPosition() SourceRange {
	return sourceRange{sr.source, sr.start, sr.end}
}

func (sr sourceRange) ContainsPosition(position SourcePosition) (bool, error) {
  
}
```
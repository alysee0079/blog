# compiler

对于 vue 中的 compiler 而言，它的核心作用就是把 template 模板 编译成 render 函数.

1. 解析 (parse) template 模板, 生成模板 AST
2. 转化 (transform) 将 AST 转化为 JavaScript AST
3. 根据 JavaScript AST 生成(generate) render 函数

#### 解析 (parse)

1. 通过自动状态机把 template 解析为 tokens
2. 通过递归下降算法把 tokens 通过栈解析成了 AST（抽象语法树）

```javascript
baseParse -> createRoot -> parseChildren
// 生成 ast
function createRoot(
  children: TemplateChildNode[],
  loc = locStub
): RootNode {
  return {
    type: NodeTypes.ROOT,
    children,
    helpers: [],
    components: [],
    directives: [],
    hoists: [],
    imports: [],
    cached: 0,
    temps: 0,
    codegenNode: undefined,
    loc
  }
}

// 解析子节点
function parseChildren(
  context: ParserContext,
  mode: TextModes,
  ancestors: ElementNode[]
): TemplateChildNode[] {
  // 获取当前节点的父节点
  const parent = last(ancestors)
  const ns = parent ? parent.ns : Namespaces.HTML
  // 这个 nodes 就是生成的 AST 中的 children
  const nodes: TemplateChildNode[] = []

  while (!isEnd(context, mode, ancestors)) {
    __TEST__ && assert(context.source.length > 0)
    const s = context.source
    let node: TemplateChildNode | TemplateChildNode[] | undefined = undefined

    if (mode === TextModes.DATA || mode === TextModes.RCDATA) {
      if (!context.inVPre && startsWith(s, context.options.delimiters[0])) {
        // 处理双大括号
        // '{{'
        node = parseInterpolation(context, mode)
      } else if (mode === TextModes.DATA && s[0] === '<') {
        // 处理 <
        // https://html.spec.whatwg.org/multipage/parsing.html#tag-open-state
        if (s.length === 1) {
          emitError(context, ErrorCodes.EOF_BEFORE_TAG_NAME, 1)
        } else if (s[1] === '!') {
          if (startsWith(s, '<!--')) {
            // 处理注释节点
            node = parseComment(context)
          } else if (startsWith(s, '<!DOCTYPE')) {
            // <!DOCTYPE html> 节点
            node = parseBogusComment(context)
          }
        } else if (s[1] === '/') {
          if (s.length === 2) {
            emitError(context, ErrorCodes.EOF_BEFORE_TAG_NAME, 2)
          } else if (s[2] === '>') {
            emitError(context, ErrorCodes.MISSING_END_TAG_NAME, 2)
            advanceBy(context, 3)
            continue
          } else if (/[a-z]/i.test(s[2])) {
            emitError(context, ErrorCodes.X_INVALID_END_TAG)
            parseTag(context, TagType.End, parent)
            continue
          }
        } else if (/[a-z]/i.test(s[1])) {
          // 处理元素标签名
          node = parseElement(context, ancestors)

          // 2.x <template> with no directive compat
          if (
            __COMPAT__ &&
            isCompatEnabled(
              CompilerDeprecationTypes.COMPILER_NATIVE_TEMPLATE,
              context
            ) &&
            node &&
            node.tag === 'template' &&
            !node.props.some(
              p =>
                p.type === NodeTypes.DIRECTIVE &&
                isSpecialTemplateDirective(p.name)
            )
          ) {
            __DEV__ &&
              warnDeprecation(
                CompilerDeprecationTypes.COMPILER_NATIVE_TEMPLATE,
                context,
                node.loc
              )
            node = node.children
          }
        } else if (s[1] === '?') {
          emitError(
            context,
            ErrorCodes.UNEXPECTED_QUESTION_MARK_INSTEAD_OF_TAG_NAME,
            1
          )
          node = parseBogusComment(context)
        } else {
          emitError(context, ErrorCodes.INVALID_FIRST_CHARACTER_OF_TAG_NAME, 1)
        }
      }
    }
    // 处理文本节点
    if (!node) {
      node = parseText(context, mode)
    }

    if (isArray(node)) {
      for (let i = 0; i < node.length; i++) {
        // 将子节点添加到节点列表
        pushNode(nodes, node[i])
      }
    } else {
      // 将子节点添加到节点列表
      pushNode(nodes, node)
    }
  }

  // 处理空格换行
  // Whitespace handling strategy like v2
  let removedWhitespace = false
  if (mode !== TextModes.RAWTEXT && mode !== TextModes.RCDATA) {
  }

  return removedWhitespace ? nodes.filter(Boolean) : nodes
}

function parseElement(
  context: ParserContext,
  ancestors: ElementNode[]
): ElementNode | undefined {
  __TEST__ && assert(/^<[a-z]/i.test(context.source))

  // 获取当前节点的父节点
  const parent = last(ancestors)
  // 处理开始标签, 获取元素对象
  const element = parseTag(context, TagType.Start, parent)

  // Children.
  // 插入当前节点作为正在处理的节点
  ancestors.push(element)
  const mode = context.options.getTextMode(element, parent)
  // 解析子节点
  const children = parseChildren(context, mode, ancestors)
  // 将当前节点删除
  ancestors.pop()

  // 给当前节点添加子节点
  element.children = children

  // End tag.
  // 处理结束标签
  if (startsWithEndTagOpen(context.source, element.tag)) {
    parseTag(context, TagType.End, parent)
  }

  return element
}
```

#### 转化(transform)

注意点:

1. 深度优先(自下而上, 因为父节点可能需要根据子节点生成)
2. 转化函数分离
3. 上下文对象

整个 transform 的逻辑可以大致分为两部分：

1. 深度优先排序，通过 traverseNode 方法，完成排序逻辑
2. 通过保存在 nodeTransforms 中的 transformXXX 方法，针对不同的节点，完成不同的处理
3. 期间创建的 context 上下文，承担了一个全局单例的作用

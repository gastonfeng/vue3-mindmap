<template>
  <div :class="style['container']">
    <div :class="style['svg']">
      <svg :class="style['svg']" ref="svgEle">
        <g ref="gEle">
          <foreignObject ref="foreignEle" style="display: none">
            <div ref="foreignDivEle" contenteditable></div>
          </foreignObject>
        </g>
      </svg>
    </div>
    <svg ref="asstSvgEle"></svg>
    <div :class="[style['button-list'], style['right-bottom']]">
      <button v-if="centerBtn" @click="centerView()"><i :class="style['gps']"></i></button>
      <button v-if="fitBtn" @click="fitView()"><i :class="style['fit']"></i></button>
      <button v-if="downloadBtn" @click="download()"><i :class="style['download']"></i></button>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, onMounted, PropType, Ref, ref, watch } from 'vue'
import { Data, Mdata } from '@/interface'
import style from './Mindmap.module.scss'
import { d3, ImData, getAddPath, getMultiline } from '@/tools'
import html2canvas from 'html2canvas'

type TspanData = { name: string, height: number }
type SelectionG = d3.Selection<SVGGElement, Mdata, SVGGElement, Mdata | null>
type SelectionRect = d3.Selection<SVGRectElement, Mdata, SVGGElement, Mdata | null>
type Transition = d3.Transition<d3.BaseType, Mdata, SVGGElement, unknown>

export default defineComponent({
  name: 'Mindmap',
  props: {
    modelValue: {
      type: Array as PropType<Data[]>,
      required: true
    },
    width: Number,
    height: Number,
    xGap: {
      type: Number,
      default: 50
    },
    yGap: {
      type: Number,
      default: 18
    },
    branch: {
      type: Number,
      default: 4
    },
    centerBtn: Boolean,
    fitBtn: Boolean,
    downloadBtn: Boolean,
    undoBtn: Boolean,
    addNodeBtn: Boolean,
    edit: Boolean,
    drag: Boolean,
    keyboard: Boolean,
    contextMenu: Boolean,
    zoom: Boolean
  },
  setup (props) {
    const svgEle: Ref<SVGSVGElement | undefined> = ref()
    const gEle: Ref<SVGGElement | undefined> = ref()
    const asstSvgEle: Ref<SVGSVGElement | undefined> = ref()
    const foreignEle: Ref<SVGForeignObjectElement | undefined> = ref()
    const foreignDivEle: Ref<HTMLDivElement | undefined> = ref()
    const svg: Ref<d3.Selection<SVGSVGElement, null, null, undefined> | undefined> = ref()
    const g: Ref<d3.Selection<SVGGElement, null, null, undefined> | undefined> = ref()
    const asstSvg: Ref<d3.Selection<SVGSVGElement, unknown, null, undefined> | undefined> = ref()
    const foreign: Ref<d3.Selection<SVGForeignObjectElement, null, null, undefined> | undefined> = ref()
    const link = d3.linkHorizontal().source((d) => d.source).target((d) => d.target)
    const zoom = d3.zoom<SVGSVGElement, null>().on('zoom', onZoomMove).scaleExtent([0.1, 8])
    const drag = d3.drag<SVGGElement, Mdata>().on('drag', onDragMove).on('end', onDragEnd)
    const observer = new ResizeObserver((arr) => {
      if (!foreign.value) { return }
      const temp = arr[0]
      const target = temp.target
      const pl = parseInt(getComputedStyle(target).paddingLeft || '0', 10)
      const b = parseInt(getComputedStyle(target.parentNode as Element).borderTopWidth || '0', 10)
      const gap = (pl + b) * 2
      foreign.value.attr('width', temp.contentRect.width + gap).attr('height', temp.contentRect.height + gap)
    })
    let mmdata: ImData
    let editFlag = false
    let showAddNodeBtn = true
    const rootTextRectRadius = 6
    const rootTextRectPadding = 10
    const textRectPadding = computed(() => Math.min(props.yGap / 2 - 1, rootTextRectPadding))
    const addBtnRect = { side: 12, padding: 2 }
    const addBtnSide = addBtnRect.side + addBtnRect.padding * 2

    onMounted(() => {
      if (!svgEle.value || !gEle.value || !asstSvgEle.value || !foreignEle.value || !foreignDivEle.value) { return }
      svg.value = d3.select(svgEle.value)
      g.value = d3.select(gEle.value)
      asstSvg.value = d3.select(asstSvgEle.value).attr('width', 0).attr('height', 0)
      foreign.value = d3.select(foreignEle.value)
      observer.observe(foreignDivEle.value)

      mmdata = new ImData(
        JSON.parse(JSON.stringify(props.modelValue[0])),
        props.xGap,
        props.yGap,
        getSize
      )

      draw()
      foreign.value.raise()
      foreignDivEle.value.addEventListener('blur', onEditBlur)
      foreignDivEle.value.addEventListener('mousedown', (e: MouseEvent) => e.stopPropagation())
      centerView()
      fitView()
      // mousedown与drag/zoom绑定的先后顺序会有影响
      svg.value.on('mousedown', () => {
        const oldSele = document.getElementsByClassName(style.selected)[0]
        oldSele?.classList.remove(style.selected)
      })
      switchZoom(props.zoom)
    })
    // watch
    watch(() => [props.branch, props.addNodeBtn], () => draw())
    watch(() => [props.xGap, props.yGap], (val) => {
      mmdata.setBoundingBox(val[0], val[1])
      draw()
    })
    watch(() => [props.drag, props.edit], (val) => {
      switchSelect(val[0] || val[1])
      switchDrag(val[0])
      switchEdit(val[1])
    })
    watch(() => props.zoom, (val) => switchZoom(val))
    // 每个属性的计算方法
    const getGClass = (d?: Mdata) => {
      const arr = ['node']
      if (d) {
        arr.push(`depth-${d.depth}`)
        if (d.depth === 0) { arr.push(style.root) }
      }
      return arr
    }
    const getGTransform = (d: Mdata) => { return `translate(${d.dx + d.px},${d.dy + d.py})` }
    const getDataId = (d: Mdata): string => { return d.id }
    const getPath = (d: Mdata) => {
      let dpw = 0
      let dph = 0
      if (d.parent) {
        dpw = d.parent.width
        dph = d.parent.height
        if (d.parent.depth === 0) {
          dpw /= 2
          dph /= 2
        }
      }
      const temp = props.branch / 2
      const source = [-d.dx + dpw - d.px, -d.dy + dph + temp - d.py] as [number, number]
      const target = [0, d.height + temp] as [number, number]
      const trp = Math.max(textRectPadding.value - 3, 0)
      return `${link({ source, target })}L${d.width + trp},${target[1]}`
    }
    const getTspanData = (d: Mdata): TspanData[] => {
      const multiline = getMultiline(d.name)
      const height = d.height / multiline.length
      return multiline.map((name) => ({ name, height }))
    }
    const getAddBtnTransform = (d: Mdata, trp: number) => {
      const gap = 8
      const y = d.depth === 0 ? d.height / 2 : d.height + props.branch / 2
      return `translate(${d.width + trp + addBtnSide / 2 + gap},${y})`
    }
    // 每个图形的绘制方法
    const attrG = (g: SelectionG, tran?: Transition) => {
      const temp1 = g.attr('class', (d) => getGClass(d).join(' ')).attr('data-id', getDataId)
      const temp2 = tran ? temp1.transition(tran) : temp1
      temp2.attr('transform', getGTransform)
    }
    const attrPath = (p: d3.Selection<SVGPathElement, Mdata, SVGGElement, Mdata | null>, tran?: Transition) => {
      const temp1 = p.attr('stroke', (d) => d.color).attr('stroke-width', props.branch)
      const temp2 = tran ? temp1.transition(tran) : temp1
      temp2.attr('d', getPath)
    }
    const attrTspan = (tspan: d3.Selection<SVGTSpanElement, TspanData, SVGTextElement, Mdata>) => {
      tspan.attr('alignment-baseline', 'text-before-edge')
        .text((d) => d.name || ' ')
        .attr('x', 0)
        .attr('dy', (d, i) => i ? d.height : 0)
    }
    const attrTextRect = (rect: SelectionRect, padding = textRectPadding.value, radius = 4) => {
      rect.attr('x', -padding).attr('y', -padding).attr('rx', radius).attr('ry', radius)
        .attr('width', (d) => d.width + padding * 2).attr('height', (d) => d.height + padding * 2)
    }
    const attrAddBtn = (g: SelectionG, trp = textRectPadding.value) => {
      g.attr('class', style['add-btn']).attr('transform', (d) => getAddBtnTransform(d, trp))
    }
    const attrAddBtnRect = (rect: SelectionRect) => {
      const side = addBtnRect.side
      const padding = addBtnRect.padding
      const radius = 4
      const temp0 = -padding - side / 2
      const temp1 = side + padding * 2
      rect.attr('x', temp0).attr('y', temp0).attr('rx', radius).attr('ry', radius).attr('width', temp1).attr('height', temp1)
    }
    const attrTrigger = (rect: SelectionRect, padding = textRectPadding.value) => {
      rect.attr('class', style.trigger)
        .attr('x', -padding)
        .attr('y', -padding)
        .attr('width', (d) => d.width + padding * 2 + 8 + addBtnSide)
        .attr('height', (d) => d.height + padding * 2)
    }
    // 绘制节点的方法
    const appendTspan = (enter: d3.Selection<d3.EnterElement, TspanData, SVGTextElement, Mdata>) => {
      const tspan = enter.append('tspan')
      attrTspan(tspan)
      return tspan
    }
    const updateTspan = (update: d3.Selection<SVGTSpanElement, TspanData, SVGTextElement, Mdata>) => {
      attrTspan(update)
      return update
    }
    const appendNode = (enter: d3.Selection<d3.EnterElement, Mdata, SVGGElement, Mdata | null>) => {
      const isRoot = !enter.data()[0]?.depth
      const enterG = enter.append('g')
      attrG(enterG)
      // 绘制线
      attrPath(enterG.append('path'))
      // 节点容器
      const gContent = enterG.append('g').attr('class', style.content)
      const gTrigger = gContent.append('rect')
      // 绘制文本
      const gText = gContent.append('g').attr('class', style.text)
      const gTextRect = gText.append('rect')
      const tspan = gText.append('text').selectAll('tspan').data(getTspanData).enter().append('tspan')
      attrTspan(tspan)
      // 绘制添加按钮
      let gAddBtn
      if (props.addNodeBtn) {
        gAddBtn = gContent.append('g')
        attrAddBtnRect(gAddBtn.append('rect'))
        gAddBtn.append('path').attr('d', getAddPath(2, addBtnRect.side))
      }

      if (isRoot) {
        attrTrigger(gTrigger, rootTextRectPadding)
        attrTextRect(gTextRect, rootTextRectPadding, rootTextRectRadius)
        if (gAddBtn) { attrAddBtn(gAddBtn, rootTextRectPadding) }
      } else {
        attrTrigger(gTrigger)
        attrTextRect(gTextRect)
        if (gAddBtn) { attrAddBtn(gAddBtn) }
      }

      bindEvent(enterG, isRoot)

      enterG.each((d, i) => {
        if (!d.children) { return }
        draw(d.children, enterG.filter((a, b) => i === b))
      })
      gContent.raise()
      return enterG
    }
    const updateNode = (update: SelectionG) => {
      const isRoot = !update.data()[0]?.depth
      const tran = makeTransition(500, d3.easePolyOut)
      attrG(update, tran)
      attrPath(update.select<SVGPathElement>(':scope > path'), tran)
      const gContent = update.select<SVGGElement>(`:scope > g.${style.content}`)
      const gTrigger = gContent.select<SVGRectElement>(':scope > rect')
      const gText = gContent.select<SVGGElement>(`g.${style.text}`)
      const gTextRect = gText.select<SVGRectElement>('rect')
      gText.select<SVGTextElement>('text').selectAll<SVGTSpanElement, TspanData>('tspan')
        .data(getTspanData)
        .join(appendTspan, updateTspan, exit => exit.remove())
      let gAddBtn = gContent.select<SVGGElement>(`g.${style['add-btn']}`)
      if (props.addNodeBtn) {
        if (!gAddBtn.node()) {
          gAddBtn = gContent.append('g')
          attrAddBtnRect(gAddBtn.append('rect'))
          gAddBtn.append('path').attr('d', getAddPath(2, addBtnRect.side))
        }
      } else {
        gAddBtn.remove()
      }

      if (isRoot) {
        attrTrigger(gTrigger, rootTextRectPadding)
        attrTextRect(gTextRect, rootTextRectPadding, rootTextRectRadius)
        attrAddBtn(gAddBtn, rootTextRectPadding)
      } else {
        attrTrigger(gTrigger)
        attrTextRect(gTextRect)
        attrAddBtn(gAddBtn)
      }

      update.each((d, i) => {
        if (!d.children) { return }
        draw(d.children, update.filter((a, b) => i === b))
      })
      gContent.raise()
      return update
    }
    // 其他
    const draw = (d = [mmdata.data], sele = g.value as d3.Selection<SVGGElement, any, any, any>) => {
      const temp = sele.selectAll<SVGGElement, Mdata>(`g.${getGClass(d[0]).join('.')}`)
      temp.data(d, (d) => d.gKey).join(appendNode, updateNode)
    }
    const getSize = (text: string): { width: number, height: number } => {
      if (!asstSvg.value) { throw new Error('asstSvg undefined') }
      const multiline = getMultiline(text)
      const t = asstSvg.value.append('text')
      t.selectAll('tspan').data(multiline).enter().append('tspan').text((d) => d).attr('x', 0)
      const tBox = (t.node() as SVGTextElement).getBBox()
      t.remove()
      return {
        width: Math.max(tBox.width, 22),
        height: Math.max(tBox.height, 22) * multiline.length
      }
    }
    const selectGNode = (d: Mdata) => {
      const ele = document.querySelector(`g[data-id='${getDataId(d)}']`)
      const oldSele = document.getElementsByClassName(style.selected)[0]
      if (ele) {
        if (oldSele) {
          if (oldSele !== ele) {
            oldSele.classList.remove(style.selected)
            ele.classList.add(style.selected)
          } else {
            editFlag = true
          }
        } else {
          ele.classList.add(style.selected)
        }
      } else {
        throw new Error(`g[data-id='${getDataId(d)}'] is null`)
      }
    }
    const moveNode = (node: SVGGElement, d: Mdata, p: [number, number], dura = 0) => {
      const tran = makeTransition(dura, d3.easePolyOut)
      d.px = p[0]
      d.py = p[1]
      d3.select<SVGGElement, Mdata>(node).transition(tran).attr('transform', getGTransform)
      d3.select<SVGPathElement, Mdata>(`g[data-id='${getDataId(d)}'] > path`).transition(tran).attr('d', getPath)
    }
    const makeTransition = (dura: number, easingFn: (normalizedTime: number) => number) => {
      return d3.transition<Mdata>().duration(dura).ease(easingFn) as d3.Transition<any, Mdata, null, undefined>
    }
    const bindEvent = (g: SelectionG, isRoot: boolean) => {
      if (props.drag || props.edit) {
        g.on('mousedown', onSelect)
        if (props.drag && !isRoot) { drag(g) }
        if (props.edit) { g.on('click', onEdit) }
      }
      if (props.addNodeBtn) {
        g.select<SVGGElement>(`:scope > g.${style.content}`).on('mouseenter', onMouseEnter).on('mouseleave', onMouseLeave)
      }
    }
    // 监听事件
    function onZoomMove (e: d3.D3ZoomEvent<SVGSVGElement, null>) {
      if (!g.value) { return }
      g.value.attr('transform', e.transform.toString())
    }
    function onDragMove (this: SVGGElement, e: d3.D3DragEvent<SVGGElement, Mdata, Mdata>, d: Mdata) {
      showAddNodeBtn = false
      if (!g.value) { return }
      moveNode(this, d, [e.x - d.x, e.y - d.y])
      // 鼠标相对gEle左上角的位置
      const mousePos = d3.pointer(e, gEle.value)
      mousePos[1] += mmdata.data.y

      const temp = g.value.selectAll<SVGGElement, Mdata>('g.node').filter((other) => {
        if (other !== d && other !== d.parent && !other.id.startsWith(d.id)) {
          const rect = {
            x0: other.x - textRectPadding.value,
            x1: other.x + other.width + textRectPadding.value,
            y0: other.y - textRectPadding.value,
            y1: other.y + other.height + textRectPadding.value
          }
          return mousePos[0] > rect.x0 && mousePos[1] > rect.y0 && mousePos[0] < rect.x1 && mousePos[1] < rect.y1
        }
        return false
      })
      const old = Array.from(document.getElementsByClassName(style.outline))
      const n = temp.node()
      old.forEach((o) => { if (o !== n) { o.classList.remove(style.outline) } })
      n?.classList.add(style.outline)
    }
    function onDragEnd (this: SVGGElement, e: d3.D3DragEvent<SVGGElement, Mdata, Mdata>, d: Mdata) {
      showAddNodeBtn = true
      // 判断是否找到了新的父节点
      const np = document.getElementsByClassName(style.outline)[0]
      if (np) {
        np.classList.remove(style.outline)
        const pid = np.getAttribute('data-id')
        if (pid) {
          d.px = 0
          d.py = 0
          reparent(pid, d.id)
        } else {
          throw new Error('outline data-id null')
        }
        return
      }
      // 判断是否需要调换节点顺序
      const p = this.parentNode as SVGGElement
      let downD = d
      let upD = d
      const brothers = d3.select<SVGGElement, Mdata>(p).selectAll<SVGGElement, Mdata>(`g.${getGClass(d).join('.')}`).filter((a) => a !== d)
      brothers.each((b) => {
        if (b.y > d.y && b.y < (d.y + d.py) && b.y > upD.y) { upD = b } // 找新哥哥节点
        if (b.y < d.y && b.y > (d.y + d.py) && b.y < downD.y) { downD = b } // 找新弟弟节点
      })
      if (downD !== d) {
        d.px = 0
        d.py = 0
        move(d.id, downD.id)
        return
      } else if (upD !== d) {
        d.px = 0
        d.py = 0
        move(d.id, upD.id, 1)
        return
      }
      // 复原
      moveNode(this, d, [0, 0], 500)
    }
    function onEdit (this: SVGGElement, e: MouseEvent, d: Mdata) {
      if (editFlag && foreign.value && foreignDivEle.value) {
        this.classList.add(style.edited)
        editFlag = false
        foreign.value.attr('x', d.x - 2).attr('y', d.y - mmdata.data.y - 2)
          .attr('data-id', d.id).attr('data-name', d.name).style('display', '')
        const div = foreignDivEle.value
        div.textContent = d.name
        div.focus()
        getSelection()?.selectAllChildren(div)
      }
    }
    const onEditBlur = () => {
      document.getElementsByClassName(style.edited)[0]?.classList.remove(style.edited, style.selected)

      if (foreignEle.value && foreignDivEle.value) {
        foreignEle.value.style.display = 'none'
        const id = foreignEle.value.getAttribute('data-id')
        const oldname = foreignEle.value.getAttribute('data-name')
        const name = foreignDivEle.value.textContent
        if (id && name && name !== oldname) {
          rename(id, name)
        }
      }
    }
    const onSelect = (e: MouseEvent, d: Mdata) => {
      e.stopPropagation()
      selectGNode(d)
    }
    function onMouseEnter (this: SVGGElement) {
      if (showAddNodeBtn) {
        const temp = this.querySelector<HTMLElement>(`g.${style['add-btn']}`)
        if (temp) {
          temp.style.opacity = '1'
        }
      }
    }
    function onMouseLeave (this: SVGGElement) {
      const temp = this.querySelector<HTMLElement>(`g.${style['add-btn']}`)
      if (temp) {
        temp.style.opacity = '0'
      }
    }
    // 插件
    const switchZoom = (zoomable: boolean) => {
      if (!svg.value) { return }
      if (zoomable) {
        zoom(svg.value)
        svg.value.on('dblclick.zoom', null)
      } else {
        svg.value.on('.zoom', null)
      }
    }
    const switchEdit = (editable: boolean) => {
      if (!foreignDivEle.value || !g.value) { return }
      const gNode = g.value.selectAll<SVGGElement, Mdata>('g.node')
      if (editable) {
        gNode.on('click', onEdit)
      } else {
        gNode.on('click', null)
      }
    }
    const switchDrag = (draggable: boolean) => {
      if (!g.value) { return }
      const temp = g.value.selectAll<SVGGElement, Mdata>(`g.node:not(.${style.root})`)
      if (draggable) {
        drag(temp)
      } else {
        temp.on('.drag', null)
      }
    }
    const switchSelect = (selectable: boolean) => {
      if (!g.value) { return }
      const temp = g.value.selectAll<SVGGElement, Mdata>('g.node')
      if (selectable) {
        temp.on('mousedown', onSelect)
      } else {
        temp.on('mousedown', null)
      }
    }
    // 一次操作
    const rename = (id: string, name: string) => {
      mmdata.rename(id, name)
      draw()
    }
    const reparent = (pid: string, id: string) => {
      mmdata.reparent(pid, id)
      draw()
    }
    const move = (id: string, referenceId: string, after = 0) => {
      mmdata.move(id, referenceId, after)
      draw()
    }
    // 辅助按钮的点击事件
    const centerView = () => {
      if (!svg.value) { return }
      const data = mmdata.data
      zoom.translateTo(svg.value, 0 + data.width / 2, 0 + data.height / 2)
    }
    const fitView = () => { // 缩放至合适大小并移动至全部可见
      // bug: 缩放后的大小与容器不一致
      if (!svg.value || !gEle.value || !svgEle.value) { return }
      const gBB = gEle.value.getBBox()
      const svgBCR = svgEle.value.getBoundingClientRect()
      const multiple = Math.min(svgBCR.width / gBB.width, svgBCR.height / gBB.height)
      const svgCenter = { x: svgBCR.width / 2, y: svgBCR.height / 2 }
      // after scale
      const gCenter = { x: gBB.width * multiple / 2, y: gBB.height * multiple / 2 }
      const center = d3.zoomIdentity.translate(
        -gBB.x * multiple + svgCenter.x - gCenter.x,
        -gBB.y * multiple + svgCenter.y - gCenter.y
      ).scale(multiple)
      zoom.transform(svg.value, center)
    }
    const download = () => {
      const svgdiv = document.querySelector<HTMLDivElement>(`div.${style.svg}`)
      if (svgdiv) {
        html2canvas(svgdiv).then((canvas) => {
          const dataUrl = canvas.toDataURL()
          const window = open()
          if (window) {
            window.document.write(`<img src='${dataUrl}'>`)
            window.document.title = mmdata.data.name
            window.document.close()
          }
        })
      }
    }

    return {
      svgEle,
      gEle,
      style,
      asstSvgEle,
      foreignEle,
      foreignDivEle,
      centerView,
      fitView,
      download
    }
  }
})
</script>

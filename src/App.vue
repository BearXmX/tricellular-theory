<template>
  <section class="three-cell-page" :class="`theme-${themePreset}`">
    <div class="bg-grid"></div>
    <div class="scan-line"></div>

    <header class="topbar">
      <div class="brand">
        <div class="logo-dot"></div>
        <div>
          <div class="eyebrow">GLOBAL ATMOSPHERIC CIRCULATION LAB</div>
          <h1>全球大气环流教学系统 · 三圈环流</h1>
        </div>
      </div>

      <div class="top-actions">
        <div class="mode-switch">
          <button :class="{ active: viewMode === '3d' }" @click="setViewMode('3d')">3D 地球</button>
          <button :class="{ active: viewMode === '2d' }" @click="setViewMode('2d')">2D 展开</button>
        </div>

        <div class="draw-switch">
          <button :class="{ active: drawMode }" @click="toggleDrawMode">
            {{ drawMode ? '退出绘制' : '绘制模式' }}
          </button>
          <input v-model="drawColor" class="color-picker" type="color" title="画笔颜色" />
          <button :class="{ active: eraserMode }" @click="eraserMode = !eraserMode">橡皮擦</button>
          <button class="ghost" @click="undoDrawing">回退</button>
          <button class="ghost" @click="clearDrawings">清空</button>
        </div>

        <div class="texture-switch">
          <button class="ghost export-btn" @click="export2DImage">导出2D图</button>
          <button
            v-for="item in textureOptions"
            :key="item.file"
            :class="{ active: currentTexture === item.file }"
            @click="setEarthTexture(item.file)"
          >
            {{ item.label }}
          </button>
        </div>
      </div>
    </header>

    <main class="content">
      <aside class="control-card left-panel">
        <div class="panel-title"><span></span>控制器</div>

        <div class="control-block">
          <div class="block-title">显示图层</div>
          <label><input v-model="layers.pressure" type="checkbox" /> 气压带环状面与标注</label>
          <label><input v-model="layers.cells" type="checkbox" /> 三圈环流路径与名称</label>
          <label><input v-model="layers.particles" type="checkbox" /> 环流小球粒子</label>
          <label><input v-model="layers.wind" type="checkbox" /> 近地面风带箭头</label>
          <label><input v-model="layers.currents" type="checkbox" /> 洋流箭头与名称</label>
          <label><input v-model="layers.graticuleLine" type="checkbox" /> 经纬线</label>
          <label><input v-model="layers.graticuleText" type="checkbox" /> 经纬度文字</label>
        </div>

        <div class="control-block">
          <div class="block-title">交互探究</div>
          <label><input v-model="featureFlags.climateLink" type="checkbox" /> 气候结果联动</label>
          <label><input v-model="featureFlags.oceanImpact" type="checkbox" /> 洋流 / 风带点击说明</label>
          <label><input v-model="featureFlags.seasonCompare" type="checkbox" /> 开启 1月 / 7月双屏对比</label>
        </div>

        <div class="control-block">
          <div class="block-title">配色预设</div>
          <div class="theme-grid">
            <button :class="{ active: themePreset === 'quantum' }" @click="themePreset = 'quantum'">量子蓝紫</button>
            <button :class="{ active: themePreset === 'aurora' }" @click="themePreset = 'aurora'">极光冷白</button>
            <button :class="{ active: themePreset === 'nebula' }" @click="themePreset = 'nebula'">星云玫紫</button>
            <button :class="{ active: themePreset === 'teal' }" @click="themePreset = 'teal'">蓝绿矩阵</button>
            <button :class="{ active: themePreset === 'gold' }" @click="themePreset = 'gold'">黑金星舰</button>
          </div>
        </div>

        <div class="control-block">
          <div class="block-title">变量参数</div>
          <label class="switch-row">
            <input v-model="params.autoRotate" type="checkbox" />
            自动旋转地球
          </label>
          <RangeRow
            label="旋转速度"
            :value="params.rotateSpeed"
            suffix="x"
            :min="0"
            :max="3"
            :step="0.1"
            @update:value="params.rotateSpeed = $event"
          />
          <RangeRow label="地轴倾角" :value="params.tilt" suffix="°" :min="0" :max="35" :step="1" @update:value="params.tilt = $event" />
          <div class="month-selector">
            <div class="month-title">季节月份</div>
            <div class="month-grid">
              <button v-for="month in 12" :key="month" :class="{ active: params.month === month }" @click="setMonth(month)">{{ month }}月</button>
            </div>
            <div class="month-tip">{{ seasonTip }}</div>
          </div>
          <RangeRow label="环流速度" :value="params.cellSpeed" suffix="x" :min="0.3" :max="3" :step="0.1" @update:value="params.cellSpeed = $event" />
          <RangeRow
            label="环流粒子数量"
            :value="params.particleDensity"
            suffix="%"
            :min="20"
            :max="120"
            :step="5"
            @update:value="params.particleDensity = $event"
          />
        </div>
      </aside>

      <div class="stage-card" @click="handleStageClick">
        <div class="star-field"></div>
        <div class="orbit-glow"></div>
        <div ref="canvasWrapRef" class="canvas-wrap"></div>
        <canvas
          ref="drawCanvasRef"
          class="draw-canvas"
          :class="{ active: drawMode }"
          @pointerdown="handleDrawStart"
          @pointermove="handleDrawMove"
          @pointerup="handleDrawEnd"
          @pointerleave="handleDrawEnd"
        ></canvas>
        <div class="corner corner-lt"></div>
        <div class="corner corner-rb"></div>

        <div class="legend-panel">
          <div class="legend-title">图例</div>
          <div class="legend-item"><span class="dot low"></span>低气压带 / 上升气流</div>
          <div class="legend-item"><span class="dot high"></span>高气压带 / 下沉气流</div>
          <div class="legend-item"><span class="dot wind"></span>风带：贴地平面箭头</div>
          <div class="legend-item"><span class="dot cell"></span>三圈环流：小球粒子</div>
          <div class="legend-item"><span class="dot warm"></span>暖流：红色短箭头</div>
          <div class="legend-item"><span class="dot cold"></span>寒流：蓝色短箭头</div>
        </div>

        <div v-if="featureFlags.seasonCompare" class="season-compare-panel real-split-compare">
          <div class="mini-title">季节对比双屏 · 气压带风带移动</div>
          <div class="split-compare-body">
            <div class="season-screen jan-screen">
              <div class="screen-head">1月 · 北半球冬季</div>
              <div class="mini-earth-map south-shift">
                <div class="mini-lat lat-60n"><span>北纬60°</span></div>
                <div class="mini-lat lat-30n"><span>北纬30°</span></div>
                <div class="mini-lat lat-0"><span>赤道0°</span></div>
                <div class="mini-lat lat-30s"><span>南纬30°</span></div>
                <div class="mini-lat lat-60s"><span>南纬60°</span></div>

                <i class="mini-band polar-n high"></i>
                <i class="mini-band subpolar-n low"></i>
                <i class="mini-band subtropical-n high"></i>
                <i class="mini-band equator low strong"></i>
                <i class="mini-band subtropical-s high"></i>
                <i class="mini-band subpolar-s low"></i>
                <i class="mini-band polar-s high"></i>

                <b class="wind-arrow trade-n a1">↙</b>
                <b class="wind-arrow trade-n a2">↙</b>
                <b class="wind-arrow trade-n a3">↙</b>
                <b class="wind-arrow westerly-n a1">↗</b>
                <b class="wind-arrow westerly-n a2">↗</b>
                <b class="wind-arrow westerly-n a3">↗</b>
                <b class="wind-arrow trade-s a1">↖</b>
                <b class="wind-arrow trade-s a2">↖</b>
                <b class="wind-arrow trade-s a3">↖</b>
                <b class="wind-arrow westerly-s a1">↘</b>
                <b class="wind-arrow westerly-s a2">↘</b>
                <b class="wind-arrow westerly-s a3">↘</b>
              </div>
              <div class="screen-note">太阳直射点偏南，气压带、风带整体南移。</div>
            </div>

            <div class="season-screen jul-screen">
              <div class="screen-head">7月 · 北半球夏季</div>
              <div class="mini-earth-map north-shift">
                <div class="mini-lat lat-60n"><span>北纬60°</span></div>
                <div class="mini-lat lat-30n"><span>北纬30°</span></div>
                <div class="mini-lat lat-0"><span>赤道0°</span></div>
                <div class="mini-lat lat-30s"><span>南纬30°</span></div>
                <div class="mini-lat lat-60s"><span>南纬60°</span></div>

                <i class="mini-band polar-n high"></i>
                <i class="mini-band subpolar-n low"></i>
                <i class="mini-band subtropical-n high"></i>
                <i class="mini-band equator low strong"></i>
                <i class="mini-band subtropical-s high"></i>
                <i class="mini-band subpolar-s low"></i>
                <i class="mini-band polar-s high"></i>

                <b class="wind-arrow trade-n a1">↙</b>
                <b class="wind-arrow trade-n a2">↙</b>
                <b class="wind-arrow trade-n a3">↙</b>
                <b class="wind-arrow westerly-n a1">↗</b>
                <b class="wind-arrow westerly-n a2">↗</b>
                <b class="wind-arrow westerly-n a3">↗</b>
                <b class="wind-arrow trade-s a1">↖</b>
                <b class="wind-arrow trade-s a2">↖</b>
                <b class="wind-arrow trade-s a3">↖</b>
                <b class="wind-arrow westerly-s a1">↘</b>
                <b class="wind-arrow westerly-s a2">↘</b>
                <b class="wind-arrow westerly-s a3">↘</b>
              </div>
              <div class="screen-note">太阳直射点偏北，气压带、风带整体北移。</div>
            </div>
          </div>
        </div>

        <div v-if="activeInfo" class="info-float">
          <button @click.stop="activeInfo = null">×</button>
          <h3>{{ activeInfo.title }}</h3>
          <p>{{ activeInfo.desc }}</p>
          <div v-if="activeInfo.tags?.length" class="info-tags">
            <span v-for="tag in activeInfo.tags" :key="tag">{{ tag }}</span>
          </div>
        </div>
      </div>

      <aside class="knowledge-panel right-panel">
        <div class="panel-title"><span></span>知识速览</div>

        <div class="knowledge-card">
          <div class="block-title">核心概念</div>
          <ul>
            <li><b>热力差异</b>：赤道受热强，空气上升；极地冷却强，空气下沉。</li>
            <li><b>地转偏向力</b>：北半球右偏、南半球左偏，塑造信风、西风、极地东风。</li>
            <li><b>气压带移动</b>：随太阳直射点季节移动，夏季北移，冬季南移。</li>
          </ul>
        </div>

        <div class="knowledge-card">
          <div class="block-title">三圈环流命名来源</div>
          <ul>
            <li><b>为什么叫“三圈”</b>：从赤道到极地，南北半球各形成三个主要环流单元，分别位于低纬、中纬、高纬。</li>
            <li><b>低纬环流</b>：又叫哈德莱环流，位于 0°—30°，赤道上升、30°附近下沉。</li>
            <li><b>中纬环流</b>：又叫费雷尔环流，位于 30°—60°，属于间接热力环流，与盛行西风联系密切。</li>
            <li><b>高纬环流</b>：又叫极地环流，位于 60°—90°，极地下沉、60°附近上升。</li>
            <li><b>命名依据</b>：既按纬度位置命名，也常按提出或研究该环流模型的科学家命名，如 Hadley、Ferrel。</li>
          </ul>
        </div>

        <div class="knowledge-card">
          <div class="block-title">易错点提醒</div>
          <ul>
            <li>三圈环流是理想模型，真实大气还会受海陆分布、地形、洋流影响。</li>
            <li>风向不能只看高低压，还要考虑地转偏向力。</li>
            <li>30°附近不一定全是沙漠，但副热带高压控制区通常更干燥。</li>
            <li>60°附近是西风与极地东风辐合，多锋面、气旋活动。</li>
          </ul>
        </div>

        <div class="knowledge-card">
          <div class="block-title">判读速记</div>
          <div class="tips-grid">
            <div><b>看纬度</b><span>0°、30°、60°、90°是关键线</span></div>
            <div><b>看垂直</b><span>上升多低压，下沉多高压</span></div>
            <div><b>看风向</b><span>信风东来，西风西来，极地东风仍东来</span></div>
            <div><b>看影响</b><span>赤道多雨，副高干燥，西风温湿</span></div>
            <div><b>看季节</b><span>气压带风带随太阳直射点移动</span></div>
            <div><b>看洋流</b><span>暖流增温增湿，寒流降温减湿</span></div>
          </div>
        </div>

        <div class="knowledge-card season-rule-card">
          <div class="block-title">气压带、风带季节移动规律</div>
          <div class="season-rule">
            <div><b>根本原因</b><span>太阳直射点随季节南北移动，热力赤道随之移动。</span></div>
            <div><b>夏季北移</b><span>北半球 6—8 月，太阳直射点偏北，赤道低压、副热带高压、盛行西风等整体北移。</span></div>
            <div><b>冬季南移</b><span>北半球 12—2 月，太阳直射点偏南，气压带和风带整体南移。</span></div>
            <div><b>春秋过渡</b><span>春分、秋分前后接近赤道附近，对称性较强。</span></div>
            <div><b>口诀</b><span>“夏北冬南，随太阳走”；看题目月份，先判断太阳直射点半球。</span></div>
          </div>
        </div>

        <div class="knowledge-card">
          <div class="block-title">三圈环流与气候联系</div>
          <div class="tips-grid">
            <div><b>赤道低压</b><span>终年上升，多对流雨，对应热带雨林气候。</span></div>
            <div><b>副热带高压</b><span>下沉增温，空气干燥，常对应热带沙漠带。</span></div>
            <div><b>盛行西风</b><span>从海洋吹向大陆时，带来温和湿润天气。</span></div>
            <div><b>极地高压</b><span>冷空气下沉，气温低、湿度小、降水少。</span></div>
          </div>
        </div>

        <div class="knowledge-card">
          <div class="block-title">进阶探究设计</div>
          <ul>
            <li><b>气候联动</b>：点击不同纬度带，弹出降水、气温、典型气候说明。</li>
            <li><b>季节对比</b>：打开双屏提示，快速比较 1 月南移与 7 月北移。</li>
            <li><b>洋流推演</b>：点击暖流、寒流标注，查看沿岸增温增湿或降温减湿。</li>
            <li><b>风带说明</b>：点击信风、西风、极地东风名称，查看风向形成机制。</li>
          </ul>
        </div>

        <div class="summary-card">
          <div class="block-title">对比总结表</div>
          <table>
            <thead>
              <tr>
                <th>纬度</th>
                <th>环流</th>
                <th>垂直运动</th>
                <th>风带</th>
                <th>影响</th>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td>0°-30°</td>
                <td>低纬</td>
                <td>赤道升<br />30°沉</td>
                <td>信风</td>
                <td>赤道多雨<br />副高干燥</td>
              </tr>
              <tr>
                <td>30°-60°</td>
                <td>中纬</td>
                <td>30°沉<br />60°升</td>
                <td>西风</td>
                <td>天气多变</td>
              </tr>
              <tr>
                <td>60°-90°</td>
                <td>高纬</td>
                <td>60°升<br />90°沉</td>
                <td>极地东风</td>
                <td>寒冷干燥</td>
              </tr>
            </tbody>
          </table>
        </div>
      </aside>
    </main>
  </section>
</template>

<script setup lang="ts">
import { computed, defineComponent, h, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'

// Vite 资源内联：?inline 会把图片打成 dataURL/base64，避免 file:// 下贴图跨域失败。
// 建议把图片放到 src/assets/image/ 目录，而不是 public/image/。
import earthTexture001 from './assets/image/Material.001_diffuse.jpg?inline'
import earthTexture002 from './assets/image/Material.002_diffuse.jpg?inline'
import earthTexture003 from './assets/image/Material.003_diffuse.jpg?inline'

const textureOptions = [
  { label: '贴图1', file: 'Material.001_diffuse.jpg', url: earthTexture001 },
  { label: '贴图2', file: 'Material.002_diffuse.jpg', url: earthTexture002 },
  { label: '贴图3', file: 'Material.003_diffuse.jpg', url: earthTexture003 },
]
const R = 4

type ViewMode = '3d' | '2d'
type ParticleItem = {
  mesh: THREE.Mesh
  halo: THREE.Mesh
  points: THREE.Vector3[]
  progress: number
  speed: number
}
type DrawGeoPoint = {
  lat: number
  lon: number
}
type DrawStroke = {
  color: string
  width: number
  points: DrawGeoPoint[]
}

const RangeRow = defineComponent({
  name: 'RangeRow',
  props: {
    label: { type: String, required: true },
    value: { type: Number, required: true },
    min: { type: Number, required: true },
    max: { type: Number, required: true },
    step: { type: Number, default: 1 },
    suffix: { type: String, default: '' },
  },
  emits: ['update:value'],
  setup(props, { emit }) {
    return () =>
      h('div', { class: 'range-row' }, [
        h('div', { class: 'range-head' }, [h('span', props.label), h('b', `${props.value}${props.suffix}`)]),
        h('input', {
          type: 'range',
          min: props.min,
          max: props.max,
          step: props.step,
          value: props.value,
          onInput: (e: Event) => emit('update:value', Number((e.target as HTMLInputElement).value)),
        }),
      ])
  },
})

const canvasWrapRef = ref<HTMLDivElement | null>(null)
const drawCanvasRef = ref<HTMLCanvasElement | null>(null)
const viewMode = ref<ViewMode>('3d')
const drawMode = ref(false)
const eraserMode = ref(false)
const drawColor = ref('#00f5ff')
const currentTexture = ref('Material.002_diffuse.jpg')
const themePreset = ref<'quantum' | 'aurora' | 'nebula' | 'teal' | 'gold'>('quantum')
const featureFlags = reactive({
  climateLink: true,
  errorDiagnosis: true,
  seasonCompare: false,
  oceanImpact: true,
})
const activeInfo = ref<{ title: string; desc: string; tags?: string[] } | null>(null)

const isDrawing = ref(false)
const currentStroke = ref<DrawStroke | null>(null)
const drawStrokes = ref<DrawStroke[]>([])
const lastDrawPoint = ref<{ x: number; y: number } | null>(null)

const layers = reactive({
  pressure: true,
  cells: true,
  wind: true,
  currents: true,
  graticuleLine: true,
  graticuleText: true,
  particles: true,
})

const params = reactive({
  autoRotate: true,
  rotateSpeed: 1,
  tilt: 23,
  month: 6,
  seasonShift: 0,
  cellSpeed: 3,
  particleDensity: 70,
})

const monthShiftMap = [-15, -12, -8, -3, 4, 10, 15, 12, 6, 0, -7, -12]

const seasonTip = computed(() => {
  if ([3, 4, 5].includes(params.month)) return '春季：气压带、风带由南向北移动。'
  if ([6, 7, 8].includes(params.month)) return '夏季：太阳直射点偏北，气压带、风带整体北移。'
  if ([9, 10, 11].includes(params.month)) return '秋季：气压带、风带由北向南移动。'
  return '冬季：太阳直射点偏南，气压带、风带整体南移。'
})

let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let controls: OrbitControls
let animationId = 0
let resizeObserver: ResizeObserver | null = null
let clock = new THREE.Clock()
let rebuildTimer: number | undefined

let earthGroup: THREE.Group
let flatMap: THREE.Group
let globe: THREE.Mesh
let graticuleGroup: THREE.Group
let labelGroup: THREE.Group
let pressureGroup: THREE.Group
let cellGroup: THREE.Group
let windGroup: THREE.Group
let currentGroup: THREE.Group
let particleGroup: THREE.Group
let drawGroup: THREE.Group

const circulationParticles: ParticleItem[] = []
const raycaster = new THREE.Raycaster()
const clickRaycaster = new THREE.Raycaster()
const pointerNdc = new THREE.Vector2()
const flatDrawPlane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0)
let lastSurfaceDrawPoint: THREE.Vector3 | null = null

const pressureBelts = [
  { lat: 90, name: '极地高气压带', type: 'high', color: 0x009dff },
  { lat: 60, name: '副极地低气压带', type: 'low', color: 0x9b2cff },
  { lat: 30, name: '副热带高气压带', type: 'high', color: 0xffb300 },
  { lat: 0, name: '赤道低气压带', type: 'low', color: 0xff2f2f },
  { lat: -30, name: '副热带高气压带', type: 'high', color: 0xffb300 },
  { lat: -60, name: '副极地低气压带', type: 'low', color: 0x9b2cff },
  { lat: -90, name: '极地高气压带', type: 'high', color: 0x009dff },
]

const windBelts = [
  { lat: 15, name: '东北信风', startLon: 18, endLon: -18, startLatOffset: 5.5, endLatOffset: -5.5, color: 0xffffff },
  { lat: -15, name: '东南信风', startLon: 18, endLon: -18, startLatOffset: -5.5, endLatOffset: 5.5, color: 0xffffff },
  { lat: 45, name: '盛行西风', startLon: -18, endLon: 20, startLatOffset: -4.5, endLatOffset: 4.5, color: 0xfef4b3 },
  { lat: -45, name: '盛行西风', startLon: -18, endLon: 20, startLatOffset: 4.5, endLatOffset: -4.5, color: 0xfef4b3 },
  { lat: 75, name: '极地东风', startLon: 17, endLon: -17, startLatOffset: 3, endLatOffset: -3, color: 0xb9f1ff },
  { lat: -75, name: '极地东风', startLon: 17, endLon: -17, startLatOffset: -3, endLatOffset: 3, color: 0xb9f1ff },
]

function setMonth(month: number) {
  params.month = month
  params.seasonShift = monthShiftMap[month - 1]!
}

function showClimateInfo(lat: number) {
  if (!featureFlags.climateLink) return
  const abs = Math.abs(lat)
  const hemi = lat >= 0 ? '北半球' : '南半球'

  if (abs < 10) {
    activeInfo.value = {
      title: '赤道低气压带 · 对流雨核心区',
      desc: `${hemi}低纬赤道附近受热强，空气膨胀上升，近地面形成低压。水汽随上升气流冷却凝结，全年多对流雨，典型气候为热带雨林气候。判读时抓住“上升、低压、多雨、赤道附近”四个关键词。`,
      tags: ['0°附近', '上升气流', '低压', '多雨', '热带雨林'],
    }
  } else if (abs < 38) {
    activeInfo.value = {
      title: '副热带高气压带 · 干旱少雨区',
      desc: `${hemi}约 30°附近是高空气流下沉区，空气下沉增温，水汽不易凝结，近地面形成高压。这里常与热带沙漠气候、地中海气候夏季干热等现象相关。注意它会随季节“夏北冬南”移动。`,
      tags: ['30°附近', '下沉气流', '高压', '干燥', '沙漠带'],
    }
  } else if (abs < 65) {
    activeInfo.value = {
      title: '中纬西风带 · 温带天气活跃区',
      desc: `${hemi}约 30°—60°之间受盛行西风影响明显。若西风从海洋吹向大陆，常带来温和湿润的气流；中纬地区锋面气旋活跃，天气变化频繁。欧洲西部温带海洋性气候与西风、暖流共同作用密切。`,
      tags: ['30°—60°', '盛行西风', '锋面气旋', '温和湿润', '天气多变'],
    }
  } else {
    activeInfo.value = {
      title: '极地高压与极地东风 · 寒冷干燥区',
      desc: `${hemi}高纬地区热量不足，冷空气收缩下沉，形成极地高压。近地面空气由极地向较低纬流动，并受地转偏向力影响形成极地东风。典型特征是寒冷、干燥、降水少。`,
      tags: ['高纬', '极地高压', '极地东风', '寒冷', '干燥'],
    }
  }
}

function showOceanImpact(type: 'warm' | 'cold', name: string) {
  if (!featureFlags.oceanImpact) return

  const currentDetails: Record<string, string> = {
    北赤道暖流:
      '北赤道暖流位于低纬海区，整体把暖水向西输送，是低纬海洋热量再分配的重要环节。它会增强沿线海域蒸发和水汽输送，并参与太平洋、大西洋低纬环流系统。',
    南赤道暖流:
      '南赤道暖流位于南半球低纬海区，主要把暖水向西输送。它与东澳大利亚暖流、巴西暖流、厄加勒斯暖流等联系紧密，对南半球低纬海区热量输送有重要作用。',
    赤道逆流: '赤道逆流位于赤道附近，方向通常与南、北赤道暖流相反，用来补偿低纬海水向西堆积造成的海面高度差。它体现了赤道海区洋流系统的闭合与平衡。',
    日本暖流: '日本暖流又称黑潮，是北太平洋西岸强大的暖流。它使日本及附近海域增温增湿，带来丰富水汽，并与千岛寒流交汇，形成著名渔场。',
    北太平洋暖流: '北太平洋暖流是日本暖流向东延伸后的暖流分支，把热量输送到北太平洋中高纬海区，对北太平洋海气交换和中纬西风带水汽输送有影响。',
    阿拉斯加暖流: '阿拉斯加暖流流向北太平洋东北部，使阿拉斯加南部沿岸相对增温增湿，减弱高纬沿岸的寒冷程度，并影响当地海雾、降水和海洋生态。',
    东澳大利亚暖流: '东澳大利亚暖流沿澳大利亚东岸向南流动，使澳大利亚东岸海域增温增湿，有利于沿岸降水和海洋生物活动。',
    加利福尼亚寒流: '加利福尼亚寒流沿北美洲西岸向低纬流动，使美国西海岸部分地区降温减湿，容易形成海雾，也与沿岸较干燥环境有关。',
    秘鲁寒流:
      '秘鲁寒流又称洪堡寒流，沿南美洲西岸向北流动。它使南半球南美西岸降温减湿，稳定大气层结，抑制降水，是秘鲁、智利沿岸荒漠形成的重要因素之一；同时上升流带来营养盐，形成世界著名渔场。',
    千岛寒流:
      '千岛寒流来自高纬冷水区，沿日本以北海域南下。它与日本暖流交汇，形成冷暖水交汇区，营养盐丰富、浮游生物多，是北海道附近渔场形成的重要原因。',
    墨西哥湾暖流: '墨西哥湾暖流是北大西洋强大的暖流，从墨西哥湾附近向东北流动，把大量热量输送到北大西洋，对欧洲西部温和湿润气候具有重要影响。',
    北大西洋暖流: '北大西洋暖流是墨西哥湾暖流向东北延伸的重要分支，使欧洲西部沿岸明显增温增湿，是欧洲西部温带海洋性气候的重要成因之一。',
    巴西暖流: '巴西暖流沿南美洲东岸向南流动，对巴西东南部沿岸有增温增湿作用。它与南大洋西风漂流系统相联系，参与南大西洋环流。',
    加那利寒流: '加那利寒流沿非洲西北岸向南流动，使沿岸降温减湿，抑制对流降水，是撒哈拉沙漠西岸干旱环境的重要影响因素之一。',
    本格拉寒流: '本格拉寒流沿非洲西南岸向北流动，使纳米比亚、安哥拉沿岸降温减湿，稳定大气，抑制降水，是纳米布沙漠沿岸干旱的重要原因之一。',
    拉布拉多寒流: '拉布拉多寒流由北冰洋和格陵兰附近南下，使加拿大东岸及纽芬兰附近海域降温。它与墨西哥湾暖流交汇，易形成海雾，并有利于渔场形成。',
    东格陵兰寒流: '东格陵兰寒流携带高纬冷水和海冰沿格陵兰东岸南下，使沿岸海域寒冷，并影响北大西洋高纬海区的海冰分布和气候环境。',
    季风洋流:
      '季风洋流主要出现在北印度洋，受冬夏季风影响显著，流向具有季节变化特点。夏季受西南季风影响，表层海水运动方向与冬季不同，是海陆热力差异和季风环流影响海洋的典型表现。',
    厄加勒斯暖流: '厄加勒斯暖流沿非洲东南岸向南流动，是印度洋西部强暖流。它使非洲东南沿岸增温增湿，并参与印度洋与南大西洋之间的水热交换。',
    索马里寒流:
      '索马里寒流位于非洲东北部索马里沿岸，夏季西南季风加强时，上升流显著，冷水上翻，使沿岸降温并带来丰富营养盐，对渔业和区域气候都有影响。',
    西澳大利亚寒流: '西澳大利亚寒流沿澳大利亚西岸向低纬流动，使澳大利亚西岸降温减湿，抑制降水，是西澳大利亚沿岸较干燥的重要因素之一。',
    西风漂流: '西风漂流又称南极绕极流，受南半球盛行西风驱动，环绕南极洲自西向东流动。它把南大洋冷水联系起来，对全球海洋环流和热量交换影响很大。',
  }

  const general =
    type === 'warm'
      ? '暖流统一规律：通常由低纬或较暖海区流向较冷海区，对沿岸起增温增湿作用，可能增加空气水汽和降水条件。'
      : '寒流统一规律：通常由高纬或较冷海区流向较暖海区，对沿岸起降温减湿作用，常使大气更稳定，抑制降水，部分寒流还伴随上升流和渔场。'

  activeInfo.value = {
    title: `${name} · ${type === 'warm' ? '暖流' : '寒流'}影响`,
    desc: `${currentDetails[name] ?? '这条洋流会通过输送冷暖海水，改变沿岸地区的热量和水汽条件，从而影响沿岸气候、海雾、降水和渔业资源。'}

${general}`,
    tags: type === 'warm' ? ['暖流', '增温', '增湿', '水汽输送', '沿岸气候'] : ['寒流', '降温', '减湿', '大气稳定', '沿岸气候'],
  }
}

function showWindInfo(name: string, lat: number) {
  if (!featureFlags.oceanImpact) return
  if (name.includes('信风')) {
    activeInfo.value = {
      title: `${name} · 低纬近地面风带`,
      desc: `${name}位于赤道低压带与副热带高压带之间。空气从副热带高压流向赤道低压，受地转偏向力影响形成偏东风：北半球为东北信风，南半球为东南信风。它是热带洋面水汽输送和低纬环流判读的关键。`,
      tags: ['0°—30°', '偏东风', '吹向赤道低压', '受地转偏向力影响'],
    }
  } else if (name.includes('西风')) {
    activeInfo.value = {
      title: `${name} · 中纬风带`,
      desc: '盛行西风位于副热带高压带与副极地低压带之间，空气由约 30°流向约 60°，受地转偏向力影响成为偏西风。西风从海洋吹向大陆时，常带来温和湿润气流，是温带海洋性气候的重要成因之一。',
      tags: ['30°—60°', '偏西风', '温带', '海洋性气候'],
    }
  } else {
    activeInfo.value = {
      title: `${name} · 高纬近地面风带`,
      desc: '极地东风位于极地高压带与副极地低压带之间，冷空气由极地向较低纬流动，并受地转偏向力影响形成偏东风。它寒冷干燥，常与极地高压、极锋等知识点联系。',
      tags: ['60°—90°', '偏东风', '寒冷干燥', '极地高压'],
    }
  }
}

function setViewMode(mode: ViewMode) {
  viewMode.value = mode

  // 切换 2D / 3D 时自动退出绘制状态。
  // 否则透明绘制 canvas 仍在接管鼠标事件，OrbitControls 收不到事件，地球就无法拖动旋转。
  drawMode.value = false
  eraserMode.value = false
  isDrawing.value = false
  currentStroke.value = null
  lastDrawPoint.value = null
  lastSurfaceDrawPoint = null

  if (controls) controls.enabled = true

  applyViewMode()
  nextTick(renderSharedDrawings)
}

function setEarthTexture(file: string) {
  currentTexture.value = file
  const option = textureOptions.find(item => item.file === file)
  if (!option) return

  // option.url 是 import ?inline 得到的 dataURL/base64，file:// 下也能被 WebGL 使用。
  new THREE.TextureLoader().load(option.url, texture => applyEarthTexture(texture))
}

function applyEarthTexture(texture: THREE.Texture) {
  const material = globe?.material as THREE.MeshBasicMaterial | undefined
  if (!material) return
  texture.colorSpace = THREE.SRGBColorSpace
  texture.anisotropy = renderer.capabilities.getMaxAnisotropy()
  material.map = texture
  material.needsUpdate = true
}

function export2DImage() {
  const oldMode = viewMode.value
  const oldDrawMode = drawMode.value

  // 导出的是 2D 模式图。若当前在 3D，先临时切到 2D 渲染一帧。
  viewMode.value = '2d'
  drawMode.value = false
  if (controls) controls.enabled = true
  applyViewMode()
  renderSharedDrawings()
  renderer.render(scene, camera)

  requestAnimationFrame(() => {
    const webglCanvas = renderer.domElement
    const drawCanvas = drawCanvasRef.value
    const output = document.createElement('canvas')
    output.width = webglCanvas.width
    output.height = webglCanvas.height
    const ctx = output.getContext('2d')
    if (!ctx) return

    drawExportBackground(ctx, output.width, output.height)
    ctx.drawImage(webglCanvas, 0, 0, output.width, output.height)
    if (drawCanvas) ctx.drawImage(drawCanvas, 0, 0, output.width, output.height)

    const link = document.createElement('a')
    link.download = `三圈环流2D模式图-${new Date().toISOString().slice(0, 10)}.png`
    link.href = output.toDataURL('image/png')
    link.click()

    // 恢复原模式。
    viewMode.value = oldMode
    drawMode.value = oldDrawMode
    applyViewMode()
    nextTick(renderSharedDrawings)
  })
}

function drawExportBackground(ctx: CanvasRenderingContext2D, width: number, height: number) {
  // 导出图片不是透明底：补一层和页面一致的科技背景。
  const bg = ctx.createLinearGradient(0, 0, width, height)
  bg.addColorStop(0, '#050612')
  bg.addColorStop(0.52, '#11102a')
  bg.addColorStop(1, '#03040a')
  ctx.fillStyle = bg
  ctx.fillRect(0, 0, width, height)

  const glowA = ctx.createRadialGradient(width * 0.22, height * 0.18, 0, width * 0.22, height * 0.18, width * 0.42)
  glowA.addColorStop(0, 'rgba(168, 85, 247, 0.26)')
  glowA.addColorStop(1, 'rgba(168, 85, 247, 0)')
  ctx.fillStyle = glowA
  ctx.fillRect(0, 0, width, height)

  const glowB = ctx.createRadialGradient(width * 0.82, height * 0.72, 0, width * 0.82, height * 0.72, width * 0.38)
  glowB.addColorStop(0, 'rgba(129, 140, 248, 0.22)')
  glowB.addColorStop(1, 'rgba(129, 140, 248, 0)')
  ctx.fillStyle = glowB
  ctx.fillRect(0, 0, width, height)

  ctx.save()
  ctx.globalAlpha = 0.42
  ctx.strokeStyle = 'rgba(226, 232, 240, 0.08)'
  ctx.lineWidth = 1
  const step = Math.max(28, Math.floor(width / 28))
  for (let x = 0; x <= width; x += step) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, height)
    ctx.stroke()
  }
  for (let y = 0; y <= height; y += step) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(width, y)
    ctx.stroke()
  }
  ctx.restore()
}

function toggleDrawMode() {
  drawMode.value = !drawMode.value
  if (controls) controls.enabled = !drawMode.value
}

function clearDrawings() {
  const canvas = drawCanvasRef.value
  const ctx = canvas?.getContext('2d')
  if (canvas && ctx) ctx.clearRect(0, 0, canvas.width, canvas.height)
  drawStrokes.value = []
  currentStroke.value = null
  if (drawGroup) clearGroup(drawGroup)
}

function undoDrawing() {
  drawStrokes.value = drawStrokes.value.slice(0, -1)
  renderSharedDrawings()
}

function handleDrawStart(e: PointerEvent) {
  if (!drawMode.value || !drawCanvasRef.value) return
  isDrawing.value = true
  drawCanvasRef.value.setPointerCapture?.(e.pointerId)

  const geoPoint = getPointerGeoPoint(e)
  if (!geoPoint) return

  if (eraserMode.value) {
    eraseDrawingAt(geoPoint)
    return
  }

  currentStroke.value = {
    color: drawColor.value,
    width: 0.018,
    points: [geoPoint],
  }

  if (viewMode.value === '3d') {
    lastSurfaceDrawPoint = geoToVec3(geoPoint, R + 0.075)
  } else {
    lastDrawPoint.value = projectGeoToCanvas(geoPoint)
  }
}

function handleDrawMove(e: PointerEvent) {
  if (!drawMode.value || !isDrawing.value) return

  const geoPoint = getPointerGeoPoint(e)
  if (!geoPoint) return

  if (eraserMode.value) {
    eraseDrawingAt(geoPoint)
    return
  }

  if (!currentStroke.value) return
  const lastGeo = currentStroke.value.points[currentStroke.value.points.length - 1]
  if (geoDistance(lastGeo!, geoPoint) < 0.35) return

  currentStroke.value.points.push(geoPoint)

  if (viewMode.value === '3d') {
    const current = geoToVec3(geoPoint, R + 0.075)
    if (!lastSurfaceDrawPoint || !drawGroup) return
    drawGroup.add(makeTubeLine([lastSurfaceDrawPoint, current], hexColorToNumber(drawColor.value), 0.018, 0.98))
    lastSurfaceDrawPoint = current
    return
  }

  const canvas = drawCanvasRef.value
  if (!canvas || !lastDrawPoint.value) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const p = projectGeoToCanvas(geoPoint)
  const last = lastDrawPoint.value
  ctx.save()
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'
  ctx.strokeStyle = drawColor.value
  ctx.shadowColor = drawColor.value
  ctx.shadowBlur = 10
  ctx.lineWidth = 3
  ctx.beginPath()
  ctx.moveTo(last.x, last.y)
  ctx.lineTo(p.x, p.y)
  ctx.stroke()
  ctx.restore()
  lastDrawPoint.value = p
}

function handleDrawEnd(e: PointerEvent) {
  if (drawCanvasRef.value) drawCanvasRef.value.releasePointerCapture?.(e.pointerId)
  isDrawing.value = false
  lastDrawPoint.value = null
  lastSurfaceDrawPoint = null

  if (currentStroke.value && currentStroke.value.points.length > 1 && !eraserMode.value) {
    const next = [...drawStrokes.value, currentStroke.value]
    drawStrokes.value = next.slice(-20)
    const avgLat = currentStroke.value.points.reduce((sum, p) => sum + p.lat, 0) / currentStroke.value.points.length
    showClimateInfo(avgLat)
  }
  currentStroke.value = null
  renderSharedDrawings()
}

function getDrawPoint(e: PointerEvent) {
  const canvas = drawCanvasRef.value!
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  return {
    x: (e.clientX - rect.left) * scaleX,
    y: (e.clientY - rect.top) * scaleY,
  }
}

function handleStageClick(e: MouseEvent) {
  if (drawMode.value) return
  const geo = getPointerGeoPoint(e as PointerEvent)
  if (geo) showClimateInfo(geo.lat)

  if (viewMode.value === '3d' && drawCanvasRef.value) {
    const rect = drawCanvasRef.value.getBoundingClientRect()
    pointerNdc.x = ((e.clientX - rect.left) / rect.width) * 2 - 1
    pointerNdc.y = -((e.clientY - rect.top) / rect.height) * 2 + 1
    clickRaycaster.setFromCamera(pointerNdc, camera)
    const hits = clickRaycaster.intersectObjects([...currentGroup.children, ...windGroup.children], true)
    const currentHit = hits.find(h => h.object.userData.currentName)
    if (currentHit) {
      showOceanImpact(currentHit.object.userData.currentType, currentHit.object.userData.currentName)
      return
    }
    const windHit = hits.find(h => h.object.userData.windName)
    if (windHit) showWindInfo(windHit.object.userData.windName, windHit.object.userData.windLat)
  }
}

function getPointerGeoPoint(e: PointerEvent): DrawGeoPoint | null {
  if (viewMode.value === '3d') {
    const p = getSurfaceDrawPoint(e)
    return p ? vec3ToGeo(p) : null
  }
  return getFlatGeoPoint(e)
}

function getSurfaceDrawPoint(e: PointerEvent) {
  if (!drawCanvasRef.value || !globe || !earthGroup) return null
  const rect = drawCanvasRef.value.getBoundingClientRect()
  pointerNdc.x = ((e.clientX - rect.left) / rect.width) * 2 - 1
  pointerNdc.y = -((e.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(pointerNdc, camera)
  const hits = raycaster.intersectObject(globe, false)
  if (!hits.length) return null

  const local = earthGroup.worldToLocal(hits[0]!.point.clone())
  return local.normalize().multiplyScalar(R + 0.075)
}

function getFlatGeoPoint(e: PointerEvent): DrawGeoPoint | null {
  if (!drawCanvasRef.value) return null
  const rect = drawCanvasRef.value.getBoundingClientRect()
  pointerNdc.x = ((e.clientX - rect.left) / rect.width) * 2 - 1
  pointerNdc.y = -((e.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(pointerNdc, camera)
  const hit = new THREE.Vector3()
  const ok = raycaster.ray.intersectPlane(flatDrawPlane, hit)
  if (!ok) return null

  const lon = THREE.MathUtils.clamp((hit.x / 10.8) * 360, -180, 180)
  const lat = THREE.MathUtils.clamp((hit.y / 5.8) * 180, -90, 90)
  return { lat, lon }
}

function vec3ToGeo(v: THREE.Vector3): DrawGeoPoint {
  const n = v.clone().normalize()
  const lat = THREE.MathUtils.radToDeg(Math.asin(n.y))
  const lon = normalizeLon(THREE.MathUtils.radToDeg(Math.atan2(n.z, -n.x)) - 180)
  return { lat, lon }
}

function geoToVec3(point: DrawGeoPoint, radius = R + 0.075) {
  return latLngToVec3(point.lat, point.lon, radius)
}

function geoDistance(a: DrawGeoPoint, b: DrawGeoPoint) {
  const dLat = a.lat - b.lat
  const dLon = normalizeLon(a.lon - b.lon)
  return Math.sqrt(dLat * dLat + dLon * dLon)
}

function hexColorToNumber(color: string) {
  return Number(color.replace('#', '0x'))
}

function eraseDrawingAt(point: DrawGeoPoint) {
  const threshold = viewMode.value === '3d' ? 4 : 7
  drawStrokes.value = drawStrokes.value.filter(stroke => !stroke.points.some(p => geoDistance(p, point) < threshold))
  renderSharedDrawings()
}

function renderSharedDrawings() {
  if (drawGroup) clearGroup(drawGroup)

  const canvas = drawCanvasRef.value
  const ctx = canvas?.getContext('2d')
  if (canvas && ctx) ctx.clearRect(0, 0, canvas.width, canvas.height)

  if (viewMode.value === '3d') {
    drawStrokes.value.forEach(stroke => {
      for (let i = 1; i < stroke.points.length; i++) {
        const a = geoToVec3(stroke.points[i - 1]!, R + 0.075)
        const b = geoToVec3(stroke.points[i]!, R + 0.075)
        drawGroup.add(makeTubeLine([a, b], hexColorToNumber(stroke.color), stroke.width, 0.98))
      }
    })
    return
  }

  if (!canvas || !ctx) return
  drawStrokes.value.forEach(stroke => {
    if (stroke.points.length < 2) return
    ctx.save()
    ctx.lineCap = 'round'
    ctx.lineJoin = 'round'
    ctx.strokeStyle = stroke.color
    ctx.shadowColor = stroke.color
    ctx.shadowBlur = 10
    ctx.lineWidth = 3
    ctx.beginPath()
    const first = projectGeoToCanvas(stroke.points[0]!)
    ctx.moveTo(first.x, first.y)
    for (let i = 1; i < stroke.points.length; i++) {
      const p = projectGeoToCanvas(stroke.points[i]!)
      ctx.lineTo(p.x, p.y)
    }
    ctx.stroke()
    ctx.restore()
  })
}

function projectGeoToCanvas(point: DrawGeoPoint) {
  const canvas = drawCanvasRef.value!
  const world = new THREE.Vector3((point.lon / 360) * 10.8, (point.lat / 180) * 5.8, 0.9)
  const ndc = world.clone().project(camera)
  return {
    x: ((ndc.x + 1) / 2) * canvas.width,
    y: ((1 - ndc.y) / 2) * canvas.height,
  }
}

function initThree() {
  if (!canvasWrapRef.value) return

  scene = new THREE.Scene()
  scene.fog = new THREE.Fog(0x07101f, 13, 30)

  camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100)
  camera.position.set(0, 2.4, 12)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, preserveDrawingBuffer: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setClearColor(0x07101f, 0)
  canvasWrapRef.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.08
  controls.minDistance = 7
  controls.maxDistance = 18
  controls.enablePan = false

  scene.add(new THREE.AmbientLight(0xffffff, 1.45))

  const sun = new THREE.DirectionalLight(0xfff2d0, 1.1)
  sun.position.set(5, 6, 8)
  scene.add(sun)

  const rim = new THREE.DirectionalLight(0x6fdcff, 1.0)
  rim.position.set(-6, 1.6, -4)
  scene.add(rim)

  earthGroup = new THREE.Group()
  flatMap = new THREE.Group()
  scene.add(earthGroup, flatMap)

  graticuleGroup = new THREE.Group()
  labelGroup = new THREE.Group()
  pressureGroup = new THREE.Group()
  cellGroup = new THREE.Group()
  windGroup = new THREE.Group()
  currentGroup = new THREE.Group()
  particleGroup = new THREE.Group()
  drawGroup = new THREE.Group()

  earthGroup.add(graticuleGroup, labelGroup, pressureGroup, cellGroup, windGroup, currentGroup, particleGroup, drawGroup)

  createEarth()
  rebuildAll()
  applyViewMode()
  bindResize()
  animate()
}

function createEarth() {
  const loader = new THREE.TextureLoader()
  loader.setCrossOrigin('anonymous')

  const fallbackTexture = createFallbackEarthTexture()
  const material = new THREE.MeshBasicMaterial({ map: fallbackTexture, color: 0xffffff })

  const currentTextureOption = textureOptions.find(item => item.file === currentTexture.value) ?? textureOptions[1]
  loader.load(
    currentTextureOption!.url,
    texture => {
      texture.colorSpace = THREE.SRGBColorSpace
      texture.anisotropy = renderer.capabilities.getMaxAnisotropy()
      material.map = texture
      material.needsUpdate = true
    },
    undefined,
    () => {
      material.map = fallbackTexture
      material.needsUpdate = true
    },
  )

  globe = new THREE.Mesh(new THREE.SphereGeometry(R, 160, 96), material)
  earthGroup.add(globe)

  const atmosphere = new THREE.Mesh(
    new THREE.SphereGeometry(R * 1.04, 160, 96),
    new THREE.MeshBasicMaterial({ color: 0x39e7ff, transparent: true, opacity: 0.08, side: THREE.BackSide }),
  )
  earthGroup.add(atmosphere)
}

function createFallbackEarthTexture() {
  const canvas = document.createElement('canvas')
  canvas.width = 1024
  canvas.height = 512
  const ctx = canvas.getContext('2d')!
  const ocean = ctx.createLinearGradient(0, 0, 0, canvas.height)
  ocean.addColorStop(0, '#143b63')
  ocean.addColorStop(0.48, '#1d6f9e')
  ocean.addColorStop(1, '#0b2b4f')
  ctx.fillStyle = ocean
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  return texture
}

function rebuildAll() {
  clearGroup(graticuleGroup)
  clearGroup(labelGroup)
  clearGroup(pressureGroup)
  clearGroup(cellGroup)
  clearGroup(windGroup)
  clearGroup(currentGroup)
  clearGroup(particleGroup)
  clearGroup(flatMap)

  circulationParticles.length = 0

  createGraticule()
  createPressureBelts()
  createCirculationCells()
  createWindBelts()
  createOceanCurrents()
  createFlatMapReference()
  applyLayerVisibility()
  renderSharedDrawings()
}

function clearGroup(group: THREE.Group) {
  group.traverse(obj => {
    const mesh = obj as THREE.Mesh
    if (mesh.geometry) mesh.geometry.dispose()
    const mat = mesh.material as THREE.Material | THREE.Material[] | undefined
    if (Array.isArray(mat)) mat.forEach(m => m.dispose())
    else mat?.dispose()
  })
  group.clear()
}

function createGraticule() {
  // 纬线：15°一条，赤道和 30°、60°关键纬线颜色更明显。
  for (let lat = -90; lat <= 90; lat += 15) {
    const color = lat === 0 ? 0xff7a7a : lat % 30 === 0 ? 0x56dfff : 0x205b84
    graticuleGroup.add(makeLatLine(lat, color, lat === 0 ? 0.95 : lat % 30 === 0 ? 0.62 : 0.36))
  }

  // 经线：15°一条，本初子午线和 30°倍数经线颜色更明显。
  for (let lon = -180; lon < 180; lon += 15) {
    const color = lon === 0 ? 0xffde77 : lon % 30 === 0 ? 0x7dffbe : 0x285d7b
    graticuleGroup.add(makeLonLine(lon, color, lon === 0 ? 0.82 : lon % 30 === 0 ? 0.5 : 0.28))
  }

  // 纬度文字：比原来更密，放在同一条可读性较好的经线附近。
  for (let lat = -75; lat <= 75; lat += 15) {
    const text = lat === 0 ? '赤道 0°' : `${Math.abs(lat)}°${lat > 0 ? 'N' : 'S'}`
    const color = lat === 0 ? '#ffb6a8' : lat % 30 === 0 ? '#7eeeff' : '#a9ddff'
    const size = lat === 0 ? 0.28 : lat % 30 === 0 ? 0.24 : 0.2
    labelGroup.add(createSpriteText(text, color, latLngToVec3(lat, -108, R + 0.22), size))
    labelGroup.add(createSpriteText(text, color, latLngToVec3(lat, 108, R + 0.22), size * 0.92))
  }

  labelGroup.add(createSpriteText('90°N', '#7eeeff', latLngToVec3(84, -108, R + 0.22), 0.22))
  labelGroup.add(createSpriteText('90°S', '#7eeeff', latLngToVec3(-84, -108, R + 0.22), 0.22))

  // 经度文字：统一放在“经线与赤道的交点”附近。
  // 为避免太拥挤，文字按 30°间隔显示，0°和 180°突出。
  for (let lon = -180; lon <= 180; lon += 30) {
    const text = lon === 0 ? '0°' : Math.abs(lon) === 180 ? '180°' : `${Math.abs(lon)}°${lon > 0 ? 'E' : 'W'}`
    const color = lon === 0 ? '#ffe59a' : Math.abs(lon) === 180 ? '#d8c7ff' : '#98ffce'
    const size = lon === 0 || Math.abs(lon) === 180 ? 0.24 : 0.2
    labelGroup.add(createSpriteText(text, color, latLngToVec3(0, lon, R + 0.34), size))
  }
}

function makeLatLine(lat: number, color: number, opacity: number) {
  const pts: THREE.Vector3[] = []
  for (let lon = -180; lon <= 180; lon += 2) pts.push(latLngToVec3(lat, lon, R + 0.023))
  return makeTubeLine(pts, color, 0.008, opacity)
}

function makeLonLine(lon: number, color: number, opacity: number) {
  const pts: THREE.Vector3[] = []
  for (let lat = -90; lat <= 90; lat += 2) pts.push(latLngToVec3(lat, lon, R + 0.025))
  return makeTubeLine(pts, color, 0.006, opacity)
}

function createPressureBelts() {
  pressureBelts.forEach(belt => {
    const shiftedLat = clampLat(belt.lat + seasonalOffset(belt.lat))
    pressureGroup.add(makeBeltBand(shiftedLat, belt.color, belt.type === 'low' ? 0.74 : 0.66))

    const labelLat = Math.abs(belt.lat) === 90 ? (belt.lat > 0 ? 82 : -82) : shiftedLat
    pressureGroup.add(createSpriteText(belt.name, belt.type === 'low' ? '#ffd2f5' : '#fff3a2', latLngToVec3(labelLat, 82, R + 0.46), 0.32))
    pressureGroup.add(createSpriteText(belt.name, belt.type === 'low' ? '#ffd2f5' : '#fff3a2', latLngToVec3(labelLat, -82, R + 0.46), 0.3))
  })
}

function makeBeltBand(lat: number, color: number, opacity: number) {
  const halfWidth = Math.abs(lat) > 84 ? 6 : 4.2
  const latMin = clampLat(lat - halfWidth)
  const latMax = clampLat(lat + halfWidth)
  const geometry = new THREE.BufferGeometry()
  const vertices: number[] = []
  const indices: number[] = []
  const latSegments = 6
  const lonSegments = 160

  for (let iy = 0; iy <= latSegments; iy++) {
    const curLat = latMin + (latMax - latMin) * (iy / latSegments)
    for (let ix = 0; ix <= lonSegments; ix++) {
      const lon = -180 + (360 * ix) / lonSegments
      const p = latLngToVec3(curLat, lon, R + 0.055)
      vertices.push(p.x, p.y, p.z)
    }
  }

  for (let iy = 0; iy < latSegments; iy++) {
    for (let ix = 0; ix < lonSegments; ix++) {
      const a = iy * (lonSegments + 1) + ix
      const b = a + 1
      const c = a + lonSegments + 1
      const d = c + 1
      indices.push(a, c, b, b, c, d)
    }
  }

  geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))
  geometry.setIndex(indices)
  geometry.computeVertexNormals()

  return new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ color, transparent: true, opacity, side: THREE.DoubleSide, depthWrite: false }))
}

function createCirculationCells() {
  const defs = [
    { a: 0, b: 30, name: '低纬环流\n哈德莱环流', color: 0xff55b8, showLabel: true },
    { a: 30, b: 60, name: '中纬环流\n费雷尔环流', color: 0xffd34f, showLabel: true },
    { a: 60, b: 88, name: '高纬环流\n极地环流', color: 0x52d9ff, showLabel: true },
    { a: 0, b: -30, name: '低纬环流\n哈德莱环流', color: 0xff55b8, showLabel: true },
    { a: -30, b: -60, name: '中纬环流\n费雷尔环流', color: 0xffd34f, showLabel: true },
    { a: -60, b: -88, name: '高纬环流\n极地环流', color: 0x52d9ff, showLabel: true },
  ]

  defs.forEach((d, cellIndex) => {
    const lon = -112
    const cellPath = createClosedCellPath(d.a, d.b, lon)
    cellGroup.add(makeTubeLine(cellPath, d.color, 0.018, 0.75))
    if (d.showLabel) {
      cellGroup.add(createSpriteText(d.name, '#ffffff', latLngToVec3((d.a + d.b) / 2, lon - 15, R + 1.16), 0.48))
    }

    const particleCount = Math.max(10, Math.floor((params.particleDensity / 100) * 18))
    for (let i = 0; i < particleCount; i++) {
      const color = new THREE.Color(d.color).lerp(new THREE.Color(0xffffff), 0.25)
      const mesh = new THREE.Mesh(
        new THREE.SphereGeometry(0.045 + Math.random() * 0.014, 12, 12),
        new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.9 }),
      )
      const halo = new THREE.Mesh(
        new THREE.SphereGeometry(0.085 + Math.random() * 0.02, 12, 12),
        new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.13, depthWrite: false }),
      )
      particleGroup.add(mesh, halo)
      circulationParticles.push({
        mesh,
        halo,
        points: cellPath,
        progress: (i / particleCount + cellIndex * 0.07) % 1,
        speed: 0.055 + Math.random() * 0.025,
      })
    }
  })
  ;[0, 60, -60].forEach(lat => cellGroup.add(createVerticalFlow(lat, -108, true)))
  ;[30, -30, 88, -88].forEach(lat => cellGroup.add(createVerticalFlow(lat, -108, false)))
}

function createClosedCellPath(latA: number, latB: number, lon: number) {
  // 三圈环流是一条连续闭合环线：
  // - 一侧为高空支，抬离地表，表现高空回流；
  // - 另一侧为近地面支，贴近地表，表现近地面风带；
  // - 两端自然转折，避免之前那种横线对折感。
  const points: THREE.Vector3[] = []
  const centerLat = (latA + latB) / 2
  const latAmp = Math.abs(latB - latA) / 2
  const lonAmp = Math.abs(latB - latA) > 40 ? 7.2 : 8.6
  const steps = 192

  for (let i = 0; i <= steps; i++) {
    const t = (i / steps) * Math.PI * 2
    const lat = centerLat + latAmp * Math.sin(t)
    const curLon = lon + lonAmp * Math.cos(t)

    // cos(t) > 0：高空一侧，远离地表。
    // cos(t) < 0：近地面一侧，贴着地表。
    // smoothstep 让高空支和近地面支过渡更柔和。
    const airSide = (Math.cos(t) + 1) / 2
    const smoothAirSide = airSide * airSide * (3 - 2 * airSide)
    const surfaceHeight = 0.1
    const upperHeight = 1.28
    const height = surfaceHeight + (upperHeight - surfaceHeight) * smoothAirSide

    points.push(latLngToVec3(lat, curLon, R + height))
  }

  return points
}

function createVerticalFlow(lat: number, lon: number, up: boolean) {
  const group = new THREE.Group()
  const p1 = latLngToVec3(lat, lon, R + 0.28)
  const p2 = latLngToVec3(lat, lon, R + 1.62)
  group.add(makeTubeLine(up ? [p1, p2] : [p2, p1], up ? 0xff46b7 : 0x52d9ff, 0.018, 0.88))
  group.add(createSpriteText(up ? '上升气流' : '下沉气流', up ? '#ff9bd8' : '#8beeff', latLngToVec3(lat, lon - 9, R + 1.14), 0.28))
  return group
}

function createWindBelts() {
  windBelts.forEach(belt => {
    const lat = clampLat(belt.lat + seasonalOffset(belt.lat))
    for (let baseLon = -160; baseLon <= 160; baseLon += 28) {
      const path = makeSurfaceWindPath(lat, baseLon + belt.startLon, baseLon + belt.endLon, belt.startLatOffset, belt.endLatOffset)
      windGroup.add(makeSurfaceArrow(path, belt.color, 0.82, 0.034))
    }
    const labelA = createSpriteText(belt.name, '#ffffff', latLngToVec3(lat, 118, R + 0.18), 0.28)
    labelA.userData.windName = belt.name
    labelA.userData.windLat = lat
    const labelB = createSpriteText(belt.name, '#ffffff', latLngToVec3(lat, -118, R + 0.18), 0.26)
    labelB.userData.windName = belt.name
    labelB.userData.windLat = lat
    windGroup.add(labelA)
    windGroup.add(labelB)
  })
}

function makeSurfaceWindPath(lat: number, lonA: number, lonB: number, latOffA: number, latOffB: number) {
  const points: THREE.Vector3[] = []
  const steps = 20
  for (let i = 0; i <= steps; i++) {
    const t = i / steps
    const lon = lerpLon(lonA, lonB, t)
    const curLat = lat + latOffA * (1 - t) + latOffB * t + Math.sin(t * Math.PI) * 1.2
    points.push(latLngToVec3(curLat, lon, R + 0.085))
  }
  return points
}

function createOceanCurrents() {
  const warm = 0xff2f2f
  const cold = 0x2435a6

  const currents = [
    { name: '北赤道暖流', color: warm, path: makeCurrentPath(11, 11, 135, -155), count: 5, labelT: 0.5 },
    { name: '南赤道暖流', color: warm, path: makeCurrentPath(-13, -13, 150, -150), count: 5, labelT: 0.5 },
    { name: '赤道逆流', color: warm, path: makeCurrentPath(4, 4, -165, 145), count: 4, labelT: 0.5 },
    { name: '日本暖流', color: warm, path: makeCurrentPath(18, 36, 122, 142), count: 3, labelT: 0.52 },
    { name: '北太平洋暖流', color: warm, path: makeCurrentPath(34, 42, 142, -150), count: 4, labelT: 0.55 },
    { name: '阿拉斯加暖流', color: warm, path: makeCurrentPath(42, 58, -152, -136), count: 3, labelT: 0.52 },
    { name: '东澳大利亚暖流', color: warm, path: makeCurrentPath(-16, -36, 154, 160), count: 3, labelT: 0.54 },
    { name: '加利福尼亚寒流', color: cold, path: makeCurrentPath(44, 20, -126, -116), count: 4, labelT: 0.45 },
    { name: '秘鲁寒流', color: cold, path: makeCurrentPath(-10, -38, -82, -76), count: 4, labelT: 0.52 },
    { name: '千岛寒流', color: cold, path: makeCurrentPath(55, 36, 158, 145), count: 3, labelT: 0.46 },

    { name: '北赤道暖流', color: warm, path: makeCurrentPath(12, 12, -20, -62), count: 4, labelT: 0.45 },
    { name: '南赤道暖流', color: warm, path: makeCurrentPath(-12, -12, 8, -42), count: 4, labelT: 0.48 },
    { name: '墨西哥湾暖流', color: warm, path: makeCurrentPath(24, 42, -80, -48), count: 4, labelT: 0.5 },
    { name: '北大西洋暖流', color: warm, path: makeCurrentPath(42, 58, -48, -18), count: 4, labelT: 0.55 },
    { name: '巴西暖流', color: warm, path: makeCurrentPath(-12, -34, -38, -48), count: 3, labelT: 0.55 },
    { name: '加那利寒流', color: cold, path: makeCurrentPath(38, 15, -14, -20), count: 4, labelT: 0.5 },
    { name: '本格拉寒流', color: cold, path: makeCurrentPath(-15, -34, 8, 14), count: 3, labelT: 0.5 },
    { name: '拉布拉多寒流', color: cold, path: makeCurrentPath(58, 42, -50, -58), count: 3, labelT: 0.45 },
    { name: '东格陵兰寒流', color: cold, path: makeCurrentPath(70, 56, -22, -34), count: 3, labelT: 0.5 },

    {
      name: '季风洋流',
      color: warm,
      path: makeCurrentPathFromPoints([
        [6, 74],
        [10, 66],
        [12, 58],
        [7, 50],
      ]),
      count: 5,
      labelT: 0.52,
    },
    {
      name: '赤道逆流',
      color: warm,
      path: makeCurrentPathFromPoints([
        [2, 52],
        [3, 66],
        [3, 82],
        [2, 98],
      ]),
      count: 5,
      labelT: 0.5,
    },
    { name: '南赤道暖流', color: warm, path: makeCurrentPath(-14, -14, 96, 48), count: 5, labelT: 0.45 },
    { name: '厄加勒斯暖流', color: warm, path: makeCurrentPath(-18, -38, 46, 28), count: 4, labelT: 0.55 },
    {
      name: '索马里寒流',
      color: cold,
      path: makeCurrentPathFromPoints([
        [10, 52],
        [5, 50],
        [-4, 48],
      ]),
      count: 3,
      labelT: 0.52,
    },
    { name: '西澳大利亚寒流', color: cold, path: makeCurrentPath(-36, -18, 108, 112), count: 4, labelT: 0.5 },

    { name: '西风漂流', color: cold, path: makeCurrentPath(-56, -56, -70, 20), count: 5, labelT: 0.45 },
    { name: '西风漂流', color: cold, path: makeCurrentPath(-56, -56, 25, 120), count: 5, labelT: 0.52 },
    { name: '西风漂流', color: cold, path: makeCurrentPath(-56, -56, 125, -150), count: 5, labelT: 0.55 },
  ]

  currents.forEach(item => {
    addShortSurfaceArrows(item.path, item.color, item.count)
    addCurrentNameLabel(item.name, item.color, item.path, item.labelT)
  })
}

function makeCurrentPath(latA: number, latB: number, lonA: number, lonB: number) {
  const points: THREE.Vector3[] = []
  const steps = 44
  for (let i = 0; i <= steps; i++) {
    const t = i / steps
    const lat = latA + (latB - latA) * t + Math.sin(t * Math.PI) * 0.8
    const lon = lerpLon(lonA, lonB, t)
    points.push(latLngToVec3(lat, lon, R + 0.34))
  }
  return points
}

function makeCurrentPathFromPoints(latLonPoints: Array<[number, number]>) {
  const controlPoints = latLonPoints.map(([lat, lon]) => latLngToVec3(lat, lon, R + 0.34))
  const curve = new THREE.CatmullRomCurve3(controlPoints)
  return curve.getPoints(48)
}

function addShortSurfaceArrows(points: THREE.Vector3[], color: number, count: number) {
  for (let i = 0; i < count; i++) {
    const startT = 0.12 + i * (0.76 / Math.max(1, count))
    const endT = Math.min(0.95, startT + 0.14)
    const segment = slicePath(points, startT, endT, 10)
    currentGroup.add(makeSurfaceArrow(segment, color, 0.86, 0.038))
  }
}

function slicePath(points: THREE.Vector3[], startT: number, endT: number, steps: number) {
  const result: THREE.Vector3[] = []
  for (let i = 0; i <= steps; i++) {
    const t = startT + (endT - startT) * (i / steps)
    result.push(samplePath(points, t))
  }
  return result
}

function addCurrentNameLabel(name: string, color: number, points: THREE.Vector3[], t: number) {
  const pos = samplePath(points, t)
    .normalize()
    .multiplyScalar(R + 0.2)
  const textColor = color === 0xff2f2f ? '#ffb18a' : '#b8d4ff'
  const label = createSpriteText(name, textColor, pos, 0.22)
  label.userData.currentName = name
  label.userData.currentType = color === 0xff2f2f ? 'warm' : 'cold'
  currentGroup.add(label)
}

function makeSurfaceArrow(points: THREE.Vector3[], color: number, opacity = 0.85, radius = 0.045) {
  const group = new THREE.Group()
  const surfacePoints = points.map(p => {
    const len = p.length()
    if (len > R * 0.85)
      return p
        .clone()
        .normalize()
        .multiplyScalar(R + 0.085)
    return p.clone()
  })

  const end = surfacePoints[surfacePoints.length - 1]
  const headSize = radius * 4.2
  const { tailPoints, basePoint } = splitPathForArrow(surfacePoints, headSize * 0.92)

  group.add(makeTubeLine(tailPoints, color, radius, opacity))

  const arrowHead = makeMapArrowHead(color, headSize)
  arrowHead.position.copy(
    end!
      .clone()
      .normalize()
      .multiplyScalar(R + 0.088),
  )
  orientFlatArrowHead(arrowHead, basePoint!, end!)
  group.add(arrowHead)

  return group
}

function splitPathForArrow(points: THREE.Vector3[], cutLength: number) {
  if (points.length <= 2) {
    return { tailPoints: points, basePoint: points[0] }
  }

  let total = 0
  const lengths: number[] = [0]
  for (let i = 1; i < points.length; i++) {
    total += points[i]!.distanceTo(points[i - 1]!)
    lengths.push(total)
  }

  const target = Math.max(0.001, total - cutLength)
  const tailPoints: THREE.Vector3[] = []
  let basePoint = points[0]!.clone()

  for (let i = 0; i < points.length; i++) {
    if (lengths[i]! < target) {
      tailPoints.push(points[i]!.clone())
      continue
    }

    const prevIndex = Math.max(0, i - 1)
    const segStartLen = lengths[prevIndex]
    const segEndLen = lengths[i]
    const segT = segEndLen === segStartLen ? 0 : (target - segStartLen!) / (segEndLen! - segStartLen!)
    basePoint = points[prevIndex]!.clone().lerp(points[i]!, THREE.MathUtils.clamp(segT, 0, 1))
    basePoint.normalize().multiplyScalar(R + 0.085)
    tailPoints.push(basePoint.clone())
    break
  }

  if (tailPoints.length < 2) tailPoints.push(basePoint.clone())
  return { tailPoints, basePoint }
}

function makeMapArrowHead(color: number, size: number) {
  const group = new THREE.Group()
  const length = size
  const halfWidth = size * 0.34

  const shape = new THREE.Shape()
  shape.moveTo(0, 0)
  shape.lineTo(-halfWidth, -length)
  shape.lineTo(halfWidth, -length)
  shape.closePath()

  const head = new THREE.Mesh(
    new THREE.ShapeGeometry(shape),
    new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.96, side: THREE.DoubleSide, depthWrite: false }),
  )
  group.add(head)
  return group
}

function orientFlatArrowHead(obj: THREE.Object3D, from: THREE.Vector3, to: THREE.Vector3) {
  const dir = to.clone().sub(from).normalize()
  if (dir.lengthSq() < 0.000001) return
  const normal = to.clone().normalize()
  const xAxis = new THREE.Vector3().crossVectors(dir, normal).normalize()
  const yAxis = dir.clone()
  const zAxis = normal.clone()
  const matrix = new THREE.Matrix4().makeBasis(xAxis, yAxis, zAxis)
  obj.quaternion.setFromRotationMatrix(matrix)
}

function createFlatMapReference() {
  const bg = new THREE.Mesh(
    new THREE.PlaneGeometry(10.8, 5.8),
    new THREE.MeshBasicMaterial({ color: 0xf7fbff, transparent: true, opacity: 0.08, side: THREE.DoubleSide }),
  )
  flatMap.add(bg)

  flatMap.add(createSpriteText('全球气压带、风带分布模式图', '#ffffff', new THREE.Vector3(0, 2.55, 0.9), 0.48))

  const globeW = 6.4
  const globeH = 4.75
  const bandData = [
    { y: 2.08, h: 0.34, color: 0x7ed7ff, label: '极地高气压带', note: '90°' },
    { y: 1.47, h: 0.28, color: 0xd49cff, label: '副极地低气压带', note: '60°' },
    { y: 0.78, h: 0.34, color: 0xffd166, label: '副热带高气压带', note: '30°' },
    { y: 0, h: 0.38, color: 0xff7a7a, label: '赤道低气压带', note: '0°' },
    { y: -0.78, h: 0.34, color: 0xffd166, label: '副热带高气压带', note: '30°' },
    { y: -1.47, h: 0.28, color: 0xd49cff, label: '副极地低气压带', note: '60°' },
    { y: -2.08, h: 0.34, color: 0x7ed7ff, label: '极地高气压带', note: '90°' },
  ]

  bandData.forEach(b => {
    flatMap.add(makeFlatBand(b.y, globeW, b.h, b.color, 0.58))
    flatMap.add(createSpriteText(b.label, '#ffffff', new THREE.Vector3(0.35, b.y, 0.85), 0.34))
    flatMap.add(createSpriteText(b.note, '#6bdcff', new THREE.Vector3(-2.95, b.y + 0.05, 0.85), 0.28))
    flatMap.add(createSpriteText(b.note, '#6bdcff', new THREE.Vector3(2.95, b.y + 0.05, 0.85), 0.28))
  })

  flatMap.add(makeFlatEllipseOutline(globeW / 2, globeH / 2, 0xffffff, 0.86))

  const wind2D = [
    {
      y: 1.78,
      name: '极地东风',
      color: 0xffffff,
      arrows: [
        [1.6, 2.2, 0.7, 1.86],
        [0.4, 2.1, -0.55, 1.84],
        [-0.8, 2.0, -1.75, 1.75],
      ],
    },
    {
      y: 1.12,
      name: '盛行西风',
      color: 0xffffff,
      arrows: [
        [-1.7, 0.95, -0.8, 1.22],
        [-0.25, 0.92, 0.75, 1.2],
        [1.2, 0.92, 2.18, 1.22],
      ],
    },
    {
      y: 0.42,
      name: '东北信风',
      color: 0xff55b8,
      arrows: [
        [1.9, 0.55, 0.95, 0.25],
        [0.65, 0.55, -0.32, 0.24],
        [-0.65, 0.55, -1.6, 0.24],
      ],
    },
    {
      y: -0.42,
      name: '东南信风',
      color: 0xff55b8,
      arrows: [
        [-1.6, -0.55, -0.65, -0.24],
        [-0.32, -0.55, 0.65, -0.24],
        [0.95, -0.55, 1.9, -0.24],
      ],
    },
    {
      y: -1.12,
      name: '盛行西风',
      color: 0xffffff,
      arrows: [
        [-1.7, -0.92, -0.8, -1.22],
        [-0.25, -0.92, 0.75, -1.2],
        [1.2, -0.92, 2.18, -1.22],
      ],
    },
    {
      y: -1.78,
      name: '极地东风',
      color: 0xffffff,
      arrows: [
        [1.6, -2.2, 0.7, -1.86],
        [0.4, -2.1, -0.55, -1.84],
        [-0.8, -2.0, -1.75, -1.75],
      ],
    },
  ]

  wind2D.forEach(w => {
    w.arrows.forEach(([x1, y1, x2, y2]) => flatMap.add(makeFlatArrow([new THREE.Vector3(x1, y1, 0.5), new THREE.Vector3(x2, y2, 0.5)], w.color)))
    flatMap.add(createSpriteText(w.name, '#ffffff', new THREE.Vector3(2.55, w.y, 0.9), 0.34))
  })

  const loopDefs = [
    { name: '高纬环流', y1: 1.35, y2: 2.15, color: 0x52d9ff },
    { name: '中纬环流', y1: 0.75, y2: 1.45, color: 0xffd34f },
    { name: '低纬环流', y1: 0, y2: 0.75, color: 0xff55b8 },
    { name: '低纬环流', y1: -0.75, y2: 0, color: 0xff55b8 },
    { name: '中纬环流', y1: -1.45, y2: -0.75, color: 0xffd34f },
    { name: '高纬环流', y1: -2.15, y2: -1.35, color: 0x52d9ff },
  ]

  loopDefs.forEach((l, index) => {
    const x = -3.12
    const midY = (l.y1 + l.y2) / 2
    const pts = [new THREE.Vector3(x, l.y1, 0.45), new THREE.Vector3(x - 0.55, midY, 0.45), new THREE.Vector3(x, l.y2, 0.45)]
    flatMap.add(makeFlatCurveArrow(pts, l.color))
    flatMap.add(createSpriteText(l.name, '#ffffff', new THREE.Vector3(x - 0.9, midY, 0.9), 0.28))
  })

  flatMap.add(createSpriteText('上升', '#ff8ed8', new THREE.Vector3(-2.68, 0, 0.9), 0.3))
  flatMap.add(createSpriteText('下沉', '#8beeff', new THREE.Vector3(-2.68, 0.78, 0.9), 0.28))
  flatMap.add(createSpriteText('下沉', '#8beeff', new THREE.Vector3(-2.68, -0.78, 0.9), 0.28))

  flatMap.visible = viewMode.value === '2d'
}

function makeFlatBand(y: number, width: number, height: number, color: number, opacity: number) {
  const rx = width / 2
  const ry = 4.75 / 2
  const xSegments = 48
  const ySegments = 5
  const vertices: number[] = []
  const indices: number[] = []

  for (let iy = 0; iy <= ySegments; iy++) {
    const ty = iy / ySegments
    const curY = y - height / 2 + height * ty
    const safeY = THREE.MathUtils.clamp(curY, -ry + 0.001, ry - 0.001)
    const curHalfW = Math.max(0.08, rx * Math.sqrt(Math.max(0, 1 - (safeY * safeY) / (ry * ry))) - 0.04)

    for (let ix = 0; ix <= xSegments; ix++) {
      const tx = ix / xSegments
      const x = -curHalfW + curHalfW * 2 * tx
      vertices.push(x, curY, 0.12)
    }
  }

  for (let iy = 0; iy < ySegments; iy++) {
    for (let ix = 0; ix < xSegments; ix++) {
      const a = iy * (xSegments + 1) + ix
      const b = a + 1
      const c = a + xSegments + 1
      const d = c + 1
      indices.push(a, c, b, b, c, d)
    }
  }

  const geometry = new THREE.BufferGeometry()
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3))
  geometry.setIndex(indices)
  geometry.computeVertexNormals()

  return new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ color, transparent: true, opacity, side: THREE.DoubleSide, depthWrite: false }))
}

function makeFlatEllipseOutline(rx: number, ry: number, color: number, opacity: number) {
  const pts: THREE.Vector3[] = []
  for (let i = 0; i <= 180; i++) {
    const a = (i / 180) * Math.PI * 2
    pts.push(new THREE.Vector3(Math.cos(a) * rx, Math.sin(a) * ry, 0.32))
  }
  return makeFlatLine(pts, color, opacity)
}

function makeFlatCurveArrow(points: THREE.Vector3[], color: number) {
  const curve = new THREE.CatmullRomCurve3(points)
  const curvePoints = curve.getPoints(28)
  const group = new THREE.Group()
  group.add(makeFlatLine(curvePoints, color, 0.9))
  group.add(makeFlatArrow([curvePoints[curvePoints.length - 4]!, curvePoints[curvePoints.length - 1]!], color))
  return group
}

function makeFlatArrow(points: THREE.Vector3[], color: number) {
  const group = new THREE.Group()
  const start = points[0]
  const end = points[1]
  const dir = end!.clone().sub(start!).normalize()
  const angle = Math.atan2(dir.y, dir.x)

  const headLength = 0.22
  const base = end!.clone().sub(dir.clone().multiplyScalar(headLength * 0.9))
  group.add(makeFlatLine([start!, base], color, 0.9))

  const shape = new THREE.Shape()
  shape.moveTo(0, 0)
  shape.lineTo(-0.1, -headLength)
  shape.lineTo(0.1, -headLength)
  shape.closePath()

  const head = new THREE.Mesh(
    new THREE.ShapeGeometry(shape),
    new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.95, side: THREE.DoubleSide }),
  )
  head.position.copy(end!)
  head.rotation.z = angle - Math.PI / 2
  group.add(head)
  return group
}

function makeTubeLine(points: THREE.Vector3[], color: number, radius = 0.01, opacity = 1) {
  const curve = new THREE.CatmullRomCurve3(points)
  const geometry = new THREE.TubeGeometry(curve, Math.max(16, points.length * 2), radius, 8, false)
  const material = new THREE.MeshBasicMaterial({ color, transparent: true, opacity, depthWrite: false })
  return new THREE.Mesh(geometry, material)
}

function makeFlatLine(points: THREE.Vector3[], color: number, opacity = 1) {
  const geo = new THREE.BufferGeometry().setFromPoints(points)
  const mat = new THREE.LineBasicMaterial({ color, transparent: true, opacity })
  return new THREE.Line(geo, mat)
}

function createSpriteText(text: string, color: string, position: THREE.Vector3, size = 0.3) {
  const lines = text.split('\n')
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')!
  const dpr = 2
  canvas.width = 560 * dpr
  canvas.height = 150 * dpr
  ctx.scale(dpr, dpr)
  ctx.clearRect(0, 0, 560, 150)
  ctx.font = '900 36px Microsoft YaHei, Arial'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'

  const w = Math.max(...lines.map(line => ctx.measureText(line).width)) + 52
  ctx.fillStyle = 'rgba(7, 18, 32, 0.82)'
  roundRect(ctx, (560 - w) / 2, 14, w, 116, 18)
  ctx.fill()
  ctx.strokeStyle = 'rgba(255, 209, 102, 0.5)'
  ctx.lineWidth = 2
  ctx.stroke()

  ctx.fillStyle = color
  lines.forEach((line, i) => ctx.fillText(line, 280, lines.length === 1 ? 74 : 54 + i * 42))

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: texture, transparent: true }))
  sprite.position.copy(position)
  sprite.scale.set(size * 3.9, size * 1.15, 1)
  return sprite
}

function roundRect(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.arcTo(x + w, y, x + w, y + h, r)
  ctx.arcTo(x + w, y + h, x, y + h, r)
  ctx.arcTo(x, y + h, x, y, r)
  ctx.arcTo(x, y, x + w, y, r)
  ctx.closePath()
}

function latLngToVec3(lat: number, lon: number, radius: number) {
  const phi = THREE.MathUtils.degToRad(90 - lat)
  const theta = THREE.MathUtils.degToRad(lon + 180)
  const x = -radius * Math.sin(phi) * Math.cos(theta)
  const y = radius * Math.cos(phi)
  const z = radius * Math.sin(phi) * Math.sin(theta)
  return new THREE.Vector3(x, y, z)
}

function samplePath(points: THREE.Vector3[], progress: number) {
  const p = ((progress % 1) + 1) % 1
  const idx = p * (points.length - 1)
  const i0 = Math.floor(idx)
  const i1 = Math.min(points.length - 1, i0 + 1)
  const t = idx - i0
  return points[i0]!.clone().lerp(points[i1]!, t)
}

function clampLat(lat: number) {
  return Math.max(-89.5, Math.min(89.5, lat))
}

function normalizeLon(lon: number) {
  let v = lon
  while (v < -180) v += 360
  while (v > 180) v -= 360
  return v
}

function lerpLon(a: number, b: number, t: number) {
  let diff = b - a
  if (diff > 180) diff -= 360
  if (diff < -180) diff += 360
  return normalizeLon(a + diff * t)
}

function seasonalOffset(lat: number) {
  if (Math.abs(lat) < 1) return params.seasonShift * 0.35
  if (Math.abs(lat) <= 35) return params.seasonShift * 0.55
  return params.seasonShift * 0.25
}

function applyViewMode() {
  if (!earthGroup || !flatMap) return
  const is3d = viewMode.value === '3d'
  earthGroup.visible = is3d
  flatMap.visible = !is3d

  if (is3d) {
    camera.position.set(0, 2.4, 12)
    controls.enabled = !drawMode.value
    controls.enableRotate = !drawMode.value
  } else {
    camera.position.set(0, 0, 12)
    controls.enabled = !drawMode.value
    controls.enableRotate = false
    controls.target.set(0, 0, 0)
  }

  controls.update()
}

function applyLayerVisibility() {
  if (pressureGroup) pressureGroup.visible = layers.pressure
  if (cellGroup) cellGroup.visible = layers.cells
  if (windGroup) windGroup.visible = layers.wind
  if (currentGroup) currentGroup.visible = layers.currents
  if (graticuleGroup) graticuleGroup.visible = layers.graticuleLine
  if (labelGroup) labelGroup.visible = layers.graticuleText
  if (particleGroup) particleGroup.visible = layers.particles
}

function bindResize() {
  const resize = () => {
    if (!canvasWrapRef.value) return
    const { clientWidth, clientHeight } = canvasWrapRef.value
    renderer.setSize(clientWidth, clientHeight)
    camera.aspect = clientWidth / clientHeight
    camera.updateProjectionMatrix()

    const drawCanvas = drawCanvasRef.value
    if (drawCanvas) {
      drawCanvas.width = Math.max(1, Math.floor(clientWidth * window.devicePixelRatio))
      drawCanvas.height = Math.max(1, Math.floor(clientHeight * window.devicePixelRatio))
      drawCanvas.style.width = `${clientWidth}px`
      drawCanvas.style.height = `${clientHeight}px`
      nextTick(renderSharedDrawings)
    }
  }
  resizeObserver = new ResizeObserver(resize)
  resizeObserver.observe(canvasWrapRef.value!)
  resize()
}

function scheduleRebuild() {
  if (rebuildTimer) window.clearTimeout(rebuildTimer)
  rebuildTimer = window.setTimeout(() => rebuildAll(), 120)
}

function animate() {
  const dt = clock.getDelta()
  const elapsed = clock.elapsedTime

  if (earthGroup && viewMode.value === '3d') {
    earthGroup.rotation.z = THREE.MathUtils.degToRad(params.tilt)
    if (params.autoRotate && !drawMode.value) {
      earthGroup.rotation.y += dt * 0.018 * params.rotateSpeed
    }
  }

  circulationParticles.forEach((p, i) => {
    p.progress += dt * p.speed * params.cellSpeed
    const pos = samplePath(p.points, p.progress)
    p.mesh.position.copy(pos)
    p.halo.position.copy(pos)

    const mat = p.mesh.material as THREE.MeshBasicMaterial
    const haloMat = p.halo.material as THREE.MeshBasicMaterial
    const flicker = 0.74 + Math.sin(elapsed * 5.5 + i * 0.35) * 0.14
    mat.opacity = layers.particles ? flicker : 0
    haloMat.opacity = layers.particles ? 0.05 + flicker * 0.09 : 0
  })

  applyLayerVisibility()
  controls?.update()
  renderer?.render(scene, camera)
  animationId = requestAnimationFrame(animate)
}

watch(
  () => [params.seasonShift, params.particleDensity],
  () => scheduleRebuild(),
)

watch(layers, applyLayerVisibility, { deep: true })
watch(viewMode, () => {
  nextTick(() => {
    rebuildAll()
    renderSharedDrawings()
  })
})
watch(drawMode, () => {
  if (!controls) return
  controls.enabled = !drawMode.value
  controls.enableRotate = viewMode.value === '3d' && !drawMode.value
})

onMounted(() => nextTick(initThree))

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId)
  if (rebuildTimer) window.clearTimeout(rebuildTimer)
  resizeObserver?.disconnect()
  controls?.dispose()
  if (drawGroup) clearGroup(drawGroup)
  renderer?.dispose()
  if (scene) clearGroup(scene as unknown as THREE.Group)
})
</script>

<style scoped>
.three-cell-page {
  position: relative;
  height: 100vh;
  padding: 12px;
  overflow: hidden;
  color: #e9fbff;
  background:
    radial-gradient(circle at 18% 14%, rgba(0, 255, 255, 0.22), transparent 28%),
    radial-gradient(circle at 82% 72%, rgba(114, 92, 255, 0.26), transparent 35%),
    radial-gradient(circle at 50% 50%, rgba(0, 173, 255, 0.12), transparent 55%), linear-gradient(135deg, #020817 0%, #071a31 46%, #01040c 100%);
}

.bg-grid {
  position: absolute;
  inset: 0;
  pointer-events: none;
  opacity: 0.42;
  background-image:
    linear-gradient(rgba(68, 217, 255, 0.09) 1px, transparent 1px), linear-gradient(90deg, rgba(68, 217, 255, 0.09) 1px, transparent 1px);
  background-size: 34px 34px;
  mask-image: radial-gradient(circle at center, black, transparent 78%);
}

.scan-line {
  position: absolute;
  inset: -30% 0 auto;
  height: 45%;
  pointer-events: none;
  opacity: 0.12;
  background: linear-gradient(to bottom, transparent, #72f6ff, transparent);
  animation: scan 6s linear infinite;
}

@keyframes scan {
  from {
    transform: translateY(-40%);
  }
  to {
    transform: translateY(330%);
  }
}

.topbar {
  position: relative;
  z-index: 2;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  height: 58px;
  padding: 0 16px;
  border: 1px solid rgba(76, 228, 255, 0.35);
  border-radius: 18px;
  background: linear-gradient(90deg, rgba(4, 28, 48, 0.9), rgba(12, 14, 48, 0.78));
  box-shadow:
    0 0 26px rgba(0, 209, 255, 0.2),
    inset 0 0 30px rgba(57, 217, 255, 0.08);
  backdrop-filter: blur(14px);
}

.brand {
  display: flex;
  align-items: center;
  gap: 14px;
  min-width: 280px;
}

.logo-dot {
  width: 38px;
  height: 38px;
  flex: 0 0 auto;
  border-radius: 50%;
  background: radial-gradient(circle at 35% 32%, #fff1cf, #45eaff 35%, #1b55ff 68%, #071329);
  box-shadow: 0 0 22px rgba(92, 237, 255, 0.58);
}

.eyebrow {
  margin-bottom: 4px;
  font-size: 11px;
  letter-spacing: 0.2em;
  color: #6beeff;
}

h1 {
  margin: 0;
  font-size: 20px;
  letter-spacing: 0.04em;
  text-shadow: 0 0 18px rgba(74, 228, 255, 0.52);
  white-space: nowrap;
}

.top-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: nowrap;
}

.mode-switch,
.draw-switch,
.texture-switch {
  display: flex;
  gap: 6px;
  padding: 4px;
  border: 1px solid rgba(102, 237, 255, 0.24);
  border-radius: 14px;
  background: rgba(0, 10, 28, 0.55);
}

.mode-switch button,
.draw-switch button,
.texture-switch button {
  height: 34px;
  padding: 0 12px;
  cursor: pointer;
  color: #aeefff;
  border: 0;
  border-radius: 11px;
  background: transparent;
  transition: 0.2s ease;
  white-space: nowrap;
}

.mode-switch button.active,
.draw-switch button.active,
.texture-switch button.ghost {
  color: #9eefff;
  background: rgba(0, 213, 255, 0.08);
}

.texture-switch button.active {
  color: #02101c;
  background: linear-gradient(135deg, #69f4ff, #ffd166);
  box-shadow: 0 0 18px rgba(96, 240, 255, 0.45);
}

.draw-switch button.ghost {
  color: #9eefff;
  background: rgba(0, 213, 255, 0.08);
}

.color-picker {
  width: 34px;
  height: 34px;
  padding: 2px;
  cursor: pointer;
  border: 1px solid rgba(102, 237, 255, 0.32);
  border-radius: 10px;
  background: rgba(0, 10, 28, 0.65);
}

.local-texture-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  height: 34px;
  padding: 0 12px;
  cursor: pointer;
  color: #c7d2fe;
  border-radius: 11px;
  white-space: nowrap;
  background: rgba(37, 99, 235, 0.12);
}

.local-texture-btn input {
  display: none;
}

.content {
  position: relative;
  z-index: 1;
  display: grid;
  grid-template-columns: 280px minmax(0, 1fr) 360px;
  gap: 12px;
  height: calc(100vh - 82px);
  margin-top: 12px;
  min-height: 0;
}

.stage-card,
.control-card,
.knowledge-panel {
  position: relative;
  overflow: hidden;
  border: 1px solid rgba(89, 229, 255, 0.28);
  border-radius: 22px;
  background: linear-gradient(135deg, rgba(3, 30, 50, 0.72), rgba(6, 9, 32, 0.92));
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.35),
    inset 0 0 40px rgba(38, 216, 255, 0.06);
  backdrop-filter: blur(14px);
}

.stage-card::before,
.control-card::before {
  content: '';
  position: absolute;
  inset: 0;
  pointer-events: none;
  border-radius: inherit;
  background: linear-gradient(120deg, rgba(0, 255, 255, 0.24), transparent 18%, transparent 76%, rgba(126, 87, 255, 0.2));
}

.canvas-wrap {
  position: absolute;
  inset: 0;
  z-index: 2;
}

.draw-canvas {
  position: absolute;
  inset: 0;
  z-index: 5;
  width: 100%;
  height: 100%;
  pointer-events: none;
  cursor: crosshair;
}

.draw-canvas.active {
  pointer-events: auto;
}

.star-field {
  position: absolute;
  inset: 0;
  z-index: 0;
  pointer-events: none;
  opacity: 0.86;
  background-image:
    radial-gradient(circle, rgba(255, 255, 255, 0.95) 0 1px, transparent 1.6px),
    radial-gradient(circle, rgba(106, 239, 255, 0.75) 0 1px, transparent 1.8px),
    radial-gradient(circle, rgba(156, 137, 255, 0.65) 0 1px, transparent 1.8px);
  background-size:
    88px 88px,
    132px 132px,
    176px 176px;
  background-position:
    12px 20px,
    54px 72px,
    90px 30px;
  animation: starFloat 16s linear infinite;
}

.orbit-glow {
  position: absolute;
  z-index: 1;
  inset: 10%;
  pointer-events: none;
  border-radius: 50%;
  border: 1px solid rgba(107, 239, 255, 0.18);
  box-shadow:
    0 0 60px rgba(0, 217, 255, 0.18),
    inset 0 0 80px rgba(126, 87, 255, 0.1);
}

@keyframes starFloat {
  from {
    transform: translate3d(0, 0, 0);
  }
  to {
    transform: translate3d(-88px, 88px, 0);
  }
}

.corner {
  z-index: 3;
  position: absolute;
  width: 78px;
  height: 78px;
  pointer-events: none;
}

.corner-lt {
  top: 14px;
  left: 14px;
  border-top: 2px solid #6df3ff;
  border-left: 2px solid #6df3ff;
}

.corner-rb {
  right: 14px;
  bottom: 14px;
  border-right: 2px solid #ffd166;
  border-bottom: 2px solid #ffd166;
}

.legend-panel {
  z-index: 4;
  position: absolute;
  left: 14px;
  bottom: 14px;
  width: 250px;
  padding: 10px;
  border: 1px solid rgba(120, 234, 255, 0.28);
  border-radius: 16px;
  background: rgba(2, 12, 31, 0.72);
  box-shadow: 0 0 20px rgba(36, 221, 255, 0.16);
}

.legend-title,
.panel-title,
.block-title {
  font-weight: 800;
  color: #dcfbff;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 6px;
  font-size: 12px;
  color: #bfeaf4;
}

.dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  box-shadow: 0 0 12px currentColor;
}

.dot.low {
  color: #ff2f2f;
  background: #ff2f2f;
}
.dot.high {
  color: #ffb300;
  background: #ffb300;
}
.dot.wind {
  color: #ffffff;
  background: #ffffff;
}
.dot.cell {
  color: #ffcf56;
  background: #ffcf56;
}
.dot.warm {
  color: #ff2f2f;
  background: #ff2f2f;
}
.dot.cold {
  color: #2435a6;
  background: #2435a6;
}

.control-card,
.knowledge-panel {
  padding: 12px;
  overflow: hidden;
  min-height: 0;
}

.control-card {
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: rgba(56, 189, 248, 0.5) rgba(4, 7, 18, 0.48);
}

.control-card::-webkit-scrollbar {
  width: 6px;
}

.control-card::-webkit-scrollbar-thumb {
  border-radius: 8px;
  background: rgba(56, 189, 248, 0.5);
}

.control-card::-webkit-scrollbar-track {
  background: rgba(4, 7, 18, 0.48);
}

.knowledge-panel {
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: rgba(97, 242, 255, 0.45) rgba(2, 12, 31, 0.35);
}

.knowledge-panel::-webkit-scrollbar {
  width: 6px;
}

.knowledge-panel::-webkit-scrollbar-thumb {
  border-radius: 8px;
  background: rgba(97, 242, 255, 0.45);
}

.knowledge-panel::-webkit-scrollbar-track {
  background: rgba(2, 12, 31, 0.35);
}

.left-panel,
.right-panel {
  display: flex;
  flex-direction: column;
}

.panel-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  font-size: 17px;
}

.panel-title span {
  width: 8px;
  height: 22px;
  border-radius: 8px;
  background: linear-gradient(#00f5ff, #8d7cff);
  box-shadow: 0 0 16px rgba(0, 245, 255, 0.9);
}

.control-block,
.knowledge-card,
.summary-card {
  position: relative;
  margin-bottom: 8px;
  padding: 9px;
  border: 1px solid rgba(97, 242, 255, 0.28);
  border-radius: 16px;
  background: linear-gradient(135deg, rgba(0, 213, 255, 0.08), rgba(125, 87, 255, 0.08)), rgba(2, 12, 31, 0.5);
}

.block-title {
  margin-bottom: 7px;
  font-size: 14px;
}

.switch-row,
.control-block label {
  display: flex;
  align-items: center;
  gap: 7px;
  margin: 5px 0;
  font-size: 12px;
  color: #c5edf7;
}

.control-block input[type='checkbox'] {
  width: 16px;
  height: 16px;
  accent-color: #5deeff;
}

:deep(.range-row) {
  margin: 6px 0;
}

:deep(.range-head) {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
  font-size: 13px;
  color: #bfeaf4;
}

:deep(.range-head b) {
  color: #ffd166;
}

:deep(input[type='range']) {
  width: 100%;
  accent-color: #63f0ff;
}

.month-selector {
  margin: 8px 0 4px;
}

.month-title {
  margin-bottom: 7px;
  font-size: 12px;
  color: #bfeaf4;
}

.month-grid,
.theme-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 6px;
}

.theme-grid {
  grid-template-columns: repeat(1, 1fr);
}

.month-grid button,
.theme-grid button {
  height: 28px;
  cursor: pointer;
  border: 1px solid rgba(97, 242, 255, 0.22);
  border-radius: 9px;
  color: #c7f7ff;
  background: rgba(4, 27, 49, 0.72);
  transition:
    transform 0.16s ease,
    background 0.16s ease,
    box-shadow 0.16s ease;
}

.month-grid button:hover,
.theme-grid button:hover {
  transform: translateY(-1px);
  background: rgba(0, 213, 255, 0.14);
}

.month-grid button.active,
.theme-grid button.active {
  color: #05101d;
  border-color: transparent;
  background: linear-gradient(135deg, #00f5ff, #8d7cff);
  box-shadow: 0 0 14px rgba(0, 245, 255, 0.36);
}

.month-tip {
  margin-top: 7px;
  padding: 7px 8px;
  border-radius: 10px;
  color: #d8fbff;
  font-size: 11px;
  line-height: 1.35;
  background: rgba(0, 213, 255, 0.08);
  border: 1px solid rgba(97, 242, 255, 0.18);
}

.knowledge-card ul {
  margin: 0;
  padding-left: 16px;
  color: #d5e7df;
  font-size: 12px;
  line-height: 1.42;
}

.knowledge-card li + li {
  margin-top: 4px;
}

.tips-grid {
  display: grid;
  gap: 5px;
}

.tips-grid div {
  display: grid;
  grid-template-columns: 62px 1fr;
  gap: 6px;
  padding: 6px;
  border-radius: 10px;
  background: rgba(255, 225, 151, 0.07);
  border: 1px solid rgba(255, 209, 102, 0.18);
}

.tips-grid b {
  color: #ffd166;
  font-size: 12px;
}

.tips-grid span {
  color: #d8eee9;
  font-size: 11px;
  line-height: 1.25;
}

.season-rule {
  display: grid;
  gap: 6px;
}

.season-rule div {
  display: grid;
  grid-template-columns: 74px 1fr;
  gap: 7px;
  padding: 7px 8px;
  border: 1px solid rgba(97, 242, 255, 0.18);
  border-radius: 10px;
  background: rgba(0, 213, 255, 0.06);
}

.season-rule b {
  color: #63f0ff;
  font-size: 12px;
}

.season-rule span {
  color: #d8eee9;
  font-size: 11px;
  line-height: 1.34;
}

.summary-card table {
  width: 100%;
  border-collapse: collapse;
  overflow: hidden;
  font-size: 10px;
  color: #d8eee9;
}

.summary-card th,
.summary-card td {
  padding: 4px 3px;
  border: 1px solid rgba(126, 231, 255, 0.18);
  text-align: center;
}

.summary-card th {
  color: #ffd166;
  background: rgba(255, 209, 102, 0.08);
}

@media (max-width: 1200px) {
  .content {
    grid-template-columns: 250px minmax(0, 1fr) 320px;
  }

  .legend-panel {
    display: none;
  }

  .texture-switch {
    display: none;
  }
}

/* ===== 新配色覆盖：量子深空 / 霓虹紫蓝 / 全息科技风 ===== */
.three-cell-page {
  color: #f8fbff;
  background:
    radial-gradient(circle at 16% 16%, rgba(99, 102, 241, 0.28), transparent 28%),
    radial-gradient(circle at 82% 72%, rgba(14, 165, 233, 0.18), transparent 34%),
    radial-gradient(circle at 50% 46%, rgba(217, 70, 239, 0.14), transparent 52%), linear-gradient(135deg, #030712 0%, #090b22 42%, #050816 100%);
}

.bg-grid {
  opacity: 0.48;
  background-image:
    linear-gradient(rgba(129, 140, 248, 0.11) 1px, transparent 1px), linear-gradient(90deg, rgba(56, 189, 248, 0.09) 1px, transparent 1px);
  background-size: 32px 32px;
}

.scan-line {
  opacity: 0.16;
  background: linear-gradient(to bottom, transparent, #818cf8, #38bdf8, transparent);
}

.topbar {
  border-color: rgba(129, 140, 248, 0.42);
  background:
    linear-gradient(90deg, rgba(5, 8, 22, 0.92), rgba(20, 18, 55, 0.84)),
    radial-gradient(circle at 20% 50%, rgba(56, 189, 248, 0.18), transparent 40%);
  box-shadow:
    0 0 28px rgba(99, 102, 241, 0.22),
    inset 0 0 32px rgba(56, 189, 248, 0.08);
}

.logo-dot {
  background: radial-gradient(circle at 35% 32%, #f8fafc, #38bdf8 32%, #6366f1 62%, #050816);
  box-shadow:
    0 0 24px rgba(129, 140, 248, 0.72),
    0 0 42px rgba(56, 189, 248, 0.25);
}

.eyebrow,
.month-title {
  color: #93c5fd;
}

h1 {
  text-shadow:
    0 0 18px rgba(129, 140, 248, 0.58),
    0 0 34px rgba(56, 189, 248, 0.2);
}

.mode-switch,
.draw-switch,
.texture-switch {
  border-color: rgba(129, 140, 248, 0.36);
  background: rgba(4, 7, 18, 0.76);
  box-shadow: inset 0 0 18px rgba(99, 102, 241, 0.08);
}

.mode-switch button,
.draw-switch button,
.texture-switch button {
  color: #c7d2fe;
}

.mode-switch button.active,
.draw-switch button.active,
.texture-switch button.active,
.texture-switch button.ghost.active,
.month-grid button.active {
  color: #020617 !important;
  background: linear-gradient(135deg, #f8fafc 0%, #38bdf8 38%, #818cf8 72%, #c084fc 100%) !important;
  box-shadow:
    0 0 18px rgba(56, 189, 248, 0.55),
    0 0 30px rgba(129, 140, 248, 0.32) !important;
}

.draw-switch button.ghost,
.texture-switch button.ghost {
  color: #93c5fd;
  background: rgba(37, 99, 235, 0.12);
}

.color-picker {
  border-color: rgba(129, 140, 248, 0.42);
  background: rgba(4, 7, 18, 0.78);
}

.stage-card,
.control-card,
.knowledge-panel {
  border-color: rgba(129, 140, 248, 0.28);
  background:
    linear-gradient(135deg, rgba(8, 13, 33, 0.92), rgba(5, 8, 22, 0.97)),
    radial-gradient(circle at 24% 16%, rgba(56, 189, 248, 0.16), transparent 36%),
    radial-gradient(circle at 76% 82%, rgba(192, 132, 252, 0.14), transparent 38%);
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.48),
    inset 0 0 44px rgba(99, 102, 241, 0.08),
    0 0 24px rgba(56, 189, 248, 0.08);
}

.stage-card::before,
.control-card::before {
  background:
    linear-gradient(120deg, rgba(56, 189, 248, 0.22), transparent 20%, transparent 72%, rgba(192, 132, 252, 0.22)),
    linear-gradient(180deg, rgba(248, 250, 252, 0.04), transparent);
}

.star-field {
  opacity: 0.82;
  background-image:
    radial-gradient(circle, rgba(248, 250, 252, 0.96) 0 1px, transparent 1.6px),
    radial-gradient(circle, rgba(56, 189, 248, 0.72) 0 1px, transparent 1.8px),
    radial-gradient(circle, rgba(192, 132, 252, 0.66) 0 1px, transparent 1.8px);
}

.orbit-glow {
  border-color: rgba(56, 189, 248, 0.22);
  box-shadow:
    0 0 70px rgba(56, 189, 248, 0.18),
    0 0 90px rgba(99, 102, 241, 0.12),
    inset 0 0 92px rgba(129, 140, 248, 0.08);
}

.corner-lt {
  border-top-color: #38bdf8;
  border-left-color: #38bdf8;
  box-shadow: -8px -8px 18px rgba(56, 189, 248, 0.14);
}

.corner-rb {
  border-right-color: #c084fc;
  border-bottom-color: #c084fc;
  box-shadow: 8px 8px 18px rgba(192, 132, 252, 0.14);
}

.legend-panel {
  border-color: rgba(56, 189, 248, 0.28);
  background: rgba(3, 7, 18, 0.76);
  box-shadow:
    0 0 22px rgba(56, 189, 248, 0.14),
    inset 0 0 20px rgba(129, 140, 248, 0.06);
}

.legend-title,
.panel-title,
.block-title {
  color: #f8fbff;
}

.legend-item,
.control-block label,
:deep(.range-head),
.month-tip,
.tips-grid span,
.season-rule span,
.knowledge-card ul,
.summary-card table {
  color: #dbeafe;
}

.control-block,
.knowledge-card,
.summary-card {
  border-color: rgba(129, 140, 248, 0.28);
  background: linear-gradient(135deg, rgba(37, 99, 235, 0.12), rgba(147, 51, 234, 0.08)), rgba(3, 7, 18, 0.62);
  box-shadow: inset 0 0 22px rgba(56, 189, 248, 0.045);
}

.panel-title span {
  background: linear-gradient(#f8fafc, #38bdf8, #818cf8, #c084fc);
  box-shadow:
    0 0 18px rgba(56, 189, 248, 0.65),
    0 0 28px rgba(129, 140, 248, 0.32);
}

:deep(input[type='range']),
.control-block input[type='checkbox'] {
  accent-color: #38bdf8;
}

.month-grid button {
  color: #dbeafe;
  border-color: rgba(129, 140, 248, 0.26);
  background: rgba(4, 7, 18, 0.78);
}

.month-grid button:hover {
  background: rgba(37, 99, 235, 0.18);
  box-shadow: 0 0 12px rgba(56, 189, 248, 0.16);
}

.tips-grid div,
.season-rule div {
  border-color: rgba(129, 140, 248, 0.22);
  background: linear-gradient(135deg, rgba(56, 189, 248, 0.07), rgba(192, 132, 252, 0.06));
}

.tips-grid b,
.summary-card th,
:deep(.range-head b) {
  color: #93c5fd;
}

.season-rule b {
  color: #67e8f9;
}

.summary-card th {
  background: rgba(129, 140, 248, 0.08);
}

.summary-card th,
.summary-card td {
  border-color: rgba(129, 140, 248, 0.18);
}

.knowledge-panel {
  scrollbar-color: rgba(56, 189, 248, 0.5) rgba(4, 7, 18, 0.48);
}

.knowledge-panel::-webkit-scrollbar-thumb {
  background: rgba(56, 189, 248, 0.5);
}

.knowledge-panel::-webkit-scrollbar-track {
  background: rgba(4, 7, 18, 0.48);
}

/* ===== 配色预设覆盖：每套主题只覆盖关键变量，激活色保持明显 ===== */
.theme-aurora {
  color: #f8fbff;
  background:
    radial-gradient(circle at 18% 18%, rgba(226, 232, 240, 0.18), transparent 28%),
    radial-gradient(circle at 82% 72%, rgba(99, 102, 241, 0.22), transparent 34%), linear-gradient(135deg, #030712 0%, #111827 48%, #020617 100%);
}

.theme-aurora .topbar,
.theme-aurora .stage-card,
.theme-aurora .control-card,
.theme-aurora .knowledge-panel {
  border-color: rgba(226, 232, 240, 0.28);
  background: linear-gradient(135deg, rgba(15, 23, 42, 0.92), rgba(3, 7, 18, 0.98));
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.48),
    inset 0 0 44px rgba(226, 232, 240, 0.055);
}

.theme-aurora .mode-switch button.active,
.theme-aurora .draw-switch button.active,
.theme-aurora .texture-switch button.active,
.theme-aurora .theme-grid button.active,
.theme-aurora .month-grid button.active {
  background: linear-gradient(135deg, #ffffff, #cbd5e1 42%, #818cf8) !important;
  box-shadow: 0 0 18px rgba(203, 213, 225, 0.48) !important;
}

.theme-nebula {
  color: #fff7fb;
  background:
    radial-gradient(circle at 16% 16%, rgba(217, 70, 239, 0.24), transparent 28%),
    radial-gradient(circle at 82% 72%, rgba(99, 102, 241, 0.22), transparent 34%),
    radial-gradient(circle at 50% 46%, rgba(244, 114, 182, 0.12), transparent 52%), linear-gradient(135deg, #090713 0%, #1a1230 45%, #05030a 100%);
}

.theme-nebula .topbar,
.theme-nebula .stage-card,
.theme-nebula .control-card,
.theme-nebula .knowledge-panel {
  border-color: rgba(216, 180, 254, 0.3);
  background: linear-gradient(135deg, rgba(30, 18, 54, 0.92), rgba(8, 5, 18, 0.98));
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.48),
    inset 0 0 44px rgba(217, 70, 239, 0.07);
}

.theme-nebula .mode-switch button.active,
.theme-nebula .draw-switch button.active,
.theme-nebula .texture-switch button.active,
.theme-nebula .theme-grid button.active,
.theme-nebula .month-grid button.active {
  background: linear-gradient(135deg, #f8fafc, #f0abfc 44%, #818cf8) !important;
  box-shadow: 0 0 18px rgba(217, 70, 239, 0.45) !important;
}

.info-float {
  z-index: 6;
  position: absolute;
  right: 18px;
  top: 18px;
  width: 260px;
  padding: 14px;
  border: 1px solid rgba(129, 140, 248, 0.32);
  border-radius: 16px;
  color: #dbeafe;
  background: rgba(3, 7, 18, 0.82);
  box-shadow:
    0 0 26px rgba(56, 189, 248, 0.16),
    inset 0 0 18px rgba(129, 140, 248, 0.08);
  backdrop-filter: blur(12px);
}

.info-float button {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 24px;
  height: 24px;
  cursor: pointer;
  color: #dbeafe;
  border: 0;
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.08);
}

.info-float h3 {
  margin: 0 28px 8px 0;
  font-size: 16px;
  color: #f8fbff;
}

.info-float p {
  margin: 0;
  font-size: 12px;
  line-height: 1.55;
}

.info-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 10px;
}

.info-tags span {
  padding: 3px 7px;
  border-radius: 999px;
  font-size: 11px;
  color: #020617;
  background: linear-gradient(135deg, #f8fafc, #38bdf8, #818cf8);
}

.season-compare-panel {
  z-index: 4;
  position: absolute;
  left: 50%;
  top: 14px;
  width: min(720px, calc(100% - 36px));
  transform: translateX(-50%);
  padding: 12px;
  border: 1px solid rgba(129, 140, 248, 0.28);
  border-radius: 18px;
  color: #dbeafe;
  background: rgba(3, 7, 18, 0.78);
  box-shadow:
    0 0 28px rgba(56, 189, 248, 0.14),
    inset 0 0 24px rgba(129, 140, 248, 0.06);
  backdrop-filter: blur(12px);
}

.mini-title {
  font-size: 13px;
  font-weight: 900;
  color: #f8fbff;
  letter-spacing: 0.04em;
}

.split-compare-body {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin-top: 10px;
}

.season-screen {
  padding: 10px;
  border: 1px solid rgba(129, 140, 248, 0.22);
  border-radius: 14px;
  background: linear-gradient(135deg, rgba(15, 23, 42, 0.78), rgba(30, 41, 59, 0.28));
}

.screen-head {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 24px;
  margin-bottom: 8px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 800;
  color: #020617;
}

.jan-screen .screen-head {
  background: linear-gradient(135deg, #dbeafe, #818cf8);
}

.jul-screen .screen-head {
  background: linear-gradient(135deg, #f8fafc, #38bdf8);
}

.mini-earth-map {
  position: relative;
  height: 210px;
  overflow: hidden;
  border: 1px solid rgba(226, 232, 240, 0.25);
  border-radius: 50% / 46%;
  background:
    linear-gradient(90deg, transparent 49%, rgba(248, 250, 252, 0.3) 50%, transparent 51%),
    linear-gradient(180deg, rgba(56, 189, 248, 0.08), rgba(129, 140, 248, 0.06));
  box-shadow: inset 0 0 34px rgba(56, 189, 248, 0.08);
}

.mini-lat {
  position: absolute;
  left: 0;
  right: 0;
  z-index: 8;
  height: 1px;
  border-top: 1px dashed rgba(226, 232, 240, 0.24);
  font-size: 10px;
  color: rgba(226, 232, 240, 0.92);
  z-index: 5;
  pointer-events: none;
}

.mini-lat span {
  position: absolute;
  left: 46px;
  top: -13px;
  z-index: 9;
  padding: 2px 7px;
  border: 1px solid rgba(226, 232, 240, 0.28);
  border-radius: 999px;
  color: #f8fbff;
  background: rgba(3, 7, 18, 0.96);
  box-shadow:
    0 0 12px rgba(3, 7, 18, 0.9),
    0 0 10px rgba(56, 189, 248, 0.18);
  white-space: nowrap;
}

.lat-60n {
  top: 18%;
}
.lat-30n {
  top: 34%;
}
.lat-0 {
  top: 50%;
  border-top-color: rgba(248, 250, 252, 0.42);
}
.lat-30s {
  top: 66%;
}
.lat-60s {
  top: 82%;
}

.mini-band {
  z-index: 1;
  position: absolute;
  left: 50%;
  height: 8px;
  transform: translateX(-50%);
  border-radius: 999px;
  opacity: 0.9;
  box-shadow: 0 0 12px currentColor;
}

.mini-band.high {
  color: #facc15;
  background: #facc15;
}

.mini-band.low {
  color: #f472b6;
  background: #f472b6;
}

.mini-band.strong {
  height: 10px;
}

.polar-n {
  top: 4%;
  width: 26%;
}
.subpolar-n {
  top: 22%;
  width: 72%;
}
.subtropical-n {
  top: 38%;
  width: 90%;
}
.equator {
  top: 52%;
  width: 96%;
}
.subtropical-s {
  top: 68%;
  width: 86%;
}
.subpolar-s {
  top: 84%;
  width: 62%;
}
.polar-s {
  top: 95%;
  width: 24%;
}

.south-shift .mini-band {
  margin-top: 11px;
}

.north-shift .mini-band {
  margin-top: -11px;
}

.wind-arrow {
  z-index: 2;
  position: absolute;
  font-size: 22px;
  font-weight: 900;
  line-height: 1;
  text-shadow: 0 0 14px currentColor;
  opacity: 0.95;
}

.trade-n {
  color: #f8fafc;
}
.westerly-n {
  color: #93c5fd;
}
.trade-s {
  color: #f8fafc;
}
.westerly-s {
  color: #93c5fd;
}

.trade-n.a1 {
  left: 24%;
  top: 39%;
}
.trade-n.a2 {
  left: 39%;
  top: 43%;
}
.trade-n.a3 {
  left: 54%;
  top: 47%;
}

.westerly-n.a1 {
  left: 24%;
  top: 31%;
}
.westerly-n.a2 {
  left: 43%;
  top: 27%;
}
.westerly-n.a3 {
  left: 62%;
  top: 23%;
}

.trade-s.a1 {
  left: 24%;
  top: 61%;
}
.trade-s.a2 {
  left: 39%;
  top: 57%;
}
.trade-s.a3 {
  left: 54%;
  top: 53%;
}

.westerly-s.a1 {
  left: 24%;
  top: 69%;
}
.westerly-s.a2 {
  left: 43%;
  top: 73%;
}
.westerly-s.a3 {
  left: 62%;
  top: 77%;
}

.screen-note {
  margin-top: 8px;
  font-size: 12px;
  line-height: 1.35;
  color: #dbeafe;
  text-align: center;
}

.compare-row,
.compare-bar {
  display: none;
}

.theme-teal {
  color: #ecfeff;
  background:
    radial-gradient(circle at 18% 18%, rgba(20, 184, 166, 0.28), transparent 28%),
    radial-gradient(circle at 82% 72%, rgba(34, 211, 238, 0.2), transparent 34%), linear-gradient(135deg, #021314 0%, #042f2e 48%, #020617 100%);
}

.theme-teal .topbar,
.theme-teal .stage-card,
.theme-teal .control-card,
.theme-teal .knowledge-panel {
  border-color: rgba(45, 212, 191, 0.34);
  background: linear-gradient(135deg, rgba(4, 47, 46, 0.9), rgba(2, 6, 23, 0.96));
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.48),
    inset 0 0 44px rgba(20, 184, 166, 0.08);
}

.theme-teal .mode-switch button.active,
.theme-teal .draw-switch button.active,
.theme-teal .texture-switch button.active,
.theme-teal .theme-grid button.active,
.theme-teal .month-grid button.active {
  color: #021314 !important;
  background: linear-gradient(135deg, #ecfeff, #2dd4bf 48%, #22d3ee) !important;
  box-shadow: 0 0 18px rgba(45, 212, 191, 0.52) !important;
}

.theme-gold {
  color: #fffbea;
  background:
    radial-gradient(circle at 16% 16%, rgba(234, 179, 8, 0.2), transparent 28%),
    radial-gradient(circle at 82% 72%, rgba(148, 163, 184, 0.14), transparent 34%), linear-gradient(135deg, #09090b 0%, #1c1917 48%, #030303 100%);
}

.theme-gold .topbar,
.theme-gold .stage-card,
.theme-gold .control-card,
.theme-gold .knowledge-panel {
  border-color: rgba(234, 179, 8, 0.32);
  background: linear-gradient(135deg, rgba(28, 25, 23, 0.94), rgba(3, 3, 3, 0.98));
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.5),
    inset 0 0 44px rgba(234, 179, 8, 0.06);
}

.theme-gold .mode-switch button.active,
.theme-gold .draw-switch button.active,
.theme-gold .texture-switch button.active,
.theme-gold .theme-grid button.active,
.theme-gold .month-grid button.active {
  color: #111827 !important;
  background: linear-gradient(135deg, #fff7ed, #facc15 50%, #94a3b8) !important;
  box-shadow: 0 0 18px rgba(234, 179, 8, 0.5) !important;
}
</style>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref, type ComponentPublicInstance } from 'vue'
import {
  BarElement,
  CategoryScale,
  Chart as ChartJS,
  type ChartOptions,
  Filler,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  Title,
  Tooltip,
} from 'chart.js'
import { Bar, Line } from 'vue-chartjs'
import storyData from '../data/space-story.json'

const clipByProgressPlugin = {
  id: 'clipByProgress',
  beforeDatasetsDraw(chart: any, _args: any, options: any) {
    const progress = typeof options?.progress === 'number' ? Math.min(Math.max(options.progress, 0), 1) : 1
    const area = chart.chartArea

    if (!area || progress >= 1) {
      return
    }

    const width = area.right - area.left
    chart.ctx.save()
    chart.ctx.beginPath()
    chart.ctx.rect(area.left, area.top, width * progress, area.bottom - area.top)
    chart.ctx.clip()
  },
  afterDatasetsDraw(chart: any, _args: any, options: any) {
    const progress = typeof options?.progress === 'number' ? Math.min(Math.max(options.progress, 0), 1) : 1
    if (progress < 1) {
      chart.ctx.restore()
    }
  },
}

ChartJS.register(
  BarElement,
  CategoryScale,
  Filler,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  Title,
  Tooltip,
  clipByProgressPlugin,
)

type ChapterId =
  | 'chapter1_notLongAgo'
  | 'chapter2_everythingChanged'
  | 'chapter3_crossedThreshold'
  | 'chapter4_expandingPossibilities'
  | 'chapter5_whatsNext'

type ActivityPoint = {
  name: string
  value: number
}

type TimelinePoint = {
  year: number
  isProjection: boolean
  launchFrequency: {
    total: number
    commercial: number
    government: number
  }
  peopleReachingOrbit: {
    total: number
    commercial: number
    government: number
  }
  averageTicketPrice: number
  commercialMissionActivity: ActivityPoint[]
}

const timeline = storyData.timeline as TimelinePoint[]
const pointByYear = new Map(timeline.map((item) => [item.year, item]))

const heroTitle = storyData.meta.story.title
const heroTheme = storyData.meta.story.theme

const getPointsBetween = (start: number, end: number) =>
  timeline.filter((point) => point.year >= start && point.year <= end)

const chapter1Points = getPointsBetween(2005, 2010)
const chapter2Points = getPointsBetween(2012, 2018)
const chapter3Points = getPointsBetween(2020, 2025)
const chapter4Points = [2022, 2025, 2030]
  .map((year) => pointByYear.get(year))
  .filter((value): value is TimelinePoint => Boolean(value))
const chapter5Points = getPointsBetween(2028, 2032)

const latestHistorical = pointByYear.get(2025)
const lastProjection = pointByYear.get(2032)

const sectionOrder: ChapterId[] = [
  'chapter1_notLongAgo',
  'chapter2_everythingChanged',
  'chapter3_crossedThreshold',
  'chapter4_expandingPossibilities',
  'chapter5_whatsNext',
]

const visibleChapters = reactive<Record<ChapterId, boolean>>({
  chapter1_notLongAgo: false,
  chapter2_everythingChanged: false,
  chapter3_crossedThreshold: false,
  chapter4_expandingPossibilities: false,
  chapter5_whatsNext: false,
})

const chapterProgress = reactive<Record<ChapterId, number>>({
  chapter1_notLongAgo: 0,
  chapter2_everythingChanged: 0,
  chapter3_crossedThreshold: 0,
  chapter4_expandingPossibilities: 0,
  chapter5_whatsNext: 0,
})

const barChapterProgress = ref(0)
let barTweenRafId: number | null = null
let barTweenStarted = false

const activeChapter = ref<ChapterId>('chapter1_notLongAgo')
const chapterElements = new Map<ChapterId, HTMLElement>()
let scrollRafId: number | null = null
let pendingScrollFrame = false

const metricTargets = {
  c1People2005: pointByYear.get(2005)?.peopleReachingOrbit.total ?? 0,
  c1Launches2005: pointByYear.get(2005)?.launchFrequency.total ?? 0,
  c1Ticket2005: pointByYear.get(2005)?.averageTicketPrice ?? 0,
  c2LaunchGrowth:
    (pointByYear.get(2018)?.launchFrequency.commercial ?? 0) -
    (pointByYear.get(2012)?.launchFrequency.commercial ?? 0),
  c2PriceDrop:
    (pointByYear.get(2012)?.averageTicketPrice ?? 0) - (pointByYear.get(2018)?.averageTicketPrice ?? 0),
  c3People2025: latestHistorical?.peopleReachingOrbit.total ?? 0,
  c3CommercialShare: latestHistorical
    ? Math.round((latestHistorical.launchFrequency.commercial / latestHistorical.launchFrequency.total) * 100)
    : 0,
  c4Infra2030:
    pointByYear.get(2030)?.commercialMissionActivity.find((entry) => entry.name === 'Infrastructure')?.value ?? 0,
  c4Medical2030:
    pointByYear.get(2030)?.commercialMissionActivity.find((entry) => entry.name === 'Medical')?.value ?? 0,
  c5People2032: lastProjection?.peopleReachingOrbit.total ?? 0,
  c5Ticket2032: lastProjection?.averageTicketPrice ?? 0,
}

const chapter2LaunchMultiple = computed(() => {
  const start = pointByYear.get(2012)?.launchFrequency.commercial ?? 1
  const end = pointByYear.get(2018)?.launchFrequency.commercial ?? 1
  return Math.round(end / start)
})

const chapter2PriceLowerPercent = computed(() => {
  const start = pointByYear.get(2012)?.averageTicketPrice ?? 1
  const end = pointByYear.get(2018)?.averageTicketPrice ?? 1
  return Math.round((1 - end / start) * 100)
})

const palette = {
  indigo: '#485696',
  light: '#E7E7E7',
  sand: '#F9C784',
  orange: '#FC7A1E',
  ember: '#F24C00',
}

const easeOutCubic = (value: number) => 1 - Math.pow(1 - value, 3)
const easeInOutCubic = (value: number) =>
  value < 0.5 ? 4 * value * value * value : 1 - Math.pow(-2 * value + 2, 3) / 2

const chapter1CounterProgress = computed(() => {
  // Slightly delay and smooth Chapter 1 counters so they read as intentional, not jumpy.
  const delayed = clamp((chapterProgress.chapter1_notLongAgo - 0.05) / 0.95)
  return easeInOutCubic(Math.pow(delayed, 1.25))
})

const animatedValues = computed(() => ({
  c1People2005: Math.round(metricTargets.c1People2005 * chapter1CounterProgress.value),
  c1Launches2005: Math.round(metricTargets.c1Launches2005 * chapter1CounterProgress.value),
  c1Ticket2005: Math.round(metricTargets.c1Ticket2005 * chapter1CounterProgress.value),
  c2LaunchGrowth: Math.round(metricTargets.c2LaunchGrowth * easeOutCubic(chapterProgress.chapter2_everythingChanged)),
  c2PriceDrop: Math.round(metricTargets.c2PriceDrop * easeOutCubic(chapterProgress.chapter2_everythingChanged)),
  c3People2025: Math.round(metricTargets.c3People2025 * easeOutCubic(chapterProgress.chapter3_crossedThreshold)),
  c3CommercialShare: Math.round(
    metricTargets.c3CommercialShare * easeOutCubic(chapterProgress.chapter3_crossedThreshold),
  ),
  c4Infra2030: Math.round(metricTargets.c4Infra2030 * easeOutCubic(chapterProgress.chapter4_expandingPossibilities)),
  c4Medical2030: Math.round(metricTargets.c4Medical2030 * easeOutCubic(chapterProgress.chapter4_expandingPossibilities)),
  c5People2032: Math.round(metricTargets.c5People2032 * easeOutCubic(chapterProgress.chapter5_whatsNext)),
  c5Ticket2032: Math.round(metricTargets.c5Ticket2032 * easeOutCubic(chapterProgress.chapter5_whatsNext)),
}))

const revealLineSeries = (_chapterId: ChapterId, values: number[]) => values

const revealBarSeriesAtProgress = (progress: number, values: number[]) => {
  const position = progress * values.length
  const fullBars = Math.floor(position)
  const fraction = position - fullBars

  return values.map((value, index) => {
    if (index < fullBars) {
      return value
    }

    if (index === fullBars) {
      return Number((value * fraction).toFixed(2))
    }

    return 0
  })
}

const sharedLineOptions: ChartOptions<'line'> = {
  responsive: true,
  maintainAspectRatio: false,
  animation: {
    duration: 1000,
    easing: 'easeOutQuart',
  },
  plugins: {
    legend: {
      labels: {
        color: palette.light,
        boxWidth: 14,
      },
    },
  },
  scales: {
    x: {
      ticks: { color: '#B9BDD4' },
      grid: { color: 'rgba(72, 86, 150, 0.28)' },
    },
    y: {
      ticks: { color: '#B9BDD4' },
      grid: { color: 'rgba(72, 86, 150, 0.28)' },
    },
  },
}

const withLineClipProgress = (progress: number, options: ChartOptions<'line'>): ChartOptions<'line'> => ({
  ...options,
  plugins: {
    ...(options.plugins ?? {}),
    clipByProgress: {
      progress,
    },
  } as any,
})

const chapter1ChartData = computed(() => ({
  labels: chapter1Points.map((item) => String(item.year)),
  datasets: [
    {
      label: 'Total launches',
      data: revealLineSeries(
        'chapter1_notLongAgo',
        chapter1Points.map((item) => item.launchFrequency.total),
      ),
      borderColor: palette.indigo,
      backgroundColor: 'rgba(72, 86, 150, 0.22)',
      fill: true,
      tension: 0.35,
      pointRadius: 2,
    },
  ],
}))

const chapter2ChartData = computed(() => ({
  labels: chapter2Points.map((item) => String(item.year)),
  datasets: [
    {
      label: 'Commercial launches',
      data: revealLineSeries(
        'chapter2_everythingChanged',
        chapter2Points.map((item) => item.launchFrequency.commercial),
      ),
      borderColor: palette.orange,
      backgroundColor: 'rgba(252, 122, 30, 0.14)',
      tension: 0.34,
      yAxisID: 'y',
    },
    {
      label: 'Average ticket price (M USD)',
      data: revealLineSeries(
        'chapter2_everythingChanged',
        chapter2Points.map((item) => item.averageTicketPrice),
      ),
      borderColor: palette.sand,
      backgroundColor: 'rgba(249, 199, 132, 0.16)',
      tension: 0.34,
      yAxisID: 'y1',
    },
  ],
}))

const chapter2Options: ChartOptions<'line'> = {
  ...sharedLineOptions,
  scales: {
    x: {
      ticks: { color: '#B9BDD4' },
      grid: { color: 'rgba(72, 86, 150, 0.28)' },
    },
    y: {
      ticks: { color: '#B9BDD4' },
      grid: { color: 'rgba(72, 86, 150, 0.28)' },
      position: 'left',
    },
    y1: {
      ticks: { color: '#B9BDD4' },
      position: 'right',
      grid: { drawOnChartArea: false },
    },
  },
}

const chapter1LineOptions = computed(() => withLineClipProgress(chapterProgress.chapter1_notLongAgo, sharedLineOptions))
const chapter2LineOptions = computed(() =>
  withLineClipProgress(chapterProgress.chapter2_everythingChanged, chapter2Options),
)
const chapter3LineOptions = computed(() => withLineClipProgress(chapterProgress.chapter3_crossedThreshold, sharedLineOptions))
const chapter5LineOptions = computed(() => withLineClipProgress(chapterProgress.chapter5_whatsNext, sharedLineOptions))

const chapter3ChartData = computed(() => ({
  labels: chapter3Points.map((item) => String(item.year)),
  datasets: [
    {
      label: 'Total people reaching orbit',
      data: revealLineSeries(
        'chapter3_crossedThreshold',
        chapter3Points.map((item) => item.peopleReachingOrbit.total),
      ),
      borderColor: palette.indigo,
      backgroundColor: 'rgba(72, 86, 150, 0.24)',
      fill: true,
      tension: 0.31,
      pointRadius: 2,
    },
  ],
}))

const chapter4CategoryNames = chapter4Points[0]?.commercialMissionActivity.map((item) => item.name) ?? []

const chapter4ChartData = computed(() => ({
  labels: chapter4CategoryNames,
  datasets: chapter4Points.map((point, index) => ({
    label: String(point.year),
    data: revealBarSeriesAtProgress(
      barChapterProgress.value,
      point.commercialMissionActivity.map((activity) => activity.value),
    ),
    borderRadius: 4,
    backgroundColor: ['rgba(72, 86, 150, 0.78)', 'rgba(252, 122, 30, 0.74)', 'rgba(249, 199, 132, 0.76)'][
      index
    ],
  })),
}))

const chapter4Options: ChartOptions<'bar'> = {
  responsive: true,
  maintainAspectRatio: false,
  animation: {
    duration: 1000,
    easing: 'easeOutQuart',
  },
  plugins: {
    legend: {
      labels: { color: palette.light },
    },
  },
  scales: {
    x: {
      ticks: { color: '#B9BDD4' },
      grid: { color: 'rgba(72, 86, 150, 0.28)' },
    },
    y: {
      ticks: { color: '#B9BDD4' },
      grid: { color: 'rgba(72, 86, 150, 0.28)' },
    },
  },
}

const chapter5ChartData = computed(() => ({
  labels: chapter5Points.map((item) => String(item.year)),
  datasets: [
    {
      label: 'Projected people reaching orbit',
      data: revealLineSeries(
        'chapter5_whatsNext',
        chapter5Points.map((item) => item.peopleReachingOrbit.total),
      ),
      borderColor: palette.sand,
      backgroundColor: 'rgba(249, 199, 132, 0.15)',
      fill: true,
      tension: 0.3,
    },
    {
      label: 'Projected commercial launches',
      data: revealLineSeries(
        'chapter5_whatsNext',
        chapter5Points.map((item) => item.launchFrequency.commercial),
      ),
      borderColor: palette.ember,
      backgroundColor: 'rgba(242, 76, 0, 0.12)',
      fill: true,
      tension: 0.3,
    },
  ],
}))

const setSectionRef =
  (chapterId: ChapterId) => (el: Element | ComponentPublicInstance | null) => {
    if (!(el instanceof HTMLElement)) {
      chapterElements.delete(chapterId)
      return
    }

    chapterElements.set(chapterId, el)
  }

const clamp = (value: number, min = 0, max = 1) => Math.min(max, Math.max(min, value))

const computeSectionProgress = (element: HTMLElement) => {
  const rect = element.getBoundingClientRect()
  const viewportCenter = window.innerHeight * 0.5
  const sectionCenter = rect.top + rect.height * 0.5
  const startDistance = window.innerHeight * 0.48
  const distanceToCenter = sectionCenter - viewportCenter
  const raw = 1 - distanceToCenter / startDistance
  return clamp(raw)
}

const startBarTween = () => {
  if (barTweenStarted) {
    return
  }

  barTweenStarted = true
  const start = performance.now()
  const duration = 760

  const tick = (time: number) => {
    const progress = Math.min((time - start) / duration, 1)
    barChapterProgress.value = easeOutCubic(progress)

    if (progress < 1) {
      barTweenRafId = requestAnimationFrame(tick)
    }
  }

  barTweenRafId = requestAnimationFrame(tick)
}

const updateScrollDrivenState = () => {
  pendingScrollFrame = false

  let bestChapter: ChapterId = activeChapter.value
  let closestDistance = Number.POSITIVE_INFINITY

  sectionOrder.forEach((chapterId) => {
    const element = chapterElements.get(chapterId)
    if (!element) {
      return
    }

    const progress = computeSectionProgress(element)
    chapterProgress[chapterId] = Math.max(chapterProgress[chapterId], progress)
    visibleChapters[chapterId] = progress > 0.16

    element.style.setProperty('--chapter-progress', chapterProgress[chapterId].toFixed(4))

    const rect = element.getBoundingClientRect()
    const sectionCenter = rect.top + rect.height * 0.5
    const distance = Math.abs(sectionCenter - window.innerHeight * 0.5)

    if (distance < closestDistance) {
      closestDistance = distance
      bestChapter = chapterId
    }

    if (chapterId === 'chapter4_expandingPossibilities' && progress >= 0.3) {
      startBarTween()
    }
  })

  activeChapter.value = bestChapter
}

const scheduleScrollUpdate = () => {
  if (pendingScrollFrame) {
    return
  }

  pendingScrollFrame = true
  scrollRafId = requestAnimationFrame(updateScrollDrivenState)
}

onMounted(() => {
  window.addEventListener('scroll', scheduleScrollUpdate, { passive: true })
  window.addEventListener('resize', scheduleScrollUpdate)
  scheduleScrollUpdate()
})

onBeforeUnmount(() => {
  window.removeEventListener('scroll', scheduleScrollUpdate)
  window.removeEventListener('resize', scheduleScrollUpdate)
  if (scrollRafId !== null) {
    cancelAnimationFrame(scrollRafId)
  }
  if (barTweenRafId !== null) {
    cancelAnimationFrame(barTweenRafId)
  }
})

const formatNumber = (value: number) => new Intl.NumberFormat('en-US').format(value)
</script>

<template>
  <div class="story-page">
    <header class="hero-block section-gap-xl">
      <div class="content-shell hero-content">
        <img class="hero-logo reveal" style="--d: 20ms" src="../images/MS_logo.png" alt="Meridian Story logo" />
        <h1 class="reveal" style="--d: 140ms">{{ heroTitle }}</h1>
        <p class="hero-theme reveal" style="--d: 240ms">{{ heroTheme }}</p>
        <p class="hero-subtext reveal" style="--d: 340ms">
          A visual story about how space quietly became routine.
        </p>
      </div>
    </header>

    <main>
      <section
        :ref="setSectionRef('chapter1_notLongAgo')"
        data-chapter="chapter1_notLongAgo"
        class="story-chapter section-gap-lg chapter-intro bg-orbit-dawn"
        :class="{ 'is-visible': visibleChapters.chapter1_notLongAgo, 'is-active': activeChapter === 'chapter1_notLongAgo' }"
      >
        <div class="chapter-background">
        </div>
        <div class="content-shell chapter-layout chapter1-layout">
          <div class="chapter-copy reveal" style="--d: 80ms">
            <p class="chapter-kicker">Chapter 1 - Not Long Ago</p>
            <h2>Space was extraordinary.</h2>
            <p class="chapter-message">
              Access to orbit remained rare, expensive, and concentrated in government programs.
            </p>
          </div>
          <div class="opening-stat reveal" style="--d: 220ms">
            <p class="impact-label">People reaching orbit in 2005</p>
            <strong class="opening-value">{{ formatNumber(animatedValues.c1People2005) }}</strong>
            <div class="chapter1-supporting-metrics">
              <span>Annual launches: {{ formatNumber(animatedValues.c1Launches2005) }}</span>
              <span>Average ticket price: {{ animatedValues.c1Ticket2005 }}M USD</span>
            </div>
          </div>
          <div class="opening-chart reveal" style="--d: 320ms">
            <Line :data="chapter1ChartData" :options="chapter1LineOptions" />
          </div>
        </div>
      </section>

      <section
        :ref="setSectionRef('chapter2_everythingChanged')"
        data-chapter="chapter2_everythingChanged"
        class="story-chapter section-gap-xl chapter-visual bg-engine-trails"
        :class="{ 'is-visible': visibleChapters.chapter2_everythingChanged, 'is-active': activeChapter === 'chapter2_everythingChanged' }"
      >
        <div class="chapter-background">
          <div class="placeholder-label">cinematic background placeholder</div>
        </div>
        <div class="content-shell chapter-layout">
          <div class="chapter-head reveal" style="--d: 70ms">
            <p class="chapter-kicker">Chapter 2 - Everything Changed</p>
            <h2>The curve bent fast.</h2>
            <p class="chapter-message">
              Commercial launch cadence climbed as ticket prices dropped, changing the trajectory of the entire
              industry.
            </p>
          </div>
          <div class="wide-chart-frame reveal" style="--d: 210ms">
            <Line :data="chapter2ChartData" :options="chapter2LineOptions" />
          </div>
          <div class="inline-facts reveal" style="--d: 330ms">
            <span>Commercial launches: {{ chapter2LaunchMultiple }}x increase</span>
            <span>Ticket price: {{ chapter2PriceLowerPercent }}% lower</span>
          </div>
        </div>
      </section>

      <section
        :ref="setSectionRef('chapter3_crossedThreshold')"
        data-chapter="chapter3_crossedThreshold"
        class="story-chapter section-gap-md chapter-threshold bg-traffic-lines"
        :class="{ 'is-visible': visibleChapters.chapter3_crossedThreshold, 'is-active': activeChapter === 'chapter3_crossedThreshold' }"
      >
        <div class="chapter-background">
          <div class="placeholder-label">cinematic background placeholder</div>
        </div>
        <div class="content-shell chapter-layout threshold-layout">
          <p class="chapter-kicker reveal" style="--d: 70ms">Chapter 3 - We've Crossed a Threshold</p>
          <h2 class="reveal" style="--d: 170ms">The transformation is already happening.</h2>
          <div class="hero-number reveal" style="--d: 300ms">
            {{ formatNumber(animatedValues.c3People2025) }}
          </div>
          <p class="hero-number-caption reveal" style="--d: 390ms">People reached orbit in a single year.</p>
          <p class="threshold-detail reveal" style="--d: 470ms">
            {{ animatedValues.c3CommercialShare }}% of launches in that year were commercial.
          </p>
          <div class="threshold-chart reveal" style="--d: 560ms">
            <Line :data="chapter3ChartData" :options="chapter3LineOptions" />
          </div>
        </div>
      </section>

      <section
        :ref="setSectionRef('chapter4_expandingPossibilities')"
        data-chapter="chapter4_expandingPossibilities"
        class="story-chapter section-gap-xl chapter-ecosystem bg-orbital-grid"
        :class="{ 'is-visible': visibleChapters.chapter4_expandingPossibilities, 'is-active': activeChapter === 'chapter4_expandingPossibilities' }"
      >
        <div class="chapter-background">
          <div class="placeholder-label">cinematic background placeholder</div>
        </div>
        <div class="content-shell chapter-layout ecosystem-layout">
          <div class="ecosystem-copy reveal" style="--d: 80ms">
            <p class="chapter-kicker">Chapter 4 - Expanding Possibilities</p>
            <h2>Space is becoming an economy.</h2>
            <p class="chapter-message">
              The destination did not change. The reasons for going did.
            </p>
            <p class="chapter-message ecosystem-secondary-copy">
              Commercial launches now support tourism, research, cargo, manufacturing, medicine, and infrastructure.
            </p>
            <p class="ecosystem-fact">
              Infrastructure missions by 2030: <strong>{{ animatedValues.c4Infra2030 }}</strong>
            </p>
            <p class="ecosystem-fact">
              Medical missions by 2030: <strong>{{ animatedValues.c4Medical2030 }}</strong>
            </p>
          </div>
          <div class="ecosystem-chart reveal" style="--d: 220ms">
            <Bar :data="chapter4ChartData" :options="chapter4Options" />
          </div>
        </div>
      </section>

      <section
        :ref="setSectionRef('chapter5_whatsNext')"
        data-chapter="chapter5_whatsNext"
        class="story-chapter section-gap-lg chapter-finale bg-horizon"
        :class="{ 'is-visible': visibleChapters.chapter5_whatsNext, 'is-active': activeChapter === 'chapter5_whatsNext' }"
      >
        <div class="chapter-background">
          <div class="placeholder-label">cinematic background placeholder</div>
        </div>
        <div class="content-shell chapter-layout finale-layout">
          <div class="finale-text reveal" style="--d: 80ms">
            <p class="chapter-kicker">Chapter 5 - What's Next?</p>
            <h2>Possibility, not prediction.</h2>
            <p class="chapter-message">
              Routine access doesn't tell us exactly what comes next.
            </p>
            <p class="chapter-message finale-secondary-copy">
              It changes what becomes plausible.
            </p>
          </div>
          <div class="finale-metrics reveal" style="--d: 210ms">
            <div>
              <span>Projected people in orbit by 2032</span>
              <strong>{{ formatNumber(animatedValues.c5People2032) }}</strong>
            </div>
            <div>
              <span>Projected average ticket price</span>
              <strong>{{ animatedValues.c5Ticket2032 }}M USD</strong>
            </div>
          </div>
          <div class="finale-chart reveal" style="--d: 330ms">
            <Line :data="chapter5ChartData" :options="chapter5LineOptions" />
          </div>
        </div>
      </section>

      <section class="story-chapter story-epilogue section-gap-xl">
        <div class="content-shell epilogue-content">
          <h2 class="epilogue-headline">Every revolution begins by becoming ordinary.</h2>
          <p class="epilogue-copy">
            Once something becomes routine, people stop asking if it's possible.
          </p>
          <p class="epilogue-copy">
            They begin asking what they can build with it.
          </p>
          <p class="epilogue-copy">Commercial spaceflight may be entering that moment now.</p>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
:global(body) {
  margin: 0;
  background: #050912;
  color: #e7eef9;
}

.story-page {
  --c-indigo: #485696;
  --c-light: #e7e7e7;
  --c-sand: #f9c784;
  --c-orange: #fc7a1e;
  --c-ember: #f24c00;
  --c-body: #cfd3e2;
  color: var(--c-light);
  background: radial-gradient(circle at 15% 0%, rgba(72, 86, 150, 0.26), transparent 44%),
    radial-gradient(circle at 80% 100%, rgba(242, 76, 0, 0.12), transparent 48%), #050912;
}

.content-shell {
  width: min(1120px, calc(100% - 3rem));
  margin: 0 auto;
}

.section-gap-xl {
  padding: 7rem 0;
}

.section-gap-lg {
  padding: 5.8rem 0;
}

.section-gap-md {
  padding: 4.7rem 0;
}

.hero-block {
  min-height: 92svh;
  display: flex;
  align-items: center;
  background: linear-gradient(180deg, rgba(5, 9, 18, 0.46), rgba(5, 9, 18, 0.64)),
    linear-gradient(90deg, rgba(5, 9, 18, 0.42), rgba(5, 9, 18, 0.06)),
    url('../images/hero.jpg');
  background-size: cover;
  background-position: center center;
  background-repeat: no-repeat;
}

.hero-content {
  max-width: 920px;
}

.hero-logo {
  width: clamp(170px, 18vw, 280px);
  height: auto;
  display: block;
  margin-bottom: 1.1rem;
}

.hero-kicker,
.chapter-kicker {
  margin: 0;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  font-size: 0.76rem;
  color: #c8b185;
}

h1,
h2 {
  margin: 2.76rem 0 0;
  line-height: 1.04;
  letter-spacing: -0.02em;
  font-family: Georgia, 'Times New Roman', serif;
}

h1 {
  max-width: 11ch;
  font-size: clamp(2.5rem, 7vw, 5.5rem);
}

h2 {
  max-width: 18ch;
  font-size: clamp(2rem, 4.4vw, 3.55rem);
}

.hero-theme {
  margin: 1rem 0 0;
  max-width: 55ch;
  font-size: clamp(1.1rem, 2.1vw, 1.55rem);
  color: var(--c-light);
}

.hero-subtext,
.chapter-message {
  margin: 1.2rem 0 0;
  max-width: 60ch;
  line-height: 1.7;
  color: var(--c-body);
}

.story-chapter {
  position: relative;
  min-height: 100svh;
  border-top: 1px solid rgba(72, 86, 150, 0.36);
  overflow: clip;
}

.chapter-layout {
  position: relative;
  z-index: 2;
}

.chapter-background {
  position: absolute;
  inset: 0;
  opacity: 0.72;
}

.chapter-background::before {
  content: '';
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(5, 9, 18, 0.52), rgba(5, 9, 18, 0.9));
}

.placeholder-label {
  position: absolute;
  top: 1rem;
  right: 1rem;
  font-size: 0.68rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: rgba(231, 231, 231, 0.78);
  border: 1px dashed rgba(249, 199, 132, 0.48);
  border-radius: 999px;
  padding: 0.35rem 0.65rem;
}

.reveal {
  --reveal-y: 34px;
  --reveal-scale: 0.982;
  opacity: 0;
  transform: translate3d(0, var(--reveal-y), 0) scale(var(--reveal-scale));
  filter: blur(1.25px);
  transition: opacity 820ms cubic-bezier(0.2, 0.8, 0.2, 1),
    transform 820ms cubic-bezier(0.2, 0.8, 0.2, 1),
    filter 820ms cubic-bezier(0.2, 0.8, 0.2, 1);
  transition-delay: var(--d, 0ms);
  will-change: transform, opacity, filter;
}

.story-chapter.is-visible .reveal,
.hero-content .reveal {
  opacity: 1;
  transform: translate3d(0, 0, 0) scale(1);
  filter: blur(0);
}

.chapter-intro .reveal {
  --reveal-y: 30px;
  transition-duration: 800ms;
}

.chapter-visual .reveal {
  --reveal-y: 38px;
  transition-duration: 860ms;
}

.chapter-threshold .reveal {
  --reveal-y: 44px;
  transition-duration: 940ms;
}

.chapter-ecosystem .reveal {
  --reveal-y: 34px;
  transition-duration: 830ms;
}

.chapter-finale .reveal {
  --reveal-y: 32px;
  transition-duration: 900ms;
}

.impact-panel.reveal,
.wide-chart-frame.reveal,
.ecosystem-chart.reveal,
.finale-metrics.reveal,
.finale-chart.reveal {
  --reveal-y: 56px;
  --reveal-scale: 0.968;
  transition-duration: 1020ms;
}

.split-layout {
  min-height: 74svh;
  display: grid;
  grid-template-columns: 1.1fr 0.9fr;
  gap: 2.4rem;
  align-items: center;
}

.chapter1-layout {
  min-height: 80svh;
  display: grid;
  grid-template-columns: 1fr;
  align-content: center;
  justify-items: center;
  text-align: center;
  gap: 1.6rem;
}

.chapter1-layout .chapter-copy {
  max-width: 760px;
  display: grid;
  justify-items: center;
}

.chapter1-layout .chapter-copy h2 {
  max-width: 11ch;
}

.chapter1-layout .chapter-message {
  max-width: 48ch;
}

.opening-stat {
  padding: 0.2rem 0;
}

.opening-value {
  display: block;
  margin-top: 0.3rem;
  font-size: clamp(4.6rem, 12vw, 8.6rem);
  line-height: 0.9;
  color: var(--c-sand);
  letter-spacing: -0.03em;
}

.opening-stat .impact-label,
.opening-stat .impact-note {
  text-align: center;
}

.opening-chart {
  height: 250px;
  width: min(780px, 100%);
  border: 1px solid rgba(72, 86, 150, 0.42);
  border-radius: 16px;
  background: rgba(18, 24, 44, 0.64);
  padding: 0.9rem;
}

.impact-panel {
  border: 1px solid rgba(151, 178, 219, 0.28);
  border-radius: 16px;
  padding: 1.4rem;
  background: rgba(8, 14, 27, 0.68);
}

.impact-label,
.impact-note {
  margin: 0;
  color: #c5c8d6;
}

.impact-panel strong {
  display: block;
  margin-top: 0.55rem;
  font-size: clamp(2.5rem, 7.5vw, 5rem);
  line-height: 0.95;
  color: #f0f5ff;
}

.impact-note {
  margin-top: 0.65rem;
}

.mini-chart {
  margin-top: 1rem;
  height: 190px;
}

.chapter-visual .chapter-layout {
  display: grid;
  gap: 1.5rem;
  min-height: 78svh;
  align-content: center;
}

.chapter-head {
  max-width: 70ch;
}

.wide-chart-frame {
  height: 380px;
  border: 1px solid rgba(72, 86, 150, 0.42);
  background: rgba(18, 24, 44, 0.68);
  border-radius: 20px;
  padding: 1rem;
}

.inline-facts {
  display: flex;
  gap: 1.5rem;
  flex-wrap: wrap;
  color: var(--c-light);
}

.inline-facts span {
  border-bottom: 1px solid rgba(249, 199, 132, 0.5);
  padding-bottom: 0.35rem;
}

.threshold-layout {
  min-height: 74svh;
  text-align: center;
  display: grid;
  justify-items: center;
  align-content: center;
  row-gap: 0.65rem;
}

.chapter-threshold {
  display: flex;
  align-items: center;
  padding-top: 0;
  padding-bottom: 0;
}

.chapter-threshold .threshold-layout {
  min-height: auto;
  width: 100%;
}

.threshold-layout h2 {
  max-width: 14ch;
}

.hero-number {
  margin-top: 0.25rem;
  font-family: Georgia, 'Times New Roman', serif;
  font-size: clamp(4rem, 13vw, 9.8rem);
  line-height: 0.9;
  letter-spacing: -0.03em;
}

.hero-number-caption,
.threshold-detail {
  margin: 0;
  color: var(--c-light);
}

.threshold-chart {
  margin-top: 0.35rem;
  width: min(760px, 100%);
  height: 215px;
}

.ecosystem-layout {
  min-height: 84svh;
  display: grid;
  grid-template-columns: 0.95fr 1.05fr;
  gap: 2rem;
  align-items: center;
}

.ecosystem-fact {
  margin: 1rem 0 0;
  color: #d5d2ca;
}

.ecosystem-secondary-copy {
  margin-top: 0.8rem;
}

.ecosystem-fact strong {
  color: var(--c-sand);
  font-size: 1.25rem;
}

.ecosystem-chart {
  height: 390px;
  border: 1px solid rgba(72, 86, 150, 0.42);
  border-radius: 18px;
  background: rgba(18, 24, 44, 0.68);
  padding: 1rem;
}

.finale-layout {
  min-height: 78svh;
  display: grid;
  gap: 1.4rem;
  align-content: center;
}

.finale-metrics {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
}

.finale-metrics > div {
  border: 1px solid rgba(72, 86, 150, 0.42);
  background: rgba(18, 24, 44, 0.68);
  border-radius: 14px;
  padding: 1rem;
}

.finale-metrics span {
  display: block;
  color: #d0d2db;
}

.finale-metrics strong {
  display: block;
  margin-top: 0.45rem;
  font-size: clamp(1.9rem, 4.6vw, 3.2rem);
  line-height: 0.95;
}

.finale-chart {
  height: 250px;
  border: 1px solid rgba(72, 86, 150, 0.42);
  border-radius: 14px;
  background: rgba(18, 24, 44, 0.68);
  padding: 0.9rem;
}

.finale-secondary-copy {
  margin-top: 0.35rem;
}

.chapter1-supporting-metrics {
  margin-top: 0.65rem;
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  justify-content: center;
  color: #d1d3dd;
  font-size: 0.94rem;
}

.story-epilogue {
  min-height: 100svh;
  display: flex;
  align-items: center;
  border-top: 1px solid rgba(72, 86, 150, 0.35);
  background: linear-gradient(180deg, rgba(5, 9, 18, 0.98), rgba(6, 11, 20, 1));
}

.epilogue-content {
  max-width: 820px;
  text-align: center;
  display: grid;
  gap: 0.7rem;
}

.epilogue-headline {
  margin: 0;
  font-size: clamp(2.1rem, 4.6vw, 4.3rem);
  max-width: 18ch;
  justify-self: center;
}

.epilogue-copy {
  margin: 0;
  color: #d5d7df;
  line-height: 1.75;
}

.bg-orbit-dawn {
  background: linear-gradient(132deg, rgba(72, 86, 150, 0.72), rgba(16, 22, 42, 0.92));
}

.bg-orbit-dawn .chapter-background {
  background-image: radial-gradient(circle at 80% 20%, rgba(249, 199, 132, 0.1), rgba(249, 199, 132, 0) 34%),
    linear-gradient(132deg, rgba(10, 16, 31, 0.1), rgba(6, 12, 24, 0.26)), url('../images/chapter1.jpg');
  background-size: cover;
  background-position: center center;
  background-repeat: no-repeat;
}

.bg-orbit-dawn .chapter-background::before {
  background: linear-gradient(180deg, rgba(5, 9, 18, 0.18), rgba(5, 9, 18, 0.38));
}

.bg-engine-trails {
  background: repeating-linear-gradient(
      114deg,
      rgba(72, 86, 150, 0.2) 0,
      rgba(72, 86, 150, 0.2) 11px,
      rgba(6, 12, 24, 0) 11px,
      rgba(6, 12, 24, 0) 24px
    ),
    linear-gradient(145deg, rgba(25, 33, 58, 0.9), rgba(11, 20, 36, 0.95));
}

.bg-traffic-lines {
  background: linear-gradient(120deg, rgba(28, 42, 79, 0.9), rgba(10, 19, 35, 0.95));
}

.bg-traffic-lines::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image: radial-gradient(circle at 20% 30%, rgba(252, 122, 30, 0.2), transparent 38%),
    radial-gradient(circle at 75% 66%, rgba(249, 199, 132, 0.17), transparent 44%);
}

.bg-orbital-grid {
  background: linear-gradient(135deg, rgba(27, 39, 71, 0.9), rgba(9, 18, 33, 0.95));
}

.bg-orbital-grid::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image: linear-gradient(rgba(72, 86, 150, 0.25) 1px, transparent 1px),
    linear-gradient(90deg, rgba(72, 86, 150, 0.25) 1px, transparent 1px);
  background-size: 38px 38px;
}

.bg-horizon {
  background: radial-gradient(circle at 50% 100%, rgba(242, 76, 0, 0.33), rgba(8, 14, 26, 0) 46%),
    linear-gradient(180deg, rgba(28, 41, 76, 0.9), rgba(7, 12, 22, 0.98));
}

@media (max-width: 980px) {
  .content-shell {
    width: min(1120px, calc(100% - 2rem));
  }

  .section-gap-xl {
    padding: 5.4rem 0;
  }

  .section-gap-lg {
    padding: 4.8rem 0;
  }

  .section-gap-md {
    padding: 4rem 0;
  }

  .story-chapter {
    min-height: auto;
  }

  .chapter-threshold {
    padding-top: 4rem;
    padding-bottom: 4rem;
  }

  .split-layout,
  .ecosystem-layout {
    grid-template-columns: 1fr;
  }

  .chapter1-layout {
    min-height: auto;
    align-content: start;
  }

  .wide-chart-frame,
  .ecosystem-chart {
    height: 320px;
  }

  .opening-chart {
    height: 220px;
  }

  .chapter-visual .chapter-layout {
    min-height: auto;
    align-content: start;
  }

  .hero-number {
    font-size: clamp(3.4rem, 18vw, 7.2rem);
  }

  .finale-metrics {
    grid-template-columns: 1fr;
  }

  .chapter1-supporting-metrics {
    flex-direction: column;
    gap: 0.35rem;
    align-items: center;
  }
}

@media (prefers-reduced-motion: reduce) {
  .reveal {
    transition: none;
    transform: none;
    opacity: 1;
    filter: none;
  }
}
</style>

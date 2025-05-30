<script setup>
  import EmotionLevelChart from '@/components/chart/EmotionLevelChart.vue'
  import EmotionCountChart from '@/components/chart/EmotionCountChart.vue'
  import HeadlineText from '@/components/text/HeadlineText.vue'
  import ActionButton from '@/components/button/ActionButton.vue'
  import DropDown from '@/components/dropdown/DropDown.vue'
  import { onMounted, ref } from 'vue'
  import { dailyChartStore } from '@/stores/chart'

  const charts = ref([])
  const chartStore = dailyChartStore()

  const labels = ref([
    '일', '월', '화', '수', '목', '금', '토',
  ])

  const barColor = '#FF3B3080'
  const lineColor = '#FF3B30'
  const averageLineColor = '#FF3B3080'

  const datasets = ref([
    {
      type: 'bar',
      label: '하루의 평균',
      data: Array(7).fill(null),
      backgroundColor: barColor,
      borderColor: barColor,
      borderWidth: 1,
    },
  ])

  const emotionDatasets = ref([
    {
      type: 'bar',
      label: '감정의 정도',
      data: Array(7).fill(null),
      backgroundColor: barColor,
      borderColor: barColor,
      borderWidth: 1,
    },
  ])

  const countDatasets = ref([])

  const getDayIndex = (dateString) => {
    const date = new Date(dateString)
    const day = date.getDay()
    return day
  }

  const formatDateToCustomString = (date) => {
    const year = String(date.getFullYear())
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    const hours = String(date.getHours()).padStart(2, '0')
    const minutes = String(date.getMinutes()).padStart(2, '0')
    const seconds = String(date.getSeconds()).padStart(2, '0')
    return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
  }

  const searchDate = ref(formatDateToCustomString(new Date()))

  const fetchData = async () => {
    try {
      console.log('Fetching data with parameters:', {
        period: 'weekly',
        searchDate: searchDate.value,
      })

      datasets.value[0].data = Array(7).fill(null)
      emotionDatasets.value[0].data = Array(7).fill(null)
      countDatasets.value = []

      const period = 'weekly'
      await chartStore.getAllCharts(period, searchDate.value)
      await chartStore.getAllCounts(period, searchDate.value)

      charts.value = chartStore.dailyCharts
      const counts = chartStore.countCharts

      console.log('Fetched dailyCharts:', charts.value)
      console.log('Fetched countCharts:', counts)

      const dailyAverages = Array(7).fill(null)
      const emotionLevels = Array(7).fill(null)
      const mainCounts = Array(7).fill(null)
      const momentCounts = Array(7).fill().map(() => [])

      chartStore.dailyCharts.forEach(chart => {
        const dayIndex = getDayIndex(chart.createdTime)
        dailyAverages[dayIndex] = chart.dailyAverage
        emotionLevels[dayIndex] = chart.dailyAverage
      })

      counts.forEach(count => {
        const dayIndex = getDayIndex(count.createdTime)
        if (count.emotionRecordState === 'main' && count.emotionRecordLevel > 0) {
          mainCounts[dayIndex] = count.emotionRecordLevel
        } else if (count.emotionRecordState === 'moment' && count.emotionRecordLevel > 0) {
          momentCounts[dayIndex].push({ x: labels.value[dayIndex], y: count.emotionRecordLevel })
        }
      })

      datasets.value[0].data = dailyAverages
      emotionDatasets.value[0].data = emotionLevels

      console.log('Updated dailyAverages:', dailyAverages)
      console.log('Updated emotionLevels:', emotionLevels)
      console.log('Updated mainCounts:', mainCounts)
      console.log('Updated momentCounts:', momentCounts)

      const mainData = mainCounts.map((count, index) => count !== null ? { x: labels.value[index], y: count } : null).filter(item => item !== null)
      const momentData = momentCounts.flat().filter(item => item !== null)

      const countDatasetsTemp = []
      countDatasetsTemp.push({
        type: 'scatter',
        label: '오늘의 감정',
        data: mainData.length > 0 ? mainData : [{ x: '', y: null }],
        backgroundColor: '#FF3B30',
        borderColor: '#FF3B30',
        borderWidth: 1,
      })

      countDatasetsTemp.push({
        type: 'scatter',
        label: '순간의 감정',
        data: momentData.length > 0 ? momentData : [{ x: '', y: null }],
        backgroundColor: 'rgba(128, 128, 128, 0.6)',
        borderColor: 'rgba(128, 128, 128, 0.6)',
        borderWidth: 1,
      })

      countDatasets.value = countDatasetsTemp
    } catch (error) {
      console.error('Error fetching data:', error)
      if (error.response) {
        console.error('Response data:', error.response.data)
      }
    }
  }

  const formatDate = (date) => {
    const year = date.getFullYear()
    const month = String(date.getMonth() + 1).padStart(2, '0')
    const day = String(date.getDate()).padStart(2, '0')
    return `${year}.${month}.${day}`
  }

  const getWeekRange = (date) => {
    const day = date.getDay()
    const diffToSunday = date.getDate() - day
    const sunday = new Date(date)
    sunday.setDate(diffToSunday)
    const saturday = new Date(sunday)
    saturday.setDate(sunday.getDate() + 6)
    return { sunday, saturday }
  }

  const currentDate = ref(new Date())
  const updateWeekRange = () => {
    const { sunday, saturday } = getWeekRange(currentDate.value)
    const formattedSunday = formatDate(sunday)
    const formattedSaturday = formatDate(saturday)
    dateRange.value = `${formattedSunday} - ${formattedSaturday}`
  }

  const dateRange = ref('')
  updateWeekRange()

  const changeWeek = async (direction) => {
    currentDate.value.setDate(currentDate.value.getDate() + direction * 7)
    searchDate.value = formatDateToCustomString(currentDate.value)
    updateWeekRange()
    await fetchData()
  }

  const valueToLabel = (value) => {
    const labels = [
      '매우 불쾌함', '불쾌함', '약간 불쾌함',
      '보통', '약간 기분 좋음', '기분 좋음', '매우 기분 좋음',
    ]
    return labels[value - 1] || ''
  }

  const emotionChartOptions = ref({
    scales: {
      y: {
        beginAtZero: true,
        max: 8,
        title: {
          display: true,
        },
        ticks: {
          stepSize: 1,
          callback: valueToLabel,
        },
      },
      x: {
        title: {
          display: true,
        },
      },
    },
    plugins: {
      legend: {
        display: true,
      },
    },
  })

  const countChartOptions = ref({
    scales: {
      y: {
        beginAtZero: true,
        max: 8,
        title: {
          display: true,
        },
        ticks: {
          stepSize: 1,
          callback: valueToLabel,
        },
      },
      x: {
        title: {
          display: true,
        },
        ticks: {
          align: 'center',
        },
      },
    },
    plugins: {
      legend: {
        display: true,
      },
    },
  })

  const isDropdownVisible = ref(false)

  const toggleDropdown = () => {
    isDropdownVisible.value = !isDropdownVisible.value
  }

  onMounted(async () => {
    await fetchData()
  })
</script>

<template>
  <div class="layout">
    <section class="main-area">
      <div class="main-title">
        <HeadlineText
          :size="'l'"
          :text="'감정 기록 통계'"
          :padding="[48, 0, 0, 0]"
        />
      </div>
      <div class="button-area">
        <div class="date">
          {{ dateRange }}
        </div>
        <div class="buttons">
          <ActionButton
            :variant="'sub'"
            :state="'secondary'"
            :icon="'chevron_left'"
            :style="{
              'border-top-right-radius': 0,
              'border-bottom-right-radius': 0,
              'border-right': 'none'
            }"
            @click="() => changeWeek(-1)"
          />
          <ActionButton
            :variant="'sub'"
            :state="'secondary'"
            :icon="'chevron_right'"
            :style="{
              'border-top-left-radius': 0,
              'border-bottom-left-radius': 0
            }"
            @click="() => changeWeek(1)"
          />
          <ActionButton
            :variant="'sub'"
            :state="'secondary'"
            :icon="'sort'"
            :text="'Weekly'"
            @click="toggleDropdown"
          />
          <DropDown
            v-if="isDropdownVisible"
            class="dropdown"
          />
        </div>
      </div>
      <div class="chart-area">
        <EmotionLevelChart
          :labels="labels"
          :datasets="emotionDatasets"
          :chart-options="emotionChartOptions"
          :bar-color="barColor"
          :line-color="lineColor"
          y-axis-title="감정의 정도"
          x-axis-title="요일"
          :average-line-color="averageLineColor"
        />
      </div>
      <div class="chart-area">
        <EmotionCountChart
          :labels="labels"
          :datasets="countDatasets"
          :chart-options="countChartOptions"
          :bar-color="barColor"
          :line-color="lineColor"
          y-axis-title="감정의 정도"
          x-axis-title="요일"
          :average-line-color="averageLineColor"
        />
      </div>
    </section>
  </div>
</template>

<style lang="scss" scoped>
.layout {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  width: 100vw;
  background-color: $lightgray-background;

  .main-area {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
    padding: 2rem;
    width: calc(100vw - 18.75rem);
    position: relative;
  }

  .main-title {
    width: 100%;
    max-width: 50rem;
    display: flex;
    justify-content: flex-start;
    flex-direction: column;
    padding-left: 0.25rem;
  }

  .button-area {
    display: flex;
    flex-direction: row;
    width: 100%;
    max-width: 50rem;
    align-items: center;
    padding-top: 15px;
    padding-left: 8px;
    justify-content: space-between;
    position: relative;
    z-index: 20;
  }

  .date {
    display: flex;
  }

  .buttons {
    display: flex;
    flex-direction: row;
  }

  .dropdown {
    position: absolute;
    top: 100%;
    right: 0;
    z-index: 10;
  }

  .chart-area {
    width: 100%;
    max-width: 50rem;
    height: 37vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 20px 20px 10px 20px;
    border: 1px solid $grayopacity30;
    border-radius: 8px;
    margin: 1rem 1rem 0.2rem 1rem;
    background-color: $white900;
  }

  @media (max-width: 567px) {
    .layout {
      max-width: 100vw;
    }

    .main-area {
      width: 100%;
      margin-left: 0;
      padding: 1rem;
    }

    .chart-area {
      width: 100%;
      max-width: 20rem;
    }
  }
}
</style>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { Bar, Bubble, Line, Scatter, PolarArea } from 'vue-chartjs'
import {
  Chart as ChartJS, Title, Tooltip, Legend, BarElement, PointElement, LineElement, CategoryScale, LinearScale, RadialLinearScale,
  ArcElement,
} from 'chart.js'

import output from './output.json'

type ProcessedPost = {
  id: string;
  createdAt: number;
  updatedAt: number;
  content: string;
  contentFormatted: string;
  commentsCount: number;
  postLikesCount: number;
  spoiler: boolean;
  nsfw: boolean;
  topLevelCommentsCount: number;
  embed: {
    url: string;
    kind: string;
  } | null;
  likeCommentRatio: number;
  isInitiallyHidden: boolean;
  hasImage: boolean;
  containsQuestion: boolean;
  length: number;
  taggedUsersCount: number;
}

ChartJS.register(Title, Tooltip, Legend, LineElement, BarElement, PointElement, CategoryScale, LinearScale, RadialLinearScale, ArcElement)
ChartJS.defaults.color = 'rgb(68,44,60)'
ChartJS.defaults.backgroundColor = 'rgba(231,94,69,255)'
ChartJS.defaults.borderColor = 'rgb(235,242,242)'


const chartOptions = {
  responsive: true,
}

const timeFormatter = {
  scales: {
    x: {
      ticks: {
        callback: function (value: number, index: number, ticks: any) {
          const dateValue = new Date(value);
          return `${dateValue.getUTCHours()}:${dateValue.getUTCMinutes()}`
        }
      }
    }
  }
}

const offset = ref(0)
const smallOutput = computed(() => {
  const result = [...(output as any[])].splice(offset.value, 2000) as Array<ProcessedPost>
  // result.sort((a, b) => b.createdAt.localeCompare(a.createdAt))

  return result
})

const lvsCAData = computed(() => {
  const data = {
    labels: [] as any,
    datasets: [
      {
        label: 'Likes',
        data: [] as any,
        fill: false,
        borderColor: 'rgba(231,94,69,255)',
        tension: 0.1
      }
    ]
  }
  const bins = {} as any// hour to like count array bins

  for (const p of smallOutput.value) {
    const post = p as ProcessedPost

    const hour = new Date(post.createdAt).getUTCHours()

    if (!bins[hour]) bins[hour] = []

    bins[hour].push(post.postLikesCount)
  }

  for (const [hour, bin] of Object.entries(bins)) {
    const averageLikes = (bin as any[]).reduce((acc: number, cur: number) => acc + cur, 0) / (bin as any[]).length
    data.labels.push(hour)
    data.datasets[0].data.push(averageLikes)
  }

  return data
})
const lvsCAOptions = {
  ...chartOptions,
}

const lCvsHIData = {
  datasets: [
    {
      pointBackgroundColor: 'black',
      label: 'Without Image',
      data: [] as any
    },
    {
      label: 'With Image',
      data: [] as any
    }
  ]
}

const lvsCQData = {
  labels: ['No Poll', 'Poll'],
  datasets: [
    {
      data: [0, 0],
      backgroundColor: [
        'black',
        'rgba(231,94,69,255)',
      ]
    }
  ]
}

const lcvsLData = {
  datasets: [{
    label: 'Likes vs Comments w/ Radius = length of post',
    data: [] as any,
  }]
}

const lvsTUData = {
  datasets: [{
    label: 'Likes vs Users Tagged',
    data: [] as any[],
  }]
}

let noQ = 0
let q = 0

let noHide = 0
let hidden = 0

let lcRSum = 0

const lvsIHData = {
  labels: ['Not Hidden', 'Hidden'],
  datasets: [{
    label: 'Likes',
    data: [0, 0]
  }]
}

for (const p of (output as ProcessedPost[])) {
  const post = p as ProcessedPost

  lcRSum += post.likeCommentRatio

  lCvsHIData.datasets[Number(post.hasImage)].data.push({
    y: post.postLikesCount,
    x: post.commentsCount,
  })

  lvsCQData.datasets[0].data[Number(post.containsQuestion)] += post.postLikesCount
  if (post.containsQuestion) q += 1
  else noQ += 1

  lvsIHData.datasets[0].data[Number(post.isInitiallyHidden)] += post.postLikesCount
  if (post.isInitiallyHidden) hidden += 1
  else noHide += 1

  lcvsLData.datasets[0].data.push({
    y: post.postLikesCount,
    x: post.commentsCount,
    r: post.length / 300
  })

  lvsTUData.datasets[0].data.push({
    y: post.postLikesCount,
    x: post.taggedUsersCount,
  })
}

lvsCQData.datasets[0].data[0] /= noQ
lvsCQData.datasets[0].data[1] /= q

lvsIHData.datasets[0].data[0] /= noHide
lvsIHData.datasets[0].data[1] /= hidden

const avgLCR = lcRSum / (output as any[]).length
</script>

<template>
  <header>
    <h1>What Does Kitsu Like?</h1>
    Here are some graphs to visualize 20,000 posts from Kitsu that were posted over the past 3 months. This data was
    gathered with
    a
    small Node
    script that requested 20 posts per request, 10 requests in parallel per batch, 100 batches. After that, I did a little
    bit of processing on each of the posts to add more attributes to each one that we can plot here e.g. whether the post
    has an image, whether it is hidden, the like/comment ratio, etc. Warning: if you're learning how to code, LOOK AWAY.
    This is the filthiest code I've ever
    written, but it's a fun 1-day project to attempt
    to answer some questions that I have:
    <br />
    <br />

    <ol>
      <li>Relationship between like count and whether or not the post has an image</li>
      <li>Relationship between like count and time of day posted</li>
      <li>Calculate average like/comment ratio</li>
    </ol>
    <br />

    Your fellow Kitsu user, <a href="https://kitsu.io/users/lczernel">@lczernel</a>
  </header>

  <main>
    <section>
      <h3>Likes vs. createdAt</h3>
      Does Kitsu like things more at certain times of day? (UTC times btw)
      <input
        v-model="offset"
        type="range"
        id="offset"
        name="offset"
        min="0"
        max="1000"
        style="display: block;"
      />
      <label for="offset">Posts Offset (viewing 2000 posts at a time)</label>
      <figure>
        <Line
          :options="lvsCAOptions"
          :data="lvsCAData"
        >
        </Line>
      </figure>
    </section>
    <section>
      <h3>Likes, comments vs. hasImage</h3>
      Does Kitsu like images (a few other users have noticed this)?
      <figure>
        <Scatter
          :options="chartOptions"
          :data="lCvsHIData"
        >
        </Scatter>
      </figure>
    </section>
    <section>
      <h3>Likes vs. containsQuestion</h3>
      Does Kitsu like polls?
      <figure>
        <PolarArea
          :options="chartOptions"
          :data="lvsCQData"
        >
        </PolarArea>
      </figure>
    </section>
    <section>
      <h3>Likes vs. Post length</h3>
      Does Kitsu like long or short posts?
      <figure>
        <Bubble
          :options="chartOptions"
          :data="lcvsLData"
        >
        </Bubble>
      </figure>
    </section>
    <section>
      <h3>Likes vs. initiallyHidden</h3>
      Does Kitsu like posts that are initially hidden (spoiler or nsfw)?
      <figure>
        <Bar
          :options="chartOptions"
          :data="lvsIHData"
        >

        </Bar>
      </figure>
    </section>
    <section>
      <h3>Likes vs. taggedUserCount</h3>
      Do Kitsu users like being tagged?
      <figure>
        <Scatter
          :options="chartOptions"
          :data="lvsTUData"
        ></Scatter>
      </figure>
    </section>
    <section>
      <h3>Likes:Comments Ratio</h3>
      Do Kitsu users comment more than they like? (this is something I have thought for a loooooong time hehe)
      <figure>
        For every 1 comment there are <span style="color: rgba(231,94,69,255); font-weight: bold; font-size: 20px;">{{
          avgLCR }}</span>
        likes!
        <br />
        <small>*Only counting top-level comments!</small>
      </figure>
    </section>
  </main>
</template>

<style scoped>
main {
  padding: 0 100px;
  padding-top: 20px;
}

section {
  margin-bottom: 50px;
}

h1 {
  margin-bottom: 16px;
}
</style>

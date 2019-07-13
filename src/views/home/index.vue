<template>
  <div class="home">
    <van-nav-bar title="首页" fixed/>
    <!--
      activeChannelIndex 绑定当前激活的标签页，使用索引
      -->
    <van-tabs
      class="channel-tabs"
      v-model="activeChannelIndex"
    >
      <div slot="nav-right" class="nav-right" @click="showPopup = !showPopup">
        <van-icon name="wap-nav" />
      </div>
      <van-tab
      v-for="item in channels"
      :key="item.id"
      :title="item.name"
      >
        <!--
          下拉刷新
          isLoading 用来控制下拉刷新的 loading 状态
          下拉刷新的时候，它会自动将 loading 设置为 true
          @refresh 当下拉刷新的时候会触发
          -->
        <van-pull-refresh v-model="item.downPullLoading" @refresh="onRefresh" :success-text="item.downPullSuccess">
          <!--
            列表组件：主要提供上拉加载更多的功能
            loading 用来控制加载中 loading 状态
            finished 用来控制是否加载完毕
            @load 加载更多的时候触发的一个事件，它自动会调用 onLoad 函数拿数据，以填充页面
                它每次调用 onLoad 会自动将 loading 设置为 true
                我们需要在 onLoad 中拿到本次加载的数据以后，将 loading 设置为 false
          -->
          <van-list
            v-model="item.upPullLoading"
            :finished="item.upPullFinished"
            finished-text="没有更多了"
            @load="onLoad"
          >
            <van-cell
              v-for="articleitem in item.articles"
              :key="articleitem.id"
              :title="articleitem.title"
            />
          </van-list>
        </van-pull-refresh>
      </van-tab>
    </van-tabs>
    <!-- 频道管理组件 -->
    <HomeChannel v-model="showPopup"></HomeChannel>
  </div>

</template>

<script>
import { getUserChannels } from '@/api/channel'
import { getArticles } from '@/api/article'
import HomeChannel from './components/channel'
export default {
  name: 'HomeIndex',
  components: {
    HomeChannel 
  },
  data () {
    return {
      activeChannelIndex: 0,
      list: [],
      loading: false,
      finished: false,
      isLoading: false,
      channels: [],
      showPopup: false // 控制遮罩层
    }
  },
  created () {
    this.loadChannels()
  },

  computed: {
    // 当前激活的频道
    activeChannel () {
      return this.channels[this.activeChannelIndex]
    }
  },

  // 用来监视user的状态，状态改变就重新获取数据
  watch: {
    '$store.state.user' () {
      // console.log(123)
      this.loadChannels()
      // 上拉加载更多设置为true，就会自动去调用 onload 事件
      this.activeChannel.upPullLoading = true
    }
  },

  methods: {
    // 获取当前频道的数据
    async loadArticles () {
      const { id: channelId, timestamp } = this.activeChannel
      const data = await getArticles({
        channelId, // 当前激活频道id
        timestamp, // 当前频道的下次事件戳
        withTop: 1 // 是否包含置顶数据
      })
      return data
    },

    // 加载频道列表
    async loadChannels () {
      const { user } = this.$store.state
      let channels = []
      // 以登陆
      if (user) {
        const data = await getUserChannels()
        channels = data.channels
      } else {
        // 未登录
        // 如果有本地存储 就是用本地存储的频道列表
        const localChannels = JSON.parse(window.localStorage.getItem('channels'))
        if (localChannels) {
          channels = localChannels
        } else {
          // 没有就使用默认的
          const data = await getUserChannels()
          channels = data.channels
        }
      }

      // 修改 channels，改为满足我们需求的结构
      channels.forEach(item => {
        item.articles = [] // 用来存储当前文章的列表
        item.downPullLoading = false // 控制当前频道的下拉刷新 loading 的状态
        item.upPullLoading = false // 控制上拉 加载更多的 loading 的状态
        item.upPullFinished = false // 控制当前频道数据加载完毕
        item.timestamp = Date.now() // 存储下一页的数据时间戳
        // downPullSuccess = ''
        // item.timestamp = 1556789000004 // 存储下一页的数据时间戳
      })
      this.channels = channels
    },

    // 上啦加载更多， push数据
    async onLoad () {
      await this.$sleep(800)
      let data = []
      data = await this.loadArticles()
      // pre_timestamp 下一页的（上次事件点推荐的数据）
      // result 就是文章列表
      
      // 如果没有 pre_timestamp 并且数据是空的，就意味着没有数据了 
      if (!data.pre_timestamp && !data.results.length) {
        // 加载完毕 取消上啦 loading状态
        this.activeChannel.upPullLoading = false
        // 设置该频道数据已加载完毕，组件会自动给出提示，并且不再 onload
        this.activeChannel.upPullFinished = true
        return
      }

      // 没有最新数据，那就加载上一次推荐数据
      if (data.pre_timestamp && !data.results.length) {
        this.activeChannel.timestamp = data.pre_timestamp
        // 加载下一页数据
        data = await this.loadArticles()
      }

      // 数据加载好以后，将 pre_timestamp 更新到当前频道的中用于下载下页数据
      this.activeChannel.timestamp = data.pre_timestamp

      // 将文章数据push更新到频道中
      this.activeChannel.articles.push(...data.results)

      // 加载完毕 取消上啦 loading状态
      this.activeChannel.upPullLoading = false
    },

    // 下拉加载更多，有新的数据 就重新数据列表
    async onRefresh () {
      const { activeChannel } = this
      const timestamp = activeChannel.timestamp
      activeChannel.timestamp = Date.now()
      const data = await this.loadArticles()
      console.log(data)
      // 如果有最新数据，就重新到当前频道的数据列表
      if (data.results.length) {
        // 将当前最新的推荐内容重置到频道文章中
        activeChannel.articles = data.results
        // 由于充值了文章列表，那么当前数据中的 per-timestamp 就是上啦加载更多的时间戳
        activeChannel.timestamp = data.timestamp
        activeChannel.downPullSuccess = '刷新成功'
        // 当下拉刷新数据并重置以后数无法满足一屏，所有使用 onload加载更多数据
        this.onLoad()
      } else {
        // 如果没有最新数据，就提示以是最新内容
        activeChannel.downPullSuccess = '已是最新数据'
      }
      activeChannel.downPullLoading = false
    }
  }
}
</script>

<style lang="less" scoped>
.channel-tabs {
  margin-bottom: 100px;
}

// 深度作用选择器：https://vue-loader.vuejs.org/zh/guide/scoped-css.html#%E6%B7%B1%E5%BA%A6%E4%BD%9C%E7%94%A8%E9%80%89%E6%8B%A9%E5%99%A8
.channel-tabs /deep/ .van-tabs__wrap {
  position: fixed;
  top: 92px;
}

.channel-tabs /deep/ .van-tabs__content {
  margin-top: 100px;
}
.nav-right {
  position: fixed;
  right: 0;
}
</style>

<template>
  <div class="login-wrap">
    <van-nav-bar title="标题"/>
    <form>
      <van-cell-group>
        <van-field
          v-model="user.mobile"
          v-validate="'required'"
          name="mobile"
          :error-message="errors.first('mobile')"
          required
          clearable
          label="手机号"
          placeholder="请输入手机号"
        />

        <van-field
          v-model="user.code"
          v-validate="'required'"
          name="code"
          :error-message="errors.first('code')"
          type="code"
          label="验证码"
          placeholder="请输入验证码"
          required
        />
      </van-cell-group>
      <div class="login-btn">
        <van-button
          class="btn"
          type="info"
          @click.prevent="handleLogin"
          :loading="loginLoading"
        >登录</van-button>
      </div>
    </form>
  </div>
</template>

<script>
import { login } from '@/api/user'

export default {
  name: 'LoginIndex',
  data () {
    return {
      user: {
        mobile: '18801185985',
        code: '246810'
      },
      loginLoading: false // 控制登录请求的 loading 状态
    }
  },

  created () {
    this.configCustomMessages()
  },

  methods: {
    async handleLogin () {
      this.loginLoading = true
      try {
        // 这个插件的 JavaScript 验证方法设计的不好，并没有在验证失败的时候抛出异常
        const valid = await this.$validator.validate()

        // 如果表单验证失败，则什么都不做
        if (!valid) {
          // 验证失败，代码不会往后执行了，所以在这里也要取消 loading
          this.loginLoading = false
          return
        }

        // 表单验证通过，提交表单
        const data = await login(this.user)

        // 通过提交 mutation 更新 Vuex 容器中的装填
        this.$store.commit('setUser', data)

        // 登录成功，先简单粗暴的跳转到首页，后面再处理跳转到来的页面
        this.$router.push({
          name: 'home'
        })
      } catch (err) {
        console.log(err)

        this.$toast.fail('登录失败！')
      }
      this.loginLoading = false
    },

    configCustomMessages () {
      const dict = {
        custom: {
          mobile: {
            required: '手机号不能为空'
          },
          code: {
            required: () => '验证码不能为空'
          }
        }
      }
      this.$validator.localize('zh_CN', dict)
    }
  }
}
</script>

<style lang="less" scoped>
.login-btn {
  padding: 10px;
  .btn {
    width: 100%;
  }
}
</style>

<template>
<div class="page-container">
  <md-app md-mode="fixed">
    <md-app-toolbar class="md-primary">
      <span class="md-title">今天吃啥？</span>
      <div class="md-toolbar-section-end">
        <md-button @click="openSetting"
                   class="md-icon-button">
          <md-icon>settings</md-icon>
        </md-button>
        <md-dialog :md-active.sync="setting.show"
                   class="setting-dialog"
                   :md-fullscreen="true">
          <div v-if="setting.state === 'store-list'" key="store-list" class="setting-view">
            <md-dialog-title>设置</md-dialog-title>
            <md-toolbar class="md-primary"
                        md-elevation="1">
              <md-button @click="closeSetting"
                         class="md-icon-button">
                <md-icon>close</md-icon>
              </md-button>
              <h3 class="md-title"
                  style="flex: 1">设置</h3>
              <md-button @click="saveSetting"
                         class="md-icon-button">
                <md-icon>save</md-icon>
              </md-button>
            </md-toolbar>
            <md-dialog-content class="view-transition">
              <md-list v-if="setting.stores.length" class="store-list md-double-line">
                <md-subheader>
                  <span>店铺</span>
                  <md-button class="md-icon-button" @click="addStore">
                    <md-icon>add</md-icon>
                    <md-tooltip md-direction="top">添加店铺</md-tooltip>
                  </md-button>
                </md-subheader>
                <template v-for="(store, index) in setting.stores">
                  <md-list-item>
                    <!-- <md-icon>store</md-icon> -->
                    <md-checkbox v-model="store.enable" />
                    <div class="md-list-item-text">
                      <span>{{store.name}}</span>
                      <p class="fafa">
                        {{store.menu.join(', ')}}
                      </p>
                    </div>
                    <md-button class="md-icon-button md-list-action" @click.stop="editStore(store)">
                      <md-icon>edit</md-icon>
                    </md-button>
                  </md-list-item>
                  <md-divider class="md-inset" v-if="index + 1 < setting.stores.length"></md-divider>
                </template>
              </md-list>
              <md-empty-state v-else
                              md-icon="store"
                              md-label="没有店铺"
                              md-description="你真是怠惰呢。">
                <md-button class="md-primary md-raised" @click="addStore">添加</md-button>
              </md-empty-state>
              <md-button class="control-button fixed-fab md-fab md-plain"
                         @click="addStore">
                <md-icon>add</md-icon>
              </md-button>
            </md-dialog-content>
            <md-dialog-actions>
              <md-button class="md-primary"
                         @click="setting.show = false">关闭</md-button>
              <md-button class="md-primary"
                         @click="saveSetting">保存</md-button>
            </md-dialog-actions>
          </div>
          <div v-else-if="setting.state === 'edit-store'" key="edit-store" class="setting-view">
            <md-dialog-title>编辑</md-dialog-title>
            <md-toolbar class="md-primary"
                        md-elevation="1">
              <md-button @click="backToStores"
                         class="md-icon-button">
                <md-icon>arrow_back</md-icon>
              </md-button>
              <h3 class="md-title"
                  style="flex: 1">编辑</h3>
              <md-button @click="deleteStore"
                         class="md-icon-button">
                <md-icon>delete</md-icon>
              </md-button>
            </md-toolbar>
            <md-dialog-content class="view-transition">
              <md-field :class="{'md-invalid': $v.setting.currentStore.name.$dirty && $v.setting.currentStore.name.$invalid}">
                <label>店名</label>
                <md-input v-model="setting.currentStore.name" @blur="$v.setting.currentStore.name.$touch" />
                <span class="md-error" v-if="!$v.setting.currentStore.name.required">必须填写</span>
              </md-field>
              <md-field>
                <label>添加菜单</label>
                <md-input placeholder="菜单：空格隔开可添加多个" v-model="setting.tmpFoods" @keypress.enter="addMenu" />
                <md-button class="md-icon-button md-input-action" @click="addMenu">
                  <md-icon>send</md-icon>
                </md-button>
              </md-field>
              <p>
                <md-chip v-for="(food, index) in setting.currentStore.menu" :key="index" md-deletable @md-delete="deleteFood(index)" class="md-layout-item">{{food}}</md-chip>
                <md-chip v-if="setting.currentStore.menu.length" md-deletable @md-delete="clearMenu" class="md-accent md-layout-item">清空</md-chip>
              </p>
            </md-dialog-content>
            <md-dialog-actions>
              <md-button class="md-primary" @click="backToStores">返回</md-button>
              <md-button class="md-accent" @click="deleteStore">删除</md-button>
            </md-dialog-actions>
          </div>
        </md-dialog>
      </div>
    </md-app-toolbar>
    <md-app-content>
      <md-card class="content-card">
        <md-card-header>
          <md-field>
            <label for="favorite-store">店铺</label>
            <md-select name="favorite-store"
                       id="favorite-store"
                       v-model="favoriteStore"
                       md-dense>
              <md-option value="all">随便，都行</md-option>
              <md-option v-for="(store, index) in stores"
                         :key="index" :value="index">{{store.name}}</md-option>
            </md-select>
          </md-field>
        </md-card-header>
        <md-card-content>
          <p class="result" v-if="foods.length">{{result || '&nbsp;'}}</p>
          <md-empty-state v-else
                          md-icon="restaurant_menu"
                          md-label="没有内容"
                          md-description="当前选择下没有内容，你可以去添加一些。">
            <md-button class="md-primary md-raised" @click="openSetting">添加</md-button>
          </md-empty-state>
        </md-card-content>
      </md-card>
      <md-button class="control-button fixed-fab md-fab md-plain"
                v-if="foods.length"
                 @click="toggle">
        <i class="action-icon"
           :class="actionState"></i>
      </md-button>
    </md-app-content>
  </md-app>
</div>

</template>

<script>
import { validationMixin } from 'vuelidate'
import { required } from 'vuelidate/lib/validators'

function storeTpl() {
  return {
    name: '',
    menu: [],
    enable: true
  }
}

let timer = null

export default {
  name: 'App',
  mixins: [validationMixin],
  data() {
    return {
      get stores() {
        const storesStr = window.localStorage.getItem('whatToEat.stores')
        return storesStr ? JSON.parse(storesStr) : []
      },
      set stores(val) {
        if (!val) return
        window.localStorage.setItem('whatToEat.stores', JSON.stringify(val))
      },
      playing: false,
      result: '',
      favoriteStore: 'all',
      setting: {
        show: false,
        stores: [],
        state: 'store-list',
        currentStore: null,
        tmpFoods: ''
      }
    }
  },
  validations: {
    setting: {
      currentStore: {
        name: {
          required
        }
      }
    }
  },
  methods: {
    randomFood() {
      const length = this.foods.length
      return this.foods[Math.floor(Math.random() * length)]
    },
    toggle() {
      this.playing = !this.playing
      const setResult = _ => {
        this.result = this.randomFood()
      }
      if (this.playing) {
        timer = window.setInterval(setResult, 16.7)
      } else {
        timer && window.clearInterval(timer)
        let [total, count, time, step] = [10, 10, 0, 16.7]
        while (count--) {
          time += step * (total - count) * 1.2
          window.setTimeout(setResult, time)
        }
      }
    },
    openSetting() {
      this.setting.stores = this.stores
      this.setting.state = 'store-list'
      this.setting.show = true
    },
    closeSetting() {
      this.setting.stores = []
      this.setting.show = false
    },
    editStore(store) {
      this.setting.currentStore = store
      this.setting.state = 'edit-store'
    },
    addStore() {
      const currentStore = storeTpl()
      this.setting.currentStore = currentStore
      this.setting.stores.push(currentStore)
      this.setting.state = 'edit-store'
    },
    backToStores() {
      this.$v.setting.currentStore.name.$touch()
      if (this.$v.setting.currentStore.name.$invalid) return
      this.$v.setting.currentStore.name.$reset()
      this.setting.state = 'store-list'
    },
    deleteStore() {
      this.setting.stores.splice(this.setting.stores.indexOf(this.setting.currentStore), 1)
      this.$v.setting.currentStore.name.$reset()
      this.setting.state = 'store-list'
    },
    addMenu() {
      const menuStr = this.setting.tmpFoods.replace(/\s+/g, ' ').trim()
      if (!menuStr) return
      console.log(menuStr.split(' '))
      Array.prototype.push.apply(this.setting.currentStore.menu, menuStr.split(' '))
      this.setting.tmpFoods = ''
    },
    deleteFood(index) {
      this.setting.currentStore.menu.splice(index, 1)
    },
    clearMenu() {
      this.setting.currentStore.menu = []
    },
    saveSetting() {
      this.stores = this.setting.stores
      this.setting.show = false
    }
  },
  computed: {
    foods() {
      const foods = []
      if (this.favoriteStore === 'all') {
        let storeI = this.stores.length
        while (storeI--) {
          if (this.stores[storeI].enable) {
            let menuI = this.stores[storeI].menu.length
            while (menuI--) {
              foods.push(`${this.stores[storeI].menu[menuI]}@${this.stores[storeI].name}`)
            }
            // Array.prototype.push.apply(foods, this.stores[storeI].menu)
          }
        }
      } else {
        const menu = this.stores[this.favoriteStore].menu
        let menuI = menu.length
        while (menuI--) {
          foods.push(menu[menuI])
        }
      }
      return foods
    },
    actionState() {
      return this.playing ? 'stop' : 'start'
    }
  },
  mounted() {
    console.log(this)
  }
}

</script>

<style lang="less">
body,
.page-container,
.md-app,
.md-app-container {
  height: 100%;
  overflow: hidden;
}

.page-container .md-app-container .md-content {
  background-color: #f1f1f1;
  overflow: auto;
  .content-card {
    margin: 0;
    .result {
      font-size: 36px;
      line-height: 1;
      color: #333;
      text-align: center;
    }
  }
  .control-button {
    .action-icon {
      display: block;
      width: 0;
      height: 0;
      box-sizing: content-box;
      border-style: solid;
      border-color: #fff;
      transition: transform .3s, border .3s;
      &.start {
        transform: rotate(-90deg) translateY(1.89px);
        border-left: 7px solid transparent;
        border-right: 7px solid transparent;
        border-top: 12.12px solid #fff;
        border-bottom: 0px solid transparent;
      }
      &.stop {
        border-width: 6px;
      }
    }
  }
}

.md-fab.fixed-fab {
  position: fixed;
  margin: 0;
  bottom: 14px;
  right: 17px;
  animation: fabAnimate .3s;
}

.md-chip {
  margin: 4px;
}

@keyframes fabAnimate {
  from {
    transform: translateY(128px);
  }
  to {
    transform: translateY(0);
  }
}

.setting-dialog {
  overflow: hidden;
  .setting-view {
    display: flex;
    flex-flow: column;
    flex: 1;
    .view-transition {
      animation: viewTransition .3s
    }
    @keyframes viewTransition {
      from {
        opacity: 0;
        transform: translateX(64px)
      }
      to {
        opacity: 1;
        transform: translateY(0)
      }
    }

  }
  .fixed-fab,
  .md-toolbar {
    display: none;
  }
  @media (max-width: 600px) {
    .fixed-fab {
      display: block;
    }
    .md-toolbar {
      display: flex;
    }
    .md-dialog-title,
    .md-dialog-actions,
    .md-subheader .md-button {
      display: none;
    }
    transition-duration: .3s;
    &.md-dialog-fullscreen {
      &.md-dialog-enter-active,
      &.md-dialog-leave-active {
        transform: translateY(100%);
      }
    }
  }
  .store-list .md-list-item-text p {
    white-space: normal;
  }
}
</style>

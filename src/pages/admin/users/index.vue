<template>
  <div>
    <el-row type="flex">
      <el-col :xs="20" :sm="12" :md="11" :lg="10" :xl="10">
        <el-card>
          <el-row id="users_header">
            <el-col :span="8">
              <h4>Пользователи</h4>
            </el-col>
            <el-col :xs="22" :sm="22" :md="22" :lg="16" :xl="16">
              <el-radio-group fill="#67c23a" v-model="type" size="small" id="user_type">
                <el-radio-button label="register">С аккаунтом</el-radio-button>
                <el-radio-button label="anonymous">Анонимные</el-radio-button>
              </el-radio-group>
            </el-col>
          </el-row>
          <div id="users_list" :key="userListKey">
            <div v-for="(user, id) in allUsers" :key="id">
              <UserListRow v-if="user.events" :id="id" :data="user" :type="type"/>
            </div>
          </div>
        </el-card>
      </el-col>
      <el-col :xs="20" :sm="12" :md="13" :lg="14" :xl="14" style="margin-left: 10px;">
        <el-card>
          <h4>Действия</h4>
          <el-popover
            v-if="$store.getters.watchedUserId"
            placement="top"
            width="220"
            v-model="deleteEventsPopover">
            <div style="margin: 12px;">
              <p style="text-align:left;">Вы действительно хотите удалить действия пользователя?</p>
              <div style="text-align: right; margin: 0">
                <el-button size="mini" type="text" @click="deleteEventsPopover = false">Отмена</el-button>
                <el-button type="danger" size="mini" @click="clearUserEvents">Удалить</el-button>
              </div>
            </div>
            <el-button slot="reference" icon="el-icon-delete" size="small"></el-button>
          </el-popover>
          <div id="users_events">
            <el-row v-for="(event, id) in userEvents" :key="id" class="user_event_row">
              <el-col style="width: 90px">
                <el-tooltip :content="new Date(event.date).toLocaleString()" placement="bottom">
                  <span class="chat_time">{{ new Date(event.date).toLocaleTimeString() }}</span>
                </el-tooltip>
              </el-col>
              <el-col :span="18">{{ event.name }}</el-col>
            </el-row>
          </div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
  import UserListRow from "@/components/admin/users/UserListRow";

  export default {
    name: "index",
    components: {UserListRow},
    layout: 'admin',
    data() {
      return {
        userListKey: '1',
        deleteEventsPopover: false,
        type: 'register' // user type: 'register' | 'anonymous'
      }
    },
    methods: {
      clearUserEvents() {
        this.deleteEventsPopover = false
        this.$store.dispatch('clearUserEvents')
      }
    },
    computed: {
      allUsers() {
        this.userListKey = new Date().getTime()
        return this.$store.getters.allUsers
      },
      userEvents() {
        if (this.$store.getters.allUsers[this.$store.getters.watchedUserId]) {
          return this.$store.getters.allUsers[this.$store.getters.watchedUserId].events
        } else {
          return {}
        }
      }
    },
    created() {
      this.$store.dispatch('fetchAllUsers')
    },
    beforeDestroy () {
      this.$store.dispatch('unsubscribeFromEvents')
    }
  }
</script>

<style scoped lang="scss">
  #users_list, #users_events {
    width: 100%;
    max-height: 600px;
    overflow: scroll;
  }

  #user_type {
    text-align: center;
  }

  #users_header {
    display: flex;
    justify-content: start;
    align-items: center;
    flex-wrap: wrap;
    margin-bottom: 5px;
  }

  .user_event_row {
    display: flex;
    font-size: 14px;
    margin: 5px;
  }

  .chat_time {
    font-size: 11px;
    padding-top: 4px;
  }
</style>

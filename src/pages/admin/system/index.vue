<template>
  <div>
    <el-tooltip placement="top" effect="light">
      <div slot="content">
        Создание архива всех данных, <br> которые вы и ваши пользователи вносили на сайте.
      </div>
      <el-button @click="getData" type="success" class="system_btn">
        Снять копию базы данных <br>
        <img src="~/assets/icons/admin/dns.svg" alt="Data Dump">
      </el-button>
    </el-tooltip>
    <!---->
    <!---->
    <!---->
    <el-tooltip placement="top" effect="light">
      <div slot="content">
        Создание файла со всеми страницами сайта для поисковых роботов.
      </div>
      <el-button @click="generateSitemap" type="success" class="system_btn">
        Сгенерировать Sitemap.xml <br>
        <img src="~/assets/icons/admin/map.svg" alt="Site map icon">
      </el-button>
    </el-tooltip>
    <!---->
    <!---->
    <!---->
    <el-tooltip placement="bottom-end" :visible-arrow="false">
      <div slot="content">
        Внимание! <br>
        Эта операция полностью перетирает старые данные в базе! <br>
        Восстановление производить загрузкой файлов по одному <br>
        с расширением <b>.json</b>! <br>
        <b>Имя файла должно соответствовать названию коллекции!</b> <br>
        Для использования операции необходим пароль разработчика.
      </div>
      <el-popover
        placement="right-start"
        width="220"
        v-model="restoreDbPopover">
        <div style="margin: 12px;">
          <p style="text-align:left;">Введите пароль:</p>
          <el-input type="password" v-model="password" auto-complete="off"></el-input>
          <div style="text-align: right; margin-top: 10px">
            <el-button size="mini" type="text" @click="restoreDbPopover = false">Отмена</el-button>
            <el-button type="danger" size="mini" @click="onPickJsonFileForRestoreDB">Продолжить</el-button>
          </div>
        </div>
        <el-button
          slot="reference" type="success" class="system_btn" style="margin-left: 8px;"
          @click="restoreDbPopover = true">
          Восстановить коллекцию <br>
          <img src="~/assets/icons/admin/history.svg" alt="Repair data">
        </el-button>
      </el-popover>
    </el-tooltip>
    <input
      type="file"
      style="display: none;"
      ref="fileInput"
      accept="json/*"
      @change="onFileJsonPicked">
    <br>
    <!---->
    <!---->
    <!---->
    <el-button @click="loadErrorLog" type="success" class="system_btn" style="margin-top: 8px;">
      Загрузить логи ошибок <br>
      <img src="~/assets/icons/admin/bug_report.svg" alt="Load err log">
    </el-button>
    <div v-if="errLog">
      <el-popover
        placement="top"
        width="220"
        v-model="clearLogPopover">
        <div style="margin: 12px;">
          <p style="text-align:left;">Введите пароль:</p>
          <el-input type="password" v-model="password" auto-complete="off"></el-input>
          <div style="text-align: right; margin-top: 10px">
            <el-button size="mini" type="text" @click="clearLogPopover = false">Отмена</el-button>
            <el-button type="danger" size="mini" @click="clearErrorLog">Удалить</el-button>
          </div>
        </div>
        <el-button
          @click="clearLogPopover = true"
          type="warning" slot="reference" class="system_btn" style="margin-top: 5px;">
          Очистить логи ошибок <br>
          <img src="~/assets/icons/admin/clear.svg" alt="Clear err log">
        </el-button>
      </el-popover>
      <el-collapse accordion id="error_accordion">
        <el-collapse-item
          v-for="(err, idx) in errLog" :key="idx" :name="idx"
          :title="`${new Date(err.time).toLocaleString()}: ${typeof err.data === 'string' ? ': ' + err.data.slice(0, 120) + '...' : 'Object error'}`">
          {{ err.data }}
        </el-collapse-item>
      </el-collapse>
    </div>
  </div>
</template>

<script>
  import {db, fs} from "@/services/fireinit";
  import JSZip from 'jszip'
  import FileSaver from 'file-saver'

  export default {
    name: "index",
    layout: 'admin',
    data() {
      return {
        errLog: '',
        clearLogPopover: false,
        restoreDbPopover: false,
        password: '',
        collections: [
          'companyInfo',
          'dictionaries',
          'orders',
          'products',
          'reviews',
          'statistics',
          'users',
          'questions',
          'userRequests'
        ]
      }
    },
    methods: {
      getData() {
        let zip = new JSZip()
        this.$store.dispatch('LOADING', true)

        let getCollectionData = function (name) {
          return fs.collection(name).get()
            .then(snap => {
              let data = {}
              snap.docs.forEach(el => {
                data[el.id] = el.data()
              })
              zip.file(name + '.json', JSON.stringify(data))
            })
        }
        let actions = []
        this.collections.forEach(name => actions.push(getCollectionData(name)))

        Promise.all(actions)
          .then(() => {
            zip.generateAsync({type: 'blob'})
              .then((content) => {
                FileSaver.saveAs(content, 'virapro_dump_' + new Date().toLocaleString() + '.zip')
                this.$message({type: 'success', message: 'Архив базы данных создан. Сохраните его в надежном месте.'})
                this.$store.dispatch('LOADING', false)
              })
              .catch(err => {
                this.$message({type: 'error', message: err})
                this.$store.dispatch('ERR', err)
                this.$store.dispatch('LOADING', false)
              })
          })
          .catch(err => {
            console.log(err)
          })
      },
      generateSitemap() {
        this.$store.dispatch('LOADING', true)
        let zip = new JSZip()
        let xml = ''
        fs.collection('products').get()
          .then(snap => {
            xml = '<?xml version="1.0" encoding="UTF-8"?><urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">'
            let URL = 'https://virapro.ru/'
            let mainPages = ['', 'catalog', 'about']
            let lastmod = new Date().toISOString()
            mainPages.forEach(el => {
              xml += `<url><loc>${URL}${el}/</loc><lastmod>${lastmod}</lastmod></url>`
            })
            snap.docs.forEach(doc => {
              doc.data()
              xml += `<url><loc>${URL}catalog/${doc.data().group}/${doc.data().category}/${doc.id}/</loc></url>`
            })
            xml += '</urlset>'
            zip.file('sitemap.xml', xml)
            zip.generateAsync({type: 'blob'})
              .then((content) => {
                let date = new Date().toISOString().substring(0, 10)
                FileSaver.saveAs(content, 'virapro_sitemap_' + date + '.zip')
                this.$store.dispatch('LOADING', false)
                this.$message({type: 'success', message: 'Архив с файлом sitemap.xml создан.'})
              })
              .catch(err => {
                this.$message({type: 'error', message: err})
                this.$store.dispatch('ERR', err)
                this.$store.dispatch('LOADING', false)
              })
          })
      },
      loadErrorLog() {
        db.ref('errLog').orderByKey().once('value', snap => {
          if (!snap.exists()) {
            return this.$message({type: 'success', message: 'Ошибки отсутствуют.'})
          }
          this.errLog = snap.val()
        })
      },
      async clearErrorLog() {
        this.clearLogPopover = false
        if (this.password !== 'panda') {
          return this.$message({type: 'error', message: 'Неверный пароль'})
        }
        await db.ref('errLog').remove()
        this.errLog = ''
      },
      onPickJsonFileForRestoreDB() {
        if (this.password !== 'panda') {
          return this.$message({type: 'error', message: 'Неверный пароль'})
        }
        this.$refs.fileInput.click()
      },
      onFileJsonPicked(event) {
        this.$store.dispatch('LOADING', true)
        const files = event.target.files
        let collectionName = files[0].name.split('.')[0]
        const fileReader = new FileReader()
        fileReader.addEventListener('load', () => {
          let actions = []
          this.imageUrl = fileReader.result
          let collection = JSON.parse(fileReader.result)
          for (let docId in collection) {
            actions.push(fs.collection(collectionName).doc(docId).set(collection[docId]))
          }
          Promise.all(actions).then(() => {
            this.$message({type: 'success', message: `Коллекция "${collectionName}" перезаписана!`})
            this.$refs.fileInput.value = ''
            this.$store.dispatch('LOADING', false)
          })
            .catch(err => {
              this.$message({type: 'error', message: err})
              this.$store.dispatch('ERR', err)
              this.$store.dispatch('LOADING', false)
            })
        })
        fileReader.readAsText(files[0])
        this.image = files[0]
      }
    }
  }
</script>

<style scoped lang="scss">
  .system_btn {
    height: 100px;
    width: 240px;
    img {
      margin-top: 8px;
    }
  }

  #error_accordion {
    margin: 10px;
  }
</style>

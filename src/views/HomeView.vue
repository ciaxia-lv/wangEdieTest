<template>
  <div>
    <div>
      <div ref="editor" id="editor" @keydown="enterEv($event)" @keyup="handleKeyUp($event)"
           @compositionstart="handleCompositionstart($event)" @compositionend="handleCompositionend($event)"></div>
      <div class="bullet-box" :style="{ left: left + 'px', top: top + 'px', visibility: showFlag }">
        <div @scroll="personOnscroll" id="personScrollId" class="bullet-box-content">
          <div @click="selectPerson(item.userName, item.userId, item.headPath)"
               :class="{ list: true, listb: index === personSelectIndex }" :id="`user${item.userId}`"
               :data-userid="item.userId" v-for="(item, index) in contactList" :key="index"
               @mouseover="personSelectOnmouserover('over', index)" @mouseleave="personSelectOnmouserover('leave', index)">
            <div class="list-item">
              <img :src="item.headPath || defaultAvatar" alt="" />
              <div class="list-item-right">
                <span class="top">{{
                    `${item?.userName}（${item?.userId != 'xxxxxxxx' ? item?.userId : '外部人员'
                    }）`
                  }}</span>
                <span class="bottom">{{ item?.orgName }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import E from 'wangeditor'
import { Tools } from '../utils/index.ts'
import {api} from '../Axios/index.ts'
export default {
  props: {
    meetingDetail: {
      type: Object
    },
    content: {
      type: Array
    },
    noticePeople: {
      type: Array
    },
    pageState: {
      type: String
    }
  },
  data() {
    return {
      editorContent: '',
      browserType: Tools.browserType(),
      editor: '',
      uploadObj: {},
      enclosure_list: [],
      left: '',
      top: '',
      showFlag: 'hidden',
      personSearchText: '',
      contactList: [],
      htmlList: [],
      searchPeopleList: [],
      page: {
        pageNum: 1,
        pageSize: 10,
        totalCount: 0,
        totalPages: 0
      },
      personSelectIndex: -1,
      notifierList: [],
      idList: [],
      chineseFlag: false,
      defaultAvatar:
          'https://oss-appupload.icomecloud.com/userInfo/faceUrl/prod/d3c26f22f46c9ae4037562494b111ba0.png-m_app_avatar.jpg'
    }
  },

  watch: {
    meetingDetail(newVal, oldVal) {
      this.contactList = newVal.attendeList
    },
    content(newVal, oldVal) {
      let _html = ''
      newVal.forEach((item, index) => {
        _html = _html + `<div data-beforeId='${item.id}' class="content-item-text">${item.styleText}</div>`
      })
      this.editor.txt.html(_html)
    },
    noticePeople(newVal, oldVal) {
      newVal.forEach((item) => {
        // 同步通知@人列表
        this.$emit('addNotifier', {
          id: item.tagId,
          name: item.tagName,
          headPath: item.headPath
        })
        this.notifierList.push(item.tagId)
      })
    }
  },
  mounted() {
    let editor = new E(this.$refs.editor)
    editor.config.onchange = this.editInputChangeHandle
    editor.config.placeholder =
        '1. 输入会议结论，可@他人<br>2. Enter换行可添加多条结论，单条结论字数不得超过1000'
    editor.config.onchangeTimeout = 500 // 修改为 500m
    editor.config.height = '60'
    editor.config.minHeight = 'auto'
    editor.create()
    this.editor = editor
    if (this.content?.length) {
      let _html = ''
      this.content.forEach((item) => {
        // console.log(item.styleText, 'item.styleText')
        _html = _html + `<div data-beforeId='${item.id}' class="content-item-text">${item.styleText}</div>`
      })
      console.log(this.content, "this.contentthis.content", _html, " _html _html")
      this.editor.txt.html(_html)
    }
  },
  methods: {
    editInputChangeHandle(val) {
      console.log(val, "vvvvvv")
      this.getHtml()
    },
    personSelectOnmouserover(type, ind) {
      type === 'over' && (this.personSelectIndex = ind)
      type === 'leave' && (this.personSelectIndex = -1)
    },

    getHtml() {
      this.htmlList = []
      this.idList = []
      Array.from(this.editor.$textElem.elems[0].children).forEach((ele, index) => {
        if (ele.innerHTML != '<br>') {
          // this.idList.push(ele.getAttribute("data-beforeid"));
          this.htmlList.push(`${ele.innerHTML}`)
        }
      })
      this.getData()
    },
    getData() {
      let newList = []
      this.htmlList.forEach((ele, index) => {
        newList[index] = {}
        newList[index].plainText = this.convertPlainText(ele)
        newList[index].textContent = this.convertTextContent(ele)
        newList[index].textUserId = this.convertTextUserId(ele).join(',')
        // newList[index].eventId =
        //     this.meetingDetail.dingCalendarId ||
        //     new URLSearchParams(window.location.search).get('eventId')
        newList[index].styleText = this.htmlList[index]
        newList[index].serialNumber = index + 1
        if (
            !(
                this.editor.$textElem.elems[0].children[index].getAttribute('data-beforeid') ===
                'undefined'
            )
        ) {
          newList[index].id =
              this.editor.$textElem.elems[0].children[index].getAttribute('data-beforeid')
        }
      })
      if (
          newList.length === 1 &&
          (newList[0].textContent === '<br>' || !newList[0].textContent.trim())
      ) {
        newList = []
      }
      this.$emit('updateConclusion', newList)
    },
    convertPlainText(parentElement) {
      const DomStr = document.createElement('div')
      DomStr.innerHTML = parentElement
      const pArr = DomStr.querySelectorAll('.my-VueTribute-span')
      pArr.forEach((item) => {
        if (item.dataset.userid != 'xxxxxxxx') {
          item.innerHTML =
              '#start' + item.dataset.name + '_' + item.dataset.userid + '#end' + item?.innerText
        }
      })
      return DomStr.textContent.replaceAll('<br>', '')
    },
    convertTextContent(parentElement) {
      const DomStr = document.createElement('div')
      DomStr.innerHTML = parentElement
      const pArr = DomStr.querySelectorAll('.my-VueTribute-span')
      pArr.forEach((item) => {
        item.innerHTML = item.dataset.name + item?.innerText
      })
      return DomStr.textContent.replaceAll('<br>', '')
    },
    convertTextUserId(text) {
      // 匹配 <span> 标签中的 data-userid 属性值
      var regex = /data-userid="(\d+)"/g
      // 提取匹配到的 data-userid 属性值
      var userIds = []
      var match
      while ((match = regex.exec(text)) !== null) {
        if (match[1] != 'xxxxxxxx') {
          userIds.push(match[1])
        }
      }
      return userIds
    },
    toHtml() {
      let html = ''
      this.htmlList.forEach((ele) => {
        html = html + `<p>${ele}</p>`
      })
      return html
    },
    enterEv(e) {
      //keydown触发事件
      this.personSearchText = ''
      let selection = getSelection()
      console.log(e.key,"e.key",e.shiftKey,"e.shiftKey")
      if (e.key === '@' && e.shiftKey) {
        // 兼容
        e.preventDefault ? e.preventDefault() : (e.returnValue = false)
        this.position = {
          range: selection.getRangeAt(0),
          selection: selection
        }
        try {
          if (this.editor.$textElem.elems[0].firstChild.firstChild.tagName === 'BR') {
            this.editor.$textElem.elems[0].firstChild.removeChild(
                this.editor.$textElem.elems[0].firstChild.firstChild
            )
          }
        } catch (e) {
        }
        let fakeNode = document.createElement('span')
        fakeNode.className = 'fake-at'
        fakeNode.innerHTML = '@'
        this.position.range.insertNode(fakeNode)
        this.position.selection.collapse(fakeNode, 1)
        this.getPosition(fakeNode)
        this.showFlag = 'visible'
        selection.collapseToEnd()
      } else if (e.code === 'Backspace' || e.key === 'Backspace') {
        // 恢复默认值
        let range = selection.getRangeAt(0)
        let removeNode = null
        if (this.browserType === 'Firefox') {
          if (range.startContainer.className !== 'my-VueTribute-span') {
            removeNode = range.startContainer.previousElementSibling
            console.dir('Firefox previousElementSibling', removeNode)
          }
          if (
              range.startContainer.parentElement.className !== 'my-VueTribute-span' &&
              range.startContainer.lastChild !== null
          ) {
            removeNode = range.startContainer.lastElementChild.previousElementSibling
            console.dir('Firefox lastElementChild.previousElementSibling', removeNode)
          }
        }
        if (this.browserType === 'Chrome') {
          if (
              range.startContainer.textContent.length === 1 &&
              range.startContainer.textContent.trim() === ''
          ) {
            removeNode = range.startContainer.previousElementSibling

            const _Id = removeNode.getAttribute('data-userid')
            this.notifierList.splice(
                this.notifierList.indexOf(removeNode.getAttribute('data-userid')),
                1
            )
            if (!this.notifierList.includes(_Id)) {
              this.$emit('removeNotifier', _Id)
            }
          }
          if (range.startContainer.parentNode.className === 'my-VueTribute-span') {
            e.preventDefault ? e.preventDefault() : (e.returnValue = false)
            removeNode = range.startContainer.parentNode
          }
          // 获取Range所在的当前内容所在的节点
          const node = range.startContainer.nodeType === 3 ? range.startContainer.parentNode : range.startContainer;
          // 如果节点内容为空，则删除该节点
          if (node.innerHTML.trim()==='<br>') {
            node.parentNode.removeChild(node);
          }
        }
        if (this.browserType === 'IE') {
          if (
              range.startContainer.nodeName !== 'P' &&
              range.startContainer.nodeValue.trim() === '' &&
              range.startContainer.previousSibling.className === 'my-VueTribute-span'
          ) {
            removeNode = range.startContainer.previousSibling
          }
          if (
              range.startContainer.parentNode.className === 'my-VueTribute-span' &&
              range.startContainer.nodeName === '#text' &&
              range.startContainer.previousSibling == null
          ) {
            removeNode = range.startContainer.parentNode
            console.log('parentNode', removeNode)
          }
        }
        if (removeNode) {
          Array.from(this.editor.$textElem.elems[0].children).forEach((ele) => {
            if (ele.querySelector(`[data-userid="${removeNode.dataset.userid}"]`)) {
              ele.removeChild(removeNode)
            }
          })
        }
      }
    },

    // 当前@人搜索
    searchPoepleHandle(e) {
      console.log(this.editor.selection.getSelectionContainerElem().elems[0].textContent, '测试')
      const _search = this.editor.selection.getSelectionContainerElem().elems[0].textContent
      if (_search.includes('@')) {
        if (_search.length > 6) {
          this.showFlag = 'hidden'
        } else if (_search.length > 1) {
          this.showFlag = 'visible'
          this.personSearchText = _search.replace('@', '')
          this.getSearchUserInfoHandle(1, this.personSearchText)
        } else {
          this.contactList = api()
          this.showFlag = 'visible'
        }
      } else {
        this.showFlag = 'hidden'
      }
    },
    handleCompositionstart(e) {
      this.chineseFlag = true
    },
    handleCompositionend(e) {
      this.chineseFlag = false
      this.searchPoepleHandle()
    },
    handleKeyUp(e) {
      console.log("KeyUp")
      if (this.chineseFlag) {
        return
      }
      if (this.searchPoepleHandleTime) {
        clearTimeout(this.searchPoepleHandleTime)
        this.searchPoepleHandleTime = null
      }
      this.searchPoepleHandleTime = setTimeout(() => {
        this.searchPoepleHandle(e)
      }, 12)
    },
    //选人
    selectPerson(name, id, headPath) {
      this.showFlag = 'hidden'
      let selection = this.position.selection
      let range = this.position.range
      // 生成需要显示的内容，包括一个 span 和一个空格。
      let spanNode1 = document.createElement('span')
      let spanNode2 = document.createElement('span')

      spanNode1.dataset.name = '@' + name
      spanNode1.dataset.userid = id
      spanNode1.className = 'my-VueTribute-span'
      spanNode1.dataset.time = new Date().valueOf
      spanNode2.innerHTML = '&nbsp'

      // 将生成内容打包放在 Fragment 中，并获取生成内容的最后一个节点，也就是空格。
      let frag = document.createDocumentFragment(),
          node,
          lastNode

      frag.appendChild(spanNode1)
      while ((node = spanNode2.firstChild)) {
        lastNode = frag.appendChild(node)
      }
      // 将 Fragment 中的内容放入 range 中，并将光标放在空格之后。
      range.insertNode(frag)
      selection.collapse(lastNode, 1)
      let fakeAtNode1 = this.editor.selection.getSelectionContainerElem().elems[0]
      if (fakeAtNode1) {
        fakeAtNode1.remove()
      }
      selection.collapseToEnd()

      // 同步通知@人列表
      if (!this.notifierList.includes(id)) {
        this.$emit('addNotifier', { id, name, headPath })
      }
      this.notifierList.push(id)
      this.contactList = api()
    },
    //获取@位置
    getPosition(fakeNode) {
      this.showFlag = 'visible'
      this.left = fakeNode.getBoundingClientRect().left
      this.top = fakeNode.getBoundingClientRect().top + 20
      const rect = fakeNode.getBoundingClientRect()
      const scrollLeft = window.pageXOffset || document.documentElement.scrollLeft
      const pageWidth = window.innerWidth
      const _left = Math.abs(scrollLeft + rect.right - pageWidth)
      if (_left < 200) {
        this.left = fakeNode.getBoundingClientRect().left - 220
      }
    },

    // 查询@人结论接口
    async getSearchUserInfoHandle(page, itcode) {
      try {
        const data = await api()
        if (page === 1) {
          this.contactList = []
        }
        data.map((item) => {
          this.contactList.push({
            ...item,
          })
        })
        this.page = { ...data }
      } catch (error) {
        console.log(error)
      }
    },
    // 分页查询
    personOnscroll(e) {
      const scrollBottom = e.target.scrollHeight - e.target.scrollTop - e.target.clientHeight
      if (scrollBottom < 1 && this.personSearchText) {
        this.page.pageNum += 1
        if (this.page.totalPages > this.page.pageNum) {
          this.getSearchUserInfoHandle(this.page.pageNum, this.personSearchText)
        }
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.bullet-box {
  width: 240px;
  max-height: 400px;
  position: fixed;
  padding: 10px;
  box-shadow: 0px 2px 10px 0px rgba(0, 0, 0, 0.16);

  // border: 1px solid #c1c6cc;
  background-color: #fff;
  z-index: 10000;
  overflow: auto;
  border-radius: 4px;

  &-content {
    overflow-y: scroll;
    max-height: 300px;

    .list {
      padding: 5px 0;
      margin: 5px 0;

      &-item {
        display: flex;
        align-items: center;

        img {
          border-radius: 50%;
          width: 32px;
          height: 32px;
        }

        &-right {
          display: flex;
          flex-direction: column;
          margin-left: 12px;

          .top {
            font-size: 14px;
            font-family: PingFangSC-Regular, PingFang SC;
            font-weight: 400;
            color: #323233;
            line-height: 22px;
          }

          .bottom {
            font-size: 12px;
            font-family: PingFangSC-Regular, PingFang SC;
            font-weight: 400;
            color: #969799;
            line-height: 18px;
          }
        }
      }
    }

    .listb {
      background: #f6f7fb;
    }
  }
}
:deep(.my-VueTribute-span:before) {
  content: attr(data-name);
  color: #487aff;
  pointer-events: auto;
}
</style>
<style lang="scss">
.w-e-toolbar {
  /* display: none; */
  height: 0 !important;
  border: 0 !important;
  visibility: hidden;
}

/* .w-e-text-container {
  height: 60px !important;
  max-height: 400px !important;
  z-index: 10 !important;
} */

.placeholder {
  left: 0 !important;
}

.w-e-text {
  padding: initial !important;

  p {
    font-size: 14px !important;
    font-family: PingFangSC-Regular, PingFang SC !important;
    font-weight: 400 !important;
    color: #3d3e42 !important;
    line-height: 24px !important;
  }
}

.my-VueTribute-span {
  color: #3c71f0;
}

.editor_defalut {
  color: #dbdcde;
}

.w-e-text-container {
  border: none !important;

  min-height: 60px !important;
  //max-height: 350px !important;
  height: auto !important;
  overflow-y: scroll;
  z-index: 10 !important;
}

.w-e-text {
  height: initial !important;
}
.content-item-text{
  margin-top: 5px !important;
}
#editor{
  border: 1px solid red;
}
</style>
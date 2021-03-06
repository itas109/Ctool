<template>
  <div>
    <Tabs v-model="current.operation">
      <TabPane label="二维码生成" name="generate">
        <Row :gutter="16">
          <Col span="14">
            <Input v-model="current.generateInput" :rows="14" type="textarea" placeholder="内容"></Input>
            <option-block>
              <FormItem>
                <Button type="primary" @click="generate()">生成</Button>
              </FormItem>
              <FormItem v-if="generateHistory.length() > 0">
                <Dropdown placement="top-start" trigger="click" style="margin-left: 10px" @on-click="history">
                  <a href="javascript:void(0)">
                    历史记录
                    <Icon type="ios-arrow-down"></Icon>
                  </a>
                  <DropdownMenu slot="list">
                    <DropdownItem v-for="(v,i) in generateHistory.lists()" :key="i" :name="i">{{substr(v.input)}}</DropdownItem>
                    <DropdownItem divided name="clear">清空历史记录</DropdownItem>
                  </DropdownMenu>
                </Dropdown>
              </FormItem>
            </option-block>
          </Col>
          <Col span="10">
            <div style="text-align: center" v-html="current.generateOutput"></div>
          </Col>
        </Row>
      </TabPane>
      <TabPane label="二维码解析" name="reader">
        <Row :gutter="16">
          <Col span="14">
            <Input v-model="current.readerInput" :rows="5" type="textarea" placeholder="请输入二维码图片地址或点击下方按钮上传图片"></Input>
            <option-block>
              <FormItem>
                <Button type="primary" @click="reader()">解析</Button>
              </FormItem>
              <FormItem>
                <Upload action="#" :before-upload="handleUpload">
                  <Button icon="ios-cloud-upload-outline">上传图片</Button>
                </Upload>
              </FormItem>
            </option-block>
            <Input v-model="current.readerOutput" :rows="5" type="textarea" placeholder="解析结果"></Input>
          </Col>
          <Col span="10" style="text-align: center" v-html="readerInputImg"></Col>
        </Row>
      </TabPane>
    </Tabs>
  </div>
</template>
<script>
    import generator from 'qrcode'
    import qrcodeParser from 'qrcode-parser'
    import cache from '../../tool/cache'
    import { trim } from '../../helper'

    const generateHistoryCacheName = 'qrCodeGenerateHistoryCacheName'
    const generateHistoryMaxLength = 15

    class generateHistory {
        data = []

        constructor () {
            this.data = cache.get(generateHistoryCacheName, [])
        }

        find (value) {
            for (let i = 0; i < this.data.length; i++) {
                if (this.data[i].input === value.input && this.data[i].isShort === value.isShort) {
                    return true
                }
            }
            return false
        }

        push (value) {
            if (this.find(value)) {
                return
            }
            if (this.data.length > generateHistoryMaxLength) {
                this.data.pop()
            }
            this.data.push(value)
            cache.set(generateHistoryCacheName, this.data, 86400)
        }

        getValue (index) {
            return this.data[index]
        }

        lists () {
            return this.data
        }

        length () {
            return this.data.length
        }

        clear () {
            this.data = []
            cache.remove(generateHistoryCacheName)
        }
    }

    export default {
        computed: {
            readerInputImg () {
                if (this.current.readerInput) {
                    return `<img style="width:300px" src="${this.current.readerInput}" />`
                }
                return ''
            },
        },
        created () {
            this.generateHistory = new generateHistory()
            this.current = Object.assign(this.current, this.$getToolData())
        },
        methods: {
            history (index) {
                if (index === 'clear') {
                    this.generateHistory.clear()
                    return
                }
                let history = this.generateHistory.getValue(index)
                this.current.generateInput = history.input
                this.generate(false)
            },
            generate (insertHistory = true) {
                if (!this.current.generateInput) return
                this.generateHandle(this.current.generateInput)
                if (insertHistory) {
                    this.generateHistory.push({
                        input: this.current.generateInput
                    })
                }
                this.$saveToolData(this.current)
            },
            reader () {
                if (!this.current.readerInput) {
                    return
                }
                qrcodeParser(this.current.readerInput).then((c) => {
                    this.current.readerOutput = c.data
                    this.$saveToolData(this.current)
                    this.$Message.success('解析成功')
                }).catch(() => {
                    return this.$Message.error('图片解析错误')
                })
            },
            generateHandle (str) {
                generator.toDataURL(str, (error, url) => {
                    if (error) return this.$Message.error('二维码生成错误:' + error)
                    this.$Message.success('生成成功')
                    this.current.generateOutput = `<img style="width:300px" src="${url}" />`
                })
            },
            handleUpload (file) {
                let r = new FileReader()
                r.readAsDataURL(file)
                r.onloadend = () => {
                    this.current.readerInput = r.result
                    this.reader()
                }
                return false
            },
            substr (str) {
                str = trim(str.replace(/[\r\n]/g, ''))
                const strLength = 100
                return str.length > strLength ? str.substr(0, strLength) + '...' : str
            },
        },
        data () {
            return {
                generateHistory: {},
                current: {
                    generateInput: '',
                    generateOutput: '',
                    readerInput: '',
                    readerOutput: '',
                    operation: 'generate',
                },
            }
        },
    }
</script>